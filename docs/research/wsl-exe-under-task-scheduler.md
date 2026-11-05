# Do exit codes and output survive `wsl.exe` launched by Task Scheduler?

Research for [das-scheduler#6](https://github.com/4lch4/das-scheduler/issues/6). Date: 2026-11-05.

## Answer

Yes, with three conditions.

1. The wrapper must give `wsl.exe` real stdout and stderr handles. Task Scheduler does not
   capture output, and an interactive console action gets a hidden console, so anything
   written to the default handles is lost [S18][L3]. Point `cmd.Stdout` and `cmd.Stderr` at
   the Run log file.
2. Set `WSL_UTF8=1` in `wsl.exe`'s environment. The Command's own output is raw bytes, but
   `wsl.exe`'s own messages are UTF-16LE by default; with `WSL_UTF8=1` they are UTF-8
   [S2][S5][S6].
3. Change the WSL invocation from `--` to `-e` (or `--exec`). `--` passes the whole command
   line through the distro's default shell (`$SHELL -c ...`) before `bash -lc` sees it, so
   `$?`, `$VAR` and command substitutions are expanded twice. `-e` execs `bash` directly
   [S2][L7].

With that, the Command's exit code comes back exactly for normal Linux exits (0-255),
stdout and stderr arrive intact as bytes with no CRLF translation, and Task Scheduler's
"Last Run Result" mirrors the wrapper's exit code [S3][S18][L1][L2]. Two codes need special
handling: `wsl.exe` uses `-1` (`0xFFFFFFFF`) as its generic failure code, and a direct child
killed by a signal returns the raw signal number (for example 15 for SIGTERM), not 128+n
[S4][L1].

The distro does not need to be running: `wsl.exe` starts it on demand [S7][S8][L4]. Budget
for a cold start (2.8 s in our test with the WSL VM already up; 23-32 s after a full WSL
shutdown has been reported) and set the timeout accordingly. A locked session still has a
live interactive session, so the task should still start: "run only when user is logged on"
uses an interactive token and locking is not logging off [S15][S16][S17]. Console commands
work; Windows interop that needs the desktop (clipboard, GUI) does not [S14].

Recommendation: keep the WSL Job design, make the three changes above, and also pass
`--cd ~`, set `cmd.Stdin = nil`, and record the exit status in SQLite as a `uint32` plus a
kind (`exited`, `signalled`, `wsl-failed`, `timed-out`). Do not switch to "run whether user
is logged on or not": WSL does not work from session 0 [S12].

## What Task Scheduler actually launches

The ticket says "with no console", but that is not quite what happens.

- "Run only when user is logged on" registers the task with the `Interactive` logon type:
  "User must already be logged on. The task will be run only in an existing interactive
  session." [S15]
- On this machine, a task registered that way ran in session 1 and its console-subsystem
  action reported `GetConsoleWindow() != 0`; stdin, stdout and stderr were console handles
  (`GetFileType` = `FILE_TYPE_CHAR`). The console window was hidden, not absent. [L3]
- `wsl.exe` works with or without a console. When its std handles are not valid it simply
  does not relay output; when a console exists it writes there. The client tolerates a
  missing `CONOUT$`/`CONIN$`. [S2][S3][S24][L1]
- Task Scheduler itself has no "capture output" feature. The Last Run Result is only the
  exit code of the last action, exposed as `IRegisteredTask::LastTaskResult`; Microsoft's
  own example calls it "the last exit code returned by a known task". [S18]

So the wrapper cannot inherit a usable stdout/stderr from Task Scheduler. It must create
the log file (or pipe) itself. That is also what makes the child's stdio non-tty, which is
usually what you want for a log.

## Exit codes

Normal Linux exit codes propagate byte-for-byte. The WSL client waits for the Linux process
and returns its status; the Linux init sends `WEXITSTATUS(Status)` for a normal exit and the
raw `waitpid` status when the process did not exit normally. [S3][S4]

Observed on WSL 3.0.3.0 across the local harnesses: [L1][L2]

| Case | `wsl.exe` exit |
| --- | --- |
| `exit 0` | 0 |
| `exit 17` | 17 |
| `exit 42` | 42 |
| `exit 255` | 255 |
| `exit 300` | 44 (bash truncates to 8 bits) |
| `kill -TERM $$` (bash itself dies) | 15 |
| `kill -KILL $$` (bash itself dies) | 9 |
| bad distro name | -1 (`0xFFFFFFFF`) |
| `-e cd /tmp` (exec fails) | 1, error on stderr |

Three traps follow from this:

1. **`-1` is the sentinel for a `wsl.exe` failure, not a Linux result.** The client uses
   exit code -1 as its generic failure code "so invokers of wsl.exe can distinguish between a
   Linux process failure and
   wsl.exe failure". [S2] A Linux command cannot produce it: Linux exit codes are 0-255.
   Store the code as `uint32` so `0xFFFFFFFF` survives; Go's `ExitCode()` returns `int` and
   turns `0xFFFFFFFF` into -1. [S22]
2. **A signal death is reported as the signal number.** If `bash -lc`'s direct child is
   killed by a signal, the raw wait status is sent, so SIGTERM looks like exit 15 and SIGKILL
   like exit 9. That is indistinguishable from a command that deliberately exits 15 or 9.
   A command killed *inside* bash normally makes bash exit 128+n instead, because bash
   reports a signalled child that way. [S4][S23] Only the direct child (`bash`) takes the
   raw-signal path.
3. **Killing `wsl.exe` is not the Command's exit code.** Go's `os.Process.Kill` calls
   `TerminateProcess(handle, 1)` on Windows, so a timeout-killed run looks like exit 1.
   Record timeouts from `ctx.Err()`, not from the code. [S21][S22]

Task Scheduler stores the value as a 32-bit result: our direct-action tests produced Last
Run Result 23, 15, 0 and 4294967295. [L2] That is a useful
cross-check, but SQLite should remain the source of truth, because the wrapper's own
failures and timeouts need a richer status than a single number.

## stdout and stderr

The Command's output is relayed as raw bytes. The client uses `WriteFile` on the relay
buffer, with no encoding conversion and no newline translation. [S26] Our tests received a
1,000,000-byte stdout stream intact, and the bytes `ff fe 00 41` arrived unchanged; a
UTF-8 `é` arrived as `c3 a9`. Child output ended in `0a`, not `0d 0a`. [L1]

`wsl.exe`'s own messages are a different stream and a different encoding:

- By default the CRT is in `_O_U16TEXT` mode, so `wsl.exe`'s own stdout/stderr are
  UTF-16LE. [S2][S5]
- `WSL_UTF8=1` switches the CRT to `_O_U8TEXT`, making those messages UTF-8. The variable
  was introduced in WSL 0.64.0 (2022-07-21) and is opt-in for compatibility. [S6]
- A missing distro produced a 241-byte UTF-8 message on **stdout** with `WSL_UTF8=1`, and a
  482-byte UTF-16LE message on **stdout** without it. The message is printed with
  `PrintMessage`, whose stream defaults to `stdout`. [S5][L1]
- An exec failure inside the distro (`-e cd /tmp`) produced its error on **stderr** and
  exit 1. [L1]

Consequences for the Go side:

- Capture both streams into the Run log. If you capture only stderr you lose "no such
  distro"; if you capture only stdout you lose exec failures.
- Set `WSL_UTF8=1` or the log will mix UTF-16LE diagnostics with UTF-8 command output and
  become unreadable.
- Write bytes, not strings. Treat the log as UTF-8 for display, with replacement for
  invalid sequences, because a Command can emit arbitrary binary.
- Do not assume ordering between stdout and stderr. They are separate relays. If order
  matters, write both to one file and accept approximate interleaving, or prefix each line
  by stream at the writer.

## Does the distro need to be running?

No. `wsl.exe -d <distro> -e ...` starts a stopped distro (and, for WSL 2, the utility VM)
on demand. The WSL FAQ describes the VM as starting and stopping automatically, and the
`instanceIdleTimeout` (default 15,000 ms) and `vmIdleTimeout` (default 60,000 ms) settings
exist precisely because WSL shuts down when idle. [S7][S8]

Measured on this machine with the WSL VM already running because Ubuntu was open: [L4]

| Case | Time |
| --- | --- |
| stopped distro (`AgentBox`), first command | 2.77 s |
| same distro, next command | 119 ms |
| running distro (`Ubuntu`) | 121 ms |
| 5 concurrent cold starts of the same stopped distro | 3.5 s each, all exit 0 |

A cold VM is slower. One report on WSL 2.7.13 measured 23-32 s after `wsl --shutdown` for
`wsl -d Ubuntu -e true`, versus about 0.13 s warm. [S9] WSL's own timeouts are
`kernelBootTimeout` = 30,000 ms and `distributionStartTimeout` = 60,000 ms, so a start that
takes longer than that fails with a `wsl.exe` error rather than hanging forever. [S7]
A `[boot] command` and systemd add to that time. [S7]

Practical budget: give a WSL Job at least the Command's own timeout plus 60-120 s of
startup allowance, or warm the distro with `wsl.exe -d <distro> -e true` before starting
the Command's clock. A 60-second job timeout applied to a cold VM will produce false
timeouts.

Concurrent starts are safe: five simultaneous `wsl.exe` invocations against one stopped
distro all started it and returned their own output; the service handled the overlap. [L4]

## Locked session

"Run only when user is logged on" needs an existing interactive session, not an unlocked
desktop. [S15] Locking the workstation keeps the logon session and its processes in place;
only logging off terminates them. Winlogon documents the locked state as keeping work in
place, and `WTS_CONNECTSTATE_CLASS` describes `WTSDisconnected` as "the user has chosen to
exit to the lock screen" while the WinStation remains active. [S16][S17]

So a WSL console job should still start and run while the screen is locked. What does not
work is Windows interop that needs the visible desktop: a 2021 report showed
`powershell.exe Get-Clipboard` failing while locked, while plain Linux commands such as
`echo` still ran. [S14] GUI apps (WSLg) and anything touching the clipboard or the active
window should be treated as unsupported for scheduled Jobs.

History: in 2019 there was a real bug where locking Windows killed WSL shells
(microsoft/WSL#4536). It was closed as resolved after an update. We did not directly test
a locked session; this section is inference from the documented session behaviour plus that
history. [S13]

One separate caveat: if the machine sleeps while locked, the task does not run until it
wakes. The spec's "run as soon as possible after a missed start" (`StartWhenAvailable`)
covers the wake-up case; waking the machine for every Job is a separate policy decision.

## Known `wsl.exe` bugs and failure modes

We found no current bug that swallows the exit code of a `wsl.exe -d ... -e ...` Command.
The failure modes worth designing around are these:

- **`--list` / `--status` output can be swallowed when redirected.** WSL 3.0.3.0 had a
  regression where `wsl -l` and `wsl --status` wrote nothing to a redirected stdout and
  still exited 0 (microsoft/WSL#41937). It was closed with the root cause unconfirmed, and
  it did not reproduce on our 3.0.3.0 install with a newer Windows build. [S11] Do not build
  distro discovery on parsing that output; if the list is empty with exit 0, treat it as
  "unknown", not "no distros".
- **A wedged WSL service can stop accepting new execs.** WSL 3.0.3.0 can leave existing
  sessions working while every new `wsl.exe -e` request hangs and eventually returns
  `Wsl/Service/WSAETIMEDOUT` (microsoft/WSL#42029). A wrapper without a timeout will hang
  the Run forever. [S10]
- **Session 0 is unsupported.** The Store release notes list "Launching Windows Subsystem
  for Linux from session zero does not currently work"; a task with "Run whether user is
  logged on or not" (S4U/password logon) runs outside the interactive session and gets
  `Access is denied`.
  [S12] This is why the user-level, logged-on design is the right one.
- **`-1` and raw signal numbers** are the two code-level traps described above. They are
  not bugs, but they are easy to misread.
- **Killing the client does not always kill the Command.** A plain `sleep 5; echo ...` died
  when we killed its `wsl.exe`, but a shell that trapped `HUP` and `TERM` survived and wrote
  its file after the client was gone. [L5] If a hard timeout matters, enforce it on the
  Linux side as well.

## What the Go side has to do

### Command line

Use `-e` and keep the script as one argv element (`-d` is the documented `--distribution`
option [S1]):

```go
cmd := exec.CommandContext(ctx,
    `C:\Windows\System32\wsl.exe`,
    "--cd", "~",
    "-d", job.Distro,
    "-e", "bash", "-lc", job.Command,
)
```

- `-e` avoids the extra `$SHELL -c` layer. Our test `-- bash -lc "false; echo $?"` printed
  0, because the outer shell expanded `$?` before bash ran; the `-e` form printed 1. [L7]
- `--cd ~` gives the Job the Linux home directory. Without it, `wsl.exe` translates the
  Windows working directory. A Task Scheduler action with no "Start in" defaulted to
  `C:\Windows\System32`, and the job started in `/mnt/c/WINDOWS/system32` in our test. [L6]
- Go's `exec.Command` quotes argv for `CommandLineToArgvW`, which is what `wsl.exe`'s `-e`
  parser uses. [S20][S2] If you ever build the command line by hand, set
  `SysProcAttr.CmdLine` instead of relying on the default quoting. [S21]
- Resolve `wsl.exe` once at startup (or use `%SystemRoot%\System32\wsl.exe`); a scheduled
  task's `PATH` is not guaranteed to be the same as the app's.

### Environment, stdio, and pipes

```go
cmd.Env = append(os.Environ(), "WSL_UTF8=1")
cmd.Stdin = nil                      // NUL: bash sees EOF instead of waiting
cmd.Stdout = logFile                 // *os.File: handle passed straight to wsl.exe
cmd.Stderr = logFile
cmd.SysProcAttr = &syscall.SysProcAttr{
    CreationFlags: windows.CREATE_NO_WINDOW, // x/sys/windows; or 0x08000000
    HideWindow:    true,
}
cmd.WaitDelay = 10 * time.Second
```

- `WSL_UTF8=1` only needs to be in the Windows child's environment. Go keeps the last value
  for a duplicated key, so `append(os.Environ(), ...)` is safe. [S20]
- `Stdin = nil` connects the child to `os.DevNull`; bash does not block on `read`. [S20]
- `Stdout`/`Stderr` as `*os.File` pass the handle directly. Any other writer makes
  `os/exec` create a pipe and copy in a goroutine; that is fine too, but `CombinedOutput`
  buffers the whole Run in memory and is wrong for long jobs. [S20]
- If you use `StdoutPipe`/`StderrPipe`, read them before calling `Wait`; `Wait` closes the
  pipes. [S20]
- `CREATE_NO_WINDOW` gives the child no console at all, which avoids a window flash if the
  app is ever started from a console. `HideWindow` alone only hides the window. [S19][S21]
- `WaitDelay` bounds `Wait` when a child exits but its pipes stay open. [S20]

### Timeouts and status

- Use `exec.CommandContext` with a deadline that covers startup plus the Command's budget.
  Check `ctx.Err() == context.DeadlineExceeded` after `Run` and store `timed-out`
  separately; do not infer it from the exit code, which will be 1 after `Kill`. [S20][S22]
- For a hard stop, run the Command under Linux `timeout` (for example
  `timeout --signal=TERM --kill-after=10s 300s bash -lc '<Command>'`) so the whole process
  group is signalled [S25]. Killing `wsl.exe` leaves a Command that ignores `HUP`/`TERM` alive.
  [L5] GNU `timeout` exits 124 when it fires, which is a convenient distinct code.
- Store the result as a `uint32` plus a kind:
  `exited` (0-255), `signalled` (raw 1-31 or 128+n), `wsl-failed` (`0xFFFFFFFF`),
  `timed-out`, or `wrapper-error` (could not start `wsl.exe`, could not open the log).
- `wsl.exe`'s own errors go to stdout, so if the first thing in the log is "There is no
  distribution with the supplied name" and the code is `0xFFFFFFFF`, the app can present a
  useful "distro missing" message rather than a generic failure. [S5][L1]

### Shell semantics

`bash -lc` is a non-interactive login shell. It reads `/etc/profile`, then the first of
`~/.bash_profile`, `~/.bash_login`, `~/.profile`; it does not read `~/.bashrc` unless one
of those files sources it. [S23] So aliases and functions defined only in `.bashrc` are not
available to a Job. Commands should be executables, scripts, or explicitly sourced. Profile
output (MOTD, version managers, errors) will land in the Run log.

## Local verification

Environment:

- Windows 11, build 10.0.26200.9711; WSL 3.0.3.0; kernel 6.18.54.1-1; WSLg 1.0.79.
- Ubuntu 26.04.1 LTS as the default distro; a second Ubuntu 26.04.1 distro (`AgentBox`)
  used for cold-start tests.
- 2026-11-05, all commands run from a Windows PowerShell started from WSL.

Methods:

- [L1] .NET `ProcessStartInfo` with `UseShellExecute = false`, `CreateNoWindow = true`,
  `RedirectStandardOutput/Error = true`; raw bytes captured from the base streams.
- [L2] Temporary tasks registered with `New-ScheduledTaskPrincipal -LogonType Interactive`
  (and one with `schtasks /create`), run, read with `Get-ScheduledTaskInfo`, then deleted.
  The `.cmd` variant redirected to files and exited with `%ERRORLEVEL%`; the direct variant
  ran `wsl.exe` itself.
- [L3] A small C# probe called `GetConsoleWindow`, `GetStdHandle`, `GetFileType`, and
  `Process.SessionId`, run as a scheduled task and directly.
- [L4] `Stopwatch` around cold and warm `wsl.exe` invocations, including five concurrent
  cold starts.
- [L5] Started `wsl.exe -e bash -lc "sleep 5; echo ... > file"`, called `Kill` after 1.2 s,
  then checked for the file after 7 s, with and without `trap '' HUP TERM`.
- [L6] `ProcessStartInfo.WorkingDirectory = C:\Windows\System32`, with and without
  `--cd ~`, plus a real Task Scheduler action with no "Start in".
- [L7] `-- bash -lc "false; echo $?"` versus `-e bash -lc "false; echo $?"`.

Results are quoted in the sections above. The test scripts and task definitions were
temporary and are not part of the repo.

## What we could not verify

- A real locked-session run. The conclusion in that section is from documented session
  behaviour, not a live lock test; locking the workstation would have interrupted the
  machine's user.
- WSL 1. All tests used WSL 2. The WSL 1 path uses the same `LXSS_WIFEXITED` logic in the
  client, but it was not exercised. [S3]
- WSL versions older than 0.64.0, which do not support `WSL_UTF8`; those will need a UTF-16
  decode fallback for `wsl.exe`'s own messages. [S6]
- Windows 10 and older Windows 11 builds. The machine runs Windows 11 build 26200; the Task
  Scheduler console behaviour may differ on older builds.
- Output larger than 1 MB and multi-hour Runs. The relay handled 1 MB without loss, but
  long-run disk and rotation behaviour is a separate concern.

## Sources

Source links for microsoft/WSL are pinned to commit 867699d (master on 2026-11-05); the
installed build was 3.0.3.0. Local-test entries [L1]-[L7] are described above.

- [S1] Microsoft Learn, "Basic commands for WSL".
  https://learn.microsoft.com/en-us/windows/wsl/basic-commands
- [S2] microsoft/WSL, `src/windows/common/WslClient.cpp` (commit 867699d), especially
  the -1 sentinel (L1566-1569), `-e` parsing (L1658-1663), the `$SHELL -c` comment
  (L1822-1827), and `WSL_UTF8` (L1883-1891).
  https://github.com/microsoft/WSL/blob/867699d/src/windows/common/WslClient.cpp
- [S3] microsoft/WSL, `src/windows/common/svccomm.cpp` (commit 867699d), console-handle
  detection (L316-325), the WSL 1 wait-status path (L410-414), and the valid-handle check
  for the output relay (L480-482).
  https://github.com/microsoft/WSL/blob/867699d/src/windows/common/svccomm.cpp
- [S4] microsoft/WSL, `src/linux/init/init.cpp` (commit 867699d), raw wait status sent for a
  non-normal exit (L2110-2123).
  https://github.com/microsoft/WSL/blob/867699d/src/linux/init/init.cpp
- [S5] microsoft/WSL, `src/windows/common/wslutil.cpp` (commit 867699d), `PrintMessage`
  defaults to stdout (L1543-1546) and `SetCrtEncoding` selects `_O_U8TEXT`/`_O_U16TEXT`
  (L1561-1590).
  https://github.com/microsoft/WSL/blob/867699d/src/windows/common/wslutil.cpp
- [S6] microsoft/WSL, release 0.64.0: "Adds the opt-in `WSL_UTF8=1` environment variable to
  force UTF8 encoding on stdout & stderr through the CRT."
  https://github.com/microsoft/WSL/releases/tag/0.64.0
- [S7] Microsoft Learn, "Advanced settings configuration in WSL": `instanceIdleTimeout`,
  `vmIdleTimeout`, `kernelBootTimeout`, `distributionStartTimeout`, `[boot] initTimeout`.
  https://learn.microsoft.com/en-us/windows/wsl/wsl-config
- [S8] Microsoft Learn, WSL FAQ: the utility VM "starts, stops and manages resources
  automatically".
  https://learn.microsoft.com/en-us/windows/wsl/faq
- [S9] microsoft/WSL#41796, "WSL 2.7.13: cold startup takes 23-32s ... warm invocation
  around 0.13s".
  https://github.com/microsoft/WSL/issues/41796
- [S10] microsoft/WSL#42029, "WSL 3.0.3.0 stops accepting new exec/session requests with
  Wsl/Service/WSAETIMEDOUT".
  https://github.com/microsoft/WSL/issues/42029
- [S11] microsoft/WSL#41937, "`wsl --list` / `wsl --status` write nothing to stdout when it
  is redirected or piped (exit code 0)", and the follow-up #41943.
  https://github.com/microsoft/WSL/issues/41937
  https://github.com/microsoft/WSL/pull/41943
- [S12] Microsoft/WSL#9231 and Microsoft Learn, WSL Store release notes: session 0 is a
  known unsupported case.
  https://github.com/microsoft/WSL/issues/9231
  https://learn.microsoft.com/en-us/windows/wsl/store-release-notes
- [S13] microsoft/WSL#4536, "WSL shell is killed when screen is locked" (2019, closed).
  https://github.com/microsoft/WSL/issues/4536
- [S14] microsoft/WSL#7001, interop/clipboard failure while Windows is locked.
  https://github.com/microsoft/WSL/issues/7001
- [S15] Microsoft Learn, `TASK_LOGON_TYPE`: `TASK_LOGON_INTERACTIVE_TOKEN` means "User must
  already be logged on. The task will be run only in an existing interactive session."
  https://learn.microsoft.com/en-us/windows/win32/api/taskschd/ne-taskschd-task_logon_type
- [S16] Microsoft Learn, "Winlogon States": the workstation-locked state keeps work in
  place; logoff terminates the session's processes.
  https://learn.microsoft.com/en-us/windows/win32/secauthn/winlogon-states
- [S17] Microsoft Learn, `WTS_CONNECTSTATE_CLASS`: `WTSDisconnected` can mean the user is
  signed in but at the lock screen.
  https://learn.microsoft.com/en-us/windows/win32/api/wtsapi32/ne-wtsapi32-wts_connectstate_class
- [S18] Microsoft Learn, `RegisteredTask.LastTaskResult` and "C/C++ Code Example Retrieving
  Task Exit Code".
  https://learn.microsoft.com/windows/win32/taskschd/registeredtask-lasttaskresult
  https://learn.microsoft.com/windows/win32/taskschd/c-c-code-example-retrieving-task-exit-code
- [S19] Microsoft Learn, "Process Creation Flags": `CREATE_NO_WINDOW` (0x08000000) runs a
  console application without a console window.
  https://learn.microsoft.com/en-us/windows/win32/procthread/process-creation-flags
- [S20] Go documentation, `os/exec`: `Command` quoting is `CommandLineToArgvW`-compatible;
  nil `Stdin`/`Stdout`/`Stderr` use the null device; `WaitDelay`; `StdoutPipe` caveat.
  https://pkg.go.dev/os/exec
- [S21] Go source, `src/syscall/exec_windows.go` (go1.27.3): `SysProcAttr` with `CmdLine`,
  `CreationFlags`, `HideWindow` (L292-302), and `makeCmdLine` (L104-116).
  https://github.com/golang/go/blob/go1.27.3/src/syscall/exec_windows.go
- [S22] Go source: `ProcessState.ExitCode` comment (exec_posix.go L155-163),
  `TerminateProcess(..., 1)` in `os.Process.Kill` (exec_windows.go L73), and
  `WaitStatus.ExitStatus` (syscall_windows.go L1156).
  https://github.com/golang/go/blob/go1.27.3/src/os/exec_posix.go
  https://github.com/golang/go/blob/go1.27.3/src/os/exec_windows.go
  https://github.com/golang/go/blob/go1.27.3/src/syscall/syscall_windows.go
- [S23] GNU Bash Reference Manual: exit status is 8-bit and 128+n for a signalled child;
  login-shell startup files; non-interactive shells do not read `.bashrc` by default.
  https://www.gnu.org/software/bash/manual/bash.html
- [S24] microsoft/WSL, `src/windows/common/ConsoleState.cpp` (commit 867699d): a missing
  `CONIN$`/`CONOUT$` is logged and ignored, so `wsl.exe` still runs without a console.
  https://github.com/microsoft/WSL/blob/867699d/src/windows/common/ConsoleState.cpp
- [S25] GNU coreutils manual, `timeout`: exit status 124 when the time limit is reached.
  https://www.gnu.org/software/coreutils/manual/html_node/timeout-invocation.html
- [S26] microsoft/WSL, `src/windows/common/relay.cpp` (commit 867699d): the output relay
  writes the buffer with `WriteFile` (L201), with no transcoding.
  https://github.com/microsoft/WSL/blob/867699d/src/windows/common/relay.cpp
- [L1]-[L7] Local test harnesses described in "Local verification" (2026-11-05).
