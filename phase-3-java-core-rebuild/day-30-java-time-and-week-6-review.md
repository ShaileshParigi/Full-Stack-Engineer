# Day 30 — java.time + Week 6 Review

_Why this matters:_ Time bugs are the quiet kind. The code works in your tests,
works in staging, and then fails twice a year when the clocks change — or for the one
customer in a different country, or after a server is moved to a different region. An
invoice due date a day off; a scheduled job that runs at 10:00 instead of 09:00; a
"created at" that's five and a half hours wrong. Almost all of them come from one
mistake: using the wrong *kind* of time for the job. `java.time` gives you the right
kind for each job — and today you learn which is which, and the one rule that prevents
most of the damage (**store UTC, show local**), which pays off again in Phase 7 as
Postgres `timestamptz`.

> **Big day.** If it runs past two hours: Parts 1–5 and hands-on 1–3 today; Parts 6–8,
> the Week 6 review, and the rest tomorrow. The three required exercises are
> hands-on 1, 2 and 3.

---

## The one-paragraph version

There are really two different things people call "a time". One is **an exact moment
on the timeline** — "the server received this request at the instant the whole world
agrees was 09:30 UTC". That's an **`Instant`**. It has no time zone and no "local"
meaning; it's the same moment for everybody. The other is **what a wall clock or
calendar shows** — "12 October", "9:30" — which means nothing precise until you say
*where*. Those are the **`Local…`** types: `LocalDate`, `LocalTime`, `LocalDateTime`.
To go between the two worlds you need a **time zone** (`ZoneId`): an `Instant` + a zone
= a `ZonedDateTime` (an instant seen on a particular wall clock). All of these are
**immutable** — every `plus…`/`with…` returns a **new** object. Amounts of time come
in two flavours: a **`Duration`** (exact seconds — "90 minutes") and a **`Period`**
(calendar units — "1 month"), and they differ because a day isn't always 24 hours and a
month isn't a fixed number of days. The rules of thumb: **store and transmit instants
in UTC; convert to a zone only when displaying; use `ZonedDateTime` (never
`LocalDateTime`) for arithmetic that crosses a daylight-saving change.**

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **timeline** | The one continuous line of real moments, same for everyone | What `Instant` points into |
| **`Instant`** | An exact moment, with no zone | Nanoseconds since 1970-01-01T00:00:00Z (the **epoch**); prints with a trailing `Z` |
| **UTC** | The world's reference clock (no daylight saving) | "Coordinated Universal Time"; offset `+00:00`, written `Z` |
| **time zone** | A region with its own clock rules, including daylight saving | `ZoneId`, named like `Asia/Kolkata`, `America/New_York` (from the IANA database) |
| **offset** | How far a clock is from UTC *right now* | `+05:30`, `-04:00` — just a number; says nothing about *rules* |
| **`LocalDate`** | A calendar date, no time, no zone | "2026-10-12" |
| **`LocalTime`** | A time on a wall clock, no date, no zone | "09:30" |
| **`LocalDateTime`** | A date and a wall-clock time, **no zone** | "2026-10-12T09:30" — not a moment! |
| **`ZonedDateTime`** | A date, time **and** zone: a real moment seen on a local clock | `2026-10-12T15:00+05:30[Asia/Kolkata]`; knows the zone's rules |
| **`OffsetDateTime`** | A date, time and *fixed offset* | `2026-10-12T15:00+05:30`; a real moment, but no zone rules |
| **`Duration`** | An exact length of time | Seconds + nanos: `PT8H45M` |
| **`Period`** | A length of calendar time | Years + months + days: `P1M`, `P31Y6M22D` |
| **DST (daylight saving time)** | Clocks jump forward in spring and back in autumn | Creates a *gap* (a local time that doesn't exist) and an *overlap* (a local time that happens twice) |
| **immutable** | Can't be changed; operations return a new object | Same idea as `String` (Day 22) |
| **ISO-8601** | The standard written format for dates and times | `2026-10-12T09:30:00Z`; what `toString()` and JSON use |
| **epoch** | The agreed zero point of computer time | 1970-01-01T00:00:00Z |
| **`Clock`** | A source of "now" you can replace | `Clock.systemUTC()`, `Clock.fixed(...)`; makes time testable |
| **`DateTimeFormatter`** | Converts between a date/time and text | Immutable and thread-safe, unlike `SimpleDateFormat` |

---

## Before you read: five questions

1. A flight lands at "2026-03-08T02:30" in New York. Is that a valid time that day?
2. `LocalDate d = LocalDate.of(2026, 10, 12); d.plusDays(7);` — what is `d` afterwards?
3. What's the difference between `Duration.ofHours(24)` and `Period.ofDays(1)`?
4. You must store "when this order was placed" in a database used by customers in
   twelve countries. What do you store?
5. `LocalDateTime` has a date and a time. Why is it *not* a moment in time?

<details><summary>Answers (try first)</summary>

1. No — in New York on 2026-03-08 the clocks jump from 02:00 to 03:00. (Part 6.)
2. Unchanged: `2026-10-12`. It returns a new object. (Part 2.)
3. 24 hours exactly vs "the same time tomorrow" — they differ across a DST change.
   (Part 5.)
4. An `Instant` (UTC). (Part 7.)
5. With no zone, the same value means a different real moment in each place. (Part 1.)

</details>

---

## Part 1 — Why `Date` and `Calendar` were replaced

**In plain words:** before Java 8 there was `java.util.Date` (which is really a
*timestamp*, despite the name) and `java.util.Calendar`. They were famously hard to use
correctly. `java.time` replaced them in Java 8, and in new code you should treat
`Date`/`Calendar`/`SimpleDateFormat` as legacy that you convert *away from* at the
boundary.

Don't take my word for the problems. This runs all three:

```java
import java.time.DateTimeException;
import java.time.LocalDate;
import java.util.Calendar;
import java.util.Date;
import java.util.GregorianCalendar;

@SuppressWarnings("deprecation")
public class OldVsNew {
    static void sneakyAdjust(Date d) {
        d.setTime(d.getTime() + 24L * 60 * 60 * 1000);
    }

    static void adjust(LocalDate d) {
        d.plusDays(1);
    }

    public static void main(String[] args) {
        System.out.println("-- 1. zero-indexed months --");
        Calendar cal = new GregorianCalendar(2026, 1, 15);
        System.out.println("new GregorianCalendar(2026, 1, 15) means month: " + (cal.get(Calendar.MONTH) + 1) + "  (February is 1!)");
        System.out.println("LocalDate.of(2026, 2, 15) = " + LocalDate.of(2026, 2, 15));

        System.out.println();
        System.out.println("-- 2. nonsense dates are silently 'fixed' --");
        Calendar bad = new GregorianCalendar(2026, Calendar.FEBRUARY, 30);
        System.out.println("Calendar Feb 30 -> month=" + (bad.get(Calendar.MONTH) + 1) + " day=" + bad.get(Calendar.DAY_OF_MONTH) + "   (rolled into March, no error)");
        try {
            LocalDate.of(2026, 2, 30);
        } catch (DateTimeException e) {
            System.out.println("LocalDate Feb 30 -> " + e.getMessage());
        }

        System.out.println();
        System.out.println("-- 3. mutability: a method can change YOUR date --");
        Date due = new Date(0L);
        long before = due.getTime();
        sneakyAdjust(due);
        System.out.println("Date changed by the callee: " + (due.getTime() != before));

        LocalDate dueNew = LocalDate.of(2026, 3, 1);
        adjust(dueNew);
        System.out.println("LocalDate after callee's plusDays(1): " + dueNew + "   (unchanged: it returns a NEW object)");
    }
}
```

```
-- 1. zero-indexed months --
new GregorianCalendar(2026, 1, 15) means month: 2  (February is 1!)
LocalDate.of(2026, 2, 15) = 2026-02-15

-- 2. nonsense dates are silently 'fixed' --
Calendar Feb 30 -> month=3 day=2   (rolled into March, no error)
LocalDate Feb 30 -> Invalid date 'FEBRUARY 30'

-- 3. mutability: a method can change YOUR date --
Date changed by the callee: true
LocalDate after callee's plusDays(1): 2026-03-01   (unchanged: it returns a NEW object)
```

| Old API problem | What you saw | `java.time` fix |
|-----------------|--------------|-----------------|
| **Zero-indexed months** (and 1900-based years in `Date`) | Month `1` means *February* | `LocalDate.of(2026, 2, 15)` or `Month.FEBRUARY` — 1 is January |
| **Lenient — invalid dates "roll over"** | Feb 30 became Mar 2 silently | Throws `DateTimeException` immediately |
| **Mutable** | A method changed the caller's `Date` | Immutable — like `String` and for the same reasons (Day 22) |
| **Not thread-safe formatters** | `SimpleDateFormat` shared between threads corrupts output (Phase 5) | `DateTimeFormatter` is immutable and thread-safe |
| **`Date` is mislabelled** | It's a timestamp, but prints in the default zone | `Instant` is an instant and says so |
| **No separate types** for "date only", "time only", "zone" | Everything is a `Date` | `LocalDate`, `LocalTime`, `ZonedDateTime`, … |

**Bridging to legacy code** (JDBC drivers, old libraries): `Date.from(instant)` and
`date.toInstant()`; `java.sql.Timestamp` has `toInstant()` / `Timestamp.from(...)`. Convert
at the edge, then use `java.time` inside.

> **Say this in an interview:** "`Date` and `Calendar` are mutable, zero-indexed for
> months, lenient about invalid dates, and `SimpleDateFormat` isn't thread-safe.
> `java.time` is immutable, strict, thread-safe, and has separate types for dates, times,
> instants and zones."

---

## Part 2 — `LocalDate`, `LocalTime`, `LocalDateTime`

**In plain words:** these are *what's written on a calendar or a clock* — nothing more.
"12 October 2026" is a `LocalDate`. "9:30" is a `LocalTime`. "12 October 2026 at 9:30" is
a `LocalDateTime`. None of them says **where**, so none of them identifies a real
moment: 9:30 in Tokyo and 9:30 in New York are about 13 hours apart. "Local" here means
"no zone attached", not "the local zone".

Use them for things that *really are* calendar facts: a birthday, a due date, an
opening time, "the 1st of each month". Run it — this includes two of today's practice
exercises (days until a date, and adding a business week):

```java
import java.time.DayOfWeek;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.temporal.ChronoUnit;
import java.time.temporal.TemporalAdjusters;

public class LocalTypes {
    static LocalDate addBusinessDays(LocalDate start, int days) {
        LocalDate d = start;
        int added = 0;
        while (added < days) {
            d = d.plusDays(1);
            if (d.getDayOfWeek() != DayOfWeek.SATURDAY && d.getDayOfWeek() != DayOfWeek.SUNDAY) {
                added++;
            }
        }
        return d;
    }

    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2026, 10, 12);
        LocalTime time = LocalTime.of(9, 30);
        LocalDateTime dateTime = LocalDateTime.of(date, time);
        System.out.println("LocalDate:     " + date + "  (" + date.getDayOfWeek() + ")");
        System.out.println("LocalTime:     " + time);
        System.out.println("LocalDateTime: " + dateTime);

        System.out.println();
        System.out.println("-- immutable: every operation returns a NEW object --");
        date.plusDays(7);
        System.out.println("after date.plusDays(7) with result ignored: " + date);
        LocalDate nextWeek = date.plusDays(7);
        System.out.println("nextWeek = date.plusDays(7):                " + nextWeek);

        System.out.println();
        System.out.println("-- month ends and leap years --");
        System.out.println("Jan 31 + 1 month = " + LocalDate.of(2026, 1, 31).plusMonths(1) + "   (clamped to the last valid day)");
        System.out.println("Feb 28 2028 + 1 day = " + LocalDate.of(2028, 2, 28).plusDays(1) + "   (2028 is a leap year)");
        System.out.println("isLeapYear 2026/2028: " + LocalDate.of(2026, 1, 1).isLeapYear() + "/" + LocalDate.of(2028, 1, 1).isLeapYear());

        System.out.println();
        System.out.println("-- practice: days until a future date --");
        LocalDate today = LocalDate.of(2026, 10, 12);
        LocalDate deadline = LocalDate.of(2026, 12, 25);
        System.out.println("days from " + today + " to " + deadline + ": " + ChronoUnit.DAYS.between(today, deadline));
        System.out.println("weeks: " + ChronoUnit.WEEKS.between(today, deadline));

        System.out.println();
        System.out.println("-- practice: add a business week --");
        LocalDate friday = LocalDate.of(2026, 10, 9);
        System.out.println("Fri " + friday + " + 5 business days = " + addBusinessDays(friday, 5) + " (" + addBusinessDays(friday, 5).getDayOfWeek() + ")");
        System.out.println("Mon " + today + " + 5 business days = " + addBusinessDays(today, 5) + " (" + addBusinessDays(today, 5).getDayOfWeek() + ")");

        System.out.println();
        System.out.println("-- adjusters --");
        System.out.println("next Monday after " + date + ":  " + date.with(TemporalAdjusters.next(DayOfWeek.MONDAY)));
        System.out.println("last day of the month:       " + date.with(TemporalAdjusters.lastDayOfMonth()));
        System.out.println("comparisons: isBefore=" + today.isBefore(deadline) + " isAfter=" + today.isAfter(deadline));
    }
}
```

```
LocalDate:     2026-10-12  (MONDAY)
LocalTime:     09:30
LocalDateTime: 2026-10-12T09:30

-- immutable: every operation returns a NEW object --
after date.plusDays(7) with result ignored: 2026-10-12
nextWeek = date.plusDays(7):                2026-10-19

-- month ends and leap years --
Jan 31 + 1 month = 2026-02-28   (clamped to the last valid day)
Feb 28 2028 + 1 day = 2028-02-29   (2028 is a leap year)
isLeapYear 2026/2028: false/true

-- practice: days until a future date --
days from 2026-10-12 to 2026-12-25: 74
weeks: 10

-- practice: add a business week --
Fri 2026-10-09 + 5 business days = 2026-10-16 (FRIDAY)
Mon 2026-10-12 + 5 business days = 2026-10-19 (MONDAY)

-- adjusters --
next Monday after 2026-10-12:  2026-10-19
last day of the month:       2026-10-31
comparisons: isBefore=true isAfter=false
```

Things to take from it:

- **Immutability, again.** `date.plusDays(7);` with the result ignored did nothing —
  the same mistake as `s.toUpperCase();` on Day 22. Every "modifier" (`plus…`,
  `minus…`, `with…`, `at…`) returns a new object. This makes them thread-safe and safe
  to share, and is why `LocalDate` is a good `HashMap` key.
- **Month arithmetic clamps.** Jan 31 + 1 month = **Feb 28**, the last valid day,
  rather than overflowing into March.
- **`ChronoUnit.X.between(a, b)`** is the all-purpose "how many of X between these"
  (`DAYS`, `WEEKS`, `HOURS`, …) — the answer to "days until a deadline".
- **Business days**: `java.time` has no built-in notion of working days (holidays vary
  by country), so you loop or write a small helper. Real systems need a holiday
  calendar.
- **`TemporalAdjusters`** answer "next Monday", "last day of month", "first Friday".
- Compare with `isBefore` / `isAfter` / `equals` (not `==`, Day 22).

> **Say this in an interview:** "`LocalDate`, `LocalTime` and `LocalDateTime` are
> date/time values with no zone, so they don't identify a point on the timeline. I use
> them for calendar concepts like birthdays and due dates, and they're immutable —
> every operation returns a new instance."

---

## Part 3 — `Instant`: a point on the timeline

**In plain words:** an `Instant` is the answer to "*when exactly* did this happen?",
with no ambiguity and no timezone drama. It counts the time since the epoch (midnight
UTC on 1 Jan 1970). Two instants can be compared, subtracted, sorted and stored without
caring where anyone is. When you print one it ends in **`Z`** — "Zulu", meaning UTC.

```
Instant.now()                           the current moment
Instant.parse("2026-10-12T09:30:00Z")   from ISO text
instant.toEpochMilli()                  1791797400000   (the number most APIs and JSON use)
```

An `Instant` is what a **log line, an audit field, `createdAt`, a token expiry, a Kafka
event time, and a JWT `exp` claim** should be. It's the only type that always identifies
one real moment and nothing else.

What you *can't* ask of an `Instant`: "what's the day of the week?" or "what month is
it?" — those are answers that depend on **where** you are. For those you need a zone —
next part.

---

## Part 4 — Zones: `ZoneId`, `ZonedDateTime`, `OffsetDateTime`

**In plain words:** "9:30 UTC" is one moment; what a clock on the wall says at that
moment depends on where the wall is. A **time zone** is the rulebook for a place: its
standard offset from UTC, and when it shifts for daylight saving. Zones are named
`Region/City` — `Asia/Kolkata`, `America/New_York`, `Europe/London` — from a database
the JDK ships and updates. Combine an `Instant` with a `ZoneId` and you get a
`ZonedDateTime`: *the same moment, shown on that place's clock*.

```java
import java.time.Clock;
import java.time.Instant;
import java.time.LocalDate;
import java.time.OffsetDateTime;
import java.time.ZoneId;
import java.time.ZoneOffset;
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;

public class InstantAndZones {
    public static void main(String[] args) {
        Instant meeting = Instant.parse("2026-10-12T09:30:00Z");
        System.out.println("Instant (always UTC): " + meeting);
        System.out.println("epoch millis:         " + meeting.toEpochMilli());

        System.out.println();
        System.out.println("-- the SAME instant, three zones (practice) --");
        DateTimeFormatter shown = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm  xxx");
        for (String zone : new String[] {"Asia/Kolkata", "America/New_York", "Europe/London", "Asia/Tokyo"}) {
            ZonedDateTime local = meeting.atZone(ZoneId.of(zone));
            System.out.printf("%-17s %s%n", zone, local.format(shown));
        }

        System.out.println();
        System.out.println("-- ZonedDateTime vs OffsetDateTime --");
        ZonedDateTime zoned = meeting.atZone(ZoneId.of("Asia/Kolkata"));
        OffsetDateTime offset = meeting.atOffset(ZoneOffset.ofHoursMinutes(5, 30));
        System.out.println("zoned:  " + zoned);
        System.out.println("offset: " + offset);
        System.out.println("both are the same instant: " + zoned.toInstant().equals(offset.toInstant()));

        System.out.println();
        System.out.println("-- two different local times, one instant --");
        ZonedDateTime ny = ZonedDateTime.of(2026, 10, 12, 5, 30, 0, 0, ZoneId.of("America/New_York"));
        ZonedDateTime delhi = ZonedDateTime.of(2026, 10, 12, 15, 0, 0, 0, ZoneId.of("Asia/Kolkata"));
        System.out.println("05:30 New York  -> " + ny.toInstant());
        System.out.println("15:00 Kolkata   -> " + delhi.toInstant());
        System.out.println("same moment? " + ny.toInstant().equals(delhi.toInstant()) + "   isEqual: " + ny.isEqual(delhi));

        System.out.println();
        System.out.println("-- Clock: make 'now' injectable (this is what Day 39 uses to test time) --");
        Clock fixed = Clock.fixed(Instant.parse("2026-10-12T00:00:00Z"), ZoneId.of("UTC"));
        System.out.println("LocalDate.now(fixedClock) = " + LocalDate.now(fixed) + "   (always, in any test, on any day)");
        System.out.println("Instant.now(fixedClock)   = " + Instant.now(fixed));
    }
}
```

```
Instant (always UTC): 2026-10-12T09:30:00Z
epoch millis:         1791797400000

-- the SAME instant, three zones (practice) --
Asia/Kolkata      2026-10-12 15:00  +05:30
America/New_York  2026-10-12 05:30  -04:00
Europe/London     2026-10-12 10:30  +01:00
Asia/Tokyo        2026-10-12 18:30  +09:00

-- ZonedDateTime vs OffsetDateTime --
zoned:  2026-10-12T15:00+05:30[Asia/Kolkata]
offset: 2026-10-12T15:00+05:30
both are the same instant: true

-- two different local times, one instant --
05:30 New York  -> 2026-10-12T09:30:00Z
15:00 Kolkata   -> 2026-10-12T09:30:00Z
same moment? true   isEqual: true

-- Clock: make 'now' injectable (this is what Day 39 uses to test time) --
LocalDate.now(fixedClock) = 2026-10-12   (always, in any test, on any day)
Instant.now(fixedClock)   = 2026-10-12T00:00:00Z
```

The **practice exercise** (one instant into several zones) is the second block of
output: `09:30Z` is `15:00` in Kolkata, `05:30` in New York, `10:30` in London (London is
on summer time, +01:00, until late October), `18:30` in Tokyo. One moment, four
different clock readings.

**Zone vs offset** — the distinction that matters for the DST section:

| | `ZonedDateTime` | `OffsetDateTime` |
|---|-----------------|------------------|
| Carries | A **zone** (`Asia/Kolkata`) *and* the current offset | Only a **fixed offset** (`+05:30`) |
| Knows DST rules? | **Yes** — adding a day across a DST change works correctly | **No** — it's just a number |
| Use for | Arithmetic and display in a place | **Transport & storage** of a moment with its offset (JSON, Postgres) |
| Prints | `…+05:30[Asia/Kolkata]` | `…+05:30` |

Practical guidance:

- Use a **region name** (`ZoneId.of("America/New_York")`), never a fixed abbreviation
  like `EST` — abbreviations are ambiguous and ignore DST.
- **`ZoneId.systemDefault()`** (what `now()` with no arguments uses) is the server's zone
  — *different on every machine*. A service whose behaviour changes when it's deployed to
  another region is a bug. Run services with `-Duser.timezone=UTC` (or `TZ=UTC` in the
  container) and never depend on the default zone in business logic.
- **`Clock`** is the injectable "now". `LocalDate.now(clock)`, `Instant.now(clock)`. In
  production you pass `Clock.systemUTC()`; in tests `Clock.fixed(...)`. This is the
  technique **Day 39 and 40** rely on to make time-dependent code testable (loan due
  dates!) — notice you can't test `LocalDate.now()` because it's different every day.

> **Say this in an interview:** "An `Instant` is a point on the timeline; a
> `ZonedDateTime` is that instant viewed on a particular zone's wall clock, with DST
> rules; an `OffsetDateTime` has just a fixed offset. I inject a `Clock` instead of
> calling `now()` so time is testable."

---

## Part 5 — `Duration` vs `Period`

**In plain words:** "How long?" has two honest answers. *"8 hours and 45 minutes"* is an
exact count of seconds — a **`Duration`**. *"One month"* or *"a year"* isn't a fixed
number of seconds (a month is 28–31 days, a day can be 23–25 hours) — it's an
instruction about the calendar — a **`Period`**. Use the one that matches what you mean.

```java
import java.time.Duration;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.Period;
import java.time.ZoneId;
import java.time.ZonedDateTime;

public class DurationPeriod {
    public static void main(String[] args) {
        System.out.println("-- Duration: an exact amount of time (hours/minutes/seconds) --");
        LocalDateTime start = LocalDateTime.of(2026, 10, 12, 9, 0);
        LocalDateTime end = LocalDateTime.of(2026, 10, 12, 17, 45);
        Duration worked = Duration.between(start, end);
        System.out.println("worked: " + worked + "  = " + worked.toHours() + "h " + worked.toMinutesPart() + "m  (" + worked.toMinutes() + " minutes)");
        System.out.println("Duration.ofMinutes(90) = " + Duration.ofMinutes(90));

        System.out.println();
        System.out.println("-- Period: a calendar amount (years/months/days) --");
        LocalDate born = LocalDate.of(1995, 3, 20);
        LocalDate onDay = LocalDate.of(2026, 10, 12);
        Period age = Period.between(born, onDay);
        System.out.println("age: " + age + "  = " + age.getYears() + " years, " + age.getMonths() + " months, " + age.getDays() + " days");
        System.out.println("Period.ofMonths(1) added to Jan 31: " + LocalDate.of(2026, 1, 31).plus(Period.ofMonths(1)));

        System.out.println();
        System.out.println("-- 'one day' is not always 24 hours --");
        ZoneId ny = ZoneId.of("America/New_York");
        ZonedDateTime saturdayNoon = ZonedDateTime.of(2026, 3, 7, 12, 0, 0, 0, ny);
        ZonedDateTime plusPeriod = saturdayNoon.plus(Period.ofDays(1));
        ZonedDateTime plusDuration = saturdayNoon.plus(Duration.ofHours(24));
        System.out.println("start:                 " + saturdayNoon);
        System.out.println("plus Period.ofDays(1): " + plusPeriod + "   (same wall-clock time)");
        System.out.println("plus Duration 24h:     " + plusDuration + "   (exactly 24h later; clocks sprang forward)");
        System.out.println("real hours between start and +1 calendar day: " + Duration.between(saturdayNoon, plusPeriod).toHours());
    }
}
```

```
-- Duration: an exact amount of time (hours/minutes/seconds) --
worked: PT8H45M  = 8h 45m  (525 minutes)
Duration.ofMinutes(90) = PT1H30M

-- Period: a calendar amount (years/months/days) --
age: P31Y6M22D  = 31 years, 6 months, 22 days
Period.ofMonths(1) added to Jan 31: 2026-02-28

-- 'one day' is not always 24 hours --
start:                 2026-03-07T12:00-05:00[America/New_York]
plus Period.ofDays(1): 2026-03-08T12:00-04:00[America/New_York]   (same wall-clock time)
plus Duration 24h:     2026-03-08T13:00-04:00[America/New_York]   (exactly 24h later; clocks sprang forward)
real hours between start and +1 calendar day: 23
```

| | `Duration` | `Period` |
|---|-----------|----------|
| Measures | Exact time: seconds and nanoseconds | Calendar: years, months, days |
| Prints as | `PT8H45M` (P = period, T = time part) | `P31Y6M22D` |
| Works with | `Instant`, `LocalTime`, `LocalDateTime`, `ZonedDateTime` | `LocalDate`, `ZonedDateTime`, `LocalDateTime` (date part) |
| Is "a day" 24h? | Yes, by definition | No — "the same time on the next calendar day" |
| Use for | Timeouts, elapsed time, TTLs, cache expiry, SLA minutes | Ages, subscriptions ("renews monthly"), "30 days from signup" |

The last block is the key surprise. From Saturday noon, **`Period.ofDays(1)`** lands on
Sunday noon — which is only **23 real hours** later, because the clocks sprang forward
that night. **`Duration.ofHours(24)`** lands on Sunday 13:00. Neither is wrong; they
mean different things. "Remind me in 24 hours" is a `Duration`. "Same time tomorrow" is
a `Period`.

> **Say this in an interview:** "A `Duration` is an exact amount of time in seconds; a
> `Period` is a calendar amount of years, months and days. They differ across daylight
> saving and month ends, so I use `Duration` for timeouts and elapsed time and `Period`
> for human calendar spans."

---

## Part 6 — Daylight saving: the bug and the fix

**In plain words:** twice a year, many regions change their clocks. In spring an hour
**disappears** (02:00 jumps straight to 03:00, so there *is no* 02:30 that night). In
autumn an hour **repeats** (01:00–02:00 happens twice). A `LocalDateTime` is just
numbers on a clock face — it has no idea any of this is happening, so it happily
describes times that never existed, can't distinguish a time that happened twice, and
does hour arithmetic that's wrong by an hour. A `ZonedDateTime` knows the rules.

All of it, for New York in 2026 (spring forward **8 March**, fall back **1 November**):

```java
import java.time.Duration;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.time.ZonedDateTime;

public class DstBug {
    public static void main(String[] args) {
        ZoneId ny = ZoneId.of("America/New_York");

        System.out.println("-- Bug 1: a local time that does not exist (spring forward: 02:00 -> 03:00 on 2026-03-08) --");
        LocalDateTime wanted = LocalDateTime.of(2026, 3, 8, 2, 30);
        System.out.println("LocalDateTime says:   " + wanted + "   (no complaint, but there was no 02:30 in New York that day)");
        ZonedDateTime fixed = wanted.atZone(ny);
        System.out.println("ZonedDateTime says:   " + fixed + "   (shifted to the next valid time)");

        System.out.println();
        System.out.println("-- Bug 2: 'add 2 hours' across the same night --");
        LocalDateTime lateNight = LocalDateTime.of(2026, 3, 8, 1, 30);
        System.out.println("LocalDateTime 01:30 + 2h = " + lateNight.plusHours(2) + "   (clock time only; 03:30 is only ONE real hour later)");
        ZonedDateTime lateNightZoned = lateNight.atZone(ny);
        ZonedDateTime realTwoHours = lateNightZoned.plusHours(2);
        System.out.println("ZonedDateTime  01:30 + 2h = " + realTwoHours.toLocalDateTime() + "  offset " + realTwoHours.getOffset()
                + "   real elapsed: " + Duration.between(lateNightZoned, realTwoHours).toHours() + "h");

        System.out.println();
        System.out.println("-- Bug 3: a time that happens TWICE (fall back: 02:00 -> 01:00 on 2026-11-01) --");
        LocalDateTime twice = LocalDateTime.of(2026, 11, 1, 1, 30);
        ZonedDateTime first = twice.atZone(ny).withEarlierOffsetAtOverlap();
        ZonedDateTime second = twice.atZone(ny).withLaterOffsetAtOverlap();
        System.out.println("01:30 first time:  " + first);
        System.out.println("01:30 second time: " + second);
        System.out.println("they are " + Duration.between(first, second).toHours() + " hour apart; a LocalDateTime cannot tell them apart");

        System.out.println();
        System.out.println("-- Bug 4: the 'every day at 09:00' job --");
        ZonedDateTime saturday9 = ZonedDateTime.of(2026, 3, 7, 9, 0, 0, 0, ny);
        ZonedDateTime sundayBad = saturday9.plus(Duration.ofHours(24));
        ZonedDateTime sundayGood = saturday9.plusDays(1);
        System.out.println("plus 24 hours: " + sundayBad.toLocalTime() + "   <- the job drifts to 10:00 after DST");
        System.out.println("plusDays(1):   " + sundayGood.toLocalTime() + "   <- stays at 09:00 local");
    }
}
```

```
-- Bug 1: a local time that does not exist (spring forward: 02:00 -> 03:00 on 2026-03-08) --
LocalDateTime says:   2026-03-08T02:30   (no complaint, but there was no 02:30 in New York that day)
ZonedDateTime says:   2026-03-08T03:30-04:00[America/New_York]   (shifted to the next valid time)

-- Bug 2: 'add 2 hours' across the same night --
LocalDateTime 01:30 + 2h = 2026-03-08T03:30   (clock time only; 03:30 is only ONE real hour later)
ZonedDateTime  01:30 + 2h = 2026-03-08T04:30  offset -04:00   real elapsed: 2h

-- Bug 3: a time that happens TWICE (fall back: 02:00 -> 01:00 on 2026-11-01) --
01:30 first time:  2026-11-01T01:30-04:00[America/New_York]
01:30 second time: 2026-11-01T01:30-05:00[America/New_York]
they are 1 hour apart; a LocalDateTime cannot tell them apart

-- Bug 4: the 'every day at 09:00' job --
plus 24 hours: 10:00   <- the job drifts to 10:00 after DST
plusDays(1):   09:00   <- stays at 09:00 local
```

This is the plan's DST exercise: demonstrate with `LocalDateTime`, fix with
`ZonedDateTime`. What each bug teaches:

| Bug | `LocalDateTime` | `ZonedDateTime` |
|-----|-----------------|-----------------|
| **1. The gap** | Accepts 02:30, a time that never existed | Moves it to the next valid time (03:30 with the new offset) |
| **2. Elapsed time** | `+2h` gives 03:30, but only **1** real hour passed | `+2h` gives 04:30: **2** real hours passed |
| **3. The overlap** | 01:30 is ambiguous: *which* 01:30? | Resolve explicitly: `withEarlierOffsetAtOverlap()` / `withLaterOffsetAtOverlap()` — one hour apart |
| **4. Recurring daily job** | n/a | `plus(Duration.ofHours(24))` drifts to 10:00; **`plusDays(1)`** keeps 09:00 |

The principles:

1. **Never use `LocalDateTime` to represent an event that has to happen at one real
   moment.** A meeting, a payment, a log entry. Use `Instant` or `ZonedDateTime`.
2. **Do date arithmetic in a `ZonedDateTime`,** and pick the right unit: *calendar* units
   (`plusDays`, `plusMonths`) for "same time tomorrow", *exact* units (`plusHours`,
   `Duration`) for "elapsed time".
3. For a *future* appointment at a local time ("9:00 in Berlin on 3 March"), store the
   **local date-time plus the zone ID**, not an `Instant` — governments change DST rules,
   and if they do, "9:00 Berlin" should still be 9:00 Berlin. (A *past* event you convert
   to an `Instant`.)
4. Where is it safe to use `LocalDateTime`? As a **display or form value** ("the user
   typed 9:30"), immediately paired with a zone before it means anything — and as the
   *local part* of Rule 3.

> **Say this in an interview:** "A `LocalDateTime` has no zone, so across a DST change it
> can represent a time that doesn't exist or happens twice, and hour arithmetic gives the
> wrong elapsed time. For anything that is a real moment I use `Instant` or
> `ZonedDateTime`, which apply the zone's rules."

---

## Part 7 — Formatting and parsing

**In plain words:** computers exchange dates as text — in JSON, CSV, logs, forms. You
convert with a **`DateTimeFormatter`**: a pattern like `dd/MM/yyyy` describing where each
part goes. For anything machine-to-machine, use the **ISO-8601** standard format and
you often need no formatter at all: `toString()` and `parse()` already speak it.

```java
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;
import java.time.format.DateTimeParseException;
import java.util.Locale;

public class Formatting {
    public static void main(String[] args) {
        LocalDateTime dt = LocalDateTime.of(2026, 10, 12, 9, 5, 7);

        System.out.println("-- formatting --");
        System.out.println("ISO (default toString): " + dt);
        System.out.println("ISO_LOCAL_DATE:         " + dt.format(DateTimeFormatter.ISO_LOCAL_DATE));
        System.out.println("custom:                 " + dt.format(DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss")));
        System.out.println("12-hour:                " + dt.format(DateTimeFormatter.ofPattern("d MMM yyyy, h:mm a", Locale.ENGLISH)));
        System.out.println("long, French:           " + dt.format(DateTimeFormatter.ofPattern("EEEE d MMMM yyyy", Locale.FRENCH)));
        System.out.println("zoned ISO:              " + ZonedDateTime.of(dt, ZoneId.of("Asia/Kolkata")).format(DateTimeFormatter.ISO_OFFSET_DATE_TIME));

        System.out.println();
        System.out.println("-- parsing --");
        LocalDate parsed = LocalDate.parse("12/10/2026", DateTimeFormatter.ofPattern("dd/MM/yyyy"));
        System.out.println("parsed dd/MM/yyyy -> " + parsed);
        System.out.println("ISO parse         -> " + LocalDate.parse("2026-10-12"));
        try {
            LocalDate.parse("12/10/2026");
        } catch (DateTimeParseException e) {
            System.out.println("wrong format      -> DateTimeParseException: " + e.getMessage());
        }

        System.out.println();
        System.out.println("-- pattern letter traps --");
        LocalDate newYearsEve = LocalDate.of(2026, 12, 31);
        System.out.println("yyyy-MM-dd for Dec 31 2026: " + newYearsEve.format(DateTimeFormatter.ofPattern("yyyy-MM-dd", Locale.US)));
        System.out.println("YYYY-MM-dd for Dec 31 2026: " + newYearsEve.format(DateTimeFormatter.ofPattern("YYYY-MM-dd", Locale.US)) + "   <- week-year! wrong year");
        LocalDateTime t = LocalDateTime.of(2026, 10, 12, 9, 5);
        System.out.println("HH:MM  (MM = month!):       " + t.format(DateTimeFormatter.ofPattern("HH:MM")));
        System.out.println("HH:mm  (mm = minutes):      " + t.format(DateTimeFormatter.ofPattern("HH:mm")));
    }
}
```

```
-- formatting --
ISO (default toString): 2026-10-12T09:05:07
ISO_LOCAL_DATE:         2026-10-12
custom:                 12/10/2026 09:05:07
12-hour:                12 Oct 2026, 9:05 AM
long, French:           lundi 12 octobre 2026
zoned ISO:              2026-10-12T09:05:07+05:30

-- parsing --
parsed dd/MM/yyyy -> 2026-10-12
ISO parse         -> 2026-10-12
wrong format      -> DateTimeParseException: Text '12/10/2026' could not be parsed at index 0

-- pattern letter traps --
yyyy-MM-dd for Dec 31 2026: 2026-12-31
YYYY-MM-dd for Dec 31 2026: 2027-12-31   <- week-year! wrong year
HH:MM  (MM = month!):       09:10
HH:mm  (mm = minutes):      09:05
```

The pattern letters worth knowing:

| Letter | Means | Careful |
|--------|-------|---------|
| `yyyy` | Year | **Not `YYYY`** — that's the *week-based* year; it shows **2027** for 31 Dec 2026 (in the US locale) and has caused real production outages around new year |
| `MM` | Month (01–12) | `mm` is **minutes**. `HH:MM` printed `09:10` — "10" is the month |
| `dd` | Day of month | `DD` is day of the *year* |
| `HH` / `hh` | Hour 00–23 / 01–12 | Pair `hh` with `a` (AM/PM) |
| `mm`, `ss` | Minutes, seconds | |
| `MMM` / `EEEE` | Month / weekday names | **Locale-dependent** — pass a `Locale` explicitly |
| `xxx` / `VV` | Offset (`+05:30`) / zone id | `z` names are locale-dependent too |

Practical rules:

- **Machine-to-machine: use ISO-8601 with `Z` or an offset.** `Instant.toString()` gives
  `2026-10-12T09:30:00Z`. In **Spring Boot 3, Jackson writes `java.time` types as ISO
  strings by default** (not numbers) — keep it that way.
- **Format at the edge** (UI, report, email), keep ISO and `Instant` everywhere inside.
- `DateTimeFormatter` constants are **immutable and thread-safe** — store them in
  `static final` fields. (The old `SimpleDateFormat` was neither — a classic bug.)
- Parsing a wrong format throws `DateTimeParseException` — catch it at the boundary
  where users type text.
- A date-only pattern can't parse a date-time string (and vice-versa) — parse into the
  matching type.

---

## Part 8 — The rule: store UTC, show local

**In plain words:** the physical world has many clocks; your data should have one. Keep
every stored moment in a single reference (UTC), because that's unambiguous and
sortable, and convert to someone's local clock only at the last step — when you show it
to them. It's the date-and-time version of "keep money as cents and format at the edge".

**Which type for which job:**

| You are storing… | Use | In Postgres (Phase 7) | In JSON |
|------------------|-----|-----------------------|---------|
| *When something happened / will happen at a moment* (`createdAt`, order placed, token expiry, event time) | **`Instant`** (UTC) | **`timestamptz`** | `"2026-10-12T09:30:00Z"` |
| A **calendar date** with no time (birthday, due date, invoice date) | `LocalDate` | `date` | `"2026-10-12"` |
| A **time of day** with no date (opening hours) | `LocalTime` | `time` | `"09:30:00"` |
| A **future appointment at a local time in a place** | `LocalDateTime` **+ `ZoneId`** (two fields) | `timestamp` + `text` zone | two fields |
| **The user's own zone** (for display) | `ZoneId` (e.g. `"Asia/Kolkata"`) in their profile | `text` | `"Asia/Kolkata"` |
| An amount of elapsed time | `Duration` (store as seconds/ISO) | `interval` or `bigint` seconds | `"PT90M"` |

The rules, in order of how much pain they save:

1. **Store and transmit `Instant` (UTC).** Never store a "local" time for something that
   happened.
2. **Convert to a zone only to display it,** using the **user's** zone (from their
   profile or browser), not the server's.
3. **Run servers with `TZ=UTC`** and never call `ZoneId.systemDefault()` in business
   logic.
4. **Use `ZonedDateTime` for arithmetic that crosses a day boundary in a zone.**
5. **Inject a `Clock`** — never scatter `Instant.now()` around code you want to test.
6. **Don't store offsets-as-zones.** `+05:30` is not `Asia/Kolkata`; the offset can't
   tell you what happens at the next DST change.
7. **Postgres `timestamptz` does *not* store a zone** — it converts the input to UTC and
   stores an instant, then renders it in the session's zone. It's exactly an `Instant`.
   `timestamp` (without time zone) is exactly a `LocalDateTime` — the type to be wary of.
   (Phase 7 depends on this — remember where it came from.)

> **Say this in an interview:** "I store moments as UTC instants — `timestamptz` in
> Postgres, ISO-8601 with `Z` in JSON — and convert to the user's zone only for display.
> I use `ZonedDateTime` for calendar arithmetic across DST, store a local date-time plus a
> zone for future local-time appointments, and inject a `Clock` so time-dependent code
> is testable."

---

## Hands-on

1. **Required — days until, and a business week.** Run `LocalTypes`. Change `today` and
   `deadline` to a real date you care about (a holiday, an exam, a release) and print the
   days, weeks and months (`ChronoUnit.MONTHS`). Then extend `addBusinessDays` to take a
   `Set<LocalDate>` of holidays and skip those too; test it with one holiday landing inside
   the week.
2. **Required — three zones.** Run `InstantAndZones`. Replace the instant with *right
   now* (`Instant.now()`) and add a zone you actually care about (yours, a colleague's, a
   customer's). Then pick a time you'd propose for a meeting (e.g. 16:00 in your zone) and
   print what that is in the others — "is it at a sociable hour for everyone?"
3. **Required — the DST bug and fix.** Run `DstBug`. Then write your own version for a
   different year: look up the fall-back date for `Europe/London` (last Sunday of October)
   and demonstrate the overlap. Explain Bug 4 in your own words — why does `plusDays(1)`
   hold the time but `plus(Duration.ofHours(24))` doesn't?
4. Run `OldVsNew` and add a fourth case of your own: print a `new Date()` and a
   `Instant.now()` — and explain why `Date` shows a zone abbreviation but is "really" an
   instant.
5. Write a `static Instant parseUserInput(String text, ZoneId userZone)` that turns
   `"12/10/2026 09:30"` (typed by a user in their zone) into the correct `Instant`.
   Call it with Kolkata and New York and observe the two different instants.
6. Show the `YYYY` bug on 31 December 2026. Then loop over every day of 2026 and count
   how many dates format differently with `YYYY` vs `yyyy` — which part of the year are
   they all in?
7. Use `Clock.fixed` to write a method `boolean isOverdue(LocalDate due, Clock clock)`
   and test it with three different "todays" without changing your computer's date.
8. Read the Postgres docs section "Date/Time Types" and write down, in your own words, what
   `timestamptz` stores (Phase 7 will need this).

---

## Gotchas

- **`LocalDateTime.now()`** gives the *server's* wall-clock time with no zone — it looks
  right on your laptop and wrong in a UTC container. Prefer `Instant.now()` or
  `ZonedDateTime.now(zone)`.
- **`ZoneId.of("EST")` / `"PST"`** — three-letter abbreviations are ambiguous and ignore
  DST. Use region IDs.
- **`plusMonths` clamps** (Jan 31 → Feb 28) and does *not* round-trip: `+1 month −1 month`
  from Jan 31 gives Jan 28.
- **`ChronoUnit.DAYS.between(a, b)` is a whole-day count** — two moments 23 hours apart can
  be 0 days. Use `Duration` for hours.
- **`Duration` cannot be added to a `LocalDate`** (throws `UnsupportedTemporalTypeException`)
  — a date has no hours. Use `Period`.
- **`java.time` instants have nanosecond precision**, Postgres `timestamptz` has
  microseconds, JavaScript milliseconds — round-tripping can change the last digits;
  tests that compare for exact equality after a database trip can fail mysteriously
  (truncate to millis or micros first).
- **`equals` on `ZonedDateTime`** compares the zone too; two `ZonedDateTime`s for the same
  instant in different zones are *not* `equals`, but `isEqual` says they are (shown in the
  demo).
- **Leap seconds and the year 2038** are not your problem: `Instant` handles them.
- **Don't store epoch *seconds* as `int`** — overflow in 2038. Use `long` or `Instant`.
- **`new Date().getYear()`** is `year − 1900`. Just don't use it.
- **A date string with no zone, parsed on a server in a different region, is the classic
  "off by a day" bug.** Always know which zone a date-time string was written in.

---

## Recap

- `Date`/`Calendar` are mutable, zero-indexed, lenient, and the formatter isn't
  thread-safe. `java.time` is immutable, strict and thread-safe.
- **`Instant`** = a moment (UTC). **`Local…`** = what a calendar/clock says, with *no
  zone*. **`ZonedDateTime`** = a moment on a zone's clock, with DST rules.
  **`OffsetDateTime`** = a moment with a fixed offset — good for transport.
- Everything is immutable: `plus…`/`with…` return new objects.
- **`Duration`** = exact seconds; **`Period`** = calendar units. A "day" is 23, 24 or 25
  hours depending on DST.
- A `LocalDateTime` can't represent a real moment: DST creates times that don't exist and
  times that happen twice. Do zone-aware arithmetic with `ZonedDateTime`.
- `DateTimeFormatter`: `yyyy` not `YYYY`, `MM` vs `mm`, pass a `Locale`; prefer ISO-8601.
- **Store UTC (`Instant` / `timestamptz`), display local, run servers in UTC, inject a
  `Clock`.**

---

## Week 6 review — Java 8: lambdas and functional programming

Five days: lambdas → the toolkit → method references → defaults → time. Here is how they
form one story.

| Day | The idea | The trap |
|-----|----------|----------|
| **26 Lambdas** | A lambda is an implementation of a **functional interface** (one abstract method), typed by its *target*. Captured locals must be **effectively final** because the lambda holds a *copy*. `this` is the enclosing object | Mutating a captured local; thinking a lambda is an anonymous class (no `.class` file, different `this`) |
| **27 The toolkit** | Pick by **shape**: `T→R` Function, `T→boolean` Predicate, `()→T` Supplier, `T→void` Consumer. Compose with `andThen`/`and`/`negate`. A **Supplier is a recipe** — that's why `orElseGet` is lazy | `andThen` vs `compose`; side effects in Function/Predicate; boxing in hot paths |
| **28 Method references** | Four kinds: static, bound, **unbound (first param = receiver)**, constructor. Your own functional interface when you need a name or a `throws` | Ambiguous `Integer::toString`; bound `null` receiver fails at creation; checked exceptions in lambdas |
| **29 Default & static methods** | A default adds behaviour to an interface without breaking implementors — which is why `and`, `thenComparing` and `stream()` exist. Resolution: **class wins → most specific interface → you decide** | Diamond; statics aren't inherited |
| **30 java.time** | **Instant** = a moment; **Local…** = clock-face, no zone; **Zoned** = a moment with rules. Store UTC, show local | `LocalDateTime` across DST; `YYYY`; `Duration` vs `Period` |

**The thread that connects them.** Days 26–29 are one idea seen four ways: *behaviour is a
value*. A lambda is a value; the functional interfaces give it a type; method references
let you name existing behaviour; default methods let interfaces carry the combinators
(`andThen`, `and`, `reversed`) that make those values composable. Week 7's streams are
nothing but a long chain of these — `filter(Predicate)`, `map(Function)`,
`reduce(BinaryOperator)`, `collect(Collector)` — so if Day 27's shapes feel solid,
streams will feel like a small step.

Day 30 is the odd one out topically, but it's the same discipline as Week 5: **choose the
right type so the compiler or the model makes the bug impossible** — `Instant` vs
`LocalDateTime` is `Optional` vs `null`, and `final` vs mutable.

### Check yourself — ten questions

1. What two things must be true for a lambda to compile? (26)
2. Why must a captured local be effectively final? Use the words *stack frame*. (26)
3. Name the interface for: `s -> s.isEmpty()`, `() -> new ArrayList<>()`,
   `(a, b) -> a + b`, `x -> list.add(x)`. (27)
4. Why is `orElseGet` lazy and `orElse` not? (27)
5. Write the lambda equivalent of `String::compareToIgnoreCase` as a `Comparator`. (28)
6. How do you handle a checked exception inside a lambda, and what must you preserve? (28)
7. Two interfaces provide the same `default` method. What happens, and how do you fix it? (29)
8. Why did adding `stream()` to `Collection` need default methods? (29)
9. When is `Duration.ofHours(24)` different from `Period.ofDays(1)`? (30)
10. What do you store for "when the order was placed", and why not a `LocalDateTime`? (30)

> **A quiz is available** and this week's material is load-bearing for Week 7. Say
> "quiz" and I'll create `day-30-week-6-quiz.md`.

---

## Say it out loud

1. "What's the difference between `Instant`, `LocalDateTime` and `ZonedDateTime`?"
2. "Why is `LocalDateTime` dangerous for scheduling, and what do you use instead?"
3. "`Duration` versus `Period` — explain with a daylight-saving example."
4. "What's the rule for storing times, and what maps to it in Postgres?"
5. "Why does `yyyy` vs `YYYY` matter?"
6. "Why inject a `Clock` instead of calling `now()`?"

---

## Q&A

_Follow-up questions from this lesson are appended here._
