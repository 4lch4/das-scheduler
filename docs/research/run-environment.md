# Run environment: matching the terminal from cron and Task Scheduler

**Ticket:** [Making Runs see the terminal's environment][ticket-8]
**Map:** [das-scheduler v1 spec][map-3]
**Date:** 2026-11-05

[ticket-8]: https://github.com/4lch4/das-scheduler/issues/8
[map-3]: https://github.com/4lch4/das-scheduler/issues/3

## Answer

Neither Backend starts a terminal. cron runs the Command with `/bin/sh -c` (or the crontab's
`SHELL`) and a minimal environment; Task Scheduler starts the executable in the task action. A
Run therefore cannot inherit the terminal's environment; `das-scheduler run` has to rebuild it.

The login-shell rule is directionally right, but `bash -lc` is not enough on Ubuntu.
`~/.profile` sources `~/.bashrc`, and Ubuntu's `~/.bashrc` returns immediately when the shell is
not interactive. The PATH lines that nvm, Bun and the OpenCode installer write to `~/.bashrc`
when bash is the detected shell sit after that guard, so a non-interactive login shell never
reaches them.

Recommended invocation per shell:

- **bash, environment in `~/.bashrc` (the common case):** `bash -ic '<Command>'`.
  This reads `~/.bashrc` the way a non-login desktop terminal does. Bash prints
  `cannot set terminal process group` and `no job control in this shell` to stderr when there
  is no TTY; those two lines can be filtered.
- **bash, environment in login files:** `bash -lc '<Command>'`. This reads `/etc/profile` and
  the first of `~/.bash_profile`, `~/.bash_login` or `~/.profile`. It avoids interactive
  `.bashrc` setup, but it does not read the body of `~/.bashrc` on Ubuntu.
- **bash, environment split across both:** `bash -lic '<Command>'`. This reads the login files
  and then `~/.bashrc` when the login file sources it. It also runs `/etc/profile` login
  scripts, which can print login noise such as `update-motd`.
- **sh:** `sh -lc '<Command>'`. This reads `/etc/profile` and `~/.profile`, never `~/.bashrc`.
- **pwsh:** `pwsh -NoProfile -NonInteractive -Command '<Command>'`. Use this when PATH and keys
  are in the Windows user environment. Drop `-NoProfile` only when the profile is the source.
  `-Login` does nothing on Windows.
- **powershell:** `powershell.exe -NoProfile -NonInteractive -Command '<Command>'`. Same rule
  for `-NoProfile`.
- **cmd:** `cmd.exe /d /c '<Command>'`. `/d` skips AutoRun. Omit `/d` only if terminal parity
  requires the user's AutoRun script.
- **WSL:** `wsl.exe -d <Distro> -- bash -ic '<Command>'`. Use `bash -lc` if the environment is
  in the Linux login files, or `zsh -ic` if zsh is the distro's shell. Windows variables cross
  only through `WSLENV` or an export inside the distro.

If one rule is preferred over per-shell flags: capture the login shell's `PATH` and an allowlist
of non-secret variables, then run every Command through a non-interactive shell with that
environment. The rest of this note gives the evidence, including why this machine's zsh
terminal cannot be matched by any bash invocation.

## 1. cron gives the Job a minimal environment

Debian's `crontab(5)` says the command portion of a line is executed by `/bin/sh` or by the
shell named in `SHELL` ([crontab(5)](https://manpages.debian.org/bookworm/cron/crontab.5.en.html)).
The daemon sets `SHELL=/bin/sh`, `HOME` and `LOGNAME`; `HOME` and `SHELL` can be overridden in
the crontab, `LOGNAME` cannot (same source).

Ubuntu 26.04 installs `cron 3.0pl1-200ubuntu1`; the Debian/Ubuntu 3.0pl1 source sets
`PATH=/usr/bin:/bin` for a job that does not set its own PATH
([entry.c](https://sources.debian.org/src/cron/3.0pl1-162/entry.c/),
[pathnames.h](https://sources.debian.org/src/cron/3.0pl1-162/pathnames.h/)). The daemon also
sets that PATH for itself ([cron.c](https://sources.debian.org/src/cron/3.0pl1-162/cron.c/)).
Other cron implementations differ in the exact default; cronie's fallback is
`/usr/bin:/bin:/usr/sbin:/sbin` unless started with `-P`
([cronie pathnames.h](https://github.com/cronie-crond/cronie/blob/master/src/pathnames.h)).
Either way, the Job does not get the terminal's PATH. On the `/usr/bin:/bin` default, a wrapper
installed in `/usr/local/bin` is not found either; the crontab entry should use its absolute
path.

That shell is non-interactive and is not a login shell, so it reads no login or interactive
startup file. POSIX
`sh` reads `/etc/profile` and `~/.profile` only for a login shell
([dash(1)](https://manpages.debian.org/bookworm/dash/dash.1.en.html)); Bash reads only
`$BASH_ENV` when it is non-interactive
([Bash startup files](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html)).
This is why a line such as `* * * * * opencode run ...` fails with "command not found" even
though the same command works in a terminal.

Setting `SHELL=/bin/bash` in the crontab changes which shell runs the command, but not the
startup behavior: cron still invokes it as `bash -c`, which is non-interactive
([crontab(5)](https://manpages.debian.org/bookworm/cron/crontab.5.en.html);
[Bash invocation](https://www.gnu.org/software/bash/manual/html_node/Invoking-Bash.html)).
The shell must be invoked with `-l` and/or `-i`, or given an explicit environment.

cron's environment is not literally four variables. Debian cron builds the job environment from
the PAM environment plus the crontab's environment lines
([do_command.c](https://sources.debian.org/src/cron/3.0pl1-162/do_command.c/)), and cron(8)
documents the PAM environment ([cron(8)](https://manpages.debian.org/bookworm/cron/cron.8.en.html)).
None of those sources is the user's shell profile, so they cannot supply the terminal's `PATH`
or a key exported in `.bashrc` / `.zshrc`.

## 2. Bash startup files: `-l`, `-i` and `-c`

Bash selects startup files by two independent properties: whether the shell is a login shell and
whether it is interactive.

- **Login, non-interactive (`bash -lc`):** reads `/etc/profile`, then the first readable of
  `~/.bash_profile`, `~/.bash_login` and `~/.profile`
  ([bash manual](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html)).
- **Interactive, non-login (`bash -ic`):** reads `~/.bashrc` (same source).
- **Non-interactive, non-login (`bash -c`):** reads only `$BASH_ENV`, if set (same source).
- **Login and interactive (`bash -lic`):** reads the login files; whether it also reads
  `~/.bashrc` depends on whether the login file sources it.

The flags are documented in
[Bash invocation](https://www.gnu.org/software/bash/manual/html_node/Invoking-Bash.html):
`-c` reads the command string, `-i` forces interactive mode, and `-l` makes the shell act as a
login shell. A login shell is one whose argument zero starts with `-` or one invoked with
`--login`; an interactive shell is one started with `-i` or one whose stdin and stderr are
terminals (same source).

Bash reads the first existing login file, not all of them. Creating `~/.bash_profile` for an
API key therefore hides `~/.profile` unless the new file sources it; that can silently remove
PATH entries that used to work
([Bash startup files](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html)).

### The Ubuntu `.bashrc` early return

Ubuntu's `bash` package ships an `/etc/skel/.bashrc` that starts with:

```bash
# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac
```

Source: [Ubuntu bashrc](https://git.launchpad.net/ubuntu/+source/bash/plain/debian/skel.bashrc).
The same guard is present in this machine's `~/.bashrc` at lines 6-9.

Ubuntu's `~/.profile` sources `~/.bashrc` when `$BASH` is set
([Ubuntu profile](https://git.launchpad.net/ubuntu/+source/base-files/plain/share/dot.profile)):

```sh
if [ "$BASH" ]; then
  if [ -f ~/.bashrc ]; then
    . ~/.bashrc
  fi
fi
```

On a default Ubuntu account with no `~/.bash_profile`, `bash -lc` does open `~/.bashrc`, but
the shell is not interactive, `$-` does not contain `i`, and the `return` exits the file before
any PATH line after the guard. The login shell gets only the `.profile` lines outside the
`.bashrc` call (plus `$BASH_ENV`, if set).

Older Debian/Ubuntu `.bashrc` files used `[ -z "$PS1" ] && return` instead of `case $-`. It has
the same effect for this question: a non-interactive shell returns before the PATH lines.

### Reproduced with a minimal home

A disposable home directory with the Ubuntu guard, an exported `TERMINAL_ONLY`, a fake tool on
`PATH`, and a `.profile` that sets `LOGIN_ONLY` and sources `.bashrc` gives:

```text
bash -lc   -> TERMINAL_ONLY=unset LOGIN_ONLY=yes  mytool-not-found
bash -ic   -> TERMINAL_ONLY=yes LOGIN_ONLY=unset  .../fakebin/mytool
bash -lic  -> TERMINAL_ONLY=yes LOGIN_ONLY=yes    .../fakebin/mytool
sh -lc     -> TERMINAL_ONLY=unset LOGIN_ONLY=yes  mytool-not-found
```

`bash -ic` also printed the two job-control warnings. `bash -lic` printed those plus the
`update-motd` notice because `/etc/profile.d/update-motd.sh` ran during login. This is the
practical trade-off: `-i` is required to get `.bashrc`, and `-l` adds login-side output.

### What to recommend for bash

- Use `bash -lc` only when the variables are in `/etc/profile`, `~/.bash_profile`,
  `~/.bash_login` or `~/.profile`. It avoids the interactive `.bashrc` side effects, though
  `/etc/profile` scripts can still print.
- Use `bash -ic` when the variables are in `~/.bashrc`, which is where the installers put them.
  It matches a non-login desktop terminal. Filter the two known job-control lines if the Run
  log must be clean.
- Use `bash -lic` when both files matter and the login file sources `~/.bashrc`. It is the
  broadest bash match, but it can run login-only scripts that print output.
- Do not rely on `BASH_ENV=~/.bashrc`: the shell is still non-interactive, so the guard returns
  before the PATH lines.

## 3. Where nvm, Bun and OpenCode add themselves to PATH

All three installers edit a shell config file, and all three prefer a file that a
non-interactive login shell may not read. The exact file depends on the shell they detect.

### nvm

nvm's installer "attempts to add the source lines ... to the correct profile file
(`~/.bashrc`, `~/.bash_profile`, `~/.zshrc`, or `~/.profile`)"
([nvm README](https://github.com/nvm-sh/nvm/blob/master/README.md)). The install script's
detection prefers `~/.bashrc` and then `~/.bash_profile` for bash; for zsh it prefers
`~/.zshrc` and then `~/.zprofile`
([install.sh](https://raw.githubusercontent.com/nvm-sh/nvm/v0.41.0/install.sh)).

nvm is a sourced shell function, not an executable
([nvm README](https://github.com/nvm-sh/nvm/blob/master/README.md)). The shell must source
`nvm.sh` before `node`, `npm` or `npx` exist on PATH. A shell that never sources it cannot find
the nvm-managed Node at all.

### Bun

Bun's installation docs tell bash users to add these lines to `~/.bashrc` (zsh users:
`~/.zshrc`):

```sh
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"
```

Source: [Bun installation](https://bun.com/docs/installation).

The install script tries `~/.bash_profile` first and falls back to `~/.bashrc` when the former
is not writable. Ubuntu's default home has no `~/.bash_profile`, so the script writes
`~/.bashrc` ([Bun install script](https://bun.com/install)). That is below the early-return
guard.

### OpenCode

The OpenCode installer sets `INSTALL_DIR=$HOME/.opencode/bin` and appends
`export PATH=$INSTALL_DIR:$PATH` to a config file chosen from `$SHELL`:

- zsh: for example, `~/.zshrc` first, then `~/.zshenv`
- bash: for example, `~/.bashrc` first, then `~/.bash_profile`, `~/.profile`
- sh: for example, `~/.ashrc`, `~/.profile`, `/etc/profile`

Source: [OpenCode install script](https://opencode.ai/install). The `--no-modify-path` option
skips the edit (same source).

For OpenCode credentials there is a better route than a shell export. `/connect` stores provider
keys in `~/.local/share/opencode/auth.json`
([OpenCode providers](https://opencode.ai/docs/providers/)). That file is read by the OpenCode
process regardless of shell startup, so a cron Job does not need the key in its environment.
OpenCode's own provider docs sometimes suggest `~/.bash_profile` for environment variables
(for example the Cloudflare and Azure sections, same page); that layout is compatible with
`bash -lc`, but it is not where the installer writes PATH.

### This machine

This machine is Ubuntu 26.04.1, the login shell is `/usr/bin/zsh`, and `~/.zshrc` contains the
nvm, Bun and OpenCode PATH lines. Local versions: bash 5.3.9, cron 3.0pl1-200ubuntu1, zsh 5.9,
WSL 3.0.3.0. zsh reads `.zshenv` always, `.zprofile` for login, `.zshrc` for interactive, and
`.zlogin` for login ([zsh files](https://zsh.sourceforge.io/Doc/Release/Files.html)); `zsh -lc`
therefore misses `.zshrc`, while `zsh -lic` reads it. Running from a cron-like empty environment:

```text
$ env -i HOME="$HOME" PATH=/usr/bin:/bin bash -lc 'command -v node bun opencode'
(no output)

$ env -i HOME="$HOME" PATH=/usr/bin:/bin bash -ic 'command -v node bun opencode'
bash: cannot set terminal process group (...): Inappropriate ioctl for device
bash: no job control in this shell
/home/alcha/.opencode/bin/opencode

$ env -i HOME="$HOME" PATH=/usr/bin:/bin zsh -lic 'command -v node bun opencode'
/home/alcha/.nvm/versions/node/v24.19.0/bin/node
/home/alcha/.bun/bin/bun
/home/alcha/.opencode/bin/opencode
```

`bash -ic` found OpenCode because this machine's `~/.bashrc` has the OpenCode PATH line at line
141; nvm and Bun are zsh-only. `zsh -lic` also printed
`(eval):1: can't change option: zle` twice on stderr because there is no TTY, but it still
executed the command and found all three. This is the concrete reason a bash/sh-only shell menu
cannot reproduce this terminal.

## 4. Windows Task Scheduler

### Task Scheduler starts the executable, not a shell

A Task's Exec action has `Path`, `Arguments` and `WorkingDirectory`; it "executes a command-line
operation ... run a script or launch an executable"
([ExecAction](https://learn.microsoft.com/windows/win32/taskschd/execaction)). There is no
shell and no profile in the action. If the action is `pwsh.exe`, PowerShell loads its profiles;
if the action is `cmd.exe`, cmd runs AutoRun; if the action is `das-scheduler.exe`, neither
happens until the wrapper starts a shell.

Task Scheduler also caches environment-variable expansions used in `Path`, `Arguments` and
`WorkingDirectory` when `taskeng.exe` starts; changes made after that are not used by the task
engine ([ExecAction](https://learn.microsoft.com/windows/win32/taskschd/execaction)).
After changing a user PATH or a key, sign out and back in (or restart the Task Scheduler
service) so the task engine is relaunched before expecting the Task to see it.

### PowerShell profiles

PowerShell loads a profile script when it starts unless `-NoProfile` is given; the profile is a
normal script that can add commands, aliases, functions, variables and modules
([profiles][profiles]).

For `pwsh`:

- `pwsh -NoProfile` does not load any profile
  ([pwsh][pwsh-doc]).
- `pwsh -NonInteractive` is intended for scheduled tasks and CI; interactive features such as
  `Read-Host` become statement-terminating errors instead of hanging (same source).
- `pwsh -Login` starts a login shell on Linux and macOS by running `/bin/sh` against
  `/etc/profile` and `~/.profile`; "On Windows, this switch does nothing" (same source). It
  cannot be used to force a Windows profile to load.
- `-Command` takes the command string; `-File` takes a script file. `-Command` must be the last
  parameter because everything after it is part of the command (same source).

For Windows PowerShell 5.1, `powershell.exe` has the same `-NoProfile`, `-NonInteractive` and
`-Command` switches ([PowerShell 5.1][powershell-exe-doc]).

Execution policy is a second gate: under `Restricted`, scripts do not run and profiles do not
run ([profiles][profiles]).
A Task that depends on a profile is therefore also dependent on the execution policy.

### `cmd`

`cmd` has no profile. Without `/d`, it looks for AutoRun commands in
`HKEY_CURRENT_USER\Software\Microsoft\Command Processor\AutoRun` and
`HKEY_LOCAL_MACHINE\Software\Microsoft\Command Processor\AutoRun` and executes them before the
command ([cmd](https://learn.microsoft.com/windows-server/administration/windows-commands/cmd)).
`/d` disables AutoRun. A scheduled Job should normally use `/d`; use plain `/c` only when the
Job is meant to mimic a terminal that relies on AutoRun.

`cmd` inherits the environment of its parent process; system variables come from
`HKEY_LOCAL_MACHINE` and user variables from `HKEY_CURRENT_USER`
([cmd](https://learn.microsoft.com/windows-server/administration/windows-commands/cmd)).
That is the environment block Task Scheduler gives the task, not a profile.

### Windows environment variables

Windows has user and system environment variables. By default a child process inherits its
parent's environment block, and `setx` writes user or system variables to the registry for
future processes; it does not change the current process
([Win32 env](https://learn.microsoft.com/windows/win32/procthread/environment-variables);
[setx](https://learn.microsoft.com/windows-server/administration/windows-commands/setx)).

The practical split:

- A variable set through System Properties, `setx`, or an installer that writes the user PATH
  in the registry is normally in the environment block of a new Task process. `pwsh
  -NoProfile` sees it.
- A variable set only inside a PowerShell session, or only by a profile script, is not in that
  block. `pwsh -NoProfile` does not see it; `pwsh -Command` (profile loaded) does.

For a Windows Job, `pwsh -NoProfile -NonInteractive -Command '<Command>'` is the deterministic
default. Keep PATH and keys in the Windows user environment. If the profile is the only source,
omit `-NoProfile` and accept that profile scripts run; moving the exports to user variables is
the cleaner fix.

### API keys on Windows

Do not put a key in the Command string. It would appear in the Task Scheduler XML, in
`schtasks /query /v` output, and in the process command line. Use a user environment variable,
the OS keyring, or OpenCode's own auth store. The same rule applies to a `.cmd` or `.ps1` file:
the file can hold the environment, but its ACL should be user-only.

## 5. WSL through Task Scheduler

`wsl.exe` behaviour, from the help output on this machine (WSL 3.0.3.0):

- With no command, it launches the default shell.
- `--exec, -e <CommandLine>` executes the command without using the default Linux shell.
- `--shell-type <standard|login|none>` executes the command with the given shell type.
- `--` passes the remaining command line as-is.

That means `wsl.exe -d <Distro> -- bash -ic '<Command>'` is explicit about the shell and its
startup mode. `--exec <Command>` runs `<Command>` without the default shell, so a bare
`opencode` gets no profile; naming `bash -lc` after `--exec` still reads the login files. The
`--shell-type login` option is a newer alternative, but spelling out `bash -ic` or `bash -lc`
is easier to reason about and works on older WSL versions.

WSL does not copy the whole Windows environment into Linux. Apart from the default PATH
translation, it shares only variables listed in `WSLENV`; by default it appends Windows PATH
entries to the Linux `$PATH` (`appendWindowsPath=true`)
([wsl.conf](https://learn.microsoft.com/windows/wsl/wsl-config);
[WSLENV](https://learn.microsoft.com/windows/wsl/filesystems)).
`WSLENV` is a colon-delimited list of variable names, each optionally followed by `/p`, `/l`,
`/u` or `/w` for path translation and direction ([WSLENV format][wslenv-format]). A Windows
`OPENAI_API_KEY` is invisible to a WSL Job unless it is listed in `WSLENV` or exported inside
the distro. With `appendWindowsPath=true`, a Windows `opencode.exe` is also on the Linux PATH;
check `command -v opencode` inside the distro to see which binary wins.

Inside WSL the same bash startup rules apply, including the Ubuntu `.bashrc` guard
([Ubuntu bashrc](https://git.launchpad.net/ubuntu/+source/bash/plain/debian/skel.bashrc)).
If the distro's default shell is zsh, use `zsh -ic` / `zsh -lic`; if it is bash and the tool
PATH is in `.bashrc`, use `bash -ic`. Moving the exports to `~/.profile` lets `bash -lc` work.

OpenCode's docs recommend WSL on Windows: its config and sessions live in
`~/.local/share/opencode/`, and its credentials live in `~/.local/share/opencode/auth.json`
after `/connect`
([OpenCode Windows/WSL](https://opencode.ai/docs/windows-wsl/);
[OpenCode providers](https://opencode.ai/docs/providers/)). Both are in the distro's home, so a
WSL Job that runs as that user gets them without any shell startup file or `WSLENV` entry.

## 6. What this means for das-scheduler

1. **Do not treat "login shell" as sufficient.** On Ubuntu, `bash -lc` is non-interactive and
   stops at the `.bashrc` guard. The v1 note should say "login shell, and interactive when the
   environment lives in `.bashrc`", or it should use an explicit environment file.
2. **Exact parity with this user's terminal needs zsh or a shared file.** `zsh -lic` found nvm,
   Bun and OpenCode; no bash variant found all three. Either add zsh to the Linux shell menu,
   run the user's `$SHELL` from `/etc/passwd`, or require the shared exports in `~/.profile`.
3. **Pick `bash -ic` for `.bashrc`, `bash -lc` for `.profile`.** Do not use `bash -lc` as a
   universal default for bash. `bash -lic` is the both-files option, at the cost of login-script
   output and possibly duplicated PATH entries.
4. **On Windows, default to `pwsh -NoProfile -NonInteractive`.** Keep PATH and keys in user
   environment variables. Load the profile only when the user says the profile is the source.
   `-Login` is not a Windows solution.
5. **Prefer OpenCode's auth store over env vars for keys.** `/connect` writes
   `~/.local/share/opencode/auth.json`, which is independent of shell startup. If a key must be
   an env var, keep it in the OS keyring or an encrypted per-Job store, not in the Command
   string or the crontab.
6. **Do not snapshot the entire environment into SQLite.** A full `env` contains API keys.
   Capture `PATH` plus an allowlist of non-secret names, or capture at Run time and discard.
7. **Preflight the command in the same shell.** Resolve it with `command -v` (or
   `Get-Command`) under the exact invocation the Job will use, and fail with the missing name.
   A green "saved" message for a Job that cannot find `opencode` is worse than a clear warning.

### Go wrapper sketch

Because the wrapper passes the Command as one argument, the calls are simple and avoid an extra
layer of shell quoting:

```go
// Linux
exec.Command("bash", "-ic", command)
exec.Command("bash", "-lc", command)
exec.Command("sh", "-lc", command)

// Windows
exec.Command("pwsh", "-NoProfile", "-NonInteractive", "-Command", command)
exec.Command("powershell.exe", "-NoProfile", "-NonInteractive", "-Command", command)
exec.Command("cmd.exe", "/d", "/c", command)

// WSL
exec.Command("wsl.exe", "-d", distro, "--", "bash", "-ic", command)
```

`cmd.exe` still applies its own quote rules to the final command line; if the Command contains
complex quotes, write a temporary `.cmd` file and run that instead of trying to escape it
through `/c`.

## Scope and confidence

The Linux and WSL behaviour was reproduced on this machine (Ubuntu 26.04.1 under WSL, WSL
3.0.3.0). The Windows Task Scheduler and PowerShell behaviour is from Microsoft's documentation;
no Task was created on this machine to test it. A bare-metal Ubuntu or Pop!_OS install has the
same `.bashrc`/`.profile` logic; WSL only adds the Windows PATH translation and `WSLENV` layer.

## Sources

- Bash startup files: <https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html>
- Bash invocation: <https://www.gnu.org/software/bash/manual/html_node/Invoking-Bash.html>
- Ubuntu `.bashrc`: <https://git.launchpad.net/ubuntu/+source/bash/plain/debian/skel.bashrc>
- Ubuntu `.profile`: <https://git.launchpad.net/ubuntu/+source/base-files/plain/share/dot.profile>
- Debian/Ubuntu crontab(5): <https://manpages.debian.org/bookworm/cron/crontab.5.en.html>
- Debian/Ubuntu cron(8): <https://manpages.debian.org/bookworm/cron/cron.8.en.html>
- Debian cron source, PATH: <https://sources.debian.org/src/cron/3.0pl1-162/entry.c/>
- Debian cron source, default path: <https://sources.debian.org/src/cron/3.0pl1-162/pathnames.h/>
- Debian cron env build: <https://sources.debian.org/src/cron/3.0pl1-162/do_command.c/>
- cronie default path: <https://github.com/cronie-crond/cronie/blob/master/src/pathnames.h>
- dash(1): <https://manpages.debian.org/bookworm/dash/dash.1.en.html>
- zsh startup files: <https://zsh.sourceforge.io/Doc/Release/Files.html>
- nvm README: <https://github.com/nvm-sh/nvm/blob/master/README.md>
- nvm installer: <https://raw.githubusercontent.com/nvm-sh/nvm/v0.41.0/install.sh>
- Bun installation: <https://bun.com/docs/installation>
- Bun installer: <https://bun.com/install>
- OpenCode installer: <https://opencode.ai/install>
- OpenCode providers and credentials: <https://opencode.ai/docs/providers/>
- OpenCode Windows/WSL: <https://opencode.ai/docs/windows-wsl/>
- PowerShell about_Profiles:
  <https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_profiles>
- PowerShell about_Pwsh:
  <https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_pwsh>
- Windows PowerShell about_PowerShell_exe:
  https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_powershell_exe
- Task Scheduler ExecAction:
  <https://learn.microsoft.com/windows/win32/taskschd/execaction>
- Task Scheduler actions:
  <https://learn.microsoft.com/windows/win32/taskschd/task-actions>
- cmd: <https://learn.microsoft.com/windows-server/administration/windows-commands/cmd>
- setx: <https://learn.microsoft.com/windows-server/administration/windows-commands/setx>
- Win32 environment variables:
  <https://learn.microsoft.com/windows/win32/procthread/environment-variables>
- WSL configuration: <https://learn.microsoft.com/windows/wsl/wsl-config>
- WSLENV: <https://learn.microsoft.com/windows/wsl/filesystems>
- WSLENV format: [Microsoft devblog][wslenv-format]

[profiles]:
https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_profiles
[pwsh-doc]:
https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_pwsh
[powershell-exe-doc]:
https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_powershell_exe
[wslenv-format]:
https://devblogs.microsoft.com/commandline/share-environment-vars-between-wsl-and-windows/
