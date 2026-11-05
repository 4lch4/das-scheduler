# Cron expressions that translate exactly to Windows Task Scheduler triggers

Research for [das-scheduler #5](https://github.com/4lch4/das-scheduler/issues/5). Product
constraints come from the [v1 spec map](https://github.com/4lch4/das-scheduler/issues/3).

## Answer

**Recommendation:** accept the four Presets and a computed subset of custom 5-field cron.
Translate by parsing every field into a set and emitting one or more Task Scheduler triggers;
reject only forms that are provably not representable, or whose exact plan exceeds Task
Scheduler's 48-trigger limit.

- **All four Presets translate exactly, one trigger each.** Hourly at minute `m` is a
  `TimeTrigger` at `00:m` with `Repetition/Interval=PT1H` and no `Duration`. Daily at `HH:MM`
  is a `DailyTrigger` with `DaysInterval=1`. Weekdays at `HH:MM` is a `WeeklyTrigger` with
  `WeeksInterval=1` and Monday through Friday. Monthly on day `d` is a `MonthlyTrigger` with
  `DaysOfMonth={d}` and all twelve months. [7][10][13][14][15]
- **Most custom forms translate exactly, often with one or two triggers.** Fixed times,
  minute and hour steps that divide their cycle, contiguous ranges, lists, month, day-of-month
  and day-of-week sets, and day-of-month OR day-of-week (two triggers) all have exact
  mappings. Forms that need more triggers still translate if the plan fits in 48 triggers. [7]
- **Two families do not translate.** (1) A stepped day field that starts with `*` combined
  with an explicit other day field, e.g. `0 0 */2 * 1`: Vixie and cronie AND the two day
  bitmaps, and Task Scheduler has no trigger that intersects day-of-month with day-of-week.
  (2) Any expression whose exact trigger plan needs more than 48 triggers. [3][5][7]
- **"Last day of month" is not a 5-field cron expression.** `28-31` is four fixed dates, not
  the last day; Windows can express the real thing with `RunOnLastDayOfMonth`, but the v1
  decision to store every Schedule as a cron string cannot represent it. The same applies to
  "first Monday": Windows `MonthlyDOWTrigger` can, cron cannot. [17][18][22]
- **Cron's OR rule is Windows-friendly, but overlaps are not free.** When both day fields are
  explicit, cron runs on either; two triggers reproduce the union. On dates where both match,
  Windows receives two start requests, so the run wrapper should dedupe by scheduled minute.
  [2][3]
- **DST is the one unavoidable semantic gap.** On the spring-forward day Task Scheduler moves
  a time in the skipped hour to 03:00, while cron skips it; cron also runs twice in the
  repeated fall-back hour. Exactness holds for wall-clock times outside the transition hour.
  [1][15]

For v1, use the platform's 48-trigger limit as the hard boundary and a lower soft cap (say 8)
for the custom editor; label rejections above the soft cap "not supported in v1", not
"impossible". A translator that emits a plan plus a next-N-runs comparison test is safer than
a hand-written allowlist.

## Scope and source semantics

This note is about schedule shape only. The wrapper, privileges, logging and missed-run policy
are settled in the map [22].

The Linux side is the user's crontab. The target is Ubuntu or Pop!_OS; the research machine's
installed `cron` package is 3.0pl1-200ubuntu1 (Vixie cron). The core grammar is the same in
cronie, and its source uses the same first-character star rule, although its manual wording is
looser. Where the descriptions differ, this note follows the installed Vixie code, because that
is what will execute the line. Verify the daemon on the target image; the open questions list
this. [1][2][3][5]

## What a 5-field cron expression means

The five fields are minute (0-59), hour (0-23), day of month (1-31), month (1-12, names
allowed) and day of week (0-7, where both 0 and 7 are Sunday, names allowed). A field may be
`*`, a range `a-b`, a list `a,b,c`, a range list `0-4,8-12`, or a step `a-b/n` or `*/n`. Steps
are evaluated inside their own field, so `*/35` in the minute field means minutes 0 and 35 of
each hour, not "every 35 minutes", and `*/23` in the hour field means hours 0 and 23, not
"every 23 hours". [1][2]

Names are aliases for numbers; cronie also allows names in lists and ranges, while Vixie's
manual says lists and ranges of names are not allowed. The app should either accept only what
the target daemon accepts, or normalise names to numbers before writing the crontab. [1][2]

### The day-of-month / day-of-week rule

The manual rule is: if both day fields are restricted (do not contain `*`), the command runs
when **either** field matches. For example, `30 4 1,15 * 5` runs at 04:30 on the 1st, the 15th,
and every Friday. [2]

The implementation is subtler. Vixie's parser sets a "star" flag whenever the field's **first
character** is `*`, even when the field is a step such as `*/2`. If either star flag is set,
the matcher ANDs the two day bitmaps; only when neither flag is set does it OR them. So a
stepped field is not "unrestricted": it contributes its actual bit set, and the other field
filters it. [3][4][5][6]

POSIX would treat `*/2` as a restricted field (not a bare `*`) and OR it with the other day
field. Ubuntu's own manual calls out the difference: `0 0 */2 * sun` runs "every Sunday that's
an uneven date" in this implementation, while POSIX "would have it run every Sunday and on
every uneven date". [23] Because the app writes a crontab that Vixie cron will execute on
Ubuntu, it must implement the first-character rule, not the POSIX wording. On a host running
systemd-cron or another POSIX-conforming implementation, the same line means something else,
which is a reason to reject this ambiguous form at the editor rather than rely on the daemon.

Practical consequences under the installed Vixie implementation. In the table, "set" means an
explicit list or range, or a value that does not start with `*`; `*/n` means a stepped field
that starts with `*`.

| Day of month | Day of week | Effective meaning | Windows plan |
| --- | --- | --- | --- |
| `*` | set | only the day-of-week set | one weekly trigger |
| set | `*` | only the day-of-month set | one monthly trigger |
| set | set | union of both sets | two triggers |
| `*/n` | `*` | the stepped day-of-month set | one monthly trigger |
| `*` | `*/n` | the stepped day-of-week set | one weekly trigger |
| `*/n` | set | intersection of the two sets | not expressible |
| set | `*/n` | intersection of the two sets | not expressible |
| `*/n` | `*/n` | intersection of the two sets | not expressible |

The intersection rows are the ones the ticket calls "day-of-month combined with day-of-week".
They are not the common OR form, and no Task Scheduler trigger can express them.

`*/2` in the day-of-month field is worth spelling out: it means days 1, 3, 5, ..., 31 of each
month, not "every second day across month boundaries". [23]

## What a Task Scheduler trigger can express

Task Scheduler 2.0 has five relevant trigger shapes. All are represented by `CalendarTrigger`
in XML except the one-shot `TimeTrigger`. [7][10]

- **TimeTrigger**: starts the task at one specific date and time. [10][11]
- **DailyTrigger** (`ScheduleByDay`): time of day plus `DaysInterval` 1-365. [10][13]
- **WeeklyTrigger** (`ScheduleByWeek`): time of day, `WeeksInterval` 1-52, day-of-week set.
  [10][14]
- **MonthlyTrigger** (`ScheduleByMonth`): time of day, day-of-month set (1-31 and `Last`),
  month set. [10][15][16]
- **MonthlyDOWTrigger** (`ScheduleByMonthDayOfWeek`): time of day, weeks of month (1-4 and
  `Last`), day-of-week set, month set. [10][18][19]

Every time or calendar trigger has a required `StartBoundary` and an optional `Repetition`
block. `Interval` is at least `PT1M` and at most `P31D`; `Duration` is optional, at least
`PT1M`, and if omitted the pattern repeats indefinitely. A task may hold up to 48 triggers, and
each trigger independently starts the task. [7][8][9][12][21]

The structural consequence is the key to the whole mapping: one trigger describes a **date
set** times a **time set**, where the time set is an arithmetic progression from the start time
with a fixed interval, bounded by the repetition window. A cron expression is translatable iff
its times can be partitioned into at most 48 such progressions, each attached to an expressible
date set.

Task Scheduler's date vocabulary covers:

- every day, every N days (daily);
- every N weeks on a day-of-week set (weekly);
- fixed days of selected months, including the last day (monthly);
- the Nth or last week of selected months on a day-of-week set (monthly DOW).

It does **not** cover an intersection of a fixed day-of-month set with a day-of-week set. It
also cannot restrict a weekly trigger to a subset of months; use a monthly DOW trigger for
that. [7][18][19]

## Translation rules

Notation: `m` is a single minute, `h` a single hour, `d` a single day of month, `M` the minute
set, `H` the hour set, `D` the day-of-month set, `Mo` the month set, `W` the day-of-week set.
A "run" is a maximal set of equally spaced values within one hour, e.g. `0,20,40` or
`10-20/2`.

### Presets

| Cron | Task Scheduler | Triggers |
| --- | --- | --- |
| `m * * * *` | `TimeTrigger` at `00:m`, `Interval=PT1H`, no `Duration` | 1 |
| `m h * * *` | `DailyTrigger`, `DaysInterval=1`, start `h:m` | 1 |
| `m h * * 1-5` | `WeeklyTrigger`, `WeeksInterval=1`, Monday-Friday, start `h:m` | 1 |
| `m h d * *` | `MonthlyTrigger`, day `d`, all months, start `h:m` | 1 |

The hourly preset can also be a daily trigger at `00:m` with `Interval=PT1H` and
`Duration=PT23H59M`; the `TimeTrigger` form is simpler and is the standard "repeat every hour"
shape. The type is one-shot, but the repetition block carries the hourly cadence and does not
expire when `Duration` is omitted. The daily trigger example in the Task Scheduler documentation
shows a daily trigger with a repetition pattern, which is the same mechanism. [9][11][20]

For the monthly preset, `d` is a fixed day. Days 29-31 exist only in some months, so a job on
day 31 has no run in February, April, June, September or November. That is exactly what cron's
`31` does, but it is not "last day"; `RunOnLastDayOfMonth` is the Windows way to say last day.
[16][17]

### Step values

The minute-step cases below assume `H=*` unless stated otherwise; the hour-step cases follow.

**Minute step `*/g`.** If `g` divides 60, the minute set is a full coset of the hour and one
`TimeTrigger` at `00:00` with `Interval=PT{g}M` and no `Duration` covers it exactly. This
includes `*` (`g=1`), `*/2`, `*/3`, `*/4`, `*/5`, `*/6`, `*/10`, `*/12`, `*/15`, `*/20` and
`*/30`. [7][9][11]

If `g` does not divide 60, the set does not wrap seamlessly: `*/7` is minutes 0, 7, ..., 56 and
then 0 again, so the last gap is 4 minutes, not 7. A continuous 7-minute repetition would drift.
The exact construction is one `TimeTrigger` per selected minute, each at `00:m` with
`Interval=PT1H`: `*/7` needs 9 triggers. That is exact and stays well under 48. [1][11]

The same rule covers lists. `0,20,40` is a full coset with `g=20`, so it is one trigger. `5,35`
is a full coset with `g=30`, so it is one trigger. `1,2` is not a single progression covering
the hour; it is two hourly triggers at `00:01` and `00:02`.

**Hour step `*/g`.** If `g` divides 24, the hour set is a full coset of the day. With a single
minute `m`, one `TimeTrigger` at `00:m` with `Interval=PT{g}H` covers it. If `g` does not divide
24, use one daily trigger per selected hour: `0 */5 * * *` is hours 0, 5, 10, 15 and 20, so it
is five `DailyTrigger`s at `00:00`, `05:00`, `10:00`, `15:00` and `20:00`. A continuous
5-hour repetition would drift at midnight. [7][13]

**Day-of-month step `*/g`.** This is a fixed day set and is exact as a `MonthlyTrigger` day
bitmask, provided the day-of-week field is bare `*` so the star/AND rule reduces to the
day-of-month set. `0 0 */2 * *` is days 1, 3, ..., 31, one monthly trigger. Combined with an
explicit day-of-week field, it becomes the unsupported intersection above. [5][15][16]

**Month step `*/g`.** Months are a bitmask in `MonthlyTrigger`/`MonthlyDOWTrigger`; any subset
is exact. `0 0 1 */3 *` is January, April, July and October on day 1, one monthly trigger. [7]

**Day-of-week step `*/g`.** This is a day-of-week set: `*/2` is Sunday, Tuesday, Thursday and
Saturday. With a bare day-of-month field it is one weekly trigger; with an explicit
day-of-month field it is the unsupported intersection above. [5][14]

### Ranges and lists

**Minute range `a-b`, every hour.** A range is a run, not a full coset unless it covers all 60
minutes. With `H=*`, use one hourly trigger per minute value (a `TimeTrigger` at `00:m` with
`Interval=PT1H`); `10-20 * * * *` needs 11 triggers. Alternatively use 24 daily triggers, one
per hour, each repeating every minute for the length of the run; that is 24 triggers and is
cheaper only when the run is longer than 24 minutes. [11]

**Minute range with a fixed hour set.** If `H` is a single hour, one daily trigger at that hour
with a repetition interval equal to the run step and a duration ending 59 seconds after the
last wanted minute is exact. For example, `10-20/2 9 * * *` is one daily trigger at 09:10
repeating every 2 minutes through 09:20. If `H` is a contiguous range of hours, the same
continuous repetition does not restart at each hour unless the run is a full coset of the hour;
use one trigger per hour, or use the full-coset form for minute sets such as `0,30`. [9][13]

**Hour range `h1-h2` with a single minute.** One daily trigger at `h1:m`, `Interval=PT1H`,
duration ending 59 seconds after `h2:m`. `0 9-17 * * *` is one daily trigger at 09:00 for
8h00m59s, giving 09:00 through 17:00. Splitting into one daily trigger per hour is equally
exact and easier to inspect; it costs nine triggers. [13]

**Lists.** A list is a union of runs. Translate each run and sum the triggers. `0,30 9-17 * * *`
is a full 30-minute coset, so one daily trigger at 09:00 with `Interval=PT30M` and
`Duration=PT8H30M59S` covers 09:00-17:30. `0 9,17 * * *` is not a progression, so it is two
daily triggers. [11][13]

**Day-of-month range or list.** Any subset of 1-31 is a `MonthlyTrigger` bitmask. `1-7` and
`1,15,28` are both one trigger. The special value `Last` exists in the Windows schema but has
no 5-field cron syntax. [15][16]

**Month range or list.** Any subset of the twelve months is a bitmask. `3,6,9,12` and
`Jan-Mar` are both one trigger. [7]

**Day-of-week range or list.** Any subset of Sunday-Saturday is a `WeeklyTrigger` day set, or a
`MonthlyDOWTrigger` day set when months are restricted. `1-5` is Monday-Friday. [14][19]

### Multiple hours and mixed hour/minute patterns

The cheapest exact plan depends on the shape of `H` and `M`:

- `H=*` and `M` a full coset: one `TimeTrigger` with no `Duration`.
- `H=*` and arbitrary `M`: one hourly trigger per minute value (at most 60), or 24 triggers per
  run if that is fewer.
- `H` a full coset with step `g` dividing 24 and `M={m}`: one `TimeTrigger` with
  `Interval=PT{g}H`.
- `H` a contiguous range and `M` a run: one daily trigger with a repetition window.
- Otherwise: one trigger per hour per run, each with a repetition window. `0,30 9,17 * * *`
  is four daily triggers (09:00, 09:30, 17:00, 17:30) or two per-hour triggers with a
  30-minute window; either is exact.

A planner that tries these constructions and picks the smallest plan is enough for v1. The
important point is that the plan is computed from sets, not by string substitution.

### Day-of-month combined with day-of-week

When both fields are explicit, the cron meaning is the union. The Windows plan is two date
classes:

- `MonthlyTrigger` for `D` with `Mo`;
- `WeeklyTrigger` for `W` with all months, or `MonthlyDOWTrigger` for `W` with `Mo` and
  `Weeks={1,2,3,4,Last}`.

`0 0 1 * 1` is one monthly trigger for the 1st plus one weekly trigger for Monday. `0 0 1 3 1`
is the same pair restricted to March, using `MonthlyDOWTrigger` for the Monday half. Both are
exact as sets of dates. [15][18][19]

Two caveats:

1. **Overlap.** When the 1st is a Monday, both triggers request a start at 00:00. Cron's single
   line runs once; Task Scheduler has two triggers. The default `MultipleInstancesPolicy` is
   `IgnoreNew`, so the second request is ignored only while the first instance is still
   running. A fast job can race. Dedupe in the wrapper by job id plus scheduled minute, or
   accept occasional double Runs. [3][7]
2. **Unsupported intersections.** The star-step rows in the table above are ANDs, not unions.
   `0 0 */2 * 1` is "Mondays on odd dates", not "odd dates plus Mondays". There is no
   `MonthlyTrigger` or `MonthlyDOWTrigger` that intersects a day-of-month set with a
   day-of-week set, so reject these with a clear message. [5][7]

### Last day of month, first Monday, and other cron gaps

Five-field cron has no `L`, `W`, or `#` operators. The day-of-month field is a set of 1-31, and
the day-of-week field is a set of weekdays; neither can say "last day" or "first Monday". [1][2]

Windows can express both: `MonthlyTrigger` with `RunOnLastDayOfMonth`, and `MonthlyDOWTrigger`
with `Weeks=First` and the chosen weekday. They are useful presets, but they cannot be stored
as a five-field cron expression. The map's current rule that every Schedule is stored as a
cron string therefore rules them out unless the storage model gains a structured schedule or a
documented extension token. [17][18][22]

`0 0 28-31 * *` is not a substitute for last day: it runs on all four dates in a 31-day month.
`0 0 1-7 * 1` is not "first Monday": it is the union of days 1-7 and every Monday. If v1 wants
these as presets, that is a storage decision, not a translation problem.

## Worked examples

| Cron | Exact Windows plan | Triggers |
| --- | --- | --- |
| `0 * * * *` | `TimeTrigger` 00:00, `PT1H`, no duration | 1 |
| `30 2 * * *` | `DailyTrigger` 02:30, `DaysInterval=1` | 1 |
| `0 22 * * 1-5` | `WeeklyTrigger` Mon-Fri 22:00 | 1 |
| `15 14 1 * *` | `MonthlyTrigger` day 1, all months, 14:15 | 1 |
| `0 0 1,15 * *` | `MonthlyTrigger` days 1 and 15 | 1 |
| `0 0 * * 0` | `WeeklyTrigger` Sunday 00:00 | 1 |
| `0 0 1 * 1` | monthly day 1 plus weekly Monday | 2 |
| `0 0 1 3 1` | monthly March 1 plus monthly-DOW March Monday | 2 |
| `0 0 */2 * *` | `MonthlyTrigger` days 1,3,...,31 | 1 |
| `0 0 */2 * 1` | intersection, no equivalent | reject |
| `*/7 * * * *` | nine hourly `TimeTrigger`s, one per minute | 9 |
| `0 */5 * * *` | five daily triggers, hours 0,5,10,15,20 | 5 |
| `0 9-17 * * *` | one daily trigger 09:00, `PT1H`, 8h00m59s | 1 |
| `0,30 9-17 * * *` | one daily trigger 09:00, `PT30M`, 8h30m59s | 1 |
| `0 9,17 * * *` | two daily triggers | 2 |
| `* * * * *` | `TimeTrigger` 00:00, `PT1M`, no duration | 1 |
| `0 0 28-31 * *` | monthly days 28-31, not last day | 1 |
| `0 0 29 2 *` | monthly Feb 29, leap years only | 1 |

The durations in the examples assume the last wanted run must be strictly inside the window.
The documentation does not define whether a repetition fires at exactly the duration boundary,
so ending 59 seconds after the last wanted run removes the ambiguity: 59 seconds is less than
the minimum 1-minute interval and greater than zero. [9]

## What does not translate

1. **The stepped-star day intersection.** `*/n` in day-of-month with an explicit day-of-week,
   or vice versa. Vixie and cronie implement this as an AND; POSIX implements it as an OR; Task
   Scheduler can express the OR but not the AND. Because the meaning also varies between cron
   implementations, the editor should reject these forms rather than pick a side. [3][5][23]
2. **Plans that need more than 48 triggers.** The schema caps a task at 48 triggers. With the
   per-minute and per-hour constructions above, most real expressions fit; pathological minute
   sets can exceed it. Compute the plan and reject by count. [7]
3. **Non-5-field forms.** `@daily` and the other nicknames, six- or seven-field expressions,
   seconds, years, `L`, `W` and `#` are not 5-field cron. Reject them at parse time with a
   message that names the unsupported construct. [1][2]
4. **Last day / first Monday as cron input.** Not expressible in the grammar; see above.

An empty date set, such as `0 0 31 2 *`, is different: cron accepts it and it never runs.
Do not emit a trigger for it, and do not report it as a translation failure. [1][2]

Everything else in a non-empty 5-field expression is representable in principle. The hard
constraints are the 48-trigger cap and the day-field AND case.

### DST caveat

A job at 02:30 local time is not bit-identical on the spring-forward day: cron never matches
the missing 02:30, while Task Scheduler runs it at 03:00. Cron also runs a matching job twice
in the repeated fall-back hour. [1][15] This is a documented difference in the two schedulers,
not a translation bug; call it out in the UI or in the docs. It affects only schedules whose
time falls in the transition hour, so it does not change the allowlist.

## Implementation notes for v1

- Parse each field to a set; keep the original text only for display and drift detection.
- Implement the Vixie star rule explicitly: a field that starts with `*` is a star field, even
  if it is `*/n`. Do not use the manual's shorthand "contains `*`", which is ambiguous for
  steps. [3][5]
- Emit explicit values for every optional list: all twelve months, all selected days, all
  selected weekdays. Do not rely on schema defaults for the schedule shape.
- Set `StartBoundary` to the next real occurrence at the wanted clock time, not a date in the
  past, so the first activation is unambiguous.
- Use a repetition window ending 59 seconds after the last wanted run. Verify the boundary
  behaviour once on a real Windows host before freezing the emitter.
- Count triggers before registering. Hard-reject above 48; for v1, soft-reject above a small
  number (8 is a reasonable starting point) with a message that distinguishes "too complex for
  v1" from "no exact translation".
- Dedupe overlapping activations in the run wrapper by job id plus scheduled minute. Two
  triggers at the same instant are otherwise two start requests. [7]
- Test the translator by generating the next N run times from the cron sets and from the
  emitted Windows plan, and compare them. Include month ends, leap day, a 28-day February and
  both DST transitions.
- Do not support `CRON_TZ` in v1 unless the Windows side also gets a time-zone story. The
  translation assumes both backends use the machine's local time. Task Scheduler's own DST
  remarks are expressed in local time. [15]

## Open questions for a follow-up

- Does Task Scheduler ever fire a repetition exactly at `Duration`, and are second-granularity
  durations honoured? Needs a real-host test.
- Does `MonthlyDOWTrigger` with `Weeks={1,2,3,4,Last}` resolve to each date once in a 28-day
  February, or can the fourth and last week both match? Needs a real-host test.
- Which cron implementation ships on the target Pop!_OS image: Vixie cron or systemd-cron?
  The app writes a user crontab; the daemon's step and day rules should be detected or
  documented, not assumed.
- Should the storage model gain a structured Schedule (or a documented `L` extension) so
  last-day and first-Monday presets are possible? That is a map decision, not a translation
  detail. [22]

## Sources

1. `crontab(5)`, cronie 1.7.1, man7.org.
   <https://man7.org/linux/man-pages/man5/crontab.5.html> (fields, ranges, lists, steps,
   step-within-field note, names, Sunday 0/7, OR rule, DST note).
2. `crontab(5)`, Vixie cron 3.0pl1 (Debian bookworm), manpages.debian.org.
   <https://manpages.debian.org/bookworm/cron/crontab.5.en.html> (grammar, OR rule, Sunday
   equivalence).
3. cronie `cron.c` (match rule: star flags select AND, otherwise OR).
   <https://github.com/cronie-crond/cronie/blob/master/src/cron.c>
4. cronie `entry.c` (sets `DOM_STAR`/`DOW_STAR` from the first character).
   <https://github.com/cronie-crond/cronie/blob/master/src/entry.c>
5. Ubuntu cron 3.0pl1 `cron.c` (same match rule as the installed package).
   <https://git.launchpad.net/ubuntu/+source/cron/plain/cron.c>
6. Ubuntu cron 3.0pl1 `entry.c` (same star-flag parsing as the installed package).
   <https://git.launchpad.net/ubuntu/+source/cron/plain/entry.c>
7. Task Scheduler Schema, Microsoft Learn. The full XSD, including
   `<xs:group ref="triggerGroup" ... maxOccurs="48"/>`, the trigger element sequences,
   `DaysInterval` 1-365, `WeeksInterval` 1-52, `DaysOfMonth` 1-31/`Last`, `Weeks` 1-4/`Last`,
   month booleans, and `Repetition/Interval` PT1M-P31D.
   <https://learn.microsoft.com/en-us/windows/win32/taskschd/task-scheduler-schema>
8. `repetitionType` complex type (Interval min PT1M, max P31D; Duration min PT1M, optional).
   <https://learn.microsoft.com/en-us/windows/win32/taskschd/taskschedulerschema-repetitiontype-complextype>
9. `Duration` (repetitionType) element (omitted duration means indefinitely; format includes
   seconds; minimum one minute).
   <https://learn.microsoft.com/en-us/windows/win32/taskschd/taskschedulerschema-duration-repetitiontype-element>
10. Trigger Types, Microsoft Learn (Time, Daily, Weekly, Monthly, MonthlyDOW descriptions).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/trigger-types>
11. `TimeTrigger` element (starts at a specific date and time; StartBoundary required).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/taskschedulerschema-timetrigger-triggergroup-element>
12. `StartBoundary` element (required for time and calendar triggers).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/taskschedulerschema-startboundary-triggerbasetype-element>
13. `DailyTrigger` object (time of day from StartBoundary; daily schedule).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/dailytrigger>
14. `WeeklyTrigger` object (day-of-week set; weeks interval).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/weeklytrigger>
15. `MonthlyTrigger` object (day-of-month and month sets; StartBoundary; DST remark).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/monthlytrigger>
16. `MonthlyTrigger.DaysOfMonth` (bitmask 1-31 plus `Last`).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/monthlytrigger-daysofmonth>
17. `MonthlyTrigger.RunOnLastDayOfMonth` (last day regardless of the actual date).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/monthlytrigger-runonlastdayofmonth>
18. `MonthlyDOWTrigger` object (weeks, day-of-week, months; DST remark).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/monthlydowtrigger>
19. `Weeks` (monthlyDayOfWeekScheduleType) element (weeks 1-4, or `Last`).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/taskschedulerschema-weeks-monthlydayofweekscheduletype-element>
20. Daily Trigger Example (XML), Microsoft Learn (a daily trigger with a repetition pattern).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/daily-trigger-example--xml->
21. `Trigger.Repetition` property (how often the task runs and how long the pattern repeats
    after the task is started).
    <https://learn.microsoft.com/en-us/windows/win32/taskschd/trigger-repetition>
22. das-scheduler v1 spec map, including the Presets, the "custom cron must translate
    exactly" rule, and the cron-string storage decision.
    <https://github.com/4lch4/das-scheduler/issues/3>
23. `crontab(5)`, cron 3.0pl1 (Ubuntu/Debian patch), POSIX-versus-implementation note and the
    `0 0 */2 * sun` example.
    <https://git.launchpad.net/ubuntu/+source/cron/plain/debian/crontab.5.in.xml>

All sources were checked on 2026-11-05. The Task Scheduler pages are the Windows 10/11
documentation set; the installed cron is Ubuntu's 3.0pl1 package.
