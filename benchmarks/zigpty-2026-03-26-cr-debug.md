# Code Review & Debugging Report: zigpty

**Repository:** zigpty — Zig + TypeScript PTY library for Node.js with N-API bindings
**Scope:** 11 Zig files (~2120 lines), 13 TypeScript files (~1568 lines), ~3700 lines total
**Reviewer:** Claude Code Review
**Date:** 2026-03-26

---

## Findings

### F-01: Use-After-Free on WinConPtyContext in Windows NAPI close/kill Path

**Severity:** Critical
**Files:** `zig/win/napi.zig` (lines 362–400, 84–115)

**Description:**
The `close()` and `kill()` functions extract a raw pointer to `WinConPtyContext` from the napi_external, then call `killProcess()` on it. However, the exit monitor thread (`winExitMonitorThread`) calls `alloc.destroy(ctx)` at line 114 after cleanup is complete. If the JS side calls `close()` or `kill()` after the exit monitor thread has already destroyed the context, it dereferences freed memory.

The napi_external was created at line 297 with **no finalizer** (`null`), so nothing prevents JS from holding a stale reference.

**Evidence:**
```zig
// win/napi.zig line 297 — no finalizer on the external
try napi.check(env, napi.napi_create_external(env, @ptrCast(ctx), null, null, &handle_val));

// win/napi.zig line 114 — exit monitor frees the context
alloc.destroy(ctx);

// win/napi.zig lines 374-377 — kill dereferences ctx unconditionally
var ctx_ptr: ?*anyopaque = null;
try napi.check(env, napi.napi_get_value_external(env, argv[0], &ctx_ptr));
const ctx: *WinConPtyContext = @ptrCast(@alignCast(ctx_ptr orelse return));
win.killProcess(ctx.spawn_result.process, 1);
```

**Impact:** Use-after-free leading to undefined behavior, potential crash, or memory corruption in the Node.js process. Exploitable in timing-dependent scenarios (process exits quickly, JS calls close/kill slightly later).

**Recommendation:**
Add an `alive` atomic flag to `WinConPtyContext`. Set it to false before `alloc.destroy`. Check it in `kill`, `write`, `resize`, and `close`. Alternatively, use a reference-counted wrapper or register a NAPI destructor finalizer that coordinates with the exit thread.

---

### F-02: Race Condition — Unix exitMonitorThread and JS Thread Accessing Same Resources

**Severity:** High
**Files:** `zig/pty_unix.zig` (lines 16–32), `src/pty/unix.ts` (lines 41–46)

**Description:**
The `exitMonitorThread` runs in a background thread and calls `napi_call_threadsafe_function` with blocking mode after `waitForExit` returns. The threadsafe function callback (`exitCallJs`) fires on the JS thread, which then invokes the JS `onExit` callback. In `unix.ts`, this callback calls `this._readable?.destroy()` and triggers `_handleExit` which clears listeners.

Meanwhile, the main JS thread may still be receiving data events on the ReadStream for the same fd. The `_handleExit` method (in `_base.ts` line 130) truncates `_dataListeners` to zero length while the ReadStream's `data` event handler (in `unix.ts` line 72) iterates over `_dataListeners` — this is not iteration-safe since `for...of` on an array being mutated can skip elements or throw.

**Evidence:**
```typescript
// _base.ts line 125-136
protected _handleExit(info: { exitCode: number; signal: number }): void {
    this._closed = true;
    this._exitCode = info.exitCode;
    // ...
    this._dataListeners.length = 0;  // <-- truncate while data handler may iterate
    this._exitListeners.length = 0;
    // ...
}

// unix.ts line 68-75
this._readable.on("data", (data: string | Buffer) => {
    // ...
    for (const listener of this._dataListeners) {  // <-- iterating same array
        listener(data);
    }
});
```

**Impact:** Data listeners may be called after the PTY is logically closed, or listeners may be skipped. In Node.js, since both callbacks run on the event loop, this is single-threaded, so the race is between microtask/macrotask ordering rather than true concurrency. However, the `destroy()` call inside the exit callback can trigger synchronous `error`/`close` events that re-enter the data path.

**Recommendation:**
Copy the listener array before iterating (as is already done for `_exitListeners` at line 129). Apply the same `[...this._dataListeners]` pattern in the data emission path in `unix.ts`.

---

### F-03: macOS kinfo_proc p_comm Offset is Hardcoded and Architecture-Fragile

**Severity:** High
**Files:** `zig/pty_darwin.zig` (lines 41–52)

**Description:**
The `getProcessName` function uses a hardcoded `P_COMM_OFFSET = 243` to extract the process name from `kinfo_proc`. This offset depends on the exact layout of `kinfo_proc` which varies across macOS versions and is not ABI-stable. Apple's headers define it as a nested struct, and any padding or field-size change breaks this.

**Evidence:**
```zig
// pty_darwin.zig lines 41-52
const P_COMM_OFFSET = 243;
const MAXCOMLEN = 16;
if (size < P_COMM_OFFSET + MAXCOMLEN) return null;

const comm = info_buf[P_COMM_OFFSET..][0..MAXCOMLEN];
const len = std.mem.indexOfScalar(u8, comm, 0) orelse MAXCOMLEN;
```

The comment says "both arm64 and x86_64" but this was presumably verified on one specific macOS version. The `kinfo_proc` struct is documented as having a `kp_proc.p_comm` field, but its offset within the struct is determined by compiler layout, not a stable ABI guarantee.

**Impact:** On a macOS update that changes `kinfo_proc` layout, `getProcessName` returns garbage data (wrong process name) or `null`. This is a data correctness issue, not a crash — the 720-byte buffer and bounds check prevent out-of-bounds reads.

**Recommendation:**
Use `proc_name()` or `proc_pidpath()` syscalls instead, which are higher-level stable APIs. Alternatively, import the Zig-translated C header for `kinfo_proc` to get the correct offset at compile time.

---

### F-04: Leaked File Descriptors in openPty on Error Path

**Severity:** High
**Files:** `zig/lib.zig` (lines 160–183)

**Description:**
In `openPtyUnix`, if `openpty()` succeeds but the subsequent `ttyname_r()` call fails, the function still returns the result with `pty_name_len = 0`. This is acceptable behavior. However, there is **no close path** for the returned master/slave fds — the caller must close them. The `open()` NAPI function at `zig/pty_unix.zig:154` returns the raw fds to JavaScript but never tracks them for cleanup.

More critically, the `Terminal` class (`src/terminal.ts:72-76`) calls `open()` and stores the fds, but if the Terminal constructor throws between the `open()` call and completing construction, the fds leak.

**Evidence:**
```typescript
// terminal.ts lines 71-76
if (!isWindows) {
    const result = (native as INativeUnix).open(this._cols, this._rows);
    this.stdin = result.master;
    this.stdout = result.slave;
    this._setupUnixReader(this.stdin);  // <-- if this throws, fds leak
}
```

**Impact:** File descriptor leak if construction fails. In long-running processes creating many terminals, this exhausts fd limits.

**Recommendation:**
Wrap the constructor body in try/catch. On exception, close both fds before re-throwing.

---

### F-05: Windows ConPTY writeInput Has No Protection Against Infinite Loop on Zero-Bytes-Written

**Severity:** High
**Files:** `zig/pty_windows.zig` (lines 349–364)

**Description:**
The `writeInput` function loops until all data is written. If `WriteFile` succeeds but reports 0 bytes written (which is valid for synchronous writes to certain handle types), the loop spins forever since `offset` never advances.

**Evidence:**
```zig
// pty_windows.zig lines 349-364
pub fn writeInput(conin: HANDLE, data: []const u8) ConPtyError!void {
    var offset: usize = 0;
    while (offset < data.len) {
        var written: DWORD = 0;
        if (WriteFile(
            conin,
            data[offset..].ptr,
            @intCast(data.len - offset),
            &written,
            null,
        ) == 0) {
            return ConPtyError.WriteFailed;
        }
        offset += written;  // if written == 0, infinite loop
    }
}
```

**Impact:** Infinite loop hangs the calling thread (which for the Windows write NAPI function is the JS main thread). The JS process becomes completely unresponsive.

**Recommendation:**
Add a check for `written == 0` — either return an error or break with an error after a small retry count.

---

### F-06: Unix fork Child Process Does Not Close Master FD Before exec

**Severity:** Medium
**Files:** `zig/lib.zig` (lines 125–143)

**Description:**
After `forkpty()`, the child process calls `platform.closeExcessFds()` which closes all fds >= 3. However, `forkpty()` returns the master fd in the parent, and the master fd is typically also present in the child's fd table (forkpty duplicates before fork). The child closes it via `closeExcessFds()`, but this happens **after** `chdir()` and `setgid()/setuid()` calls. If `setuid()` fails (returns non-zero), the child calls `exit(1)` — but between `forkpty` returning and `closeExcessFds`, the master fd is open in the child, which is a brief window where the child (potentially running as a different user if setgid succeeded but setuid failed) has access to the master side of the PTY.

**Evidence:**
```zig
// lib.zig lines 125-143
if (pid == 0) {
    _ = std.c.sigprocmask(std.c.SIG.SETMASK, @ptrCast(&sigset_old), null);
    platform.resetSignalHandlers();

    if (std.c.chdir(opts.cwd) != 0) {
        std.process.exit(1);
    }

    if (opts.gid) |gid| {
        if (std.c.setgid(gid) != 0) std.process.exit(1);
    }
    if (opts.uid) |uid| {
        if (std.c.setuid(uid) != 0) std.process.exit(1);
    }

    platform.closeExcessFds();  // <-- master fd still open until here

    platform.execChild(opts.file, opts.argv, opts.envp);
    std.process.exit(1);
}
```

**Impact:** Minor security concern — the child briefly holds the master fd with potentially changed group/user permissions. In practice, `forkpty()` should set `CLOEXEC` on the master fd, and the child calls `exec` soon after. But if `exec` fails, the child enters `exit(1)` with the master fd still open (since `closeExcessFds` runs before exec, this is actually handled). The ordering issue is that privilege-dropping happens before fd cleanup.

**Recommendation:**
Move `closeExcessFds()` to immediately after `resetSignalHandlers()`, before `chdir`/`setgid`/`setuid`. This follows the principle of dropping capabilities before dropping privileges.

---

### F-07: Signal Number Mapping in signalNumber() is Incomplete and Linux-Specific

**Severity:** Medium
**Files:** `src/pty/unix.ts` (lines 141–152)

**Description:**
The `signalNumber` function hardcodes signal numbers. `SIGUSR1 = 10` and `SIGUSR2 = 12` are Linux-specific values. On macOS/BSD, `SIGUSR1 = 30` and `SIGUSR2 = 31`. The fallback to `os.constants.signals` handles this, but only if the signal name is passed as a standard name (e.g., "SIGUSR1"). If a user passes a signal name that matches the hardcoded table first, they get the wrong number on macOS.

**Evidence:**
```typescript
// unix.ts lines 141-152
function signalNumber(signal: string): number {
    const signals: Record<string, number> = {
        SIGHUP: 1,
        SIGINT: 2,
        SIGQUIT: 3,
        SIGTERM: 15,
        SIGKILL: 9,
        SIGUSR1: 10,   // Linux-only! macOS: 30
        SIGUSR2: 12,   // Linux-only! macOS: 31
    };
    return signals[signal] ?? (os.constants.signals as Record<string, number>)[signal] ?? 1;
}
```

**Impact:** Sending `SIGUSR1` or `SIGUSR2` on macOS sends the wrong signal (10 = SIGBUS on macOS, 12 = SIGSYS on macOS). This could crash the child process instead of delivering the intended user signal. `SIGBUS` in particular would appear as a crash rather than a graceful signal.

**Recommendation:**
Remove the hardcoded table entirely and use `os.constants.signals` as the primary source. It is always available in Node.js and has correct values for the running platform.

---

### F-08: WriteQueue Does Not Handle EPIPE or EIO — Silent Data Loss

**Severity:** Medium
**Files:** `src/pty/_writeQueue.ts` (lines 35–63)

**Description:**
The `_process` method handles `EAGAIN` by retrying, but all other errors (including `EPIPE` from a dead process, `EIO` from a broken PTY) silently clear the queue without notifying the caller. Data is lost with no error propagation.

**Evidence:**
```typescript
// _writeQueue.ts lines 44-51
if (err) {
    if ("code" in err && err.code === "EAGAIN") {
        this._immediate = setImmediate(() => this._process());
        return;
    }
    this._queue.length = 0;  // <-- all pending data silently dropped
    return;
}
```

**Impact:** Writes to a closed/dead PTY silently succeed from the caller's perspective. The caller has no way to know that data was lost.

**Recommendation:**
Add an error callback to `WriteQueue` and invoke it when a non-retryable write error occurs. At minimum, expose the error via an event or store it for the caller to check.

---

### F-09: macOS closeExcessFds Brute-Force Close Has O(n) Cost With High FD Limits

**Severity:** Medium
**Files:** `zig/pty_darwin.zig` (lines 67–77)

**Description:**
On macOS, `closeExcessFds()` calls `sysconf(_SC_OPEN_MAX)` and then loops closing every fd from 3 to that limit. The default `OPEN_MAX` on macOS can be 12288 or higher (and users can set it to millions via `ulimit`). This is called in the forked child between `fork` and `exec`, where async-signal-safety matters and excessive syscalls increase fork-to-exec latency.

**Evidence:**
```zig
// pty_darwin.zig lines 67-77
pub fn closeExcessFds() void {
    const SC_OPEN_MAX = 5;
    const max_fd = sysconf(SC_OPEN_MAX);
    const limit: c_int = if (max_fd > 0) @intCast(max_fd) else 256;
    var fd: c_int = 3;
    while (fd < limit) : (fd += 1) {
        _ = std.c.close(fd);
    }
}
```

**Impact:** Fork-to-exec delay proportional to the fd limit. For `ulimit -n 1048576` (common in server environments), this is over 1 million syscalls. While each `close()` on an invalid fd is fast, the aggregate time is noticeable — potentially hundreds of milliseconds.

**Recommendation:**
Use `closefrom(3)` if available (macOS 13.x+), or iterate `/dev/fd/` directory entries (which exists on macOS) similar to the Linux `/proc/self/fd` approach. Cap the brute-force fallback to a reasonable maximum (e.g., 4096).

---

### F-10: TIOCSWINSZ Constant for macOS is Hardcoded Rather Than Derived

**Severity:** Medium
**Files:** `zig/lib.zig` (lines 98–102)

**Description:**
The `TIOCSWINSZ` ioctl constant is hardcoded as `0x80087467` for macOS. This value is correct for current macOS versions, but using a magic number rather than deriving it from headers means any future change would silently break resize functionality.

**Evidence:**
```zig
// lib.zig lines 98-102
const TIOCSWINSZ: c_ulong = switch (builtin.os.tag) {
    .linux => @as(c_ulong, @bitCast(@as(c_long, @intCast(std.posix.T.IOCSWINSZ)))),
    .macos => 0x80087467,
    else => 0,
};
```

Note the asymmetry: Linux uses the stdlib constant, macOS uses a magic number.

**Impact:** If Apple ever changes the ioctl number (unlikely but not impossible in a major OS restructuring), resize silently fails or does something unexpected.

**Recommendation:**
Use `@cImport` with `<sys/ioctl.h>` to get `TIOCSWINSZ` at compile time, or use Zig's `std.posix.T.IOCSWINSZ` if it's available for macOS in the Zig standard library version being used.

---

### F-11: Windows spawn Error Path Leaks data_tsfn When Process Start Fails After Read Thread Launch

**Severity:** Medium
**Files:** `zig/win/napi.zig` (lines 241–253)

**Description:**
When `win.startProcess` fails after the read thread has been started, the error handling path at lines 243-252 closes `conout` to unblock the read thread, then joins the read thread. The read thread releases `data_tsfn` via `napi_release_threadsafe_function` at line 80. However, this release uses `.release` mode, not `.abort` mode. If there are queued calls in the TSFN, they may still fire on the JS thread after the error has been reported — potentially accessing freed memory.

**Evidence:**
```zig
// win/napi.zig line 80
_ = napi.napi_release_threadsafe_function(ctx.data_tsfn, .release);

// win/napi.zig lines 241-253 (error path)
const proc = win.startProcess(setup.hpc, cmd_line, ...) catch {
    win.closeHandle(setup.conout);
    ctx.spawn_result.conout = win.INVALID_HANDLE;
    if (ctx.read_thread) |rt| rt.join();
    // data_tsfn already released by read thread via winReadThread defer
    win.closeHandle(setup.conin);
    win.closePseudoConsole(setup.hpc);
    alloc.destroy(ctx);  // <-- freed, but queued tsfn calls may still reference ctx
    ...
};
```

**Impact:** Potential use-after-free if there are pending TSFN calls when the context is destroyed. In practice, since the read thread saw EOF immediately (conout closed), there are likely zero or very few queued calls.

**Recommendation:**
Use `.abort` mode in the error path to ensure no pending calls fire. The read thread should detect the error condition and use `.abort` instead of `.release` when `conout` is invalid.

---

### F-12: No Validation of cols/rows = 0 Before Passing to ConPTY / ioctl

**Severity:** Medium
**Files:** `zig/pty.zig` (line 68–71), `zig/pty_windows.zig` (lines 206, 368)

**Description:**
The `clampU16` function allows 0 for cols and rows. `CreatePseudoConsole` on Windows does not accept 0 for either dimension — it returns `E_INVALIDARG`. On Unix, `TIOCSWINSZ` accepts zero dimensions (some programs treat it as "unknown"), but it can cause division-by-zero in applications that compute characters-per-line.

**Evidence:**
```zig
// pty.zig lines 68-71
pub fn clampU16(val: i32) u16 {
    if (val <= 0) return 0;  // allows 0
    if (val > std.math.maxInt(u16)) return std.math.maxInt(u16);
    return @intCast(val);
}
```

**Impact:** On Windows, passing `cols=0` or `rows=0` causes `CreatePseudoConsole` to fail, and the error is reported generically as "ConPTY creation failed." On Unix, zero dimensions may cause applications in the PTY to misbehave.

**Recommendation:**
Clamp minimum to 1 for both cols and rows: `if (val <= 0) return 1;`. This matches the behavior expected by both ConPTY and most terminal applications.

---

### F-13: Unix close() Sends SIGHUP After closeSync — Signal May Go to Wrong Process (PID Reuse)

**Severity:** Medium
**Files:** `src/pty/unix.ts` (lines 120–138)

**Description:**
The `close()` method closes the fd first, then sends SIGHUP to the child process. Between `closeSync` and `kill`, the child process may have already exited (closing the master fd causes SIGHUP to the terminal's session leader). If the child has exited and the PID has been reused by a new process, the `kill` call sends SIGHUP to an unrelated process.

**Evidence:**
```typescript
// unix.ts lines 120-138
close(): void {
    if (this._closed) return;
    this._closed = true;
    this._wq.close();
    try { this._readable?.destroy(); } catch {}
    try { fs.closeSync(this._fd); } catch {}     // <-- child likely gets SIGHUP here
    try {
        process.kill(this.pid, 0);                // <-- check if alive
        process.kill(this.pid, "SIGHUP");         // <-- but PID may be reused
    } catch {}
}
```

The `kill(pid, 0)` check is a TOCTOU race: the process could exit and PID could be reused between the check and the SIGHUP.

**Impact:** An unrelated process receives SIGHUP and may terminate unexpectedly. This is a classic PID reuse vulnerability. The probability is low in normal usage but increases under high process-creation load.

**Recommendation:**
Send SIGHUP **before** closing the fd, not after. This way the child is still alive (or just exiting) and PID reuse hasn't occurred. Alternatively, remove the explicit SIGHUP entirely — closing the master fd already sends SIGHUP to the session leader via the kernel.

---

### F-14: Terminal._attachUnixFd Closes Standalone FDs Without Checking If Reader Is Using Them

**Severity:** Medium
**Files:** `src/terminal.ts` (lines 152–165)

**Description:**
When `_attachUnixFd` is called (from `UnixPty` constructor), it calls `_destroyReader()` to destroy the existing ReadStream, then closes the standalone fds. However, `ReadStream.destroy()` is asynchronous — the underlying fd may still be referenced by the event loop until the `close` event fires. Calling `fs.closeSync(this.stdin)` immediately after `destroy()` can cause EBADF errors in the event loop's poll.

**Evidence:**
```typescript
// terminal.ts lines 152-165
_attachUnixFd(fd: number): void {
    this._standalone = false;
    this._destroyReader();          // destroy is async
    if (this.stdout >= 0) {
        try { fs.closeSync(this.stdout); } catch {}   // close slave fd
        this.stdout = -1;
    }
    if (this.stdin >= 0) {
        try { fs.closeSync(this.stdin); } catch {}     // close master fd — reader may still use it!
    }
    this.stdin = fd;
    this._setupUnixReader(fd);
}
```

**Impact:** Potential EBADF errors logged to stderr or swallowed by the error handler. The new reader on the different fd works correctly, but there may be transient errors.

**Recommendation:**
Wait for the ReadStream's `close` event before closing the fd, or at minimum, set the ReadStream's internal fd to -1 before closing. Since the try/catch swallows errors, the impact is limited in practice.

---

### F-15: Windows WindowsPty.close() Only Kills — Does Not Release NAPI External Reference

**Severity:** Medium
**Files:** `src/pty/windows.ts` (lines 104–115)

**Description:**
The `close()` method only calls `kill()` via the native binding. It relies on the exit monitor thread to clean up. But if the process doesn't exit promptly after `TerminateProcess` (e.g., stuck in kernel mode), the `WinConPtyContext` is never freed and the NAPI external reference leaks.

Additionally, there's no mechanism for the JS side to know when cleanup is complete. The `_handle` reference continues to point at potentially stale native memory.

**Evidence:**
```typescript
// windows.ts lines 104-115
close(): void {
    if (this._closed) return;
    this._closed = true;
    this._deferredCalls.length = 0;
    try {
        this._native.kill(this._handle);  // only kills, no cleanup
    } catch {}
}
```

**Impact:** Resource leak if process termination is delayed. The handle remains valid in JS but the underlying context may be in an inconsistent state.

**Recommendation:**
Set `this._handle` to null after calling kill. Add null checks in all native call sites. Consider adding a timeout mechanism that force-cleans resources if the exit monitor thread doesn't complete within a reasonable time.

---

### F-16: macOS execChild Mutates Global `environ` Pointer — Not Thread-Safe

**Severity:** Medium
**Files:** `zig/pty_darwin.zig` (lines 10–15)

**Description:**
On macOS, since `execvpe` doesn't exist, the code sets the global `environ` pointer to the new environment, then calls `execvp`. This runs in the forked child process, so thread safety in the child is not normally a concern (only one thread survives fork). However, if `execvp` fails and returns, the global `environ` pointer has been permanently modified in the child process. The subsequent `exit(1)` at `lib.zig:143` will run atexit handlers with the wrong environment.

**Evidence:**
```zig
// pty_darwin.zig lines 10-15
pub fn execChild(file: [*:0]const u8, argv: [*:null]const ?[*:0]const u8, envp: [*:null]const ?[*:0]const u8) void {
    const environ: *[*:null]const ?[*:0]const u8 = @extern(*[*:null]const ?[*:0]const u8, .{ .name = "environ" });
    environ.* = envp;
    _ = execvp(file, argv);
}
```

**Impact:** If exec fails, atexit handlers run with a potentially invalid environment (the arena allocator in the parent may have freed the env strings, but in the child, the arena copy is still valid since fork copies memory). Practically low impact since `exit(1)` in the child is fast.

**Recommendation:**
Use `_exit(1)` (or `std.process.exit` which uses `_exit`) instead of relying on atexit handlers. This is already the case (`std.process.exit` calls `_exit` on Unix). No change needed — but document the assumption.

---

### F-17: Arena Allocator in forkImpl Frees Memory That Child Process Still References

**Severity:** Medium
**Files:** `zig/pty_unix.zig` (lines 39–106)

**Description:**
The `forkImpl` function uses an `ArenaAllocator` (`defer arena.deinit()` at line 50). The arena allocates strings for `file`, `exec_argv`, `exec_envp`, and `cwd`. After `lib.forkPty` returns in the parent, the arena is freed. However, `forkPty` calls `forkpty()` which forks the process. In the child, the arena memory is part of the child's copied address space and is valid. But in the parent, `arena.deinit()` runs after `forkPty` returns — this is correct because the child has its own copy.

This is actually safe due to `fork()` semantics (copy-on-write), but it's worth noting that the child process uses pointers into arena-allocated memory for `execvp`/`execvpe` arguments, which is valid because `exec` happens before the arena could be freed in the child.

**Impact:** None in practice — this is safe. Documenting it as informational.

**Recommendation:**
Add a comment explaining that this is safe because fork gives the child its own copy of the arena memory.

---

### F-18: `waitFor` Promise Leaks Timer and Listener on Garbage Collection

**Severity:** Low
**Files:** `src/pty/_base.ts` (lines 73–123)

**Description:**
The `waitFor` method creates a Promise with a `setTimeout`. If the promise is created but never awaited and the PTY is closed, the timer continues running until it fires. The cleanup function clears the timer and disposes the listener, but only when the timeout fires or the pattern matches. If the PTY exits before either condition, the timer and listener remain until timeout.

**Evidence:**
```typescript
// _base.ts lines 96-99
const timer = setTimeout(() => {
    cleanup();
    reject(new Error(`waitFor("${pattern}") timed out after ${timeout}ms`));
}, timeout);
```

There is no cleanup path triggered by PTY exit.

**Impact:** Minor memory/timer leak. The timer keeps the Node.js event loop alive for up to `timeout` ms (default 30 seconds) even after the PTY is closed.

**Recommendation:**
Add a listener on `onExit` that calls `cleanup()` and rejects the promise with an appropriate error ("process exited before pattern matched").

---

### F-19: Flow Control Check Only Works for Single-Character Chunks

**Severity:** Low
**Files:** `src/pty/unix.ts` (lines 69–71)

**Description:**
The flow control check compares the entire data chunk to the pause/resume character. If the terminal sends the flow control character embedded in a larger data chunk (e.g., `"text\x13moretext"`), the flow control character is not intercepted.

**Evidence:**
```typescript
// unix.ts lines 69-71
if (this.handleFlowControl && typeof data === "string") {
    if (data === this._flowControlPause || data === this._flowControlResume) return;
    // Only exact-match, not substring match
}
```

**Impact:** Flow control characters that arrive in the same chunk as other data are not intercepted. This reduces the reliability of software flow control.

**Recommendation:**
Filter flow control characters from within the data string, not just match the entire chunk. Use `data.replace()` or similar to strip the characters and only suppress the event if the resulting string is empty.

---

### F-20: No CLOEXEC on Pipe Handles in Windows ConPTY

**Severity:** Low
**Files:** `zig/pty_windows.zig` (lines 206–221)

**Description:**
The `CreatePipe` calls pass `null` for the security attributes, which means the handles are inheritable by default on Windows. While the `CreateProcessW` call uses `bInheritHandles = 0`, any other process spawned by the Node.js process (via `child_process.exec` etc.) would inherit these pipe handles.

**Evidence:**
```zig
// pty_windows.zig lines 210-211
if (CreatePipe(&pipe_in_read, &pipe_in_write, null, 0) == 0) {
    return ConPtyError.CreatePipeFailed;
}
```

**Impact:** Handle leak to child processes spawned by other means. The leaked handles could prevent ConPTY cleanup from completing (the pipe stays open because a child inherited it).

**Recommendation:**
Pass `SECURITY_ATTRIBUTES` with `bInheritHandle = FALSE`, or call `SetHandleInformation` to remove the `HANDLE_FLAG_INHERIT` flag after creation.

---

### F-21: `environ` Extern Declaration Uses Incorrect Mutability Semantics

**Severity:** Low
**Files:** `zig/pty_darwin.zig` (line 12)

**Description:**
The `environ` pointer is declared as `*[*:null]const ?[*:0]const u8`, but it's immediately mutated (`environ.* = envp`). This works in Zig because the outer pointer is mutable, but the type signature suggests the pointed-to data is const, which is semantically misleading. The actual C `environ` is `char **` (fully mutable).

**Evidence:**
```zig
const environ: *[*:null]const ?[*:0]const u8 = @extern(*[*:null]const ?[*:0]const u8, .{ .name = "environ" });
environ.* = envp;
```

**Impact:** No runtime impact — it works correctly. Code readability/maintainability issue.

**Recommendation:**
Adjust the type to reflect the intended mutability, or add a comment explaining the cast.

---

### F-22: Linux closeExcessFds Fallback Closes Only 3..256 — Misses Higher FDs

**Severity:** Low
**Files:** `zig/pty_linux.zig` (lines 57–60)

**Description:**
If both `close_range` and `/proc/self/fd` are unavailable (unlikely on any modern Linux, but possible in containers with restricted /proc), the brute-force fallback only closes fds 3 through 255. This misses any fds above 255.

**Evidence:**
```zig
// pty_linux.zig lines 57-60
// Last resort: brute-force close FDs 3..256
var fd: c_int = 3;
while (fd < 256) : (fd += 1) _ = linux.close(@intCast(fd));
return;
```

**Impact:** In extremely constrained environments, fds 256+ leak to the child process. Very unlikely in practice.

**Recommendation:**
Increase the fallback limit or use `sysconf(_SC_OPEN_MAX)` as macOS does.

---

### F-23: No Input Validation on uid/gid Values in Fork NAPI Binding

**Severity:** Low
**Files:** `zig/pty_unix.zig` (lines 85–101)

**Description:**
The `uid` and `gid` values are extracted as `i32` from JavaScript and only checked for `>= 0` before casting to `u32` with `@intCast`. While the NAPI layer correctly converts `-1` to `null` (meaning "don't change"), there's no validation that the provided uid/gid actually exists on the system. This is arguably the caller's responsibility, but passing uid=0 (root) without any additional checks could be a privilege escalation vector if the Node.js process runs as root.

**Evidence:**
```zig
.uid = if (uid_i32 >= 0) @intCast(uid_i32) else null,
.gid = if (gid_i32 >= 0) @intCast(gid_i32) else null,
```

**Impact:** If the Node.js process runs as root and a user-controlled uid/gid is passed, the spawned process runs as that user. This is by design for PTY libraries, but worth documenting.

**Recommendation:**
Document that uid/gid are passed through without validation. If the library is used in multi-tenant contexts, callers must validate uid/gid themselves.

---

### F-24: WindowsPty.process Returns Static File Name Instead of Foreground Process

**Severity:** Low
**Files:** `src/pty/windows.ts` (lines 61–63)

**Description:**
On Windows, the `process` getter returns `this._file` (the originally spawned executable), not the current foreground process. On Unix, `native.process(fd)` queries the actual foreground process via tcgetpgrp + /proc or sysctl. This inconsistency means Windows users get stale process information.

**Evidence:**
```typescript
// windows.ts lines 61-63
get process(): string {
    return this._file;
}
```

**Impact:** API inconsistency between platforms. On Windows, `pty.process` always returns the shell name, never the actual foreground process (e.g., if the user runs `node` inside the PTY, `process` still returns `cmd.exe`).

**Recommendation:**
Document the limitation. Implementing foreground process detection on Windows would require `QueryFullProcessImageName` on the console's attached processes, which is non-trivial.

---

### F-25: Terminal Constructor on Windows Creates a Non-Functional Terminal

**Severity:** Low
**Files:** `src/terminal.ts` (lines 59–77)

**Description:**
On Windows, the `Terminal` constructor does nothing platform-specific — `stdin` and `stdout` remain `-1`. The standalone terminal is essentially non-functional on Windows. There is no error or warning.

**Evidence:**
```typescript
// terminal.ts lines 67-77
this.stdin = -1;
this.stdout = -1;

// Standalone: open a bare PTY pair immediately (Unix only)
if (!isWindows) {
    const result = (native as INativeUnix).open(this._cols, this._rows);
    this.stdin = result.master;
    this.stdout = result.slave;
    this._setupUnixReader(this.stdin);
}
```

**Impact:** On Windows, `new Terminal()` creates an object where `write()` returns 0 (no-op), `stdin = -1`, `stdout = -1`. Users get no feedback that the terminal is non-functional.

**Recommendation:**
Either throw an error on Windows for standalone terminals, or document that standalone `new Terminal()` is Unix-only. The `spawn()` path with `terminal` option works on Windows.

---

---

## Statistics Summary

| Metric | Count |
|---|---|
| **Total findings** | 25 |
| **Critical** | 1 |
| **High** | 4 |
| **Medium** | 10 |
| **Low** | 10 |

### Severity Breakdown

| ID | Severity | Category | File(s) |
|---|---|---|---|
| F-01 | Critical | Use-After-Free | `zig/win/napi.zig` |
| F-02 | High | Race Condition | `zig/pty_unix.zig`, `src/pty/_base.ts`, `src/pty/unix.ts` |
| F-03 | High | Platform Fragility | `zig/pty_darwin.zig` |
| F-04 | High | Resource Leak | `zig/lib.zig`, `src/terminal.ts` |
| F-05 | High | Infinite Loop | `zig/pty_windows.zig` |
| F-06 | Medium | Security (fd leak in child) | `zig/lib.zig` |
| F-07 | Medium | Platform Bug | `src/pty/unix.ts` |
| F-08 | Medium | Silent Data Loss | `src/pty/_writeQueue.ts` |
| F-09 | Medium | Performance | `zig/pty_darwin.zig` |
| F-10 | Medium | Maintainability | `zig/lib.zig` |
| F-11 | Medium | Resource Leak | `zig/win/napi.zig` |
| F-12 | Medium | Input Validation | `zig/pty.zig` |
| F-13 | Medium | PID Reuse / Signal Safety | `src/pty/unix.ts` |
| F-14 | Medium | Resource Leak | `src/terminal.ts` |
| F-15 | Medium | Resource Leak | `src/pty/windows.ts` |
| F-16 | Medium | Thread Safety | `zig/pty_darwin.zig` |
| F-17 | Medium | Informational (Safe) | `zig/pty_unix.zig` |
| F-18 | Low | Timer/Listener Leak | `src/pty/_base.ts` |
| F-19 | Low | Logic Bug | `src/pty/unix.ts` |
| F-20 | Low | Handle Inheritance | `zig/pty_windows.zig` |
| F-21 | Low | Code Quality | `zig/pty_darwin.zig` |
| F-22 | Low | Edge Case | `zig/pty_linux.zig` |
| F-23 | Low | Input Validation | `zig/pty_unix.zig` |
| F-24 | Low | API Inconsistency | `src/pty/windows.ts` |
| F-25 | Low | Platform Limitation | `src/terminal.ts` |

### Category Distribution

| Category | Count |
|---|---|
| Resource Leak | 5 |
| Platform Bugs/Fragility | 4 |
| Memory Safety | 2 |
| Race Condition | 2 |
| Input Validation | 2 |
| Silent Data Loss | 1 |
| Security | 1 |
| Performance | 1 |
| Infinite Loop | 1 |
| Logic Bug | 1 |
| API Design | 2 |
| Code Quality | 2 |
| Informational | 1 |

### Key Recommendations (Priority Order)

1. **F-01 (Critical):** Add lifetime management to `WinConPtyContext` — either a reference count, atomic alive flag, or NAPI destructor finalizer. This is the most dangerous bug.
2. **F-05 (High):** Add zero-written guard in `writeInput` loop to prevent infinite hangs.
3. **F-07 (Medium but easy fix):** Remove hardcoded signal table, use `os.constants.signals` exclusively.
4. **F-13 (Medium):** Reorder close() to send SIGHUP before closing the fd.
5. **F-03 (High):** Replace hardcoded `kinfo_proc` offset with `proc_name()` syscall.
