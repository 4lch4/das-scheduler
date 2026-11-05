# Go backends and SQLite for das-scheduler

Research for [issue 7](https://github.com/4lch4/das-scheduler/issues/7), part of the
[das-scheduler v1 spec](https://github.com/4lch4/das-scheduler/issues/3). Researched 2026-11-05
against primary sources: Microsoft Learn, the Vixie cron and cronie sources, the SQLite
documentation and the library repositories. Bracketed numbers point at [Sources](#sources).

## Answer

**Windows Task Scheduler: build the Task XML in Go and call the COM API through
`github.com/go-ole/go-ole`.** `ITaskFolder::RegisterTask` takes a whole task definition as an XML
string, so the app needs only a handful of COM calls: `GetFolder`/`CreateFolder`, `RegisterTask`,
`DeleteTask`, `Run`, `put_Enabled` and `get_Xml`. `go-ole` is MIT, cgo-free and maintained. Keep
`schtasks /create /xml` as the documented fallback. Do not depend on `capnspacehook/taskmaster`: it
is MIT, but has no tagged release and no commit since 2021, its README calls the API unstable, and
it has neither an enable/disable call nor an XML export, so it cannot serve Drift.

**crontab: use the `crontab(1)` command, not a Go library.** Read with `crontab -l`, write the whole
file back on the standard input of `crontab -`, keep every other line byte-for-byte, identify each
Job with a marker comment line directly above its entry, end the file with a newline, and serialise
read-modify-write with a lock file. Vixie cron validates the file and renames it into place, so a
rejected edit leaves the previous crontab installed.

**SQLite: `modernc.org/sqlite`.** It is a cgo-free build of SQLite (BSD-3-Clause for the Go code,
public domain for SQLite itself), released regularly, and it documents windows/amd64, windows/arm64
and windows/386 alongside the Linux targets, so cross-compiling from Linux works with
`CGO_ENABLED=0`. WAL does handle one GUI plus several `run` processes: readers never block the
writer, writes serialise, and every process must set `busy_timeout` and open write transactions with
`BEGIN IMMEDIATE`.

At a glance:

| Area | Use | Version | Licence | Status |
| --- | --- | --- | --- | --- |
| Task Scheduler | Task XML + COM via `go-ole` | v1.3.0 | MIT | maintained |
| crontab | `crontab -l` / `crontab -` | OS `crontab(1)` | n/a | OS component |
| SQLite | `modernc.org/sqlite` | v1.61.1 | BSD-3-Clause | active |

## Windows Task Scheduler

### Option 1: COM through go-ole (recommended)

Task Scheduler exposes a COM automation server under the program ID `Schedule.Service` [42]
(`Schedule.Service.1` in the COM class table [23]). `ITaskService::Connect` runs on the local
computer when its server argument is empty and uses the current token when no user is given [8].
`GetFolder` returns an `ITaskFolder`; the root is `\` and a subfolder is `\das-scheduler` [9].
`CreateFolder` creates a missing folder, including a whole missing tree [11].

`ITaskFolder::RegisterTask` "registers (creates) a new task in the folder using XML to define the
task" [10]. Its arguments are the task name, the XML text, a `TASK_CREATION` flag, user, password,
logon type and SDDL [10]. The flags that matter are `TASK_CREATE_OR_UPDATE` (0x6) for apply, and
`TASK_VALIDATE_ONLY` (0x1) for a dry run [10]. `TASK_LOGON_INTERACTIVE_TOKEN` (3) means "User must
already be logged on. The task will be run only in an existing interactive session." [17], which is
exactly the v1 decision to store no password.

The rest of the surface is small:

| Operation | COM call |
| --- | --- |
| Read a task | `ITaskFolder::GetTask` [13] |
| Read its definition | `IRegisteredTask::get_Xml` [14] |
| Enable or disable | `IRegisteredTask::put_Enabled` [15] |
| Run now | `IRegisteredTask::Run`, with `VT_NULL` when there are no parameters [16] |
| Delete | `ITaskFolder::DeleteTask` [12] |

None of that needs the `ITaskDefinition` object graph: the XML is the whole definition, which is why
`RegisterTask` is enough.

`go-ole` is "Go bindings for Windows COM using shared libraries instead of cgo" [21]. It is MIT
[21], tagged v1.3.0 on 2023-08-15 [22], with commits through 2026-10-12 [21], and it has non-Windows
stub files, so a Linux machine can cross-compile the Windows binary. COM is apartment-threaded: the
README says to lock the goroutine to an OS thread and to call `CoUninitialize` for every
`CoInitialize` [21]. Give the backend one dedicated goroutine, lock it with `runtime.LockOSThread`,
call `CoInitializeEx(0, COINIT_APARTMENTTHREADED)` once, and send requests to it over a channel.

### Option 2: a dedicated library

`capnspacehook/taskmaster` is the main Go library that covers Task Scheduler [23]. It wraps the same
COM API and offers `Connect`, `CreateTask`, `UpdateTask`, `DeleteTask`, `Run` and `Stop` [23]. But:

- The README says "the API is not stable ... pin a specific commit" [23].
- It has no tagged release; the module resolves to `v0.0.0-20210519235353-1629df7c85e9`, and the
  last commit is 2021-05-19 [23].
- It registers through `RegisterTaskDefinition` with a COM `ITaskDefinition`, not through
  `RegisterTask` with XML, and its `XMLText` fields are declared but never populated [23].
- `RegisteredTask` has no enabled setter [23].

It is a good reference for the COM call shapes and a poor dependency for the job.

### Option 3: schtasks with XML

`schtasks /create /tn "\das-scheduler\<id>" /xml <file> /f` creates a task from an XML file and
overwrites an existing one; `/tn` accepts `folder\task` [1]. `/query /tn <name> /xml` prints a task
definition as XML [2]; `/change /tn <name> /ENABLE` or `/DISABLE` toggles it [3]; `/run` starts it
[5]; `/delete /tn <name> /f` removes it [4]. This is fully documented and needs no COM, but every
operation is a child process (a console flash in a GUI app, which the separate console-flash
research covers), and the XML has to travel through a temp file. Keep it as the break-glass path and
as the thing a user can run by hand.

### Option 4: PowerShell

The `ScheduledTasks` module has `Register-ScheduledTask` with an `-Xml` parameter set,
`Get-ScheduledTask`, `Export-ScheduledTask`, `Enable-ScheduledTask`, `Disable-ScheduledTask`,
`Start-ScheduledTask` and `Unregister-ScheduledTask` [19][20]. Each call is a child PowerShell
process, and the XML has to survive command-line quoting. It is the slowest option and adds nothing
over COM for this app.

### The task XML

The schema namespace is `http://schemas.microsoft.com/windows/2004/02/mit/task` [6]. A task that
runs daily at a fixed time, in the user's interactive session, and catches up after a missed start:

```xml
<?xml version="1.0"?>
<Task xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <Description>das-scheduler job 01J8Z6Q3K4</Description>
  </RegistrationInfo>
  <Triggers>
    <CalendarTrigger>
      <StartBoundary>2026-11-05T03:17:00</StartBoundary>
      <ScheduleByDay><DaysInterval>1</DaysInterval></ScheduleByDay>
    </CalendarTrigger>
  </Triggers>
  <Principals>
    <Principal>
      <UserId>BENICIA\dleam</UserId>
      <LogonType>InteractiveToken</LogonType>
    </Principal>
  </Principals>
  <Settings>
    <Enabled>true</Enabled>
    <StartWhenAvailable>true</StartWhenAvailable>
    <AllowStartOnDemand>true</AllowStartOnDemand>
  </Settings>
  <Actions>
    <Exec>
      <Command>C:\Users\dleam\AppData\Local\das-scheduler\das-scheduler.exe</Command>
      <Arguments>run 01J8Z6Q3K4</Arguments>
    </Exec>
  </Actions>
</Task>
```

`StartBoundary` is required for a calendar trigger [7]. `LogonType` `InteractiveToken` in the
`Principal` is the XML spelling of the no-password logon type [7][17]. `StartWhenAvailable` is the
"run as soon as possible after a missed start" setting, and it applies only to timed tasks [18]. The
app writes its own absolute executable path into `Command`.

### Drift

Read the registered definition with `get_Xml` [14] and compare the fields the app owns (command,
arguments, trigger, enabled) after parsing. Do not compare raw bytes: the call returns the
registration information, which includes fields the app never set. A `GetTask` that returns a
not-found HRESULT is Drift with the entry missing [13].

## crontab

### Reading

`crontab -l` prints the current user's crontab on stdout [24]. With no crontab installed, Vixie
cron's `list_cmd` prints `no crontab for <user>` on stderr and exits with `ERROR_EXIT`, which is `1`
[26][27]. Treat that exact case as an empty crontab, not as a failure.

On Debian and Ubuntu, `-l` strips the three-line "DO NOT EDIT THIS FILE" header by default, which is
what makes `crontab -l | crontab -` idempotent [24]. Ubuntu ships the Debian `cron` package, Vixie
cron 3.0pl1 [28], and Pop!_OS inherits it.

### Writing

Never edit `/var/spool/cron/crontabs/<user>` directly: the man page says users may not, and that the
setgid `crontab` command exists to validate syntax [24]. Write the complete new file to the standard
input of `crontab -`. In the source, the replace path copies stdin to a temporary file in the spool
directory, runs the syntax check, then renames the temp file over the real crontab [26]. A syntax
error prints `errors in crontab file, can't install.` and leaves the old file untouched [26]. The
rename makes the swap atomic for cron's reader.

Two formatting rules the writer must honour:

- Every line, including the last, must end in a newline; cron refuses a crontab whose last entry
  lacks one [24].
- `%` is special in a command: unless escaped as `\%`, cron turns it into a newline and feeds the
  rest to the command as standard input [25]. The wrapper line (`das-scheduler run <id>`) has no
  `%`, so this only matters if a command is ever written inline.

### Preserving everything else

Read the whole file, split on newlines, drop only the marker block for the Job being changed, and
re-join. Keep blank lines, comments, `SHELL`/`MAILTO`/`PATH` assignments and the user's own entries
exactly as they were. Do not round-trip through a cron parser: parsers normalise fields and lose the
user's formatting.

### Marker convention

A comment is a line whose first non-space character is `#`; comments are not allowed on the same
line as a command, because the rest of the line is taken as part of the command [25]. The marker
therefore needs its own line, directly above the entry:

```cron
# das-scheduler:01J8Z6Q3K4
17 3 * * * /home/dleam/.local/bin/das-scheduler run 01J8Z6Q3K4
```

Rules that follow:

- The prefix is exactly `# das-scheduler:`; the rest is the Job id. Ids must not contain whitespace.
- Exactly one marker line per Job; duplicates are Drift.
- Drift detection: find the marker, then require the next non-comment line to equal the expected
  entry. Marker with a different entry is "changed"; marker with no entry, or an entry whose marker
  is gone, is "missing".
- An entry without a marker is not ours. Leave it alone; importing is out of scope.

### Serialising writes

`crontab -l` followed by `crontab -` is a read-modify-write with a gap. Take an exclusive `flock` on
a lock file in the app data directory around the pair, so the GUI and every `run` process queue up.
A user running `crontab -e` bypasses the lock; because that write is also a whole-file replace,
re-read after writing and surface the conflict instead of assuming success.

`crontab -n` on Debian and Ubuntu is a dry run that checks syntax without writing [24]; it is a
cheap pre-flight before the real install.

### No library

Go cron packages are in-process schedulers: `robfig/cron` is "a cron spec parser and job
runner" [39], and `gronx` and `mileusna/crontab` are the same shape. None of them reads or writes
the system crontab. Use one for parsing and next-run preview (a separate ticket), not for
installation.

## SQLite

### Driver

`modernc.org/sqlite` is "a sql/database driver using a CGo-free port of the C SQLite3 library" [34].
It registers the driver name `sqlite` [34]. The current release is v1.61.1, published 2026-10-31
[36]. The package documents SQLite 3.54.1 and supported targets that include windows/386,
windows/amd64, windows/arm64, linux/386, linux/amd64, linux/arm64, darwin/amd64 and darwin/arm64
[34]. Its Go code is BSD-3-Clause and the embedded SQLite is public domain [34][33].

Alternatives:

- `github.com/ncruces/go-sqlite3` compiles SQLite to Wasm and translates it with `wasm2go`; MIT,
  v0.36.0 (2026-10-29), active but still pre-1.0 [37].
- `github.com/mattn/go-sqlite3` is the long-standing cgo driver; MIT, v1.14.52 (2026-09-05) [38]. It
  needs a C toolchain and `CGO_ENABLED=1`, so a Linux-to-Windows build needs mingw.

Pick modernc: the mature, actively released pure-Go option with a plain `database/sql` API.

### Cross-compiling

cgo is "disabled by default when cross-compiling" [40], so a pure-Go driver is what makes
`GOOS=windows GOARCH=amd64 go build` work from Linux without a C cross toolchain. `go-ole` is
cgo-free too [21].

### WAL with a GUI and several run processes

WAL allows many readers and one writer at the same time: "since there is only one WAL file, there
can only be one writer at a time" [29]. All processes must be on the same host; WAL does not work
over a network filesystem [29]. The database lives in the per-user data directory on local disk, so
that holds. `PRAGMA journal_mode=WAL` is persistent, staying in effect across connections and
reopens [30].

That matches this app: the GUI reads lists and writes settings, and each `run` process inserts a Run
row and updates it at exit. Writes are short and rare, so they serialise cleanly. Three settings
make contention painless:

- `PRAGMA busy_timeout=5000` installs a busy handler, so a writer waits instead of failing at once
  with `SQLITE_BUSY` [30][32].
- `BEGIN IMMEDIATE` takes the write lock at the start of the transaction; a deferred transaction
  that upgrades later can fail with `SQLITE_BUSY` once another connection has written [31].
- `PRAGMA synchronous=NORMAL` is "safe from corruption" in WAL and the documented balance for most
  applications in WAL mode [30].

The DSN, using the syntax the driver documents [35]:

```go
dsn := "file:" + dbPath +
    "?_pragma=busy_timeout(5000)" +
    "&_pragma=journal_mode(WAL)" +
    "&_pragma=foreign_keys(1)" +
    "&_txlock=immediate"
```

`_pragma` values run as SQL text on every new connection, and `_txlock=immediate` makes
`database/sql` transactions start with `BEGIN IMMEDIATE` [35]. Keep the DSN a compile-time constant;
if it could ever come from outside, call `sqlite.StrictPragmas(true)` first [35].

Bound the pool as well. `database/sql` defaults to unlimited connections [41], and the driver's own
docs warn that every connection carries its own page cache and that slow periodic queries can pile
up [34]. A single writer connection plus a small read pool is plenty.

WAL adds a `-wal` and a `-shm` file beside the database. Opening the database then needs write
access to the `-shm` file, or to the directory when that file does not exist yet [29].

### Caveats

- Keep `modernc.org/libc` at the exact version in `modernc.org/sqlite`'s `go.mod`; the package
  documents that dependency as fragile [34].
- The transpiled SQLite is slower than C on CPU-bound work, by roughly 1.3x to 2.0x on the workloads
  the package documents [34]. For a table of Jobs and Runs that is irrelevant.

## Sources

All accessed 2026-11-05. Long URLs use the locale-less form, which redirects to `en-us`.

1. Microsoft Learn, "schtasks create".
   <https://learn.microsoft.com/windows-server/administration/windows-commands/schtasks-create>
2. Microsoft Learn, "schtasks query".
   <https://learn.microsoft.com/windows-server/administration/windows-commands/schtasks-query>
3. Microsoft Learn, "schtasks change".
   <https://learn.microsoft.com/windows-server/administration/windows-commands/schtasks-change>
4. Microsoft Learn, "schtasks delete".
   <https://learn.microsoft.com/windows-server/administration/windows-commands/schtasks-delete>
5. Microsoft Learn, "schtasks run".
   <https://learn.microsoft.com/windows-server/administration/windows-commands/schtasks-run>
6. Microsoft Learn, "Task Scheduler Schema".
   <https://learn.microsoft.com/windows/win32/taskschd/task-scheduler-schema>
7. Microsoft Learn, "Daily Trigger Example (XML)".
   <https://learn.microsoft.com/windows/win32/taskschd/daily-trigger-example--xml->
8. Microsoft Learn, "ITaskService::Connect".
   <https://learn.microsoft.com/windows/win32/api/taskschd/nf-taskschd-itaskservice-connect>
9. Microsoft Learn, "ITaskService::GetFolder".
   <https://learn.microsoft.com/windows/win32/api/taskschd/nf-taskschd-itaskservice-getfolder>
10. Microsoft Learn, "ITaskFolder::RegisterTask".
    <https://learn.microsoft.com/windows/win32/api/taskschd/nf-taskschd-itaskfolder-registertask>
11. Microsoft Learn, "ITaskFolder::CreateFolder".
    <https://learn.microsoft.com/windows/win32/api/taskschd/nf-taskschd-itaskfolder-createfolder>
12. Microsoft Learn, "ITaskFolder::DeleteTask".
    <https://learn.microsoft.com/windows/win32/api/taskschd/nf-taskschd-itaskfolder-deletetask>
13. Microsoft Learn, "ITaskFolder::GetTask".
    <https://learn.microsoft.com/windows/win32/api/taskschd/nf-taskschd-itaskfolder-gettask>
14. Microsoft Learn, "IRegisteredTask::get_Xml".
    <https://learn.microsoft.com/windows/win32/api/taskschd/nf-taskschd-iregisteredtask-get_xml>
15. Microsoft Learn, "IRegisteredTask::put_Enabled".
    <https://learn.microsoft.com/windows/win32/api/taskschd/nf-taskschd-iregisteredtask-put_enabled>
16. Microsoft Learn, "IRegisteredTask::Run".
    <https://learn.microsoft.com/windows/win32/api/taskschd/nf-taskschd-iregisteredtask-run>
17. Microsoft Learn, "TASK_LOGON_TYPE enumeration".
    <https://learn.microsoft.com/windows/win32/api/taskschd/ne-taskschd-task_logon_type>
18. Microsoft Learn, "StartWhenAvailable (settingsType) Element".
    <https://learn.microsoft.com/windows/win32/taskschd/taskschedulerschema-startwhenavailable-settingstype-element>
19. Microsoft Learn, "Register-ScheduledTask".
    <https://learn.microsoft.com/powershell/module/scheduledtasks/register-scheduledtask>
20. Microsoft Learn, "ScheduledTasks Module".
    <https://learn.microsoft.com/powershell/module/scheduledtasks/>
21. go-ole, README and LICENSE (MIT), github.com/go-ole/go-ole. <https://github.com/go-ole/go-ole>
22. Go module proxy, versions of github.com/go-ole/go-ole.
    <https://proxy.golang.org/github.com/go-ole/go-ole/@v/list>
23. capnspacehook/taskmaster, README, source and commit history (MIT).
    <https://github.com/capnspacehook/taskmaster>
24. Debian manpage, crontab(1), cron 3.0pl1-162 (bookworm).
    <https://manpages.debian.org/bookworm/cron/crontab.1.en.html>
25. Debian manpage, crontab(5), cron 3.0pl1-162 (bookworm).
    <https://manpages.debian.org/bookworm/cron/crontab.5.en.html>
26. Vixie cron source: `crontab.c` (`list_cmd`, `replace_cmd`) and `macros.h` (`ERROR_EXIT`).
    <https://github.com/vixie/cron>
27. cronie source: `src/crontab.c` (`list_cmd`) and `src/macros.h` (`ERROR_EXIT`).
    <https://github.com/cronie-crond/cronie>
28. Ubuntu package index, `cron` 3.0pl1-184ubuntu2 in noble.
    <https://packages.ubuntu.com/noble/cron>
29. SQLite, "Write-Ahead Logging". <https://sqlite.org/wal.html>
30. SQLite, "PRAGMA" (`journal_mode`, `busy_timeout`, `synchronous`).
    <https://sqlite.org/pragma.html>
31. SQLite, "Transactions" (`BEGIN IMMEDIATE`, deferred upgrade).
    <https://sqlite.org/lang_transaction.html>
32. SQLite, "Result and Error Codes" (`SQLITE_BUSY`). <https://sqlite.org/rescode.html>
33. SQLite, "SQLite Is Public Domain". <https://sqlite.org/copyright.html>
34. modernc.org/sqlite, package documentation (`doc.go`) and README.
    <https://gitlab.com/cznic/sqlite/-/blob/master/doc.go>
35. modernc.org/sqlite, `driver.go` DSN documentation and `sqlite.go` parameter parsing.
    <https://gitlab.com/cznic/sqlite/-/blob/master/driver.go>
36. Go module proxy, versions of modernc.org/sqlite.
    <https://proxy.golang.org/modernc.org/sqlite/@v/list>
37. ncruces/go-sqlite3, README (MIT). <https://github.com/ncruces/go-sqlite3>
38. mattn/go-sqlite3, repository and releases (MIT). <https://github.com/mattn/go-sqlite3>
39. robfig/cron, README and package documentation (MIT). <https://github.com/robfig/cron>
40. Go documentation, `cmd/cgo`, "Cgo is disabled by default when cross-compiling".
    <https://pkg.go.dev/cmd/cgo>
41. Go documentation, `database/sql`, `DB.SetMaxOpenConns`.
    <https://pkg.go.dev/database/sql#DB.SetMaxOpenConns>
42. Microsoft Learn, "Time Trigger Example (Scripting)", `CreateObject("Schedule.Service")`.
    <https://learn.microsoft.com/windows/win32/taskschd/time-trigger-example--scripting->
