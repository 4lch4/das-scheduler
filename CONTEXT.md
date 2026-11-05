# das-scheduler

A small desktop app for creating and managing recurring shell commands on Windows and Linux, using
each OS's own scheduler.

## Language

**Job**: A recurring unit of work created in das-scheduler: one **Command** paired with one
**Schedule**. _Avoid_: task, entry, cron job

**Command**: The shell command a **Job** executes on each **Run**, such as an `opencode run`
invocation. _Avoid_: action, script

**Schedule**: When a **Job** runs: either a **Preset** or a custom cron expression. _Avoid_: trigger,
timing

**Preset**: One of the named **Schedule** shortcuts: hourly, daily, weekly or monthly. _Avoid_:
frequency, interval

**Backend**: The OS scheduler that actually fires a **Job**: Windows Task Scheduler or the user's
crontab on Linux. _Avoid_: engine, provider

**Run**: One execution of a **Job**, with a start time, an exit status and captured output. _Avoid_:
execution, invocation

**Task**: Reserved for an entry in Windows Task Scheduler, never for a **Job**.
