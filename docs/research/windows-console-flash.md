# Windows console flashing when Task Scheduler runs `das-scheduler run`

- Ticket: [4lch4/das-scheduler#9](https://github.com/4lch4/das-scheduler/issues/9)
- Date: 2026-11-05
- Method: desk research against Microsoft docs, Windows Terminal/OpenConsole and PowerShell
  source, and the Go toolchain source. The only local check was the Windows build number.

## Answer

Build the Windows executable as a **console-subsystem** binary (Go's default) and embed an
application manifest with `consoleAllocationPolicy=detached`. Then launch every Job Command
with `CREATE_NO_WINDOW`.

That combination gives:

- **No console window when Task Scheduler starts `das-scheduler run <job-id>`.** The manifest
  tells Windows to skip console allocation for console-subsystem programs launched from
  Explorer or Task Scheduler [5][12][17].
- **Normal CLI behavior.** Because the PE subsystem stays `IMAGE_SUBSYSTEM_WINDOWS_CUI`,
  shells still wait for the process, exit codes propagate, and stdout/stderr work when the
  process inherits a terminal [5].
- **No window for the child Command either.** `CREATE_NO_WINDOW` gives the child a console
  host whose window is invisible, while stdout/stderr still go to the handles you pass
  [10][17].
- **Output and exit code captured by `das-scheduler` itself.** Point the child's stdout and
  stderr at the Run log file and read `cmd.ProcessState.ExitCode()` after `cmd.Wait()`
  [20][21]. Task Scheduler does not capture output; the wrapper has to do it [16].

The manifest is supported only on Windows 11 24H2 (build 26100) and Windows Server 2025 or
later [5][12]. The Windows machine in this session is Windows 11 Pro build 26200, so this
path is available here (`cmd.exe /c ver` and `Get-CimInstance Win32_OperatingSystem` both
reported 10.0.26200).

Do **not** make `-H windowsgui` the primary build of a binary that is also a CLI. It removes
the wrapper's console, but it also makes the process a GUI-subsystem app: shells no longer
wait for it, and its own stdout/stderr are usually invalid [5][7][8]. `AttachConsole` can
restore output, but not the shell's wait or exit-code behavior [8][17].

`wscript`, `pwsh -WindowStyle Hidden`, and `conhost --headless` do not solve the problem.
The first two hide a window that has already been created, so a brief flash remains, and
they add a second process (and, for VBScript, a dependency Windows is removing). The third
is the internal ConPTY host, not a supported hidden launcher [17][23][25][29].

If Windows versions older than 24H2 must be supported, use one of these instead:

1. Ship two binaries: a GUI-subsystem `das-scheduler.exe` for Task Scheduler, and a
   console-subsystem CLI for people (see "Separate console-subsystem binary" below).
2. Accept the `-H windowsgui` compromise and treat `run` as an internal entry point that no
   human invokes; output still goes to the Run log, but a manual CLI invocation will not
   block or print reliably.

## What actually causes the flash

A PE image declares a subsystem in its optional header: `IMAGE_SUBSYSTEM_WINDOWS_GUI` (2) or
`IMAGE_SUBSYSTEM_WINDOWS_CUI` (3) [4]. When Windows starts a CUI image as a process, it
allocates a console and a console host (`conhost.exe`); a GUI image gets no console [5][6].
This is not a window that the program creates and can suppress; it is allocated when the
process starts, before the program's entry point runs, and the window belongs to the console
host [17][33].

The same rule applies to a shell:

- A CUI program run inside an existing console session inherits that console [5][6].
- A CUI program started with no console (Explorer, Task Scheduler) gets a new console and
  console host [5][6].
- A GUI program is never allocated a console and does not block the shell [5].

Task Scheduler's `ExecAction` starts the executable [16]. Under the map's user-level,
logged-on-only constraint, the task uses an interactive token, which Microsoft documents as
running "only in an existing interactive session" [15].
The `ExecAction` exposes only `Path`, `Arguments`, `WorkingDirectory` (plus `Id` and `Type`)
[16], and the task's `Hidden` setting only hides the task from the Task Scheduler UI, not
the program's window [13][14]. So a CUI `das-scheduler run` gets a console window exactly like
any other console program launched from Explorer.

## `-H windowsgui`

Go's linker flag is documented as: "On Windows, `-H windowsgui` writes a 'GUI binary'
instead of a 'console binary'" [1]. The linker implements it by setting the PE subsystem to
`IMAGE_SUBSYSTEM_WINDOWS_GUI` instead of `IMAGE_SUBSYSTEM_WINDOWS_CUI` [2][3].

### Effect on stdout and stderr

- A GUI-subsystem process is not attached to a console when created [6].
- Its standard handles are not filled automatically. They are valid only if the launcher
  passed them by handle inheritance; `GetStdHandle` returns `NULL` when the process has no
  associated handles [7][8].
- Go captures `GetStdHandle(STD_OUTPUT_HANDLE)` and `STD_ERROR_HANDLE` once, at package
  initialization, and builds `os.Stdout`/`os.Stderr` from those handles [35][36].
- Writes from `das-scheduler run` itself therefore fail or go nowhere when Task Scheduler
  launches it [7][35][36]. The child's output is unaffected: it comes from the pipe or file
  handles that `os/exec` creates, not from a console handle [11][19][20].
- Attaching to a console later does not repair Go's cached `os.File` values. `AttachConsole`
  replaces the process's standard handles [8], but `os.Stdout` still wraps the pre-attach
  handle; the program must reopen `CONOUT$` (or rebuild `os.Stdout` from the new handle)
  after attaching [8][35][36].

So `-H windowsgui` does not prevent capturing the child's output. It prevents the wrapper
itself having a usable console, which is a different thing.

### Effect on the single-binary GUI+CLI design

Microsoft's console-allocation write-up says a GUI-subsystem application "will neither be
allocated a console, nor block execution inside a shell" [5]. Concretely:

- `das-scheduler run 123` typed in a terminal returns the prompt immediately; the shell did
  not wait for the process.
- `$LASTEXITCODE` in PowerShell (or `%ERRORLEVEL%` in `cmd.exe`) is not the Run's exit code.
- If the program attaches to the parent console and prints, the output can interleave with
  the next prompt. Microsoft's Terminal spec shows the same problem and calls it out
  explicitly [17].
- `AttachConsole(ATTACH_PARENT_PROCESS)` can join the parent console [8], but the shell's
  decision to wait was made from the PE subsystem before the program ran [5][17].

Verdict: good for a GUI-only executable. It is also the simplest option if `run` is
strictly an internal entry point that people never type. It is a poor fit for a real CLI.

## Separate console-subsystem binary

Ship two artifacts from one codebase:

- `das-scheduler.exe`: GUI subsystem (`-H windowsgui`). Task Scheduler's action points at
  this for `run <job-id>`. It has no console [5][6] and writes the Run log itself.
- `das-scheduler-cli.exe`: console subsystem (Go's default). People invoke this. It has
  normal wait, output, and exit-code behavior.

This is correct on every Windows version and needs no manifest. The costs:

- Two binaries to build, sign, and install.
- Two entry points, so the command parsing and `run` implementation must be shared and
  must work without stdout.
- The scheduled action must point at the GUI binary. Pointing it at the console binary
  restores the flash.

Verdict: the clean fallback when pre-24H2 Windows is in scope. Redundant when the manifest
is available.

## `conhost --headless`

`conhost.exe` is the Windows console host: the server for the Windows Console APIs and the
classic console UI [33]. `--headless` is an internal switch parsed by `ConsoleArguments`
alongside `--width`, `--height`, `--signal`, and `--server` [30]. The OpenConsole/ConPTY
code builds this command line to start a console host for a pseudoconsole:

```
"<conhost>" --headless ... --width <n> --height <n> --signal 0x... --server 0x...
```

It passes the input/output pipes and the server handle through `STARTUPINFOEX`, then talks
to the host over the ConPTY protocol [29].

The documented public API for that is `CreatePseudoConsole`, which returns a pseudoconsole
handle and produces UTF-8 text interleaved with virtual-terminal sequences [31][32]. A
pseudoconsole deliberately has no hosting window; the caller owns presentation [32].

`conhost --headless` is therefore not a drop-in hidden launcher for `das-scheduler run`:

- It needs a console server handle and a client command to host [29][30].
- Its output is a VT stream over pipes the caller must create and service synchronously
  [29][31].
- The switch is not part of the documented console API; it is an implementation detail of
  ConPTY/OpenConsole [29][30].

Using `CreatePseudoConsole` directly would also be more work than plain `exec.Cmd` with
redirected output, and it would give the Run log VT escape sequences instead of plain text.
Verdict: not a fit for a scheduler wrapper.

## `wscript` and `pwsh -WindowStyle Hidden`

### `wscript`

Windows Script Host has two hosts: `wscript.exe` is the Windows-based host, and
`cscript.exe` is the command-prompt host [24]. A task can run:

```
wscript.exe //nologo //B run-hidden.vbs
```

The VBScript calls `WScript.Shell.Run(command, 0, True)`. Window style `0` is documented as
"Hides the window", and `bWaitOnReturn=True` makes `Run` return the child's exit code
[23]. Limits:

- `Run` exposes only the exit code, not stdout/stderr, so output must be redirected to a
  file by the child command [23].
- Microsoft warns that "not all programs make use of this information" [23], so the console
  host may ignore the requested style.
- It adds a second deployed file, plus a scripting host that shows script errors in dialogs
  unless `/B` is used [37].
- VBScript is deprecated. It is a feature-on-demand, on by default in Windows 11 24H2,
  expected to be off by default around 2027, and removed later [25][39].

Verdict: works today, but it is not durable and not a single binary.

### `pwsh -WindowStyle Hidden`

`pwsh` documents `-WindowStyle Hidden` as setting the window style for the session [26].
PowerShell is a console-subsystem application; Microsoft's allocation spec uses PowerShell
as the example of a program that would need a separate `pwshw.exe` to avoid a console [17].
At startup, PowerShell applies the style with `GetConsoleWindow()` followed by
`ShowWindow(hwnd, SW_HIDE)` [27][28]. By then the console exists and has been shown. The
same spec says that when the runtime hides the window after allocation, "there will still
be a brief period during which that window is visible. It is inescapable" [17].

It also adds a PowerShell startup and a second process. Task Scheduler's action has no output
property [16], so PowerShell would still have to redirect to a file; and preserving a specific
exit code needs `exit $LASTEXITCODE` [26].

Verdict: still flashes, so it fails the requirement.

## `CREATE_NO_WINDOW` for children

`CREATE_NO_WINDOW` (0x08000000) is documented as: "The process is a console application
that is being run without a console window" [10]. It is ignored for non-console
applications and cannot be combined with `CREATE_NEW_CONSOLE` or `DETACHED_PROCESS` [10].
The Windows Terminal spec describes the mechanism: a console host is spawned, but the
first connection packet says the window should be invisible [17]. `DETACHED_PROCESS` is
different: the child gets no console at all and may call `AllocConsole` later [6][9][10].

In Go:

```go
cmd := exec.Command(shell, args...)
cmd.Stdout = logWriter
cmd.Stderr = logWriter
cmd.SysProcAttr = &syscall.SysProcAttr{
    CreationFlags: windows.CREATE_NO_WINDOW, // 0x08000000
}
err := cmd.Run()
exitCode := cmd.ProcessState.ExitCode()
```

Go passes `SysProcAttr.CreationFlags` straight into `CreateProcess` [19];
`windows.CREATE_NO_WINDOW` is the constant in `golang.org/x/sys/windows` [18]. `os/exec`
sets `STARTF_USESTDHANDLES` and supplies the `Stdout`/`Stderr` handles, so the child still
writes to the log file or pipe [19][20]. `ExitCode()` returns the exit code, or -1 if the
process has not exited [21].

Two things to remember:

- The flag fixes the child's window, not the wrapper's. If the wrapper is launched with a
  visible console, that window still flashes. The wrapper must be GUI-subsystem or
  CUI+detached.
- Children of the child inherit the invisible console by default (a `CreateProcess` with
  flags 0 inherits the parent's console) [11], so normally only the first child needs the
  flag. A grandchild that explicitly requests `CREATE_NEW_CONSOLE` can still create a
  visible window; that is outside the wrapper's control.

Prefer `CREATE_NO_WINDOW` over Go's `SysProcAttr.HideWindow`. Go implements `HideWindow` as
`STARTF_USESHOWWINDOW` plus `SW_HIDE` [19], which is a window-show hint. `CREATE_NO_WINDOW`
is the flag Microsoft documents for running a console application with no console window
[10].

## Single binary that is both GUI and CLI

| Build | Task window | CLI wait/exit | Own stdout | Child output | 1 binary |
|---|---|---|---|---|---|
| `-H windowsgui` | none | no | usually invalid | capturable | yes |
| default CUI | flashes | yes | yes | capturable | yes |
| CUI + detached manifest | none (24H2+) | yes | yes from a shell | capturable | yes |
| GUI + separate CLI | none (GUI) | yes (CLI) | CLI yes, GUI no | capturable | no |
| `wscript` wrapper | hidden, best effort | VBS waits | no | file only | no |
| `pwsh -WindowStyle Hidden` | brief flash | yes | yes | file redirect only | no |
| `conhost --headless` | none | n/a | VT stream | yes, as VT | no |

The CUI+manifest row is the only one that satisfies all of: no window under Task Scheduler,
normal CLI semantics, and one executable [5][12][17].

A note for the GUI toolkit ticket: the console-allocation policy applies only to CUI images
[17]. If the chosen toolkit forces `-H windowsgui`, the manifest will do nothing and the
single-binary choice reverts to the `-H windowsgui` row. That is a toolkit constraint, not
something the manifest can fix.

## Recommended design for das-scheduler v1

1. **Windows build:** use Go's default (CUI); do not pass `-H windowsgui`. Embed a manifest
   with `consoleAllocationPolicy=detached` [5][12]. Merge the element into whatever manifest
   the GUI toolkit already needs; Windows reads the manifest from the `.exe` resource, so it
   must be linked in [12].
2. **Manifest packaging:** Go collects `.syso` files in the package directory and the go
   command passes them to the linker [22][38], so generate one PE resource object containing
   the manifest and let `go build` link it. If embedding is inconvenient at first, Microsoft
   allows an external `das-scheduler.exe.manifest` beside the executable [12], but embedding
   is the preferred form.
3. **`run` path:** spawn the Job's shell with `CREATE_NO_WINDOW`; point the child's stdout
   and stderr at the Run log file; record start, end, and exit code from
   `ProcessState.ExitCode()` [10][18][19][20][21]. Do not depend on Task Scheduler to
   capture output; its `ExecAction` has no output property [16]. Keep the `run` path out of
   GUI initialization so it never creates the toolkit window.
4. **CLI path:** when a console is present, optionally tee the child's output to the
   terminal. Build the writer so a failed terminal write cannot abort the file write:
   `io.MultiWriter` stops at the first failing writer [34]. A scheduled run normally has no
   stdout handle, so skip the tee there, or use a writer that ignores stdout errors.
5. **Older Windows:** if a pre-24H2 release is in scope, ship the two-binary layout above.
   Otherwise set Windows 11 24H2 / Server 2025 as the Windows baseline and keep one
   executable.
6. **Do not add `wscript`, `pwsh -WindowStyle Hidden`, or `conhost --headless`:** they
   either leave the flash, add a process and a deprecated dependency, or require ConPTY
   plumbing [17][23][25][29][31].

### Manifest snippet

```xml
<assembly manifestVersion="1.0" xmlns="urn:schemas-microsoft-com:asm.v1">
  <application>
    <windowsSettings>
      <consoleAllocationPolicy xmlns="http://schemas.microsoft.com/SMI/2024/WindowsSettings"
        >detached</consoleAllocationPolicy>
    </windowsSettings>
  </application>
</assembly>
```

This is the element and namespace from the Microsoft docs [5][12]. Keep the element's text
exactly `detached`; the value is case-insensitive, but the element name is not.

## Sources

URLs are left unwrapped so they remain clickable; every other line is wrapped at 100
columns.

- [1] Go `cmd/link` documentation, `-H type`:
  https://pkg.go.dev/cmd/link
- [2] Go linker source, `-H windowsgui` flag handling:
  https://github.com/golang/go/blob/master/src/cmd/link/internal/ld/main.go
- [3] Go linker source, PE subsystem selection:
  https://github.com/golang/go/blob/master/src/cmd/link/internal/ld/pe.go
- [4] Microsoft, PE format, Windows subsystem values:
  https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#windows-subsystem
- [5] Microsoft, Console Allocation Policy:
  https://learn.microsoft.com/en-us/windows/console/console-allocation-policy
- [6] Microsoft, Creation of a Console:
  https://learn.microsoft.com/en-us/windows/console/creation-of-a-console
- [7] Microsoft, GetStdHandle:
  https://learn.microsoft.com/en-us/windows/console/getstdhandle
- [8] Microsoft, AttachConsole:
  https://learn.microsoft.com/en-us/windows/console/attachconsole
- [9] Microsoft, AllocConsole:
  https://learn.microsoft.com/en-us/windows/console/allocconsole
- [10] Microsoft, Process Creation Flags:
  https://learn.microsoft.com/en-us/windows/win32/procthread/process-creation-flags
- [11] Microsoft, CreateProcessW:
  https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw
- [12] Microsoft, Application manifests, consoleAllocationPolicy:
  https://learn.microsoft.com/en-us/windows/win32/sbscs/application-manifests#consoleallocationpolicy
- [13] Microsoft, TaskSettings.Hidden:
  https://learn.microsoft.com/en-us/windows/win32/taskschd/tasksettings-hidden
- [14] Microsoft, Hidden (settingsType) element:
  https://learn.microsoft.com/en-us/windows/win32/taskschd/taskschedulerschema-hidden-settingstype-element
- [15] Microsoft, Principal.LogonType:
  https://learn.microsoft.com/en-us/windows/win32/taskschd/principal-logontype
- [16] Microsoft, ExecAction object:
  https://learn.microsoft.com/en-us/windows/win32/taskschd/execaction
- [17] Microsoft Terminal spec #7335, Console Allocation Policy:
  https://github.com/microsoft/terminal/blob/main/doc/specs/%237335%20-%20Console%20Allocation%20Policy.md
- [18] golang.org/x/sys/windows constants:
  https://github.com/golang/sys/blob/master/windows/types_windows.go
- [19] Go `syscall/exec_windows.go` (`SysProcAttr`, `HideWindow`, creation flags):
  https://github.com/golang/go/blob/master/src/syscall/exec_windows.go
- [20] Go `os/exec` documentation:
  https://pkg.go.dev/os/exec
- [21] Go `os.ProcessState.ExitCode`:
  https://pkg.go.dev/os#ProcessState.ExitCode
- [22] Go `go/build` `.syso` collection:
  https://github.com/golang/go/blob/master/src/go/build/build.go
- [23] Microsoft, WScript.Shell.Run method:
  https://learn.microsoft.com/en-us/previous-versions/d5fk67ky(v=vs.85)
- [24] Microsoft, Windows Script Host overview (wscript vs cscript):
  https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc738350(v=ws.10)
- [25] Microsoft, Deprecated features, VBScript row:
  https://learn.microsoft.com/en-us/windows/whats-new/deprecated-features
- [26] Microsoft, about_pwsh, `-WindowStyle`:
  https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh#-windowstyle--w
- [27] PowerShell source, `ConsoleControl.SetConsoleMode`:
  https://github.com/PowerShell/PowerShell/blob/master/src/Microsoft.PowerShell.ConsoleHost/host/msh/ConsoleControl.cs
- [28] PowerShell source, startup call site:
  https://github.com/PowerShell/PowerShell/blob/master/src/Microsoft.PowerShell.ConsoleHost/host/msh/ConsoleHost.cs
- [29] Windows Terminal/OpenConsole source, `winconpty.cpp`:
  https://github.com/microsoft/terminal/blob/main/src/winconpty/winconpty.cpp
- [30] Windows Terminal/OpenConsole source, `ConsoleArguments.cpp`:
  https://github.com/microsoft/terminal/blob/main/src/host/ConsoleArguments.cpp
- [31] Microsoft, CreatePseudoConsole:
  https://learn.microsoft.com/en-us/windows/console/createpseudoconsole
- [32] Microsoft, Pseudoconsoles:
  https://learn.microsoft.com/en-us/windows/console/pseudoconsoles
- [33] Microsoft, Windows Console and Terminal Definitions:
  https://learn.microsoft.com/en-us/windows/console/definitions
- [34] Go `io.MultiWriter` documentation:
  https://pkg.go.dev/io#MultiWriter
- [35] Go source, `os.Stdout`/`os.Stderr` initialization:
  https://github.com/golang/go/blob/master/src/os/file.go
- [36] Go source, standard handles captured at package init:
  https://github.com/golang/go/blob/master/src/syscall/syscall_windows.go
- [37] Microsoft, `wscript` command reference (`/B`):
  https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/wscript
- [38] Go source, `.syso` files passed to the link step:
  https://github.com/golang/go/blob/master/src/cmd/go/internal/work/exec.go
- [39] Microsoft/Windows IT Pro, VBScript deprecation: timelines and next steps:
  https://techcommunity.microsoft.com/blog/windows-itpro-blog/vbscript-deprecation-timelines-and-next-steps/4148301
