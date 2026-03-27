# AFM Radiate v3.2 Report: zigpty

**Target**: zigpty — Zig + TypeScript PTY library for Node.js (N-API bindings)
**Scope**: 11 .zig files (~2120 lines), 13 .ts files (~1568 lines), ~3700 lines total
**Date**: 2026-03-26
**Protocol**: AFM Radiate v3.2 — Empty prompt blind test

---

## Phase 1: Seed Sweep

After reading all source files, the following seeds were identified:

| # | Seed | Location | Initial Signal |
|---|------|----------|----------------|
| S01 | `openPty` returns slave fd to JS but never closes it | `zig/lib.zig:160-183`, `zig/pty_unix.zig:154-188` | FD leak — slave fd handed to JS, no close path |
| S02 | `napi_create_external` with no finalize callback for WinConPtyContext | `zig/win/napi.zig:297` | Use-after-free if JS handle GC'd while threads running |
| S03 | `setuid`/`setgid` ordering: gid set before uid | `zig/lib.zig:133-138` | Correct order (gid first), but no supplementary groups call |
| S04 | macOS `P_COMM_OFFSET = 243` hardcoded for kinfo_proc | `zig/pty_darwin.zig:42` | Assumes struct layout is stable across macOS versions |
| S05 | `closeExcessFds` on macOS brute-force closes up to sysconf limit | `zig/pty_darwin.zig:67-77` | Closes FDs that may belong to other threads; perf issue |
| S06 | `closeHandle` does not check for INVALID_HANDLE before calling CloseHandle | `zig/pty_windows.zig:415-417` | Double-close on INVALID_HANDLE_VALUE |
| S07 | `closePty` can double-close `conin` (already closed by `closeConin`) | `zig/pty_windows.zig:404-411`, `zig/win/napi.zig:91,100` | conin closed in exit monitor, then closed again in closePty |
| S08 | Unix fork NAPI: arena allocator used for argv/envp passed to child process | `zig/pty_unix.zig:49-76`, `zig/lib.zig:140-143` | Arena freed after fork returns in parent, but child uses execvp with same pointers |
| S09 | Windows write/resize/kill/close access ctx after exit thread may have freed it | `zig/win/napi.zig:304-400` | JS can call write/resize/kill/close on handle after context destroyed |
| S10 | Unix `exitMonitorThread` never unrefs the tsfn (only releases) | `zig/pty_unix.zig:16-32` | Node.js process may hang waiting for tsfn ref |
| S11 | `waitFor` in BasePty accesses `terminal._dataListeners` directly | `src/pty/_base.ts:88,116` | Reaches into private state, bypasses encapsulation |
| S12 | Windows `WindowsPty.close()` calls `kill` but never calls `native.close()` | `src/pty/windows.ts:104-115` | Resources leak if process doesn't exit |
| S13 | `WriteQueue` silently drops data on write error (non-EAGAIN) | `src/pty/_writeQueue.ts:48-50` | No error propagation to caller |
| S14 | `Terminal.close()` calls `_onExit` with exitCode=0 unconditionally | `src/terminal.ts:144` | Misleading exit code if process crashed |
| S15 | `UnixPty` constructor: when terminal attached, `_readable` set to `undefined!` | `src/pty/unix.ts:61` | Non-null assertion on a value that is `undefined` |
| S16 | Flow control only filters exact single-char matches, not embedded chars | `src/pty/unix.ts:69-71` | XOFF/XON within larger data chunks pass through unfiltered |
| S17 | `decodeWaitStatus` treats stopped process (0x7f) as exit_code=-1 | `zig/lib.zig:214-223` | Stopped (WUNTRACED) returns misleading -1 instead of signal info |
| S18 | `openPtyUnix` does not apply termios settings | `zig/lib.zig:160-183` | Unlike forkPty which configures terminal, openPty leaves defaults |
| S19 | macOS `execChild` mutates global `environ` pointer | `zig/pty_darwin.zig:10-15` | Not thread-safe if multiple forks happen concurrently |
| S20 | `buildCmdLine` Windows command injection via special characters | `zig/pty_windows.zig:443-498` | Quoting logic may not cover all edge cases (e.g. `%VAR%` expansion) |
| S21 | `resizeImpl` reads argv[3]/argv[4] without argc check for >=4/>=5 first | `zig/pty_unix.zig:206-210` | argc check uses `>= 4` and `>= 5` which is correct, but no check for minimum 3 args |
| S22 | `getProcessName` on Linux reads `/proc/{pgrp}/cmdline` — pgrp is foreground group, not necessarily a valid PID | `zig/pty_linux.zig:22-37` | Process group ID used as path component without validation |
| S23 | `pid` returned via `napi_create_int32` — Windows DWORD pid can exceed i32 range | `zig/win/napi.zig:293` | PID > 2^31 truncated/wrapped on Windows |

---

## Phase 2: Full Interrogation

### S01: `openPty` slave fd leak

**REVERSE**: "The caller will close the slave fd." — If the caller (JS side) never closes it, the fd leaks permanently. The `open()` function in `index.ts` returns `{master, slave, pty}` as a plain object. There is no close/cleanup API provided. The `Terminal` class stores `result.slave` as `this.stdout` and closes it in `close()`, but the raw `open()` API has no lifecycle management.

**FORWARD**: If unclosed slave fds accumulate, the process will hit fd limits. Each `open()` call leaks one fd if the caller doesn't manually `fs.closeSync(slave)`.

**RADIATE**: Searched for all fd lifecycle paths:
- `forkPty` — slave fd is consumed by the child process (no leak)
- `openPty` via `Terminal` constructor — slave stored in `this.stdout`, closed in `Terminal.close()` (OK)
- `openPty` via raw `open()` export — **no close path provided**

**Verdict**: **VULNERABLE**

**WHY**: The developer designed `open()` as a low-level API assuming callers manage their own fds. But the exported interface (`IOpenResult`) provides no `close()` method and no documentation about cleanup responsibility. The `Terminal` class handles it correctly, but the raw `open()` export is a foot-gun.

---

### S02: No finalize callback on `napi_create_external` for WinConPtyContext

**REVERSE**: "The exit monitor thread always runs to completion and frees the context." — If the JS handle is garbage collected before the exit monitor thread finishes, or if the process never exits (zombie), the napi_external has no safety net.

**FORWARD**: The context is freed by `winExitMonitorThread` (line 114 of `win/napi.zig`). If that thread completes normally, no leak. But the external has `null` finalize_cb (line 297), meaning if the JS handle is collected and the thread never finishes, the context leaks.

**RADIATE**: Checked all `napi_create_external` calls — only one instance (Windows spawn). The Unix path uses `alloc.create(ExitContext)` freed in the defer of `exitMonitorThread`, which is safer because `waitpid` always eventually returns. On Windows, `WaitForSingleObject(INFINITE)` truly blocks forever if the process zombifies.

**Verdict**: **VULNERABLE**

**WHY**: The developer assumed the exit monitor thread always runs to completion. On Windows, a zombie process (rare but possible with debugger-attached processes) means the thread blocks forever and the context is never freed. A finalize callback would be the safety net.

---

### S03: `setuid`/`setgid` without supplementary groups

**REVERSE**: "Setting uid/gid is sufficient for privilege drop." — If the process needs to drop privileges properly, `setgroups()` or `initgroups()` should be called first to clear supplementary groups. Without it, the forked process retains the parent's supplementary groups.

**FORWARD**: If the parent process runs as root and has supplementary group memberships (docker, sudo, etc.), the child process inherits all of them even after setuid/setgid.

**RADIATE**: No other privilege-related calls found. `node-pty` (the reference implementation mentioned in termios comments) also does not call `initgroups`/`setgroups`, so this is an inherited design choice from the ecosystem.

**Verdict**: **VULNERABLE**

**WHY (multi-level)**:
1. Immediate: Developer followed the node-pty pattern which also omits supplementary group clearing.
2. Underlying: PTY libraries are typically not used in privilege-escalation contexts, so this is low-priority. But if someone does use uid/gid options to drop from root, the incomplete privilege drop is a real issue.
3. Root cause: The uid/gid feature exists for compatibility with node-pty's API, not because privilege management is a core concern.

---

### S04: Hardcoded `P_COMM_OFFSET = 243` for macOS kinfo_proc

**REVERSE**: "The kinfo_proc struct layout is stable at offset 243 across all macOS versions and architectures." — Apple does not guarantee struct layout stability across OS versions. The comment says "both arm64 and x86_64" but this is an empirical observation, not a documented API contract.

**FORWARD**: If Apple changes the struct layout in a future macOS version, `getProcessName` will return garbage or wrong process names silently (the bounds check at line 44 prevents a crash, but returns wrong data).

**RADIATE**: The `info_buf` size is 720 bytes (line 35) while "kinfo_proc is large (~648 bytes on arm64 macOS)" per the comment. The 720-byte buffer has safety margin, but the offset itself is the fragile assumption. Checked Linux path — uses `/proc/{pgrp}/cmdline` which is a stable kernel interface. No similar hardcoded offset issue there.

**Verdict**: **VULNERABLE**

**WHY**: Using raw `sysctl` with hardcoded struct offsets instead of including the proper C header and accessing `kp_proc.p_comm` is a deliberate choice to avoid C headers in a pure-Zig project. The tradeoff is ABI fragility. This works until Apple changes the struct layout.

---

### S05: macOS `closeExcessFds` brute-force close

**REVERSE**: "Closing all fds >= 3 in the child is safe." — After `forkpty()`, the child has a copy of all parent fds. Closing them is standard practice. But the brute-force approach (iterating to sysconf limit) can be slow if `_SC_OPEN_MAX` is high (some systems return 1048576).

**FORWARD**: Performance issue on systems with high fd limits. Not a correctness bug since this runs in the child process after fork, before exec.

**RADIATE**: Linux path tries `close_range(3, UINT_MAX, 0)` first (O(1) kernel call), falls back to `/proc/self/fd` enumeration, then brute-force to 256. macOS has no `close_range` equivalent. The asymmetry means macOS is slower for high fd limits.

**Verdict**: **HOLDS** (challenged: the performance concern is real but bounded — this runs once per spawn in the child process, and macOS `sysconf(_SC_OPEN_MAX)` typically returns 12544, not millions)

---

### S06: `closeHandle` does not guard against INVALID_HANDLE_VALUE

**REVERSE**: "CloseHandle on INVALID_HANDLE_VALUE is harmless." — Actually, calling `CloseHandle(INVALID_HANDLE_VALUE)` on Windows returns FALSE and sets last error, but does NOT crash. However, `CloseHandle(NULL)` on some Windows versions can raise an exception if invalid handle debugging is enabled.

**FORWARD**: The code sets handles to `INVALID_HANDLE` after closing in some paths (`closeConin`, `closePty`) but not all. `closePty` calls `closeHandle(result.conin)` which may already be INVALID_HANDLE from `closeConin`.

**RADIATE**: Found multiple close paths:
- `closeConin` — guards with `!= INVALID_HANDLE`, sets to INVALID_HANDLE (safe)
- `closePty` — calls `closeHandle` on all three handles **without** INVALID_HANDLE guard
- `ConPtySetup.deinit` — calls `closeHandle` directly, then sets to INVALID_HANDLE
- `closeHandle` itself — no guard

The pattern: some callers guard, some don't. `closePty` is called in `winExitMonitorThread` after `closeConin` has already closed conin, so `closeHandle(result.conin)` is called on INVALID_HANDLE.

**Verdict**: **VULNERABLE**

**WHY**: The developer was inconsistent — `closeConin` correctly checks for INVALID_HANDLE before closing, but `closePty` does not. This results in a double-close of `conin` (once in `closeConin`, once in `closePty`). While Windows tolerates CloseHandle(INVALID_HANDLE_VALUE) silently, this is still a latent bug that could manifest if handle values change or in debug builds.

---

### S07: Double-close of `conin` handle (radiation from S06)

**REVERSE**: "The exit monitor thread's close sequence is safe: closeConin → closePseudoConsole → closePty." — But `closePty` closes conin *again* (already set to INVALID_HANDLE by `closeConin`, so `closeHandle(INVALID_HANDLE)` is called).

**FORWARD**: In `winExitMonitorThread` (win/napi.zig lines 91-100):
1. `win.closeConin(&ctx.spawn_result)` → closes conin, sets to INVALID_HANDLE
2. `win.closePseudoConsole(...)` → closes hpc
3. read_thread join
4. `win.closePty(&ctx.spawn_result)` → calls `closeHandle(result.conin)` on INVALID_HANDLE, then `closeHandle(result.conout)`, then `closeHandle(result.process)`

Step 4 double-closes conin (now INVALID_HANDLE). This is a confirmed double-close, though mitigated by Windows tolerating CloseHandle on INVALID_HANDLE_VALUE.

**RADIATE**: Also found: if `close()` is called from JS (killImpl) while the exit monitor thread is running, a race exists. `close` calls `killProcess`, which triggers process exit, which causes the exit monitor thread to run cleanup. But there's no synchronization between JS-thread and exit-monitor-thread access to the spawn_result handles.

**Verdict**: **VULNERABLE** (confirmed double-close, and potential race condition with JS-thread close)

**WHY**: The developer implemented `closeConin` with a proper guard but didn't notice that `closePty` (called later) re-closes the same handle. The close sequence works in practice because Windows silently ignores CloseHandle(INVALID_HANDLE_VALUE), but the logic is wrong.

---

### S08: Arena-allocated argv/envp used across fork boundary

**REVERSE**: "Arena memory allocated before fork is valid in both parent and child." — After `fork()`, the child gets a copy-on-write copy of the parent's address space, including the arena memory. The parent's `defer arena.deinit()` runs in the parent process only. The child calls `execvp`/`execvpe` which replaces the address space entirely.

**FORWARD**: The sequence is: allocate in arena → forkpty → child uses exec_argv/exec_envp for execvp → parent defers arena.deinit(). This is **correct** because:
1. Child gets its own copy of the memory (COW)
2. Child calls exec immediately, which replaces the address space
3. Parent frees the arena after fork returns

**Verdict**: **HOLDS** (challenged: fork() semantics guarantee COW memory is independent. The arena pointers remain valid in the child until exec replaces the address space. This is the standard Unix fork+exec pattern.)

---

### S09: Windows use-after-free: JS calls write/resize/kill/close after context freed

**REVERSE**: "JS code will not call write/resize/kill/close after the exit callback fires." — The exit callback fires asynchronously via tsfn. There is a window where JS code could call write() on a handle whose context has been freed by the exit monitor thread.

**FORWARD**: The timeline:
1. Process exits
2. Exit monitor thread: closeConin → closePseudoConsole → join read thread → closePty → fire exit tsfn → **alloc.destroy(ctx)** (line 114)
3. JS thread receives exit callback
4. But JS code might call `write(handle)` between steps 2 and 3

The `napi_external` wrapping the context has no finalize callback and no validity check. After `alloc.destroy(ctx)` on line 114, any subsequent JS call to `write/resize/kill/close` with this handle dereferences freed memory.

**RADIATE**: Checked all Windows NAPI functions that dereference the external:
- `writeImpl` (line 316): `@ptrCast(@alignCast(ctx_ptr))` — no validity check
- `resizeImpl` (line 345): same pattern
- `killImpl` (line 375): same pattern
- `closeImpl` (line 393): same pattern

All four functions dereference the external pointer without checking if the context is still alive.

**Verdict**: **VULNERABLE**

**WHY (multi-level)**:
1. Immediate: No synchronization between exit monitor thread freeing context and JS thread accessing it.
2. Design gap: The `napi_external` is created without a finalize callback, so there's no GC-based cleanup and no way to invalidate the handle.
3. The `_closed` flag in `WindowsPty` (TypeScript side) provides partial protection — `write()` checks `this._closed` before calling native. But `_handleExit` sets `_closed = true` in the exit listener, which fires AFTER the context is already freed. There's a race between context destruction and the JS-side closed flag being set.

---

### S10: Unix exitMonitorThread never unrefs the tsfn

**REVERSE**: "Releasing the tsfn is sufficient to allow Node.js to exit." — `napi_release_threadsafe_function` decrements the thread count. When it reaches 0, the tsfn is cleaned up. But if the tsfn still has a **reference** (ref count > 0), it prevents Node.js from exiting even after release.

**FORWARD**: On Unix, the tsfn created in `forkImpl` (line 112-124) is **not** followed by `napi_unref_threadsafe_function`. Compare with Windows (line 225, 273) which calls `napi_unref_threadsafe_function` on both data and exit tsfns. This means the Unix exit tsfn keeps Node.js alive even if the user has no other active handles.

**RADIATE**: Windows path (win/napi.zig):
- Line 225: `napi_unref_threadsafe_function(env, ctx.data_tsfn)` — YES
- Line 273: `napi_unref_threadsafe_function(env, ctx.exit_tsfn)` — YES

Unix path (pty_unix.zig):
- After creating tsfn at line 112-124: **NO unref call**

This asymmetry means a Unix PTY spawn keeps Node.js alive until the child process exits, even if the user has called `close()` and removed all listeners.

**Verdict**: **VULNERABLE**

**WHY**: The developer correctly unreffed the tsfn on Windows but missed doing the same on Unix. This is a copy-paste oversight — the Windows code was likely written later with the lesson learned, but the fix was never backported to Unix.

---

### S11: `waitFor` accesses `terminal._dataListeners` directly

**REVERSE**: "The `_dataListeners` array on Terminal is a stable internal interface." — It's prefixed with `_` (internal convention) but accessed from `BasePty.waitFor()` which is in a different class. If Terminal's internal data listener mechanism changes, waitFor breaks.

**FORWARD**: This is a code quality/encapsulation issue, not a runtime bug. The `_dataListeners` array exists and is used consistently.

**Verdict**: **HOLDS** (design smell, not a vulnerability — both classes are internal to the same library)

---

### S12: Windows `WindowsPty.close()` only kills, never calls `native.close()`

**REVERSE**: "Killing the process is sufficient for cleanup." — The comment in the code (lines 108-111) explicitly explains why: calling `native.close()` from the JS thread would call `ClosePseudoConsole`, which blocks until the output pipe is drained, but draining requires the tsfn callback on the JS thread — deadlock.

**FORWARD**: The design relies on the exit monitor thread for cleanup. Kill → process exits → exit monitor runs → closeConin → closePseudoConsole → closePty → free context. If the kill doesn't cause the process to exit (e.g., a process that catches/ignores TerminateProcess — which is impossible on Windows, TerminateProcess always succeeds), cleanup happens normally.

**Verdict**: **HOLDS** (challenged: the comment correctly identifies the deadlock risk and the design is intentional. TerminateProcess on Windows is not catchable, so the process will exit.)

---

### S13: WriteQueue silently drops data on write error

**REVERSE**: "Write errors should be propagated to the caller." — The `_process()` method (line 48-50) clears the entire queue on non-EAGAIN errors without notifying anyone. The `enqueue()` call has already returned the buffer length as "bytes written."

**FORWARD**: If the fd becomes invalid (e.g., PTY closed from the other side), all queued data is silently dropped. The user gets no error callback or event.

**RADIATE**: The `WriteQueue` is used in:
- `UnixPty` (unix.ts:54): `new WriteQueue(this._fd)` — no drain or error callback
- `Terminal._setupUnixReader` (terminal.ts:205): `new WriteQueue(fd, () => this._onDrain?.(this))` — drain callback but no error callback

Neither consumer gets notified of write failures.

**Verdict**: **VULNERABLE**

**WHY**: The developer optimized for the common case (writes succeed) and treated errors as "PTY is closing anyway." But there's no error event in the API, so callers cannot distinguish between "data was written" and "data was silently dropped."

---

### S14: `Terminal.close()` fires `_onExit` with exitCode=0

**REVERSE**: "Exit code 0 is appropriate when the terminal is explicitly closed." — When `close()` is called by the user, they intentionally closed it. A 0 exit code is reasonable. But the docstring says "exitCode is PTY lifecycle status (0=EOF, 1=error)" which conflates user-initiated close with normal EOF.

**FORWARD**: If the terminal closes due to an error in the read stream, `_onExit` is not called (the readable's error handler is a no-op at terminal.ts:204). Only explicit `close()` fires `_onExit(this, 0, null)`. This means errors are swallowed.

**Verdict**: **VULNERABLE**

**WHY**: The developer defined `exit` callback semantics (0=EOF, 1=error) but never implements the error=1 case. Read stream errors are silently eaten by `this._readable.on("error", () => {})`. The `_onExit` callback only fires on explicit close, always with code 0.

---

### S15: `_readable` set to `undefined!` when terminal attached

**REVERSE**: "When a terminal is attached, _readable is never accessed." — The exit handler at line 43-44 does `this._readable?.destroy()` using optional chaining. The `pause()` and `resume()` methods use `this._readable?.pause()`. So `undefined` is safe due to optional chaining.

**FORWARD**: The `undefined!` non-null assertion is misleading but harmless — the variable is immediately used with `?.` optional chaining everywhere. TypeScript's type system sees it as defined, but runtime accesses all use `?.`.

**Verdict**: **HOLDS** (ugly code, but safe at runtime due to consistent use of optional chaining on all access paths)

---

### S16: Flow control only filters exact single-char matches

**REVERSE**: "XOFF/XON always arrive as individual characters." — In a PTY, terminal data arrives in chunks. An XOFF byte (0x13) could be embedded in a larger string: `"text\x13more"`. The filter at unix.ts:70 checks `data === this._flowControlPause` which only matches if the entire chunk is exactly one XOFF character.

**FORWARD**: Embedded XOFF/XON characters within larger data chunks pass through unfiltered to listeners. If the application relies on flow control interception, it will miss cases where XOFF/XON arrives in the same read() as other data.

**RADIATE**: Checked node-pty source (this is a known limitation there too). The `handleFlowControl` feature is documented as intercepting flow control characters, but the implementation only handles the case where they arrive as standalone chunks.

**Verdict**: **VULNERABLE**

**WHY**: The developer implemented a simplified flow control filter that works for the common case (flow control characters arriving as separate PTY reads) but fails for the edge case of batched reads. This is inherited from node-pty's design pattern.

---

### S17: `decodeWaitStatus` mishandles stopped processes

**REVERSE**: "Stopped processes (WIFSTOPPED) won't occur because waitpid is called with options=0." — Correct. `waitpid(pid, &status, 0)` without WUNTRACED means stopped processes are NOT reported. The `termsig == 0x7f` case in `decodeWaitStatus` would only occur if someone changed the waitpid flags to include WUNTRACED.

**FORWARD**: The function is technically correct for the current usage (options=0). The 0x7f branch is dead code that handles a case that cannot occur.

**Verdict**: **HOLDS** (the code is correct for the calling context; the 0x7f handling is defensive dead code)

---

### S18: `openPty` does not apply termios settings

**REVERSE**: "The terminal opened by openPty will have sane defaults from the OS." — The OS-provided defaults may differ from the configured termios in `forkPty`. The `Terminal` class uses `openPty` for standalone mode and gets raw OS defaults, while `spawn()` uses `forkPty` which applies configured termios.

**FORWARD**: A `Terminal` created standalone will have different terminal behavior (line discipline, echo, etc.) than one created via `spawn()`. The user may experience inconsistent behavior depending on creation path.

**Verdict**: **UNCLEAR** (the termios configuration mainly matters for the slave side of the PTY; the master side behavior is controlled by the kernel. Whether this causes observable differences depends on the use case.)

---

### S19: macOS `execChild` mutates global `environ` pointer

**REVERSE**: "Multiple forks won't happen simultaneously." — `forkPtyUnix` blocks signals around the fork call (line 113-115), but after fork returns in the parent, the signal mask is restored. If two Node.js operations trigger `forkPty` concurrently from different threads (e.g., worker threads), the `environ` mutation in the child is safe (each child has its own address space after fork). But the real concern is: could the parent's `environ` be corrupted?

**FORWARD**: The `environ.* = envp` write happens in the child process (pid == 0 branch). After fork, the child has a copy-on-write address space. Modifying `environ` in the child does NOT affect the parent. This is safe.

**Verdict**: **HOLDS** (challenged: the environ mutation is in the child process only, which has an independent copy-on-write address space. No cross-process corruption possible.)

---

### S20: Windows command line injection via `buildCmdLine`

**REVERSE**: "The quoting logic handles all special characters." — The `appendQuoted` function handles spaces, tabs, quotes, and backslashes per the Windows command-line quoting convention (Microsoft's rules for `CommandLineToArgvW`). However, `cmd.exe` has its own metacharacters (`%`, `^`, `|`, `&`, `<`, `>`) that are NOT handled.

**FORWARD**: If the spawned process is `cmd.exe` and args contain `%PATH%`, the environment variable is expanded. If args contain `| malicious_command`, pipe injection could occur. However, `CreateProcessW` receives the command line directly — it does NOT invoke `cmd.exe` for interpretation unless the target IS `cmd.exe`.

**RADIATE**: The command line goes to `CreateProcessW` which passes it directly to the process. The quoting convention is for `CommandLineToArgvW` parsing. `cmd.exe` builtins use different parsing. This is a known design constraint of Windows process creation, not specific to this library.

**Verdict**: **HOLDS** (challenged: the quoting is correct per `CommandLineToArgvW` spec. The `cmd.exe` metacharacter issue is a Windows platform limitation, not a library bug. Any caller spawning `cmd.exe /c <untrusted>` already has this problem regardless of quoting.)

---

### S21: `resizeImpl` minimum argument check

**REVERSE**: "The caller always provides at least 3 arguments (fd, cols, rows)." — The NAPI layer initializes argv as a [5] array and sets `argc = 5`. `napi_get_cb_info` sets argc to the actual argument count. If fewer than 3 args are passed, `argv[0..2]` may contain uninitialized napi_value handles.

**FORWARD**: If JS calls `resize()` with fewer than 3 args, `napi_get_value_int32` on uninitialized napi_value would likely return an error status, caught by `napi.check()`, throwing a JS error. Not a crash, but not a clean error message either.

**Verdict**: **HOLDS** (The NAPI check functions provide safety — napi_get_value_int32 on invalid values returns an error status. Not ideal UX but not a vulnerability.)

---

### S22: Linux `getProcessName` with process group ID as path

**REVERSE**: "The foreground process group ID is a valid PID for /proc lookup." — Process group ID equals the PID of the process group leader. `/proc/{pgid}/cmdline` will return the cmdline of the group leader, which is typically the foreground process. This is the standard approach.

**FORWARD**: If the group leader has exited but the group still has members, `/proc/{pgid}/cmdline` will fail to open (ENOENT), and the function returns null. This is correct error handling.

**Verdict**: **HOLDS**

---

### S23: Windows PID truncation via `napi_create_int32`

**REVERSE**: "Windows PIDs fit in i32." — Windows PIDs are DWORD (u32). While most PIDs are small, the theoretical maximum is 4,294,967,295. PIDs above 2,147,483,647 (i32 max) would be truncated/wrapped by `napi_create_int32`.

**FORWARD**: In practice, Windows PIDs rarely exceed a few thousand. The maximum PID on Windows 10/11 is configurable but defaults to much less than 2^31. However, the `@intCast(proc.pid)` at line 293 would panic in Zig debug mode if the value exceeds i32 range.

**RADIATE**: Same pattern on Unix side (pty_unix.zig:143-144): `napi.createI32(env, @intCast(result.fd))` and `napi.createI32(env, result.pid)`. Unix PIDs are limited to `pid_t` (i32), so no truncation issue there. But Unix fd values can theoretically exceed i32 on 64-bit systems (unlikely in practice).

**Verdict**: **UNCLEAR** (theoretical issue — Windows PIDs fitting in i32 is a practical invariant but not guaranteed by the OS. The Zig `@intCast` would trap in safe/debug builds if violated.)

---

## Board Summary

### Hot Leads (confirmed VULNERABLE)

| Seed | Finding | Severity | Impact |
|------|---------|----------|--------|
| S09 | **Use-after-free**: Windows JS calls dereference freed context after exit monitor destroys it | **HIGH** | Memory corruption, potential crash |
| S10 | **Missing tsfn unref on Unix**: Node.js process hangs after PTY close | **MEDIUM** | Process hang — cannot exit cleanly |
| S07 | **Double-close of conin handle**: closePty re-closes already-closed INVALID_HANDLE | **MEDIUM** | Latent handle corruption (silent on current Windows) |
| S02 | **No finalize callback on napi_external**: context leak if exit thread never completes | **MEDIUM** | Memory leak, resource leak on zombie process |
| S01 | **Slave fd leak from raw open() API**: no close path for returned slave fd | **LOW-MEDIUM** | FD exhaustion under repeated open() calls |
| S13 | **WriteQueue silent data loss**: write errors drop data without notification | **LOW-MEDIUM** | Data loss without caller awareness |
| S14 | **Terminal.close() always reports exitCode=0**: errors never reported | **LOW** | Misleading exit status |
| S16 | **Flow control bypass**: embedded XOFF/XON in chunks not intercepted | **LOW** | Flow control unreliable for batched reads |
| S03 | **Incomplete privilege drop**: no supplementary group clearing | **LOW** | Privilege escalation in root-to-user scenarios |
| S04 | **Hardcoded kinfo_proc offset**: macOS struct layout may change | **LOW** | Silent wrong process name on future macOS |
| S06 | **closeHandle no INVALID_HANDLE guard**: inconsistent with closeConin | **LOW** | Code defect, benign on current Windows |

### Warm Leads (HOLDS after challenge)

| Seed | Finding | Why it holds |
|------|---------|-------------|
| S05 | macOS closeExcessFds brute-force | Runs in child process, bounded by sysconf |
| S08 | Arena memory across fork | COW semantics make this safe |
| S12 | WindowsPty.close() only kills | Intentional deadlock avoidance, TerminateProcess always works |
| S15 | `_readable = undefined!` | Safe due to consistent `?.` optional chaining |
| S17 | decodeWaitStatus 0x7f branch | Dead code — waitpid flags exclude stopped |
| S19 | macOS environ mutation | Happens in child process only (post-fork) |
| S20 | Windows command injection | Quoting correct per CommandLineToArgvW spec |
| S21 | resizeImpl argument count | NAPI check functions catch invalid values |
| S22 | Linux pgrp as /proc path | Standard approach, error handling correct |

### Cold Leads (UNCLEAR)

| Seed | Finding | Why unclear |
|------|---------|-------------|
| S18 | openPty no termios config | Depends on use case; may be intentional |
| S23 | Windows PID > i32 | Theoretical; practical PIDs always fit |

---

## Key Patterns Observed

### Pattern 1: Unix/Windows Asymmetry in Lifecycle Management
The Windows code path has more sophisticated lifecycle management (deferred calls, atomic hpc_closed flag, explicit thread coordination) while the Unix path is simpler but has gaps (missing tsfn unref, no arena concern because fork semantics save it). The developer invested more effort in Windows correctness but still has the S09 use-after-free and S07 double-close.

### Pattern 2: Close Sequence Inconsistency
Multiple close-related bugs stem from a single pattern: some close paths guard against INVALID_HANDLE while others don't. The developer implemented cleanup correctly in individual pieces (closeConin checks, closePty does full cleanup) but the composition of these pieces creates double-closes.

### Pattern 3: Missing Error Propagation
WriteQueue (S13) and Terminal (S14) both silently swallow errors. Read stream errors are caught by empty error handlers. Write errors clear the queue. Exit callbacks report 0. This is a consistent pattern of "errors are not worth reporting because the PTY is closing anyway" — but the user has no way to distinguish clean shutdown from error shutdown.

### Pattern 4: NAPI External Without Safety Net
The Windows path creates a `napi_external` wrapping a heap-allocated context but provides no finalize callback and no validity flag. This is the root cause of S09 (use-after-free) and S02 (leak on zombie). A single fix — adding a finalize callback and an `alive` atomic flag — would resolve both.

---

## Radiation Map

```
S06 (closeHandle no guard)
  └──► S07 (double-close conin) — same pattern, composites into a real bug
        └──► S09 (use-after-free) — same root: no synchronization between cleanup thread and JS access
              └──► S02 (no finalize callback) — same root: NAPI external has no safety mechanism

S10 (missing tsfn unref on Unix)
  └──► Compared with Windows path which correctly unrefs — ASYMMETRY pattern

S13 (WriteQueue silent drop)
  └──► S14 (Terminal.close exitCode=0) — same pattern: errors silently swallowed

S16 (flow control bypass)
  └──► Standalone finding, inherited from node-pty design

S01 (slave fd leak from open())
  └──► Standalone finding, API design gap

S03 (no supplementary groups)
  └──► S04 (hardcoded offset) — different issue but same root: following node-pty patterns without questioning assumptions
```

**Central Theme**: The codebase is well-structured and handles many edge cases correctly (signal blocking around fork, COW-safe arena usage, Windows deadlock avoidance). The vulnerabilities cluster around **lifecycle synchronization** (S02/S06/S07/S09) and **silent error swallowing** (S13/S14). The highest-impact finding (S09: use-after-free) is a genuine memory safety issue in the Windows NAPI bridge.
