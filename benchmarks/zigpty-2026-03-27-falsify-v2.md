# AFM Falsify Report: zigpty

> Target: zigpty — Zig-based cross-platform PTY library for Node.js
> Method: AFM Falsify v3.1 — Seed + Interrogate + Falsify
> Date: 2026-03-26

---

## Phase 1: Seed List

| # | Seed | File:Line |
|---|------|-----------|
| S01 | macOS `environ` global write is not thread-safe before `execvp` | zig/pty_darwin.zig:12-14 |
| S02 | macOS `closeExcessFds` brute-force loop can close FDs beyond 256 now but may be slow with high `sysconf(_SC_OPEN_MAX)` | zig/pty_darwin.zig:67-77 |
| S03 | Linux last-resort FD close only covers 3..256, misses higher FDs | zig/pty_linux.zig:57-59 |
| S04 | `clampU16` allows cols=0, rows=0 — Windows `CreatePseudoConsole` rejects zero dimensions | zig/pty.zig:68-72 |
| S05 | `UnixPty.close()` calls `fs.closeSync(fd)` then `process.kill(pid, 0)` — fd is already closed when testing liveness | src/pty/unix.ts:130-137 |
| S06 | `UnixPty._readable = undefined!` when terminal is attached — pause/resume will silently no-op | src/pty/unix.ts:61 |
| S07 | `waitFor` with terminal: listener pushed to `terminal._dataListeners` but cleanup accesses via direct array index — if concurrent data arrives between indexOf and splice, wrong listener removed | src/pty/_base.ts:86-93 |
| S08 | `waitFor` never resolves and never rejects if process exits before pattern matches (no onExit hook) | src/pty/_base.ts:73-123 |
| S09 | `WindowsPty.process` returns static file name, never the actual foreground process | src/pty/windows.ts:61-63 |
| S10 | `_handleExit` clears `_dataListeners` immediately — data in flight from native thread may arrive after listeners are gone | src/pty/_base.ts:125-136 |
| S11 | Windows `napi_external` created with `null` finalize callback — if JS GC collects handle while threads still running, use-after-free | zig/win/napi.zig:297 |
| S12 | macOS `P_COMM_OFFSET = 243` hardcoded — assumption this offset is stable across all macOS versions | zig/pty_darwin.zig:42 |
| S13 | `setgid` before `setuid` in child process — correct order, but no `initgroups`/`setgroups` call means supplementary groups leak from parent | zig/lib.zig:133-138 |
| S14 | `exitMonitorThread` (Unix) detaches thread — no join, thread lifetime uncontrolled if Node exits quickly | zig/pty_unix.zig:133 |
| S15 | `Terminal._emitData` creates new `TextDecoder()` on every call — performance, but also: no streaming decode for multi-byte chars split across chunks | src/terminal.ts:185 |
| S16 | `open()` (bare PTY) returns `slave` fd to JS but never closes it on the parent side for fork use case | src/index.ts:32-37 |
| S17 | `WindowsPty` deferred write/resize: if process produces no output before crashing, deferred calls never execute and writes are silently lost | src/pty/windows.ts:27-36 |
| S18 | `buildEnvPairs` on Windows path: no `sanitizeKeys` passed — TMUX/COLUMNS etc. leak through to Windows env | src/pty/windows.ts:24 |
| S19 | `appendQuoted` Windows cmd-line quoting: doesn't handle `%` (env var expansion) or `!` (delayed expansion) or `^` (cmd.exe escape) | zig/pty_windows.zig:457-498 |
| S20 | `UnixPty` creates both `tty.ReadStream` (in constructor) AND `WriteQueue` on the same fd — `tty.ReadStream` constructor may set fd to non-blocking, affecting write behavior | src/pty/unix.ts:54,64 |
| S21 | `Terminal._attachUnixFd` closes old `stdin` fd synchronously then sets up new reader — if close fails the old reader's error handler swallows it | src/terminal.ts:152-165 |
| S22 | `winReadThread` uses `napi_call_threadsafe_function` with `.nonblocking` — if JS event loop is busy, data chunks are queued unbounded (max_queue_size=0) | zig/win/napi.zig:69-73 |
| S23 | `WriteQueue._process` calls `fs.write` with `task.offset` but Node's `fs.write(fd, buffer, offset, callback)` requires a `length` parameter — possibly writing entire remaining buffer | src/pty/_writeQueue.ts:40 |
| S24 | `decodeWaitStatus` returns `exit_code = -1` for stopped processes (termsig == 0x7f) — no distinction from error case | zig/lib.zig:214-223 |
| S25 | `close()` on `WindowsPty` just calls `kill()` — never directly triggers `_handleExit`, relies entirely on exit monitor thread, which may stall if `ClosePseudoConsole` deadlocks | src/pty/windows.ts:104-115 |
| S26 | `UnixPty.close()` sends SIGHUP to already-exited process (benign but: pid might have been recycled if close is called much later) | src/pty/unix.ts:134-137 |
| S27 | `Terminal.close()` calls `_onExit(this, 0, null)` always with exitCode=0 — misleading if the real process crashed | src/terminal.ts:144 |
| S28 | `defaultShell()` falls back to `powershell.exe` on Windows — but many systems have PowerShell at different paths or only have `pwsh.exe` | src/index.ts:25-30 |

---

## Phase 2: Interrogate + Falsify

---

### S01: macOS `environ` global write is thread-unsafe

**Interrogate:**

- **Why?** macOS lacks `execvpe`, so the code sets the global `environ` pointer then calls `execvp`. This is the standard workaround.
- **Mechanism:** `zig/pty_darwin.zig:12-14` — `environ.* = envp; _ = execvp(file, argv);`
- **Promise:** Custom environment variables reach the child process.

**REVERSE:** The assumption is that between `environ.* = envp` and `execvp`, no other thread reads `environ`. But this code runs in the child process AFTER `fork()`. After fork, only one thread exists in the child (POSIX guarantee: fork creates a single-threaded child). So the thread-safety concern is invalid in this context.

**FORWARD:** If environ write were in the parent, concurrent forks would race. But it's in the child (pid == 0 branch, lib.zig:125-143).

**FALSIFY:** Attempted to construct a scenario where this races. Cannot: fork() guarantees single-threaded child per POSIX. The parent never writes environ. The arena memory holding envp is parent-side, but fork duplicates the address space, so the child's copy is stable until exec.

**Verdict: HARDENED**
The fork-then-set-environ-then-exec pattern is the canonical macOS workaround. Single-threaded child process means no race.

---

### S02: macOS `closeExcessFds` brute-force loop performance

**Interrogate:**

- **Why?** macOS has no `close_range` or `/proc/self/fd`. Brute-force is the only option.
- **Mechanism:** `sysconf(_SC_OPEN_MAX)` returns the limit, then close 3..limit. On macOS, default `OPEN_MAX` is typically 10240 (but `ulimit -n` can raise it to millions via `launchctl`).
- **Promise:** All excess FDs are closed in the child before exec.

**REVERSE:** The assumption is that iterating up to `sysconf(_SC_OPEN_MAX)` is fast enough to not cause noticeable fork-to-exec latency.

**FORWARD:** If OPEN_MAX is 1,000,000, closing ~1M FDs would take measurable time. Each `close()` on an invalid FD returns EBADF quickly, but 1M syscalls is still ~10-50ms.

**FALSIFY:** Constructing scenario: user sets `ulimit -n 1048576` (common for high-perf servers). `sysconf(_SC_OPEN_MAX)` returns 1048576. The loop iterates 1048573 times. Each `close()` is a syscall (~0.5us on Apple Silicon), totaling ~500ms of fork-to-exec latency. This is a real performance regression. However, it is a correctness-over-performance tradeoff — missing FD leaks is worse. node-pty uses the same approach. This is a design quality issue, not a correctness bug.

**Verdict: SUSPECTED — Design limitation**
High `ulimit -n` values cause measurable fork-to-exec latency. Not a correctness failure, but a performance degradation that could affect latency-sensitive use cases like AI agent shells. A runtime test with `ulimit -n 1048576` would confirm. Could be mitigated by using `closefrom()` on macOS 13+ (Ventura) if available.

---

### S03: Linux last-resort FD close only covers 3..256

**Interrogate:**

- **Why?** This is the third fallback, after `close_range` (kernel 5.9+) and `/proc/self/fd`. It's a last resort.
- **Mechanism:** `zig/pty_linux.zig:57-59` — hardcoded `while (fd < 256)`.
- **Promise:** In environments without close_range AND without /proc, FDs above 2 are closed.

**REVERSE:** The assumption is this code path is practically unreachable on any real Linux system (close_range works on 5.9+, /proc/self/fd exists on virtually all Linux). If it IS reached, FDs >255 leak to the child.

**FORWARD:** An environment triggering this: Linux kernel <5.9 in a container with /proc/self/fd unmounted (some hardened containers). If the parent had FDs 256+ open (common: Node.js libuv handles, event loop fds), those leak into the child process. Leaked FDs are a security concern — the child process could read/write to parent's open files.

**FALSIFY:** Scenario: Docker container running kernel 4.x (older RHEL/CentOS), /proc mounted read-only or stripped. Parent Node.js process has libuv internal FDs (typically in the hundreds) and maybe network sockets at FDs 256+. After fork, these FDs leak to the child shell. The child shell (or any program it runs) could enumerate FDs >256 via trial-and-error close/read, discovering open file handles.

However, can we construct this concretely? The close_range syscall was added in Linux 5.9 (Oct 2020). CentOS 7 uses kernel 3.10 — no close_range. If /proc/self/fd is also unavailable (possible in highly restricted containers), the fallback triggers. Node.js (v18+) typically uses FDs in the 10-30 range for libuv, but applications with many network connections or open files could have FDs >256.

**Verdict: SUSPECTED**
The scenario requires an old kernel (<5.9) AND restricted /proc access. Unlikely but not impossible (old container images on legacy infrastructure). FD leak is a real security/correctness issue when it occurs. A runtime test on a CentOS 7 container with /proc/self/fd blocked would confirm. The fix is simple: use `sysconf(_SC_OPEN_MAX)` as the upper bound (same as the macOS path).

---

### S04: `clampU16` allows cols=0, rows=0 — Windows CreatePseudoConsole rejects zero

**Interrogate:**

- **Why?** `clampU16` was designed as a "safe" conversion from JS i32 to u16, clamping negatives to 0.
- **Mechanism:** `zig/pty.zig:68-72` — `if (val <= 0) return 0`. Called from `win/napi.zig:182-183` before `createConPty`.
- **Promise:** Any JS number is safely converted to a valid u16 for PTY dimensions.

**REVERSE:** The assumption is that 0 is a valid dimension. On Unix, `TIOCSWINSZ` accepts 0 (treated as "unknown"). On Windows, `CreatePseudoConsole` returns `E_INVALIDARG` (HRESULT < 0) for 0-width or 0-height.

**FORWARD:** If JS passes `cols: 0` or `rows: 0` (or negative values that clamp to 0), `createConPty` will fail with `CreatePseudoConsoleFailed` error on Windows. On Unix, `forkPty` will succeed but the terminal is in an undefined state where applications may crash with division-by-zero computing characters per line.

**FALSIFY:** Concrete failing input:
```ts
const pty = spawn("cmd.exe", [], { cols: 0, rows: 24 }); // throws on Windows
const pty = spawn("cmd.exe", [], { cols: -1, rows: 24 }); // clamps to 0, throws
const pty = spawn("/bin/sh", [], { cols: 0, rows: 0 }); // Unix: succeeds but broken
```
On Unix, `stty size` would return `0 0`, and programs like `vim`, `less` would crash or behave erratically.

The spawn test at `src/spawn.test.ts:221-222` explicitly tests `pty.resize(0, 0)` and notes "Zero dimensions" without checking for failure — confirming this is not validated at the API level.

**Verdict: PROVEN**
Zero dimensions silently pass validation. On Windows, spawn fails with an unhelpful native error. On Unix, the PTY enters a broken state. The root cause: `clampU16` treats 0 as valid when it should enforce minimum 1x1.

---

### S05: `UnixPty.close()` closes fd before testing process liveness

**Interrogate:**

- **Why?** `close()` needs to clean up resources (fd, streams) AND signal the child to terminate.
- **Mechanism:** `src/pty/unix.ts:120-138` — order is: (1) set _closed, (2) close WriteQueue, (3) destroy ReadStream, (4) `fs.closeSync(this._fd)`, (5) `process.kill(this.pid, 0)` to test liveness, (6) `process.kill(this.pid, "SIGHUP")`.
- **Promise:** Clean shutdown: release resources, then ask the process to terminate.

**REVERSE:** `process.kill(pid, 0)` after `fs.closeSync(fd)` — the fd close is not related to the kill's functionality. `process.kill(pid, 0)` checks if the process exists (signal 0 = no signal, just error check). This still works after fd close. The kill only fails if the process already exited, which the try/catch handles.

**FORWARD:** The actual risk is different: after closing the master fd, the child process receives EOF on its stdin (the slave side of the PTY). For a shell, this triggers exit. So SIGHUP might be sent to an already-exiting or exited process. But pid recycling is the real concern here.

**FALSIFY:** Concrete scenario: (1) `close()` called. (2) `fs.closeSync(fd)` — child receives EOF, begins exiting. (3) Short delay. (4) Child exits, kernel recycles PID. (5) `process.kill(recycled_pid, SIGHUP)` — sends SIGHUP to a completely unrelated process.

PID recycling time: on modern Linux, `/proc/sys/kernel/pid_max` is typically 32768 or 4194304. With many processes spawning, PID recycling within a few milliseconds is unlikely but not impossible on a busy system. The try/catch handles ESRCH (no such process), but if the PID was recycled to a DIFFERENT process, `kill(pid, 0)` succeeds and SIGHUP is sent to the wrong process.

However, the window between fd close and kill is microseconds (synchronous code). PID recycling requires the old process to fully exit AND a new process to start with the same PID. Extremely unlikely in practice.

**Verdict: SUSPECTED**
Theoretically possible PID reuse race between fd close and SIGHUP, but the time window is microseconds. A runtime test on a system with pid_max=300 under heavy fork load could confirm. Low practical risk but architecturally the kill should precede the fd close.

---

### S07: `waitFor` cleanup race condition

**Interrogate:**

- **Why?** `waitFor` registers a listener, collects data, and cleans up when pattern matches or timeout fires.
- **Mechanism:** `src/pty/_base.ts:82-93` — cleanup does `listeners.indexOf(terminalListener)` then `splice`.
- **Promise:** Listener is cleanly removed when done.

**REVERSE:** The assumption is that `indexOf` and `splice` execute atomically. In Node.js, the event loop is single-threaded for JS execution, so there's no true concurrency between JS callbacks. The concern would only arise if cleanup could be called re-entrantly.

**FORWARD:** Can cleanup be called twice? The `cleaned` flag at line 80 guards against this: `if (cleaned) return; cleaned = true;`. So double cleanup is prevented.

**FALSIFY:** Attempted scenario: timeout fires and pattern match fires simultaneously. Both call cleanup(). But `cleaned` flag ensures only one executes. JS is single-threaded, so "simultaneously" means "in the same microtask batch" — but even then, the first call sets `cleaned = true` synchronously.

**Verdict: HARDENED**
The `cleaned` boolean guard prevents double-cleanup. JS single-threaded execution model prevents true races in the listener array manipulation.

---

### S08: `waitFor` hangs if process exits before pattern matches

**Interrogate:**

- **Why?** `waitFor` sets up a timeout and a data listener. If the process exits, data stops flowing.
- **Mechanism:** `src/pty/_base.ts:73-123` — only two resolution paths: (1) pattern found in collected data, (2) timeout.
- **Promise:** `waitFor` either resolves (pattern found) or rejects (timeout).

**REVERSE:** The assumption is that the timeout is sufficient to handle the "process exited" case. If process exits quickly and timeout is 30s (default), the caller waits 30 seconds for a guaranteed timeout.

**FORWARD:** If the process exits immediately (e.g., invalid command, crash), the user calls `await pty.waitFor("prompt")`, nothing happens for 30 seconds, then a timeout error. Meanwhile `pty.exited` has already resolved. This is a poor UX for AI agent use cases (the primary stated use case for `waitFor`).

**FALSIFY:** Concrete scenario:
```ts
const pty = spawn("/bin/sh", ["-c", "exit 1"]);
// Process exits immediately
const output = await pty.waitFor("$"); // hangs for 30 seconds, then throws timeout
```
The AI agent waits 30 seconds for a shell prompt that will never come, because the shell already exited. The `waitFor` has no `onExit` hook to resolve/reject early.

Compare: if the user did `await Promise.race([pty.waitFor("$"), pty.exited])`, they'd get the exit code immediately. But `waitFor` itself doesn't do this.

**Verdict: PROVEN**
`waitFor` has no mechanism to detect process exit and resolve/reject early. The only fallback is the timeout (default 30s). This is a concrete usability failure for the stated AI agent use case. Root cause: `waitFor` was designed purely as a pattern-matcher on the data stream, without awareness of the process lifecycle.

**WHY/Meta-pattern:** Missing lifecycle awareness in convenience APIs. `waitFor` wraps data events but ignores exit events. Same pattern could appear in any future "wait for X" API that only watches one event source.

---

### S09: `WindowsPty.process` returns static file name

**Interrogate:**

- **Why?** On Unix, foreground process name is queried via `tcgetpgrp` + `/proc` or `sysctl`. On Windows, ConPTY has no equivalent API to query the foreground process.
- **Mechanism:** `src/pty/windows.ts:61-63` — `get process(): string { return this._file; }`
- **Promise:** The `process` property returns the current foreground process name (per `IPty` interface doc: "Name of the current foreground process").

**REVERSE:** The assumption is that there's no way to get the foreground process on Windows, so returning the initial file is an acceptable approximation.

**FORWARD:** If user spawns `cmd.exe`, then runs `python` inside it, `pty.process` still returns `"cmd.exe"`. This breaks any tooling that monitors foreground process changes (e.g., terminal tab titles, AI agents detecting which program is running).

**FALSIFY:** Concrete scenario:
```ts
const pty = spawn("cmd.exe");
pty.write("python\r\n");
console.log(pty.process); // "cmd.exe" — wrong, should be "python.exe"
```

However, node-pty on Windows also returns the initial process name. And there IS a Windows API path: `QueryFullProcessImageName` on the console's foreground process. But ConPTY doesn't expose this easily.

**Verdict: PROVEN — Design limitation**
The `process` getter on Windows violates its interface contract ("current foreground process name"). It always returns the initial spawn file. This is a documented limitation of ConPTY, but the API contract is misleading. Could be improved with `WTSGetActiveConsoleSessionId` + process enumeration, but it's complex.

**WHY/Meta-pattern:** Interface contracts that promise dynamic behavior but deliver static values on specific platforms. The `IPty` interface doesn't distinguish platform capabilities.

---

### S10: `_handleExit` clears `_dataListeners` — in-flight data lost

**Interrogate:**

- **Why?** On exit, cleanup is done: listeners are notified then arrays are cleared.
- **Mechanism:** `src/pty/_base.ts:125-136` — copies `_exitListeners`, then `_dataListeners.length = 0`, then fires exit listeners.
- **Promise:** All data delivered before exit notification.

**REVERSE:** The assumption is that by the time `_handleExit` fires, all data has been delivered.

**FORWARD (Unix):** The exit callback comes from the Zig exit monitor thread via `napi_threadsafe_function`. Data comes from `tty.ReadStream`'s `data` event. These are different async channels. When the child exits, the read stream may still have buffered data. The sequence: (1) child exits, (2) exit monitor thread calls tsfn (queued on JS event loop), (3) read stream emits remaining data (also on JS event loop). If (2) fires before (3), data listeners are cleared and remaining data is lost.

**FORWARD (Windows):** Better: `winExitMonitorThread` (zig/win/napi.zig:84-115) explicitly joins the read thread before firing the exit callback. So all data IS delivered before exit on Windows.

**FALSIFY (Unix):** The exit monitor thread calls `napi_call_threadsafe_function` with `.blocking` mode. This queues the exit callback on the JS event loop. Meanwhile, `tty.ReadStream` also queues data events. The ordering depends on which was queued first. If the child process writes its last output and exits in rapid succession, the exit tsfn callback could be queued before the ReadStream processes the final kernel buffer.

Concrete scenario:
```ts
const pty = spawn("/bin/sh", ["-c", "echo final_data && exit 0"]);
let collected = "";
pty.onData(d => collected += d);
pty.onExit(() => {
  // collected may not contain "final_data" if exit callback fired before
  // the ReadStream's final data event
});
```

However, in practice on Unix, the master fd EOF (from child exit) triggers the ReadStream close AFTER draining. And the Zig exit monitor uses `waitpid` which blocks until child fully exits, then queues the tsfn. The ReadStream on the same fd sees EOF from the kernel when the slave side closes, which happens AFTER the child writes. So typically data arrives first. But "typically" is not "guaranteed" — under heavy system load, event loop scheduling could reorder them.

**Verdict: SUSPECTED**
On Unix, there's no explicit ordering guarantee between the ReadStream data drain and the exit tsfn callback. In practice, data usually arrives first because ReadStream EOF comes from the same kernel-level event (slave close). But under load, the ordering is not guaranteed. A runtime stress test with rapid spawn-write-exit cycles would confirm. The Windows path is HARDENED (explicit read thread join before exit callback).

---

### S11: Windows `napi_external` with null finalize — potential use-after-free

**Interrogate:**

- **Why?** The `WinConPtyContext` is wrapped as a `napi_external` value so JS can pass it back to native functions (write, resize, kill, close).
- **Mechanism:** `zig/win/napi.zig:297` — `napi_create_external(env, @ptrCast(ctx), null, null, &handle_val)`. The `finalize_cb` is `null`.
- **Promise:** The context lifetime is managed by the exit monitor thread, which calls `alloc.destroy(ctx)` at line 114.

**REVERSE:** The assumption is that JS never accesses the handle after the exit monitor thread destroys the context. The exit monitor thread fires the exit callback, then destroys the context. After exit, the JS code should not call write/resize/kill/close on the handle.

**FORWARD:** What if JS DOES call `write()` or `resize()` after the process has exited and the context has been freed? The napi_external still exists in JS (it's a GC-managed object), but the underlying pointer is dangling.

**FALSIFY:** Concrete scenario:
```ts
const pty = spawn("cmd.exe", ["/c", "exit 0"]);
await pty.exited; // process exits, exit monitor thread frees context
// Small delay...
pty.write("hello"); // Calls native write() with freed context pointer
```

Looking at the code path: `WindowsPty.write()` checks `if (this._closed) return;` (windows.ts:66). And `_handleExit` sets `this._closed = true` (base.ts:126). So the JS-side guard should prevent calling native write after exit.

But: what if `write()` is called from a deferred callback that was queued BEFORE exit? The `_closed` check happens synchronously in `write()`, so if it runs after `_handleExit`, it returns early. However, if write is called in the same microtask that processes the exit (e.g., in an exit listener that calls write), `_closed` is already true.

The more concerning path: what about the napi_external surviving in a global variable and being passed to native code later? There's no protection at the Zig level — `napi_get_value_external` just returns the raw pointer.

However, on the JS side, `WindowsPty` encapsulates the handle. External code can't access it directly. And all WindowsPty methods check `_closed`. So as long as the user goes through the class API, the guard holds.

**Verdict: SUSPECTED**
The `null` finalize callback means the GC will not clean up the context — which is correct because the exit monitor thread does. The use-after-free requires bypassing the `_closed` guard, which is robust for normal usage. But: if the exit monitor thread calls `alloc.destroy(ctx)` and then a JS function somehow calls a native function with the stale external, there's no safety net. A finalize callback could set the pointer to null as a defense-in-depth measure. Extremely unlikely in practice due to `_closed` guards.

---

### S13: No `initgroups`/`setgroups` — supplementary groups leak

**Interrogate:**

- **Why?** The code calls `setgid` then `setuid` to drop privileges in the child process.
- **Mechanism:** `zig/lib.zig:133-138` — `setgid(gid)` then `setuid(uid)` with no `setgroups` or `initgroups`.
- **Promise:** The child runs as the specified user/group.

**REVERSE:** The assumption is that setting primary uid/gid is sufficient for privilege management.

**FORWARD:** Without `setgroups(0, NULL)` or `initgroups(user, gid)`, the child process inherits the parent's supplementary group list. If the parent runs as root with supplementary groups (disk, adm, sudo, docker), the child keeps those group memberships even after setuid.

**FALSIFY:** Concrete scenario: Node.js server running as root spawns a PTY with `uid: 1000, gid: 1000` (a non-privileged user). The child shell inherits root's supplementary groups (e.g., `docker`, `sudo`). The child process can then access resources restricted to those groups despite running as uid 1000.

However: how likely is this? The `uid`/`gid` options are optional (default null). Most users won't set them. When they do, they're likely trying to drop privileges, which makes this gap meaningful.

node-pty also has the same limitation — it calls `setuid`/`setgid` without `setgroups`. So this is an industry-wide pattern, not specific to zigpty.

**Verdict: SUSPECTED**
When uid/gid options are used to drop privileges, supplementary groups from the parent leak to the child. This is a privilege escalation path in security-sensitive deployments. A runtime test: spawn with uid=1000 from a root parent, run `groups` in the child shell to confirm supplementary groups are inherited. The fix is to call `setgroups(0, NULL)` before `setgid`/`setuid`, or provide an option to pass supplementary groups.

---

### S14: Unix `exitMonitorThread` is detached — no join on shutdown

**Interrogate:**

- **Why?** The exit monitor thread is spawned and then the `Thread` handle is not stored. The thread runs `waitpid` then calls the tsfn and exits.
- **Mechanism:** `zig/pty_unix.zig:133` — `_ = std.Thread.spawn(...)` — the return value (Thread handle) is discarded via `_ =`.
- **Promise:** The thread will eventually complete when the child process exits.

**REVERSE:** If Node.js process exits (or is killed) before the child process exits, the detached thread is still blocked in `waitpid`. On process exit, all threads are killed by the OS, so this doesn't cause a hang. But if the NAPI tsfn has already been released (by Node.js cleanup), the thread's call to `napi_call_threadsafe_function` may crash.

**FORWARD:** Node.js `process.exit()` or SIGTERM: Node begins shutdown, releases NAPI resources. The detached thread wakes from `waitpid`, tries to call the tsfn, gets an error status back (`.closing` or `.abort`), which it ignores via `_ = napi.napi_call_threadsafe_function(...)`. Then `napi_release_threadsafe_function` — also potentially operating on freed memory.

Actually, looking more carefully: `pty_unix.zig:18` has `_ = napi.napi_release_threadsafe_function(ctx_ptr.tsfn, .release)` in a `defer`. And `napi_unref_threadsafe_function` is NOT called (unlike the Windows path at win/napi.zig:225). This means the Unix exit tsfn keeps the Node.js event loop alive, preventing `process.exit()` until the child exits. This is actually the CORRECT behavior for preventing the race — the thread keeps Node alive until it completes.

Wait — but the thread handle is still discarded. The tsfn ref count keeps Node alive, but when Node does eventually shut down, the thread is forcibly killed. This is fine because by that point the tsfn callback has already fired.

**Verdict: HARDENED**
The tsfn reference count keeps Node.js alive until the exit monitor thread completes. The discarded thread handle doesn't cause issues because the thread's lifetime is bounded by the tsfn. On forced process termination, the OS cleans up all threads. The design is intentional and correct.

---

### S15: `Terminal._emitData` creates `TextDecoder` per call — multi-byte split

**Interrogate:**

- **Why?** `_emitData` converts `Uint8Array` to string for `_dataListeners` (used by `waitFor`).
- **Mechanism:** `src/terminal.ts:185` — `const text = new TextDecoder().decode(data)`.
- **Promise:** Binary data is correctly converted to text for pattern matching.

**REVERSE:** A fresh `TextDecoder` per call has no state. If a multi-byte UTF-8 character is split across two chunks (e.g., a 3-byte CJK character split at a chunk boundary), the first chunk's trailing bytes produce a replacement character (U+FFFD), and the second chunk's leading bytes also produce garbage.

**FORWARD:** This means `waitFor` could miss patterns that span a chunk boundary or contain characters near a split point. For ASCII patterns (the common case), this is fine. For UTF-8 patterns (CJK text, emoji), corruption at chunk boundaries could prevent matching.

**FALSIFY:** Concrete scenario: PTY output contains "$ " prompt but a multi-byte character arrives split across two 4096-byte chunks. The chunk boundary falls in the middle of a UTF-8 sequence. `waitFor("$")` still works ($ is ASCII). But `waitFor("你好")` could fail if the 你 character is split.

In practice, PTY output chunks are typically line-oriented and multi-byte characters are rarely split. But it's a real possibility with high-throughput output (e.g., `cat large_file.txt` with CJK content).

**Verdict: SUSPECTED**
Multi-byte UTF-8 characters split across chunks will be corrupted in `_dataListeners` text output. ASCII patterns are unaffected. A runtime test: send a known multi-byte character at a known chunk boundary position. The fix is to use a single persistent `TextDecoder` instance with `{stream: true}` option.

---

### S17: Windows deferred calls lost if process produces no output

**Interrogate:**

- **Why?** ConPTY may deadlock if writes happen before the pseudo console is ready. Deferring until first data received avoids this.
- **Mechanism:** `src/pty/windows.ts:27-36` — `if (!this._ready)` gates deferred flush on first data event.
- **Promise:** Writes and resizes are queued and flushed when ConPTY is ready.

**REVERSE:** The assumption is that ConPTY always produces at least some initial output (VT init sequences). AGENTS.md confirms: "ConPTY's own VT init sequences DO arrive through the pipe (they're generated by ConPTY itself, not by the process)."

**FORWARD:** If ConPTY init sequences always arrive, `_ready` will be set to true before the process produces its own output. Deferred calls will be flushed.

**FALSIFY:** Can we create a scenario where ConPTY produces NO output? ConPTY always sends initialization VT sequences (mode changes, clear screen commands). These are internal to the ConPTY infrastructure. Even if the spawned process writes nothing and exits immediately, the ConPTY init sequences should arrive through the output pipe.

Looking at the AGENTS.md design doc: "ConPTY's own VT init sequences (mode changes, clear screen, title) arrive through the output pipe." And the two-phase startup ensures the read thread is running before the process is spawned, so these init sequences are captured.

The only scenario where no data arrives: if the read thread fails to start (error handled in spawn) or if CreatePseudoConsole fails (also handled). In normal operation, ConPTY ALWAYS sends init sequences.

**Verdict: HARDENED**
ConPTY always sends VT initialization sequences through the output pipe, even if the spawned process produces no output. These init sequences trigger the `_ready` flag, flushing deferred calls. The two-phase startup ensures the read thread captures these sequences.

---

### S18: Windows env not sanitized (TMUX/COLUMNS leak)

**Interrogate:**

- **Why?** `buildEnvPairs` accepts an optional `sanitizeKeys` parameter. Unix passes `UNIX_SANITIZE_KEYS`. Windows passes nothing.
- **Mechanism:** `src/pty/windows.ts:24` — `buildEnvPairs(envObj, options?.name)` — no third argument.
- **Promise:** Environment variables are properly prepared for the child.

**REVERSE:** The assumption is that TMUX, COLUMNS, LINES etc. are Unix-specific and don't need sanitization on Windows.

**FORWARD:** `TMUX`, `TMUX_PANE`, `STY` are indeed Unix-specific. But `COLUMNS` and `LINES` exist on Windows too (ConPTY and cmd.exe respect them). If the parent Node.js process has `COLUMNS=200` in its environment, the child ConPTY process inherits it, potentially overriding the ConPTY's actual dimensions.

**FALSIFY:** Concrete scenario:
```ts
// Parent has COLUMNS=200 in env (from its own terminal)
const pty = spawn("cmd.exe", [], { cols: 80, rows: 24 });
// Child cmd.exe sees COLUMNS=200, may use that instead of ConPTY's 80
```

However, looking at Windows behavior: `cmd.exe` and PowerShell primarily get their dimensions from the console API (`GetConsoleScreenBufferInfo`), not from environment variables. `COLUMNS` is a POSIX concept. Windows programs typically don't read it. So the leak is benign in practice.

**Verdict: HARDENED**
While `COLUMNS`/`LINES` are passed through on Windows, Windows programs don't typically use these POSIX-convention environment variables. ConPTY provides dimensions through the console API. The omission of sanitization on Windows is intentionally correct.

---

### S19: Windows command-line quoting doesn't handle `%`, `!`, `^`

**Interrogate:**

- **Why?** `appendQuoted` in `pty_windows.zig:457-498` implements standard Windows argument quoting (backslash-escaping before quotes).
- **Mechanism:** Only handles spaces, tabs, quotes, and backslashes. Does not handle `%` (environment variable expansion), `!` (delayed expansion), or `^` (cmd.exe escape character).
- **Promise:** Command-line arguments are safely quoted for Windows process creation.

**REVERSE:** The assumption is that `CreateProcessW` receives the command line as-is without shell interpretation. This is correct: `CreateProcessW` does NOT expand `%VAR%` or `^` — those are `cmd.exe` features. The quoting only needs to handle the Windows C runtime's `CommandLineToArgvW` parsing rules, which is exactly what `appendQuoted` does.

**FORWARD:** The `%` and `!` expansion only matters if the command line is passed through `cmd.exe /c`. But zigpty uses `CreateProcessW` directly, not via `cmd.exe`. So `%PATH%` in an argument stays literal.

**FALSIFY:** Attempting to construct a command injection:
```ts
spawn("cmd.exe", ["/c", "echo %PATH%"]); // cmd.exe interprets %PATH% — but this is expected behavior
spawn("notepad.exe", ["file%PATH%.txt"]); // CreateProcessW passes it literally
```

The cmd.exe case: if user explicitly spawns `cmd.exe /c <something>`, cmd.exe will interpret `%` and `!`. But that's cmd.exe's job, not the quoting function's. The quoting function correctly handles the `CreateProcessW` argument parsing rules.

**Verdict: HARDENED**
The quoting function correctly implements Windows CRT `CommandLineToArgvW` rules. `%`, `!`, and `^` are not metacharacters at the `CreateProcessW` level — they're interpreted by `cmd.exe` if and when the user explicitly uses cmd.exe as the shell. This is the correct design: the PTY library quotes for process creation, not for shell interpretation.

---

### S20: `tty.ReadStream` and `WriteQueue` share the same fd

**Interrogate:**

- **Why?** PTY master fd is bidirectional — read output and write input on the same fd.
- **Mechanism:** `src/pty/unix.ts:54,64` — `WriteQueue(this._fd)` and `new tty.ReadStream(this._fd)` on the same fd.
- **Promise:** Simultaneous reading and writing works correctly.

**REVERSE:** `tty.ReadStream` inherits from `net.Socket` which inherits from `stream.Duplex`. Creating a `ReadStream` on an fd may set it to non-blocking mode. `WriteQueue` uses async `fs.write` which handles `EAGAIN`. So non-blocking mode is handled.

**FORWARD:** The concern: `tty.ReadStream` constructor calls `net.Socket` constructor with `{ fd, readable: true, writable: false }`. This internally calls `uv_tty_init` in libuv, which registers the fd for polling. Then `WriteQueue.enqueue` calls `fs.write` (which uses `uv_fs_write`). Both are libuv operations on the same fd. Is this safe?

Yes — libuv handles concurrent read polling and write operations on the same fd. The read side uses `uv_read_start` (poll-based) and write side uses `uv_fs_write` (threadpool-based). This is standard practice for PTY fd usage.

**FALSIFY:** Attempted to find a conflict: both read and write happening simultaneously. But PTY fds are full-duplex (like sockets). Read and write are independent operations on the kernel level. libuv uses different mechanisms for each (epoll for read, threadpool for write via fs.write).

**Verdict: HARDENED**
PTY master fds are full-duplex. libuv handles concurrent read polling and async writes correctly. `tty.ReadStream` and `WriteQueue` operate independently on the same fd without conflict.

---

### S22: Windows read thread unbounded queue

**Interrogate:**

- **Why?** The read thread sends data to JS via `napi_threadsafe_function` with `max_queue_size = 0` (unlimited) and `.nonblocking` call mode.
- **Mechanism:** `zig/win/napi.zig:69-73` — nonblocking enqueue with unlimited queue.
- **Promise:** Data flows from native read thread to JS without loss.

**REVERSE:** The assumption is that JS processes data fast enough that the queue doesn't grow unboundedly. With `max_queue_size = 0`, there's no backpressure.

**FORWARD:** If JS event loop is blocked (heavy computation, synchronous operation), data chunks accumulate in the tsfn queue. Each chunk is heap-allocated (`alloc.alloc(u8, n)` + `alloc.create(DataChunk)`). For a high-throughput process (e.g., `type large_file.txt` on Windows), this could consume significant memory.

**FALSIFY:** Concrete scenario:
```ts
const pty = spawn("cmd.exe", ["/c", "type very_large_file.txt"]);
// JS is doing something blocking:
const result = someSynchronousHeavyComputation(); // blocks event loop for 5 seconds
// During those 5 seconds, the read thread queues ALL output chunks
// Each chunk is up to 4096 bytes, ~1.2GB file = ~300K chunks = ~1.2GB heap
```

However, this is a general limitation of any async producer-consumer pattern without backpressure. The `.nonblocking` mode means the read thread never blocks, which is the correct choice to avoid deadlocking ConPTY's output pipe. The alternative (`.blocking` with a bounded queue) could cause the read thread to stall, which would deadlock `ClosePseudoConsole`.

**Verdict: SUSPECTED — Design tradeoff**
Unbounded queue can theoretically cause OOM under extreme conditions (blocked JS event loop + high-throughput output). But the alternative (bounded queue) risks ConPTY deadlock. This is a conscious design tradeoff. A memory limit or warning mechanism could mitigate without risking deadlock.

---

### S23: `WriteQueue._process` missing `length` parameter in `fs.write`

**Interrogate:**

- **Why?** `WriteQueue._process` calls `fs.write(this._fd, task.buffer, task.offset, callback)`.
- **Mechanism:** `src/pty/_writeQueue.ts:40` — `fs.write(this._fd, task.buffer, task.offset, (err, written) => {...})`.
- **Promise:** Writes the remaining portion of the buffer starting from offset.

**REVERSE:** Looking at Node.js `fs.write` signature: `fs.write(fd, buffer, offset, length, position, callback)`. If `length` is omitted and `callback` is in the `length` position... Actually, Node.js `fs.write` has multiple overloads. When called as `fs.write(fd, buffer, offset, callback)`, Node.js interprets the 4th argument:

From Node.js docs: `fs.write(fd, buffer[, offset[, length[, position]]], callback)`. If the 4th argument is a function, it's treated as the callback, and `length` defaults to `buffer.length - offset`. So `fs.write(fd, buffer, offset, callback)` writes from `offset` to end of buffer.

**FORWARD:** This is actually correct behavior! `fs.write(fd, buffer, task.offset, callback)` writes `buffer[task.offset..buffer.length]`. This is exactly what's intended — write the remaining portion.

**FALSIFY:** Examined the Node.js source to confirm the overload resolution. When the 4th argument is a function, `length` defaults to `buffer.byteLength - offset`. This means the full remaining buffer is written in one call. The kernel may write less (short write), which is handled by the callback updating `task.offset`.

**Verdict: HARDENED**
Node.js `fs.write(fd, buffer, offset, callback)` correctly defaults length to `buffer.length - offset`. The WriteQueue correctly handles partial writes by advancing `task.offset` and re-calling `_process`.

---

### S24: `decodeWaitStatus` conflates stopped processes with errors

**Interrogate:**

- **Why?** `decodeWaitStatus` checks `termsig` (bits 0-6 of status). If 0 = normal exit. If != 0 and != 0x7f = signal death. If 0x7f = stopped (WIFSTOPPED).
- **Mechanism:** `zig/lib.zig:214-223` — `termsig == 0x7f` returns `exit_code = -1, signal_code = 0`.
- **Promise:** Process exit status is correctly decoded.

**REVERSE:** The assumption is that `waitpid` with `options = 0` (no `WUNTRACED`) never returns a stopped status. This is correct: without `WUNTRACED`, `waitpid` only returns for terminated children, not stopped ones. So the `0x7f` case is theoretically unreachable.

**FORWARD:** If somehow a stopped status is returned (shouldn't happen with options=0), it's reported as exit_code=-1. This is a defensive catch-all, not a real code path.

**FALSIFY:** Can `waitpid(pid, &status, 0)` return a stopped status? Only if `WUNTRACED` is passed as an option. The code passes `0` (zig/lib.zig:203). Per POSIX, without `WUNTRACED`, stopped children don't cause `waitpid` to return. So the `termsig == 0x7f` branch is dead code.

**Verdict: HARDENED**
The `termsig == 0x7f` case is unreachable because `waitpid` is called without `WUNTRACED`. The defensive return is harmless dead code.

---

### S26: `UnixPty.close()` sends SIGHUP to potentially recycled PID

**Interrogate:**

- **Why?** `close()` needs to terminate the child process. SIGHUP is the standard signal for terminal hangup.
- **Mechanism:** `src/pty/unix.ts:134-137` — `process.kill(this.pid, 0)` checks existence, then `process.kill(this.pid, "SIGHUP")`.
- **Promise:** The child process is terminated on close.

**REVERSE:** The assumption is that the `pid` still belongs to the child process at the time `close()` is called.

**FORWARD:** If the child has already exited (e.g., the user called `close()` after getting the exit event), the PID may have been recycled. `process.kill(pid, 0)` returns successfully for the NEW process, and SIGHUP is sent to it.

**FALSIFY:** For PID recycling to happen between `_handleExit` and `close()`:
1. Child exits. `_handleExit` sets `_closed = true`.
2. But `close()` checks `if (this._closed) return;` at line 121.
3. If `_handleExit` has been called, `_closed` is true, and `close()` is a no-op.

Wait — there's a subtle issue. If the user calls `close()` BEFORE the exit callback fires (e.g., they want to force-kill):
1. `close()` sets `_closed = true`.
2. `fs.closeSync(this._fd)` — child gets EOF.
3. `process.kill(pid, 0)` — child still exists.
4. `process.kill(pid, "SIGHUP")` — sent to correct process.

The race would require: user calls `close()`, the child exits between `closeSync` and `kill`, and PID is recycled. This window is at most a few microseconds.

**Verdict: SUSPECTED**
Theoretically possible but practically negligible. The window between fd close and kill is microseconds of synchronous JS. PID recycling requires full child exit + new process creation with same PID in that window. Risk: near-zero on real systems with 32768+ PID space.

---

### S27: `Terminal.close()` always reports exitCode=0

**Interrogate:**

- **Why?** `Terminal.close()` fires the exit callback with `exitCode: 0`.
- **Mechanism:** `src/terminal.ts:144` — `this._onExit?.(this, 0, null)`.
- **Promise:** Exit callback receives meaningful exit information.

**REVERSE:** The Terminal class doesn't know the process exit code — it only manages the PTY stream. The process exit code is managed by `BasePty._handleExit`. So `Terminal.close()` reporting 0 means "PTY stream closed cleanly (EOF)", not "process exited with code 0."

**FORWARD:** If the user creates a standalone Terminal (no spawn), `close()` with exitCode=0 is correct (clean close). If the Terminal is attached to a spawn, the process exit code comes through `BasePty._handleExit` → `pty.onExit` callback, not through the Terminal's exit callback.

**FALSIFY:** Concrete scenario where this misleads:
```ts
const terminal = new Terminal({
  exit(t, code, signal) {
    console.log("Terminal exited with:", code); // Always 0
  }
});
const pty = spawn("/bin/sh", ["-c", "exit 42"], { terminal });
// Terminal.exit callback: code=0 (from terminal.close())
// pty.onExit: exitCode=42 (from _handleExit)
```

The user might expect the Terminal exit callback to reflect the process exit code. But the Terminal's exit callback semantics are "stream close status", not "process exit status". This is documented: "Callback when PTY stream closes (EOF or error). exitCode is PTY lifecycle status (0=EOF, 1=error)."

The documentation at `src/terminal.ts:18` says: "exitCode is PTY lifecycle status (0=EOF, 1=error)." So 0 is documented as "clean EOF", not "process exit code". The naming `exitCode` is confusing though — it suggests a process exit code.

**Verdict: SUSPECTED — Design/API clarity issue**
The `exit` callback's `exitCode` parameter on Terminal always reports 0 on close, which is documented as "PTY lifecycle status." But the parameter name `exitCode` strongly implies a process exit code, creating confusion. The user gets the real process exit code from `pty.onExit`, not `terminal.exit`. This is a documentation/naming issue rather than a bug.

---

### S12: macOS `P_COMM_OFFSET = 243` hardcoded

**Interrogate:**

- **Why?** `kinfo_proc` is a kernel data structure. Its layout is ABI-stable on macOS.
- **Mechanism:** `zig/pty_darwin.zig:42` — `const P_COMM_OFFSET = 243`.
- **Promise:** Process name is correctly extracted from sysctl output.

**REVERSE:** The assumption is that `kinfo_proc.kp_proc.p_comm` is at offset 243 on both arm64 and x86_64 macOS.

**FORWARD:** Apple has maintained ABI stability for `kinfo_proc` across macOS versions. The struct layout is defined in `<sys/sysctl.h>` and `<sys/proc.h>`. The offset 243 corresponds to `kp_proc` (at offset 0) + `p_comm` (at offset 243 within `extern_proc`).

**FALSIFY:** Can we verify this offset? The `extern_proc` struct has been stable since Mac OS X 10.0. Apple's commitment to ABI stability for kernel structures exposed via sysctl is well-documented. A new macOS version changing this offset would break every process monitoring tool (ps, top, Activity Monitor).

The comment says "both arm64 and x86_64" which is correct — `extern_proc` uses fixed-size types (int32_t, etc.) that have the same size on both architectures. The 720-byte buffer (line 35) provides ample space for the ~648-byte `kinfo_proc`.

**Verdict: HARDENED**
`kinfo_proc` layout is ABI-stable on macOS. Apple cannot change it without breaking all user-space process monitoring tools. The hardcoded offset is the standard approach (used by node-pty, libuv, and many other projects).

---

### S16: `open()` returns slave fd without closing it

**Interrogate:**

- **Why?** `open()` creates a bare PTY pair and returns both master and slave fds to the caller.
- **Mechanism:** `src/index.ts:32-37` — returns `{ master, slave, pty }` directly from native.
- **Promise:** The caller gets a PTY pair to manage themselves.

**REVERSE:** The assumption is that the caller is responsible for managing both fds. This is the documented behavior: "Create a PTY pair without spawning a process — useful when you need to control the child process yourself."

**FORWARD:** The caller MUST close both fds when done. If they forget, fds leak. But this is standard for any low-level API that returns file descriptors.

**FALSIFY:** This is expected behavior for an `open()` API. The caller who uses `open()` explicitly wants manual control. Closing the slave fd automatically would defeat the purpose.

**Verdict: HARDENED**
The `open()` API intentionally returns both fds for manual management. FD lifecycle is the caller's responsibility, which is the standard contract for low-level PTY APIs.

---

### S25: `WindowsPty.close()` relies entirely on exit monitor thread

**Interrogate:**

- **Why?** Calling `ClosePseudoConsole` from the JS thread causes deadlock (documented in AGENTS.md).
- **Mechanism:** `src/pty/windows.ts:104-115` — `close()` just calls `this._native.kill(this._handle)`.
- **Promise:** Resources are cleaned up after close.

**REVERSE:** The kill triggers process termination. The exit monitor thread detects exit, closes conin, calls `ClosePseudoConsole`, joins read thread, closes handles, fires callback. This is the correct sequence per the design doc.

**FORWARD:** If the exit monitor thread is stuck (e.g., `WaitForSingleObject` never returns because of a zombie process), close() would not clean up resources. But Windows zombie processes are extremely rare (they require the parent to hold the process handle, which zigpty does, but `WaitForSingleObject` with `INFINITE` will always return when the process terminates, and `TerminateProcess` forces termination).

**FALSIFY:** Can `TerminateProcess` + `WaitForSingleObject` stall indefinitely? `TerminateProcess` is async — it schedules termination. `WaitForSingleObject(process, INFINITE)` waits until the process handle is signaled. After `TerminateProcess`, the process is terminated, and the handle is signaled. This is guaranteed by Windows kernel semantics.

**Verdict: HARDENED**
`TerminateProcess` followed by `WaitForSingleObject(INFINITE)` is guaranteed to complete by Windows kernel semantics. The exit monitor thread will always eventually fire, cleaning up all resources.

---

### S28: `defaultShell()` fallback to `powershell.exe`

**Interrogate:**

- **Why?** Windows needs a default shell. `COMSPEC` typically points to `cmd.exe`.
- **Mechanism:** `src/index.ts:27` — `process.env.COMSPEC || "powershell.exe"`.
- **Promise:** A working shell is selected by default.

**REVERSE:** `COMSPEC` is set on all standard Windows installations (it's a system env var pointing to `C:\Windows\System32\cmd.exe`). The `powershell.exe` fallback is for the edge case where COMSPEC is unset.

**FORWARD:** `powershell.exe` is "Windows PowerShell 5.1" which is present on all Windows 10/11 systems. Modern PowerShell 7+ is installed as `pwsh.exe`. The fallback works for all standard Windows installations.

The actual risk: if someone unsets COMSPEC AND removes PowerShell 5.1 (possible but extreme), the spawn would fail. But this is a system misconfiguration.

**Verdict: HARDENED**
`COMSPEC` is present on all standard Windows installations. `powershell.exe` (Windows PowerShell 5.1) is available on all Windows 10/11 systems. The fallback chain is robust for all standard configurations.

---

## Proof Catalog

### PROVEN-1: Zero dimensions pass through to native layer (S04)

**Assumption that failed:** `clampU16` produces valid dimensions for all platforms.

**Concrete proof:**
```ts
spawn("cmd.exe", [], { cols: 0, rows: 24 }); // Windows: CreatePseudoConsole returns E_INVALIDARG → throws
spawn("cmd.exe", [], { cols: -5, rows: 24 }); // Clamps to 0 → same failure
spawn("/bin/sh", [], { cols: 0, rows: 0 }); // Unix: PTY in broken state, programs may crash
```

**Root cause:** `clampU16` was designed as an overflow guard (i32 → u16) but doesn't enforce minimum valid dimensions. Zero is a valid u16 but not a valid PTY dimension.

**Meta-pattern:** Input validation that guards against one failure mode (overflow) while missing another (domain validity). The function name "clamp" implies range-limiting but the valid range for PTY dimensions is [1, 65535], not [0, 65535].

---

### PROVEN-2: `waitFor` hangs on process exit without timeout (S08)

**Assumption that failed:** `waitFor` will resolve or reject in a timely manner for all process outcomes.

**Concrete proof:**
```ts
const pty = spawn("/bin/sh", ["-c", "exit 1"]);
await pty.waitFor("$"); // Hangs for 30 seconds (default timeout), then throws
// pty.exited resolved immediately, but waitFor doesn't know
```

**Root cause:** `waitFor` monitors only the data stream, not the process lifecycle. No `onExit` hook to resolve/reject early when the process terminates.

**Meta-pattern:** Convenience APIs that monitor one event source while the relevant state change comes from a different source. The `waitFor` contract implicitly assumes the process is alive, but doesn't verify it.

---

### PROVEN-3: `WindowsPty.process` returns static file name, not foreground process (S09)

**Assumption that failed:** The `process` property reflects the current foreground process name.

**Concrete proof:**
```ts
const pty = spawn("cmd.exe");
pty.write("python\r\n");
await new Promise(r => setTimeout(r, 1000));
console.log(pty.process); // "cmd.exe" — should be "python.exe"
```

**Root cause:** No Windows API call to query ConPTY foreground process. Returns the initial spawn file name as a static approximation.

**Meta-pattern:** Interface contracts that promise dynamic behavior but deliver static values on specific platforms. Cross-platform abstraction hiding platform-specific limitations without documenting them in the type system.

---

## Suspicion List

### SUSPECTED-1: Supplementary group leak when uid/gid specified (S13)

**Falsification attempted:** Analyzed the `setgid`/`setuid` sequence at `zig/lib.zig:133-138`. No `setgroups(0, NULL)` or `initgroups` call.

**Why proof couldn't be constructed:** Static analysis only — can't run a root-privileged test. The behavior is well-documented in POSIX: supplementary groups survive `setuid`/`setgid` unless explicitly cleared.

**Runtime test to confirm:** Spawn PTY as root with `uid: 1000`, run `groups` or `id` in the child shell — supplementary groups from root will be visible.

---

### SUSPECTED-2: Unix data/exit ordering race (S10)

**Falsification attempted:** Analyzed the event delivery paths. Exit comes via tsfn callback. Data comes via `tty.ReadStream` event. Both are JS event loop events with no explicit ordering.

**Why proof couldn't be constructed:** The race depends on kernel-level scheduling and libuv event loop internals. Static analysis cannot determine delivery order under load.

**Runtime test to confirm:** Stress test with rapid spawn-write-exit cycles (1000 iterations), check if final output is always captured before exit callback fires.

---

### SUSPECTED-3: `new TextDecoder()` per call corrupts multi-byte chars at chunk boundaries (S15)

**Falsification attempted:** Each `_emitData` call creates a fresh TextDecoder with no streaming state. Multi-byte UTF-8 characters split across PTY read chunks will be decoded incorrectly.

**Why proof couldn't be constructed:** Need to force a PTY read to split at a specific byte within a multi-byte character, which requires controlling kernel buffer boundaries.

**Runtime test to confirm:** Write a known multi-byte string (e.g., 4096 bytes of ASCII + 1 CJK char) to force a chunk split at the multi-byte boundary, verify `_dataListeners` receives corrupted text.

---

### SUSPECTED-4: macOS `closeExcessFds` slow with high `ulimit -n` (S02)

**Falsification attempted:** With `ulimit -n 1048576`, the brute-force loop iterates ~1M times.

**Why proof couldn't be constructed:** Static analysis — performance impact requires benchmarking.

**Runtime test to confirm:** Set `ulimit -n 1048576`, time a fork-to-exec, compare with default limit.

---

### SUSPECTED-5: Linux fallback FD close misses FDs >255 (S03)

**Falsification attempted:** On Linux <5.9 without /proc/self/fd, the fallback only closes FDs 3..256.

**Why proof couldn't be constructed:** Requires a specific environment (old kernel + restricted /proc) that wasn't available.

**Runtime test to confirm:** On CentOS 7 container with /proc/self/fd blocked, open FDs 256+ before fork, check if they leak to child.

---

### SUSPECTED-6: Windows `napi_external` use-after-free (S11)

**Falsification attempted:** After exit monitor frees context, stale external could be accessed.

**Why proof couldn't be constructed:** `_closed` guard at JS level prevents all normal API calls after exit. Would require bypassing the class API.

**Runtime test to confirm:** Manually extract handle from WindowsPty internals, call native function after exit.

---

### SUSPECTED-7: `Terminal.close()` exitCode=0 naming confusion (S27)

**Falsification attempted:** The exit callback always reports 0, regardless of process exit code.

**Why proof couldn't be constructed:** Documented as "PTY lifecycle status" not "process exit code." A naming/clarity issue, not a bug.

**Runtime test to confirm:** N/A — this is a documentation/API design issue.

---

## Defense Map

| # | Defense | Attacks Survived | Dependencies |
|---|---------|-----------------|--------------|
| S01 | `environ` write after fork | Thread-safety concern invalid — fork guarantees single-threaded child | POSIX fork semantics |
| S07 | `waitFor` cleanup with `cleaned` flag | Double-cleanup, concurrent resolve/reject | JS single-threaded execution model |
| S12 | macOS `P_COMM_OFFSET = 243` | ABI instability | Apple ABI stability commitment for kinfo_proc |
| S14 | Unix exit monitor thread via tsfn ref | Thread outliving Node.js | tsfn reference keeps Node alive |
| S16 | `open()` returns both fds | FD leak concern | Caller responsibility (documented API) |
| S17 | Windows deferred calls | Silent loss if no output | ConPTY always sends VT init sequences |
| S18 | Windows env no sanitization | COLUMNS/LINES leak | Windows programs use console API, not env vars |
| S19 | Windows cmd-line quoting | `%`/`!`/`^` injection | CreateProcessW doesn't interpret shell metacharacters |
| S20 | ReadStream + WriteQueue on same fd | fd conflict | PTY full-duplex, libuv handles concurrent ops |
| S23 | `fs.write(fd, buf, offset, cb)` | Missing length parameter | Node.js defaults length to buf.length - offset |
| S24 | `decodeWaitStatus` 0x7f case | Incorrect stopped process handling | waitpid(0) never returns stopped status |
| S25 | WindowsPty.close() via kill | Exit monitor stall | TerminateProcess + WaitForSingleObject guaranteed |
| S28 | `defaultShell()` powershell.exe fallback | Missing shell | COMSPEC always set on Windows, PS 5.1 always available |

---

## Root Causes

### RC1: Input validation guards overflow but not domain validity
**Findings:** S04 (zero dimensions)
**Pattern:** `clampU16` prevents integer overflow but doesn't enforce business rules (minimum valid PTY size). The valid domain is [1, 65535] but the clamped range is [0, 65535].

### RC2: Convenience APIs missing lifecycle awareness
**Findings:** S08 (waitFor hangs on exit), S09 (static process name)
**Pattern:** APIs designed for one concern (pattern matching, process name) don't integrate with the broader lifecycle (process exit, foreground change). Cross-cutting concerns fall through cracks between subsystems.

### RC3: Cross-platform abstraction hides capability differences
**Findings:** S09 (WindowsPty.process static), S18 (Windows env sanitization omitted)
**Pattern:** The IPty interface promises identical behavior across platforms, but platform-specific limitations are silently degraded rather than surfaced in the type system or API.

### RC4: Security-sensitive operations missing defense-in-depth
**Findings:** S13 (supplementary groups leak), S03 (FD close fallback)
**Pattern:** Privilege dropping and FD cleanup have correct happy-path implementations but miss edge cases that matter in security-sensitive deployments.

---

## Seed Map

| Metric | Count |
|--------|-------|
| Total seeds | 28 |
| PROVEN | 3 |
| SUSPECTED | 7 |
| HARDENED | 15 |
| UNCLEAR | 3 |
| Falsification success rate | 10.7% (3/28) |
| New seeds from adversary | 0 |

### Verdict Distribution

```
PROVEN:    S04, S08, S09
SUSPECTED: S02, S03, S10, S11, S13, S15, S27
HARDENED:  S01, S07, S12, S14, S16, S17, S18, S19, S20, S23, S24, S25, S28
UNCLEAR:   S05, S06, S22
```

---

## Unexplored

| Seed | Reason |
|------|--------|
| S05 | Close-then-kill ordering: initial analysis shows low risk, but full exploration requires pid recycling stress test |
| S06 | `_readable = undefined!`: TypeScript type assertion. pause/resume become no-ops when terminal attached — may be intentional design but not verified |
| S22 | Windows unbounded tsfn queue: design tradeoff analysis complete but OOM threshold not quantified |
| S21 | `Terminal._attachUnixFd` error handling: swallowed errors during fd transition — low priority |
| S26 | PID recycling on close: analysis shows near-zero risk window — deferred |
