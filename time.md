# Time

Spite stores time in one form: an `Instant`, an exact point on the universal time line. A time zone never
changes what is stored. It is a presentation layer that turns an `Instant` into the date and clock time someone
in that zone would read, and turns such a reading back into an `Instant`. The calendar has its own types
(`Date`, `Time`, `DateTime`) that cannot be mistaken for an instant, and the two kinds of length
are separate: a `Duration` is an exact amount of time, and a `Period` is an amount of calendar (years, months,
days), whose real length depends on where it is applied.

All of it is library code, written in Spite, with nothing running behind it: a program that never names a time
class carries none of it, and one that only measures with `Clock()` carries only that.
Each value is an ordinary small object, and arithmetic makes a new one for its result, which lives wherever the
compiler places it ([memory.md](memory.md#where-a-value-lives-memory)).

| Class | What it is | Stored as |
|---|---|---|
| `Instant` | a point on the universal time line | a `Duration` since 1970-01-01T00:00:00Z |
| `Duration` | an exact length of time, to the nanosecond | whole seconds and nanoseconds |
| `Date` | a calendar date with no zone: `2024-03-10` | year, month, day |
| `Time` | a clock reading with no date and no zone: `02:30:00` | hour, minute, second, nanosecond |
| `DateTime` | a date and a clock reading, with no zone | a `Date` and a `Time` |
| `Period` | an amount of calendar: `P1Y2M3D` | years, months, days |
| `TimeZone` | the rules that map instants to local readings | an IANA identifier and its rules |
| `TimeZones()` | the time zone database, a singleton | the operating system's |
| `Clock()` | `now()` for the wall clock, and elapsed time for measuring | |
| `Benchmark(work)` | one run of a function, timed | what it answered and a `Duration` |

What cannot happen, by construction:

- **A local reading is never an instant.** `DateTime` has no function that answers an `Instant`, and
  comparing one with an `Instant` is a type error. The only way across is `local.to_instant_in(zone, ...)`, which
  names the zone.
- **A day is never 24 hours by accident.** `Duration` has no days; `Period(1, 'days')` is one calendar day and is
  added to dates, never to instants. Adding "one day" to an instant is written through a zone, where it means
  what the zone says it means.
- **A daylight-saving gap or overlap is never resolved silently by a default you did not choose.** Every
  `to_instant_in` names the rule for the ambiguous case.

The first of them is the type checker's to enforce, so the mistake never runs:

```gdscript title=time_reading_is_not_instant/time_reading_is_not_instant.spite entry error
var console = Console()

func TimeReadingIsNotInstant() {
    load "../spite_standard_library/windows"
    var meeting_day = Date(2024, 3, 10)
    var meeting_time = Time(9, 30, 0, 0)
    var meeting = DateTime(meeting_day, meeting_time)
    var launch_since_1970 = Duration(1710054000, 'seconds')
    var launch = Instant(launch_since_1970)
    console.print(meeting == launch)
}
```
```diagnostic
an Instant cannot be used where a DateTime is needed
```

## Instants and durations

`Instant(since_1970)` is the instant a `Duration` after 1970-01-01T00:00:00Z (before it, for a negative one).
`clock.now()` is the current one. `+` adds a `Duration`, `-` between two instants answers the `Duration` between
them, and to go back you add a negative `Duration`: `moment + -hour`. An `Instant` prints in UTC, with a `Z`.

`Duration(amount, unit)` takes `'nanoseconds'`, `'microseconds'`, `'milliseconds'`, `'seconds'`, `'minutes'` or
`'hours'`. `total(unit)` is the whole length in one unit, and `part(unit)` is one component of it written as
hours, minutes, seconds and a fraction; both round toward zero and keep the sign.

```gdscript title=time_instants/time_instants.spite entry
var console = Console()

func TimeInstants() {
    load "../spite_standard_library/windows"
    var launch_since_1970 = Duration(1710054000, 'seconds')
    var launch = Instant(launch_since_1970)
    var hour = Duration(1, 'hours')
    var later = launch + hour
    var earlier = launch + -hour
    var between = later - earlier
    console.print(launch, later, earlier, between)
    var lap = Duration(90500, 'milliseconds')
    var minutes = lap.total('minutes')
    var seconds = lap.part('seconds')
    var half = Duration(-500, 'milliseconds')
    console.print(lap, minutes, seconds, half, lap * 2)
    var in_order = earlier < launch and launch < later
    var two_hours = Duration(120, 'minutes')
    console.print(in_order, between == two_hours)
}
```
```output
2024-03-10T07:00:00Z 2024-03-10T08:00:00Z 2024-03-10T06:00:00Z PT2H
PT1M30.5S 1 30 -PT0.5S PT3M1S
true true
```

`Clock()` keeps its monotonic readings, `elapsed_nanoseconds()` and `elapsed_milliseconds()`, for measuring a
frame or a request: the wall clock can jump when the machine's time is corrected, and a monotonic clock cannot. A
reading is a `Long` of nanoseconds and allocates nothing, so a profiler can take one around every system of every
frame; a `Duration` reading would be an object made per read, so the monotonic clock answers the number.
`now()` is the wall clock, and each system reads it its own way (`GetSystemTimeAsFileTime` on Windows,
`clock_gettime` on Linux and macOS).

## Measuring a piece of work

To time one piece of work, hand it to `Benchmark(work)`: it reads the monotonic clock, runs `work` once, reads the
clock again, and keeps both what `work` answered, in `answer`, and how long it took, in `duration`, a `Duration`.
`work` is a function of yours that takes nothing, passed by name, so what is measured is that function and nothing
around it: the setup before the line and the printing after it are not timed.

```gdscript title=time_benchmark/time_benchmark.spite entry
var console = Console()

func TimeBenchmark() {
    load "../spite_standard_library/windows"
    var benchmark = Benchmark(sum_to_a_million)
    console.print("sum", benchmark.answer)
    var took = benchmark.duration
    console.print("measured:", not took.is_negative)
}

func sum_to_a_million(): Long {
    var total: Long = 0
    var index = 0
    while index < 1000000 {
        total = total + index
        index = index + 1
    }
    return total
}
```
```output
sum 499999500000
measured: true
```

What the work answered is kept, so the work is never left out as unused. Work that answers nothing makes a
`Benchmark<Nothing>`, and what it computes is whatever it stores, read once the line has run. `Benchmark` is
library code like the rest: a program that never names it carries none of it, and the work is called once, through
its function value, between the two readings. To print the time, ask the `Duration` for the unit you want:
`benchmark.duration.total('microseconds')` ([spec](https://github.com/Spite-Language/spite_spec/blob/master/specs/time.md#time-one-stored-instant-zones-for-presentation)).
In a program that also starts a `Parallel`, the work is treated as code another thread may run, so the objects it
counts are counted atomically and the singletons it calls keep their locks; to measure exactly what those cost,
read the clock twice around the code instead.

## Dates and periods

`Date(year, month, day)` is a date in the proleptic Gregorian calendar, the one ISO 8601 uses, for any year
(year 0 is 1 BC). A date that does not exist (`Date(2023, 2, 29)`) halts the program, since a program that
builds one from its own numbers has a bug; text from outside is read with `text.to_date()` instead, which
answers `null` ([Text](#text)). `Time(hour, minute, second, nanosecond)` is the same for a clock reading, and
`DateTime(date, time)` joins the two.

A `Period` adds to a date by months first and then by days, and a day of the month that the new month does not
have becomes its last day: January 31 plus one month is February 29 in a leap year. `period_until` answers the
years, months and days between two dates the same way, so adding it back lands on the later date.

```gdscript title=time_calendar/time_calendar.spite entry
var console = Console()

func TimeCalendar() {
    load "../spite_standard_library/windows"
    var end_of_january = Date(2024, 1, 31)
    var month = Period(1, 'months')
    var leap = end_of_january + month
    var common = Date(2023, 1, 31) + month
    var fortnight = leap + Period(2, 'weeks')
    console.print(leap, common, fortnight, leap.weekday, leap.day_of_year)
    var born = Date(1990, 5, 15)
    var today = Date(2024, 3, 10)
    var age = born.period_until(today)
    var days = born.days_until(today)
    console.print(age, days, today.is_leap_year, today.days_in_month)
    var half_past_nine = Time(9, 30, 0, 0)
    var meeting = DateTime(today, half_past_nine)
    var next_meeting = meeting + Period(1, 'months')
    console.print(meeting, next_meeting)
}
```
```output
2024-02-29 2023-02-28 2024-03-14 thursday 60
P33Y9M24D 12353 true 31
2024-03-10T09:30:00 2024-04-10T09:30:00
```

A `Period` is not compared with `<`: whether one month is longer than 30 days depends on the month. Two periods
are equal when their years, months and days are.

## Time zones

`TimeZones()` is the database. `find(identifier)` answers the zone with an IANA identifier such as
`"America/New_York"`, or `null` when the database has none by that identifier; `utc()` is UTC;
`fixed_offset(offset)` is a zone that is
always the same distance from UTC; `system()` is the zone the machine is set to, for showing times to the person
at it. Nothing uses it unless it is asked for by name, so a program never depends on the machine's zone by
accident.

A zone is one class, `TimeZone`, and all it answers is its own: `offset_at(instant)`, its distance from UTC at
that instant as a `Duration`, and `identifier`, the IANA identifier it was found by. A conversion is asked of the
value being converted, naming the zone:

| Call | Answers |
|---|---|
| `instant.to_date_time_in(zone)` | the `DateTime` a clock in the zone reads at that instant |
| `local.to_instant_in(zone, ambiguity)` | the `Instant` at which the zone's clocks read `local` |
| `instant.to_string_in(zone)` | the local reading with its offset: `2024-03-10T03:00:00-04:00` |

A local reading is not always one instant. When clocks spring forward, a reading in the gap never happens; when
they fall back, a reading in the overlap happens twice. `to_instant_in` takes what to do about it, and there is
no default:

| Ambiguity | In an overlap | In a gap |
|---|---|---|
| `'earlier'` | the first of the two instants | the reading taken with the offset after the gap, which lands before it |
| `'later'` | the second of the two instants | the reading taken with the offset before the gap, which lands after it |
| `'compatible'` | the first, like `'earlier'` | the one after, like `'later'` |

`'compatible'` is what RFC 5545 calendars, `java.time` and JavaScript do by default: a meeting at 02:30 on the
night the clocks skip 02:00-03:00 happens at 03:30, and one at 01:30 on the night 01:00-02:00 happens twice is the
first 01:30. It is still written at every call.

```gdscript title=time_zones_named/time_zones_named.spite entry
var console = Console()
var zones = TimeZones()

func TimeZonesNamed() {
    load "../spite_standard_library/windows"
    var new_york = zones.find("America/New_York")
    crash new_york
    var launch_since_1970 = Duration(1710054000, 'seconds')
    var launch = Instant(launch_since_1970)
    var shown = launch.to_string_in(new_york)
    var offset = new_york.offset_at(launch)
    console.print(launch, shown, offset)
    var skipped_day = Date(2024, 3, 10)
    var skipped_time = Time(2, 30, 0, 0)
    var skipped = DateTime(skipped_day, skipped_time)
    var repeated_day = Date(2024, 11, 3)
    var repeated_time = Time(1, 30, 0, 0)
    var repeated = DateTime(repeated_day, repeated_time)
    var after_gap = skipped.to_instant_in(new_york, 'compatible')
    var first = repeated.to_instant_in(new_york, 'earlier')
    var second = repeated.to_instant_in(new_york, 'later')
    var after_gap_text = after_gap.to_string_in(new_york)
    var first_text = first.to_string_in(new_york)
    var second_text = second.to_string_in(new_york)
    console.print(after_gap_text, first_text, second_text)
    var nowhere = zones.find("Atlantis/Capital")
    if nowhere {
        console.print("found", nowhere.identifier)
    } else {
        console.print("no such zone")
    }
}
```
```output
2024-03-10T07:00:00Z 2024-03-10T03:00:00-04:00 -PT4H
2024-03-10T03:30:00-04:00 2024-11-03T01:30:00-04:00 2024-11-03T01:30:00-05:00
no such zone
```

"The same time tomorrow" is a question for a zone, because tomorrow may be 23 or 25 hours away: read the local
date and time, add a `Period`, and turn it back into an instant.

```gdscript title=time_tomorrow/time_tomorrow.spite entry
var console = Console()
var zones = TimeZones()

func TimeTomorrow() {
    load "../spite_standard_library/windows"
    var new_york = zones.find("America/New_York")
    crash new_york
    var saturday_since_1970 = Duration(1709996400, 'seconds')
    var saturday = Instant(saturday_since_1970)
    var local = saturday.to_date_time_in(new_york)
    var tomorrow = local + Period(1, 'days')
    var sunday = tomorrow.to_instant_in(new_york, 'compatible')
    var elapsed = sunday - saturday
    var sunday_text = sunday.to_string_in(new_york)
    console.print(local, sunday_text, elapsed)
}
```
```output
2024-03-09T10:00:00 2024-03-10T10:00:00-04:00 PT23H
```

### Fixed offsets and UTC

A fixed offset has no rules at all: `zones.fixed_offset(india_offset)`, with `india_offset` a
`Duration(330, 'minutes')`, is identified as `+05:30` and is five
and a half hours ahead of UTC at every instant. It is the right zone for text that carries its own offset and
nothing more, and the wrong one for a place, whose offset changes. It is `TimeZone(identifier, offset)` with the
offset written as its identifier: the constructor makes only fixed offsets (under a day either way, or the
program halts), and a zone with rules comes from the database.

```gdscript title=time_offsets/time_offsets.spite entry
var console = Console()
var zones = TimeZones()

func TimeOffsets() {
    load "../spite_standard_library/windows"
    var launch_since_1970 = Duration(1710054000, 'seconds')
    var launch = Instant(launch_since_1970)
    var india_offset = Duration(330, 'minutes')
    var india = zones.fixed_offset(india_offset)
    var universal = zones.utc()
    var india_text = launch.to_string_in(india)
    var universal_text = launch.to_string_in(universal)
    console.print(india.identifier, india_text, universal.identifier, universal_text)
}
```
```output
+05:30 2024-03-10T12:30:00+05:30 UTC 2024-03-10T07:00:00+00:00
```

### Where zones come from

The rules are the operating system's, so a change in a country's daylight-saving law reaches every Spite program
with the system's own updates, and a program carries no copy of the database:

- **Windows** reads them through `icu.dll`, the ICU library Windows 10 (version 1903 and later) and Windows 11
  ship and update, which holds the IANA database under its IANA identifiers (`windows/time_zone.spite`, over
  `DynamicLibrary`). Windows' own registry zones are not used: they have Windows names, and less history.
- **Linux and macOS** read the compiled database in `/usr/share/zoneinfo`, one TZif file per zone (RFC 8536),
  through `File` and parsed in Spite (`TimeZone`'s TZif reading, in `common/time_zone.spite`); `system()` follows `TZ`, then the
  `/etc/localtime` link.

What a zone costs when it runs: on Windows, `icu.dll` is loaded only by a program that asks for a named zone or
`system()`, a zone holds an ICU calendar open until it is dropped, and each offset it answers is one call into
ICU; on Linux and macOS, `find` reads and parses the zone's file once, and each answer looks through the
transitions it read. UTC and fixed offsets need neither.

`zones.read_tzif(identifier, data)` reads a TZif file you bring, its bytes as `File.read_as_buffer()` gives them,
which is the answer for a machine with no database (a
bare container) or a program that must pin one version of the rules. It is the same reader Linux and macOS use,
so it runs on Windows too.

A TZif file has the transitions its zone has had, and a POSIX rule for the years after them
(`EST5EDT,M3.2.0,M11.1.0`), which is how the reader answers for 2100. A time zone's rules are only
known up to when they were written, on every system: a future law change is not in them yet.

## Text

Time text is ISO 8601 as RFC 3339 and RFC 9557 profile it. Writing is every type's `to_string()`; reading is a
`to_<type>()` on the text, as `to_integer()` and `to_color()` are, each answering `null` for text that is not
exactly that type:

| Function | Reads | Refuses |
|---|---|---|
| `text.to_instant()` | `2024-03-10T07:00:00Z`, `2024-03-10T03:00:00-04:00[America/New_York]` | a reading with no offset |
| `text.to_date_time()` | `2024-03-10T02:30:00`, `2024-03-10 02:30` | a reading with an offset |
| `text.to_date()` | `2024-02-29`, `+012345-06-07` | `2023-02-29` |
| `text.to_time()` | `09:30`, `23:59:59.5` | `24:00`, the leap second `23:59:60` |
| `text.to_duration()` | `PT1H30M`, `-PT0.5S` | `P1D`: a day is a `Period` |
| `text.to_period()` | `P1Y2M3D`, `P2W` | `PT1H`: an hour is a `Duration` |

The refusals are the point: text that carries an offset is an instant, and reading it as a local reading would
drop the one fact that makes it exact, while text without one is a local reading and needs a zone to become an
instant. A bracketed zone after the offset must agree with it: `-04:00[America/New_York]` is read, while
`-05:00[America/New_York]` on a day New York is four hours behind, a zone the database does not have, and a
second zone are `null`, since one of the two would be wrong and the text does not say which. After `Z`, which
says the offset is not known, any zone the database has agrees. The one other annotation read is the ISO
calendar, `[u-ca=iso8601]`; any other is `null`, since Spite has no other calendar. A leap second (`23:59:60`)
is `null` too: no type holds it, and reading it as `23:59:59` would change the value read. `T` may be written
`t` or a space, and a fraction may follow `,` as well as `.`.

```gdscript title=time_text_reading/time_text_reading.spite entry
var console = Console()

func TimeTextReading() {
    load "../spite_standard_library/windows"
    var exact = "2024-03-10T03:00:00-04:00[America/New_York]".to_instant()
    crash exact
    var local = "2024-03-10 02:30".to_date_time()
    crash local
    var lap = "PT90M".to_duration()
    crash lap
    console.print(exact, local, lap)
    var no_offset = "2024-03-10T02:30:00".to_instant()
    if no_offset {
        console.print("unexpected")
    } else {
        console.print("a local reading is not an instant")
    }
    var wrong_zone = "2024-03-10T03:00:00-05:00[America/New_York]".to_instant()
    if wrong_zone {
        console.print("unexpected")
    } else {
        console.print("New York was not five hours behind")
    }
    var no_such_day = "2023-02-29".to_date()
    if no_such_day {
        console.print("unexpected")
    } else {
        console.print("2023 has no February 29")
    }
}
```
```output
2024-03-10T07:00:00Z 2024-03-10T02:30:00 PT1H30M
a local reading is not an instant
New York was not five hours behind
2023 has no February 29
```

## What time does not cover

There are no calendars other than the ISO one, no leap seconds (text with one is `null`, never moved to the second
before it), no formatting patterns or localized month names, and no zone abbreviations (`EST`).

## Why this design

The goal is the best date support there is, so that Spite never repeats JavaScript's `Date` (a type that is
an instant and a local reading at once, whose months count from zero and whose replacement took most of a decade,
with a wrapper library in every project meanwhile), with one stored format and zones only for presentation. Five
modern designs were compared:

- **JavaScript's `Temporal`** (the replacement for `Date`, in browsers from 2025) gets the split right:
  `Instant` for exact time, `PlainDate`/`PlainTime`/`PlainDateTime` for readings, zones by IANA name, RFC 9557
  text, and the `'compatible'`/`'earlier'`/`'later'`/`'reject'` choice for gaps and overlaps, which Spite copies.
  It gets two things wrong for Spite: its `Duration` holds years and months beside hours and nanoseconds, so
  whether a day is 24 hours depends on a `relativeTo` option; and `ZonedDateTime` makes an instant-plus-zone a
  thing to store, which is the opposite of zones as presentation. Its surface (non-ISO calendars, year-month
  and month-day types, rounding options) is several ways to do each thing.
- **Java's `java.time`** (JSR-310, from Joda-Time) has the separation Spite wants between `Duration` (exact
  seconds and nanoseconds) and `Period` (years, months, days), immutable values, and zone rules that answer
  transitions, read from the JDK's own copy of the database; its zone-less types are `LocalDate`, `LocalTime`
  and `LocalDateTime`. It also has `ZonedDateTime`,
  `OffsetDateTime` and `OffsetTime` besides, and it resolves a gap or an overlap by a default nobody chose at the
  call (`atZone`), with `ZoneId.systemDefault()` a line away.
- **Rust's `jiff`** is Temporal in Rust: `Timestamp`, `civil::Date`/`Time`/`DateTime`, `Zoned`, `Span` and
  `SignedDuration`, and it reads `/usr/share/zoneinfo` rather than shipping a copy, the approach Spite copies
  for Linux and macOS. It keeps Temporal's two faults (`Span` mixes calendar and exact units; `Zoned` is a stored
  type). **`chrono`**, its predecessor, has `NaiveDate`/`NaiveDateTime` and a `LocalResult` that makes an
  ambiguous reading a value, but carries the zone as a type parameter, added month arithmetic late, and left the
  database to competing crates, the ecosystem split Spite's design avoids.
- **.NET's NodaTime** (by Jon Skeet, written because `DateTime` has the same fault as JavaScript's `Date`) is the
  most principled: `Instant`, NodaTime's `LocalDate`/`LocalTime`/`LocalDateTime`, `Duration` apart from `Period`, a clock
  that is a service rather than a global, and gap and overlap resolution that is always explicit
  (`AtStrictly`, `AtLeniently`, a resolver). It ships its own copy of the database, and it too has
  `ZonedDateTime` and `OffsetDateTime`.
- **Go's `time`** is small and has the right monotonic-clock idea (a `Time` carries a monotonic reading for
  measuring), but one `Time` is both an instant and a local reading with a location attached, so `==` compares
  locations too and `Equal` must be remembered; there is no date-only type; a `Duration` is nanoseconds in 64 bits
  (292 years); formatting is by a magic reference date. `LoadLocation` reads the system database, with an
  embeddable copy as a fallback.

**Spite copies NodaTime's model with Temporal's vocabulary.** From NodaTime and `java.time`: an `Instant`, the
three zone-less types, and a `Duration` that can never hold a day beside a `Period` that can never hold an hour.
From Temporal: the ambiguity words, the ISO 8601 and RFC 9557 text, and printing an `Instant` in UTC with `Z`.
From jiff: reading the operating system's database. What all of them have and Spite leaves out is a stored
zoned type (`ZonedDateTime`, `Zoned`, `OffsetDateTime`): Spite makes a zone presentation, so an instant is stored
and a zone is applied when it is shown, and the one conversion that needs a zone, local reading to instant,
names it and names its ambiguity rule every time. The zone-less types are plainly `Date`, `Time` and `DateTime`,
jiff's words, rather than NodaTime's and `java.time`'s `Local*` or Temporal's `Plain*`: a reader takes
"local" to mean "has a zone", which is the confusion Temporal's `Plain` avoided, and a type that does carry a zone
for presentation gets a prefix.

**Why not embed a copy of the database**, as NodaTime, Go's fallback and jiff on Windows do: a copy is out of date
the day a country changes its law, and every program built with it stays wrong until it is rebuilt, while the
operating system's copy is updated for every program at once. It would also add a few hundred kilobytes to every
program that names a zone. The costs are a machine with no database, where `find` answers `null` and
`read_tzif` takes a file you ship, and Windows before 10 version 1903, which has no `icu.dll`.

---

Next: [Game maths](game_maths.md), vectors, matrices and the maths a game loop needs.
