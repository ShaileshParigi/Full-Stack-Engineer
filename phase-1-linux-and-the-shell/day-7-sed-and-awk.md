# Day 7 — `sed` & `awk`

_Why this matters:_ Day 6 taught you to **find** lines. Today you **change**
them and **compute** over them. `sed` is how config gets rewritten when nobody
is there to open an editor — the `RUN sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/'`
in a Dockerfile, the CI step that bumps a version string, the entrypoint script
that swaps `localhost` for a real database host. `awk` is how you answer "which
client is hammering us?", "how many bytes did we serve per status code?" and
"which endpoint is slow?" from a raw access log in thirty seconds, on a box that
has no Elasticsearch, no Grafana and no time. These two show up in every
Dockerfile in Phase 4 and every Kubernetes init script in Phase 17.

---

## The one-paragraph version

`sed` is a **stream editor**: text flows through it one line at a time, it
applies your edit — almost always a find-and-replace written `s/old/new/` — and
prints the result. By default it **never touches the file**; you only see the
edited copy on screen. Add `-i` and it rewrites the file for real, with no undo,
which is why you always do a dry run first and keep a backup. `awk` is a small
programming language built around one idea: **for each line, split it into
columns, then run your "if this, do that" rules against it**. Columns are
`$1`, `$2`, … and `$NF` is the last one. Two special rules run once — `BEGIN`
before the first line and `END` after the last — and that's where totals get
printed. awk's arrays are lookup tables keyed by any string, basically a
`HashMap`, so "count requests per IP" or "sum bytes per status code" is one
line. **Rule of thumb: `grep` finds lines, `sed` edits lines, `awk` does anything
involving columns or arithmetic.**

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **stream editor** | Edits text as it flows past, one line at a time, never loading the whole file | `sed`: applies a script to each input line and writes the result to stdout |
| **script** | The list of commands you give `sed` or `awk` | The program text, normally single-quoted on the command line |
| **pattern space** | `sed`'s workbench — the line it's currently editing | The buffer holding the current input line (without its newline) while commands run |
| **address** | Which lines a `sed` command applies to | A line number, `$`, a `/regex/`, or a range `start,end`; negated with `!` |
| **range** | "From this line through that line" | A two-address selection, inclusive at both ends |
| **substitution** | Find-and-replace | `s/regex/replacement/flags` |
| **flag** | A letter after the substitution that changes how it behaves | `g` every match, `2` the second match, `I` ignore case, `p` print if changed |
| **delimiter** | The character separating the three parts of `s///` | Any character following `s`; `/` only by convention |
| **in-place edit** | Change the actual file instead of printing a copy | `-i`: write to a temp file, then rename it over the original |
| **inode** | The file's real identity on disk; the filename is just a label pointing at it | The on-disk object holding a file's metadata and data blocks |
| **hard link** | A second filename pointing at the same inode | An extra directory entry for an existing inode |
| **idempotent** | Running it twice leaves the same result as running it once | f(f(x)) = f(x) — the property every provisioning step needs |
| **record** | awk's word for a line | One unit of input, separated by `RS` (newline by default) |
| **field** | One column of a line | A piece of the record split by `FS`: `$1` … `$NF` |
| **`FS` / `OFS`** | What separates columns coming in / going out | Field Separator (default: runs of whitespace) / Output Field Separator (default: one space) |
| **`NR` / `NF`** | Which line number we're on / how many columns this line has | Number of Records read so far / Number of Fields in the current record |
| **pattern → action** | An "if this, do that" rule | `pattern { action }` — no pattern means every line; no action means `print` |
| **`BEGIN` / `END`** | Code that runs before the first line / after the last one | Special patterns that match before input is read and after EOF |
| **associative array** | A lookup table keyed by any string — awk's `HashMap` | String-indexed array, created on first reference, iterated in no fixed order |
| **strnum** | A field that *looks* like a number, so awk compares it like one | An input string that looks numeric: compared numerically with numbers, as text with quoted strings |
| **`printf`** | Print with a format template, like Java's `String.format` | Formatted output; adds **no** newline unless you write `\n` |

---

## Before you read: three questions

1. You run `sed 's/8080/9090/' application.properties`. Afterwards, what port
   does the file on disk say?
2. `echo "a,b,c" | awk '{print $2}'` — what prints?
3. You ran `sed -i` with the wrong pattern and mangled a config file. How do you
   get the original back?

---

## Part 1 — `sed`: the model and the substitute command

**In plain words:** picture a conveyor belt. Each line of the file rides past
`sed`, `sed` applies its edits to that one line, and drops it onto an output
belt — your screen. The original file is never touched unless you explicitly
tell `sed` to write back (`-i`, Part 3). So your first `sed` commands are
completely safe: they're previews.

Precisely, for every input line `sed`:

1. reads the line into the **pattern space** (dropping its newline),
2. runs each command in the script, in order, against the pattern space,
3. prints the pattern space — unless you passed `-n`,
4. moves to the next line.

Every command has the shape `[address]command`. The one you'll use 90% of the
time is `s` — substitute.

```
sed 's/FIND/REPLACE/FLAGS' file
```

### Flags

| Command | Effect |
|---|---|
| `s/cat/dog/` | Replace the **first** `cat` on each line |
| `s/cat/dog/g` | Replace **every** `cat` on each line |
| `s/cat/dog/2` | Replace only the **second** `cat` on each line |
| `s/cat/dog/I` | Case-insensitive (GNU `sed`) |
| `sed -n 's/cat/dog/p'` | Print **only** the lines where a replacement happened |

Without `g`, only the **first match on each line** changes. Forgetting `g` is
the first `sed` bug everyone writes.

```bash
echo "cat cat cat" | sed 's/cat/dog/'      # dog cat cat
echo "cat cat cat" | sed 's/cat/dog/g'     # dog dog dog
echo "cat cat cat" | sed 's/cat/dog/2'     # cat dog cat
```

### The regex is Day 6's regex — including the dialect trap

The FIND side is a regular expression. Like `grep`, plain `sed` speaks **BRE**,
so `+`, `?`, `( )`, `{ }` and the pipe need backslashes. **Use `sed -E`**, same
rule as `grep -E`.

### Using what you matched: `&` and `\1`

In the REPLACE side:

- **`&`** = the entire matched text.
- **`\1`, `\2`, …** = what the first, second, … parenthesised group matched.

```bash
echo "port=8080" | sed 's/[0-9][0-9]*/[&]/'                        # port=[8080]
echo "2026-09-11" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3.\2.\1/'   # 11.09.2026
```

### Pick a different delimiter

The character right after `s` is the delimiter. It doesn't have to be `/`. For
anything containing slashes — paths, URLs, JDBC strings — use `|`, `#` or `,`:

```
sed 's/\/usr\/local\/app/\/opt\/app/' f     # "leaning toothpick syndrome"
sed 's|/usr/local/app|/opt/app|' f          # same thing, readable
```

### Three characters are special in REPLACE — and they bite

The replacement is *not* a regex, but three characters still mean something
there: **`&`**, **`\`** and **the delimiter**. The failures are silent:

```bash
echo 'redirect=OLD' | sed 's/OLD/a=1&b=2/'      # redirect=a=1OLDb=2   ← & inserted the match
echo 'redirect=OLD' | sed 's/OLD/a=1\&b=2/'     # redirect=a=1&b=2     ← escaped
```

This one gets worse when the replacement comes from a **shell variable** — now
you need double quotes (so the shell expands it), and whatever is *inside* the
variable can break the command:

```bash
path=/opt/app/logs
echo 'log.dir=/tmp' | sed "s/=.*/=$path/"     # sed: -e expression #1, char 9: unknown option to `s'
echo 'log.dir=/tmp' | sed "s|=.*|=$path|"     # log.dir=/opt/app/logs
```

The shell turned the first into `s/=.*/=/opt/app/logs/` — four slashes, so `sed`
saw `opt` as flags. Pick a delimiter that can't appear in the value, and if the
value is truly arbitrary (user input, passwords with `&` or `|`), `sed` is the
wrong tool — use `awk -v` (Part 4) or a real language.

> **Say this in an interview:** "`sed` is a stream editor — it reads each line
> into the pattern space, runs the script against it and prints the result, so
> by default it's non-destructive. The workhorse is `s/regex/replacement/flags`:
> without `g` it replaces only the first match per line, `&` and `\1` reuse the
> matched text, and the delimiter can be any character, which I change to `|`
> for paths. The classic bug is splicing a shell variable into the replacement —
> a `/` or `&` in the value silently breaks or corrupts the edit."

---

## Part 2 — Addresses: choosing which lines

**In plain words:** by default every command runs on every line. An **address**
in front of the command narrows that down: "only line 3", "only lines matching
ERROR", "from the line mentioning START to the line mentioning END". Same
command, targeted.

| Address | Selects |
|---|---|
| `3` | Line 3 |
| `$` | The last line |
| `2,5` | Lines 2 through 5 |
| `/ERROR/` | Lines matching the regex |
| `/START/,/END/` | From a line matching START through the next line matching END |
| `5,$` | Line 5 to the end |
| `1~2` | Every other line, starting at 1 (GNU) |
| `/regex/!` | Every line that does **not** match |
| `\|/api/health|` | A regex with a custom delimiter — for patterns containing `/` |

### The commands you'll pair with addresses

| Command | Does | Example |
|---|---|---|
| `p` | Print (use with `-n`, or lines appear twice) | `sed -n '10,20p' f` — lines 10–20 |
| `d` | Delete (don't print) | `sed '/^#/d' f` — drop comments |
| `q` | Quit — stop reading the file | `sed 5q f` — like `head -5` |
| `s` | Substitute, only on addressed lines | `sed '/^#/!s/foo/bar/' f` |
| `a text` | Append a line after (GNU one-line form) | `sed '/^\[db\]/a port=5432' f` |
| `i text` | Insert a line before | `sed '1i # generated file' f` |
| `c text` | Replace the whole line | `sed '/^version=/c version=2.0' f` |
| `=` | Print the line number | `sed -n '$=' f` — counts lines |

Combine commands with `;` or repeat `-e`: `sed -e '/^#/d' -e '/^$/d' f`.

### Printing a window of a log

This is the most useful `sed` address in incident work — pull out everything
between two timestamps:

```
sed -n '/08:15:/,/08:17:/p' access.log
```

Two gotchas with ranges:

- **The end is the first match *after* the start**, inclusive. If the end
  pattern never appears, the range runs **to the end of the file** — no error.
- **The start pattern can re-arm it.** After a range closes, the next line
  matching START opens a new one.

### Editing only inside a section

Addresses and `s` together give you surgical config edits — for example,
changing `port` only inside the `[database]` block of an INI file:

```
sed '/^\[database\]/,/^\[/ s/^port=.*/port=5433/' config.ini
```

"From the `[database]` header through the next section header, on those lines
only, replace the port."

> **Say this in an interview:** "A `sed` command can be prefixed with an address
> — a line number, `$`, a regex, or a `start,end` range, optionally negated with
> `!` — so `sed -n '/08:15/,/08:17/p'` prints a time window from a log and
> `'/^#/!s/a/b/'` edits only uncommented lines. Ranges are inclusive and, if the
> end never matches, run to EOF."

---

## Part 3 — `-i`: editing the file for real, safely

**In plain words:** `-i` tells `sed` "don't show me — change the file". There's
no undo and no recycle bin. So the professional workflow is always: preview
without `-i`, then run with `-i.bak` so a backup is kept, then diff to confirm.

```
sed 's/^server.port=.*/server.port=9090/' app.properties         # 1. preview
sed -i.bak 's/^server.port=.*/server.port=9090/' app.properties  # 2. edit, keep app.properties.bak
diff app.properties.bak app.properties                            # 3. confirm exactly what changed
```

### What `-i` actually does — it replaces the file

`sed` doesn't edit the file in place at all, whatever the flag's name says. It
writes the output to a **new temporary file** in the same directory, then
**renames** that over the original. The name is the same; the file is new — a
different **inode** (the file's real identity on disk; the name is only a label
pointing at it). Consequences you will hit:

- **Hard links break.** A second name for the old inode keeps the *old*
  content. (Hands-on 7 proves this.)
- **Symlinks get replaced** by a regular file, unless you pass
  `--follow-symlinks`.
- **A process holding the file open** — `tail -f`, or an app that read its
  config at startup — still sees the old file.
- **In Docker, `sed -i` on a bind-mounted single file fails** with
  `Device or resource busy`, because you can't rename over a mount point. You'll
  meet this in Phase 4.
- GNU `sed` copies the permissions across, but only root can preserve a
  different owner.

### Portability: GNU vs macOS

`sed -i 's/a/b/' f` works on Linux. On macOS (BSD `sed`), `-i` **requires** a
suffix argument, so the same line fails or eats your next argument. `sed -i ''`
works on macOS and fails on Linux. The portable form is to always give a suffix:
`sed -i.bak … && rm …bak`.

### Make edits idempotent

Scripts get re-run — a retried CI job, a rebuilt container. An edit is
**idempotent** if running it twice gives the same file as running it once.

| Edit | Run twice | Idempotent? |
|---|---|---|
| `s/app/app-v2/` | `app-v2-v2` | No |
| `s/8080/9090/` | fine — but also changes `18080` to `19090` | Yes, but sloppy |
| `s/^server\.port=.*/server.port=9090/` | same result | **Yes** — anchored to the whole line |

Anchor to the key and replace the whole line. And remember `sed` does **nothing,
silently**, if the line isn't there — so for a real "set this value", use the
upsert pattern:

```
grep -q '^management.port=' f && sed -i 's/^management\.port=.*/management.port=9091/' f || echo 'management.port=9091' >> f
```

"If the key exists, rewrite it; otherwise append it." Idempotent either way.

> **Say this in an interview:** "`sed -i` isn't really in-place: it writes a
> temp file and renames it over the original, so you get a new inode — which
> breaks hard links, replaces symlinks, fails on Docker bind-mounted files, and
> leaves processes that had the file open reading the old version. I preview
> without `-i`, use `-i.bak` and `diff`, and write substitutions anchored to the
> whole line so they're idempotent when a pipeline re-runs."

---

## Part 4 — `awk`: the model

**In plain words:** awk reads a line, chops it into columns, then walks through
your list of "if this, then do that" rules. Think of it as a SQL
`SELECT … WHERE` that runs over each line of a text file — plus variables that
**survive from one line to the next**, so you can keep running totals and print
them at the end.

The shape of every awk program:

```
awk 'BEGIN { setup }   pattern { action }   pattern { action }   END { report }' file
```

For each line (a **record**), awk:

1. splits it into **fields** `$1`, `$2`, … using `FS`,
2. goes through the rules in order — if a rule's pattern is true, it runs the
   action,
3. moves on.

`BEGIN` runs once before any input; `END` runs once after the last line.

Two defaults make awk short:

- **No pattern** → the action runs on every line: `awk '{print $1}'`
- **No action** → the default is `{ print }`, the whole line: `awk '/ERROR/'`
  behaves exactly like `grep ERROR`.

awk is a real language: variables, arithmetic, `if`/`else`, `for`, `while`,
`printf`, string functions (`length`, `substr`, `split`, `sub`, `gsub`,
`toupper`, `tolower`), user functions. Variables need no declaration — an unset
variable is `""` as text and `0` as a number, which is why `sum += $10` works
with no setup.

The name is its authors' initials: **A**ho, **W**einberger, **K**ernighan, 1977.

### Which awk do you have?

```bash
readlink -f "$(command -v awk)"
```

Yours resolves to **gawk** (GNU awk). A stock Ubuntu install gives you **mawk**
instead, Alpine containers give you BusyBox awk, and macOS has its own. Everything
in this lesson is POSIX awk and runs on all of them. Functions like `gensub`,
`asort` and `strftime` are gawk-only — fine at your desk, a surprise inside a
container.

### Single quotes. Always.

awk's `$1` and the shell's `$1` look identical. In double quotes, the shell gets
there first:

```bash
echo "alpha beta" | awk "{print $1}"     # prints "alpha beta" — the shell replaced $1 with nothing
echo "alpha beta" | awk '{print $1}'     # prints "alpha"
```

The shell expanded `$1` (the script's empty first argument), so awk received
`{print }` and printed the whole line. No error. Same lesson as Day 6: the shell
reads the command first.

**To get a shell value into awk, use `-v`**, never string splicing:

```
awk -v min="$THRESHOLD" '$10 > min' access.log
```

`-v` sets an awk variable before the program runs — no quoting puzzles, and
nothing in the value can be mistaken for awk code.

> **Say this in an interview:** "awk is a data-driven language: for each record
> it splits fields on `FS`, then evaluates each `pattern { action }` rule in
> order, with `BEGIN` and `END` blocks running before input and after EOF.
> Missing pattern means every record, missing action means print. Variables
> persist across records, which is what makes aggregation possible, and I pass
> shell values in with `-v` rather than interpolating them into the program."

---

## Part 5 — Fields and built-in variables

**In plain words:** awk numbers the columns for you. `$1` is the first, `$NF` is
the last — however many there are. By default columns are separated by **any
run of spaces or tabs**, which is exactly what human-formatted output like `ps`,
`df` and `ls -l` looks like.

| Variable | Meaning |
|---|---|
| `$0` | The whole line |
| `$1`, `$2`, … | Field 1, 2, … |
| `NF` | Number of fields on this line |
| `$NF` | The **last** field (`$` of the number `NF`) |
| `$(NF-1)` | The second-to-last field |
| `NR` | Line number, counted across all input |
| `FNR` | Line number within the current file |
| `FS` | Input field separator — set with `-F` |
| `OFS` | Output field separator — what `print a, b` puts between values |

### Why `awk` beats `cut` on real output

`cut -d' '` splits on **exactly one** space, so column-aligned output — padded
with several spaces — gives you empty fields. awk's default `FS` treats a
**run** of whitespace as one separator and ignores leading spaces:

```bash
printf 'root        1  /sbin/init\nshailesh  742  -bash\n' > /tmp/cols.txt
echo "--- cut -d' ' -f2:";  cut -d' ' -f2 /tmp/cols.txt
echo "--- awk '{print \$2}':"; awk '{print $2}' /tmp/cols.txt
rm -f /tmp/cols.txt
```

`cut` printed blank lines — field 2 is the empty string between two adjacent
spaces. This is why every "get the PID column" one-liner uses awk.

### Custom separators with `-F`

```
awk -F: '{print $1, $7}' /etc/passwd          # username and login shell
awk -F, '{print $3}' data.csv                  # third CSV column
awk -F'[=:]' '{print $2}' f                    # a regex: split on = or :
```

**awk is not a CSV parser.** `-F,` breaks on a quoted field containing a comma,
like `"Smith, John",42`. For real CSV, use a real parser.

### Printing: comma vs space

```bash
echo "a b c" | awk '{print $1, $3}'     # a c    ← comma inserts OFS (a space)
echo "a b c" | awk '{print $1 $3}'      # ac     ← no comma = concatenation
echo "a b c" | awk -v OFS=, '{print $1, $2, $3}'   # a,b,c
```

And one classic surprise: changing `OFS` alone doesn't reformat `$0`. Assigning
to any field forces awk to rebuild the line with `OFS`:

```bash
echo "a b c" | awk -v OFS=, '{print}'          # a b c   ← $0 untouched
echo "a b c" | awk -v OFS=, '{$1=$1; print}'   # a,b,c   ← rebuilt
```

### `$NF` survives messy lines

In an access log, the user-agent field contains spaces
(`"Mozilla/5.0 (X11; Linux x86_64)"`), so field numbers after it shift from line
to line. If the value you want is **last** — response time usually is — `$NF`
finds it regardless.

> **Say this in an interview:** "Fields are `$1` to `$NF`, `NF` is the field
> count and `NR` the record number. The default `FS` splits on runs of
> whitespace and trims leading blanks, which is why awk is more reliable than
> `cut -d' '` on column-aligned output like `ps` or `df`. `print a, b` joins with
> `OFS`; `print a b` concatenates."

---

## Part 6 — Patterns, `BEGIN` and `END`

**In plain words:** the pattern is the "WHERE" clause. It can be a regex, a
comparison on a column, or any true/false expression. `BEGIN` and `END` are the
before-and-after hooks: set things up, and report totals.

### Patterns

| Pattern | Selects lines where… |
|---|---|
| `/ERROR/` | the whole line matches the regex |
| `$9 ~ /^5/` | field 9 matches the regex (status 5xx) |
| `$9 !~ /^2/` | field 9 does **not** match |
| `$9 >= 500` | field 9, as a number, is at least 500 |
| `$1 == "10.0.3.14"` | field 1 is exactly that string |
| `$9 >= 500 && $NF > 1` | both conditions |
| `NR > 1` | every line but the first — skip a header |
| `NF` | the line isn't blank (`NF` is 0 on a blank line → false) |
| `/START/,/END/` | a range, same as `sed` |

### The number-vs-string trap

A field that *looks* like a number is compared **as a number** against a
number, but **as text** against a quoted string. Text comparison goes character
by character, so `"812" > "1000"` is **true** — `8` sorts after `1`.

```
awk '$10 > 1000'      # numeric — correct
awk '$10 > "1000"'    # string comparison — 812 counts as "bigger"
```

Worse: with fixed-width values like HTTP status codes, string comparison
*happens to give the right answer*, so the bug survives testing. **Never quote a
number in an awk comparison.**

### `BEGIN` and `END`

```
awk 'BEGIN { print "ip,status" }  { print $1 "," $9 }' access.log   # header, then rows
awk '{ sum += $10 } END { print sum }' access.log                  # total of column 10
awk '{ sum += $NF } END { if (NR) printf "%.3f\n", sum / NR }' access.log   # average — guarded
```

- `sum` starts at 0 with no declaration.
- `END` sees the final values of your variables, and `NR` equals the total
  number of lines.
- **Guard divisions.** On an empty file `NR` is 0, and gawk stops with
  `fatal: division by zero attempted`.
- A non-numeric field in arithmetic silently counts as **0** — Nginx/Apache log
  `-` for "no body", and `"-" + 0` is `0`. Convenient, and exactly how bad data
  hides inside an awk total without anyone noticing.

### `printf`

`printf` works like Java's `String.format` — and like Java's, it adds **no
newline**:

```
printf "%-15s %6d %8.1f KB\n", $1, count, bytes / 1024
```

`%-15s` left-aligns a string in 15 columns, `%6d` right-aligns an integer in 6,
`%8.1f` is a decimal with one digit after the point, `%%` is a literal percent
sign.

> **Say this in an interview:** "Patterns can be regexes, field matches with
> `~`, numeric comparisons, or boolean combinations. Fields that look numeric
> compare numerically against numbers but lexically against string constants, so
> `$10 > \"1000\"` is a silent bug. `BEGIN` sets up — separators, headers — and
> `END` reports accumulated values; I guard divisions by `NR` there because an
> empty input is a fatal divide-by-zero in gawk."

---

## Part 7 — Associative arrays: `GROUP BY` in one line

**In plain words:** an awk array is a lookup table where the key can be any
string — an IP address, a status code, a URL. It's a `HashMap<String, Number>`
that creates an entry the first time you touch a key, starting at 0. That one
feature turns awk into a tiny group-by engine.

```
awk '{ count[$1]++ } END { for (ip in count) print count[ip], ip }' access.log
```

- `count[$1]++` — "the counter for this line's IP, plus one". A new IP starts at
  0 automatically.
- `for (ip in count)` — loop over every key.
- It's this SQL:

```
SELECT ip, COUNT(*)  FROM access_log GROUP BY ip;
SELECT status, SUM(bytes) FROM access_log GROUP BY status;   -- awk: bytes[$9] += $10
```

Several arrays can share a key, which gives you averages per group:

```
awk '{ sum[$7] += $NF; n[$7]++ } END { for (p in sum) printf "%-20s %.3f\n", p, sum[p]/n[p] }' access.log
```

That's "average response time per endpoint" — the headline number on an APM
dashboard, from a raw file.

### Array rules that bite

- **`for (k in arr)` has no defined order** — just like iterating a `HashMap`.
  Pipe the output through `sort` when order matters. (Sorting inside awk needs
  gawk's `asort`, which isn't portable.)
- **Testing with `arr[k] == ""` creates `arr[k]`.** Touching a key is enough to
  create it. Use `(k in arr)`, which tests without creating.
- `delete arr[k]` removes one key.

### The join trick: two files, one pass

The most famous awk idiom — a lookup from one file applied to another:

```
awk 'NR == FNR { owner[$1] = $2; next }  ($1 in owner) { print $0, owner[$1] }' owners.txt access.log
```

While awk reads the first file, `NR` (overall line count) equals `FNR` (line count
in this file), so the first rule loads the lookup table and `next` skips to the
next line. From the second file on, `NR != FNR`, and the second rule looks
things up. It's a hash join, written in one line. Hands-on 14 runs it.

> **Say this in an interview:** "awk arrays are associative — string-keyed and
> auto-vivified at 0 — so `count[$1]++` in the main block and a `for (k in
> count)` in `END` is a group-by. Iteration order is unspecified, so I pipe into
> `sort -rn`, and I test membership with `(k in arr)` because referencing an
> element creates it. `NR==FNR` with `next` is the standard way to join a lookup
> file against a data file in a single pass."

---

## Part 8 — Choosing: grep, sed, awk, or something else

**In plain words:** they overlap, and any of them can be bent into doing the
others' jobs. Don't. Pick by what you're doing to the text.

| You want to… | Reach for |
|---|---|
| Find lines | `grep` (fastest at pure filtering) |
| Change text within lines, or edit a file | `sed` |
| Work with columns, compare numbers, count, sum, group | `awk` |
| Parse JSON | `jq` (Day 8) |
| Logic that doesn't fit on a screen | A script (Day 9) — or Java/Python |

### Collapse pipelines — when it helps

```
grep ' 500 ' access.log | awk '{print $1}'          # two processes, and ' 500 ' also matches 500 bytes
awk '$9 == 500 { print $1 }' access.log            # one process, and it tests the actual status column
```

The awk version isn't just shorter — it's **more correct**, because it tests the
status *field* instead of "the text 500 anywhere in the line".

```
grep ERROR app.log | cut -d' ' -f4 | sort | uniq -c | sort -rn     # five tools
awk '/ERROR/ { n[$4]++ } END { for (k in n) print n[k], k }' app.log | sort -rn   # two
```

But don't make a religion of it. `sort` stays in the pipeline — awk can't sort
portably. And on multi-gigabyte files, a `grep -F` prefilter in front of awk is
often **faster**, because grep's literal search skips through text much faster
than awk evaluates rules line by line.

### Buffering, again

Like `grep` (Day 6), awk block-buffers when its output goes to a pipe.
`tail -f log | awk '…' | tee out` shows nothing for a while. Call `fflush()` in
the action, or make awk the last command in the pipe.

> **Say this in an interview:** "grep filters, sed edits, awk computes. When a
> pipeline does `grep | cut | sort | uniq -c`, a single awk with an associative
> array usually replaces the middle of it — and testing a specific field, like
> `$9 == 500`, is more correct than grepping for the text. For huge files I'll
> still prefilter with `grep -F` because it's faster at pure matching."

---

## Hands-on

Open the **Ubuntu (WSL)** tab. **Run block 1 first** — it creates the files
everything else uses. **Blocks 1, 6, 11 and 12 are the plan's required
practice** (rewrite a config in place; bytes per status; top 5 IPs). The rest
build up to them — do them if you have the time.

### 1. Build an access log and a config file

```bash
mkdir -p /tmp/day7 && cd /tmp/day7
cat > access.log <<'EOF'
10.0.3.14 - - [11/Sep/2026:08:14:02 +0000] "GET /api/orders HTTP/1.1" 200 5123 "-" "curl/8.5.0" 0.012
10.0.3.14 - - [11/Sep/2026:08:14:03 +0000] "GET /api/orders/42 HTTP/1.1" 200 812 "-" "curl/8.5.0" 0.009
10.0.3.91 - - [11/Sep/2026:08:14:05 +0000] "POST /api/orders HTTP/1.1" 201 256 "-" "Mozilla/5.0 (X11; Linux x86_64)" 0.143
192.168.10.7 - - [11/Sep/2026:08:14:07 +0000] "GET /api/health HTTP/1.1" 200 17 "-" "kube-probe/1.29" 0.001
10.0.3.14 - - [11/Sep/2026:08:14:09 +0000] "GET /api/orders HTTP/1.1" 500 1024 "-" "curl/8.5.0" 2.043
172.16.4.2 - - [11/Sep/2026:08:15:11 +0000] "GET /api/users HTTP/1.1" 200 20480 "-" "Mozilla/5.0 (Windows NT 10.0)" 0.087
10.0.3.14 - - [11/Sep/2026:08:15:40 +0000] "GET /api/orders HTTP/1.1" 503 0 "-" "curl/8.5.0" 30.001
10.0.3.91 - - [11/Sep/2026:08:16:02 +0000] "GET /static/app.js HTTP/1.1" 304 - "-" "Mozilla/5.0 (X11; Linux x86_64)" 0.002
192.168.10.7 - - [11/Sep/2026:08:16:30 +0000] "GET /api/health HTTP/1.1" 200 17 "-" "kube-probe/1.29" 0.001
10.0.3.55 - - [11/Sep/2026:08:16:45 +0000] "DELETE /api/orders/7 HTTP/1.1" 403 128 "-" "curl/8.5.0" 0.004
10.0.3.14 - - [11/Sep/2026:08:17:02 +0000] "GET /api/orders HTTP/1.1" 200 5123 "-" "curl/8.5.0" 0.011
172.16.4.2 - - [11/Sep/2026:08:17:33 +0000] "GET /api/reports/2026 HTTP/1.1" 200 1048576 "-" "Mozilla/5.0 (Windows NT 10.0)" 4.512
10.0.3.91 - - [11/Sep/2026:08:18:04 +0000] "GET /api/orders HTTP/1.1" 404 64 "-" "Mozilla/5.0 (X11; Linux x86_64)" 0.003
192.168.10.7 - - [11/Sep/2026:08:18:20 +0000] "GET /api/health HTTP/1.1" 200 17 "-" "kube-probe/1.29" 0.001
10.0.3.55 - - [11/Sep/2026:08:19:00 +0000] "POST /api/login HTTP/1.1" 401 96 "-" "curl/8.5.0" 0.020
10.0.3.14 - - [11/Sep/2026:08:19:59 +0000] "GET /api/orders HTTP/1.1" 500 1024 "-" "curl/8.5.0" 1.877
10.0.3.201 - - [11/Sep/2026:08:20:15 +0000] "GET /api/orders HTTP/1.1" 200 5123 "-" "python-requests/2.31" 0.013
172.16.4.2 - - [11/Sep/2026:08:21:00 +0000] "GET /api/users HTTP/1.1" 200 20480 "-" "Mozilla/5.0 (Windows NT 10.0)" 0.090
172.16.4.2 - - [11/Sep/2026:08:21:30 +0000] "GET /api/users HTTP/1.1" 200 20480 "-" "Mozilla/5.0 (Windows NT 10.0)" 0.101
EOF
cat > app.properties <<'EOF'
# Order service
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/orders
spring.datasource.username=orders
#spring.profiles.active=dev
logging.level.root=INFO
feature.newCheckout=false
EOF
wc -l access.log app.properties
echo "--- field numbers on line 1:"
head -1 access.log | awk '{ for (i = 1; i <= NF; i++) printf "$%d = %s\n", i, $i }'
```

That last command is worth keeping: it's how you work out which `$N` you need
before writing any awk against an unfamiliar log. Note `$9` is the status,
`$10` the bytes, and `$NF` the response time.

### 2. `sed` substitution — and prove it's a preview

```bash
cd /tmp/day7
echo "--- first match per line vs every match:"
echo "8080 and 8080" | sed 's/8080/9090/'
echo "8080 and 8080" | sed 's/8080/9090/g'
echo "--- preview a port change:"
sed 's/^server\.port=.*/server.port=9090/' app.properties | grep port
echo "--- the file on disk is untouched:"
grep port app.properties
```

### 3. Addresses: windows, deletions, extraction

```bash
cd /tmp/day7
echo "--- lines 2 to 4:"; sed -n '2,4p' access.log | cut -c1-70
echo "--- time window 08:15 through the first 08:17 line:"
sed -n '/08:15:/,/08:17:/p' access.log | cut -c1-70
echo "--- without health checks (custom delimiter because the pattern has /):"
sed '\|/api/health|d' access.log | wc -l
echo "--- config minus comments and blanks:"
sed -e '/^#/d' -e '/^$/d' app.properties
echo "--- count lines, sed-style:"; sed -n '$=' access.log
```

### 4. Capture groups: pull the method and path out

```bash
cd /tmp/day7
sed -nE 's/.*"([A-Z]+) ([^ ]+) HTTP[^"]*".*/\1 \2/p' access.log | sort | uniq -c | sort -rn
```

`-n` plus the `p` flag means "print only lines where the substitution worked" —
`grep -o` and a rewrite in one step.

### 5. The replacement-side failures, on purpose

```bash
echo "--- & in the replacement means 'the whole match':"
echo 'redirect=OLD' | sed 's/OLD/a=1&b=2/'
echo 'redirect=OLD' | sed 's/OLD/a=1\&b=2/'
echo "--- a / inside a shell variable breaks the command:"
path=/opt/app/logs
echo 'log.dir=/tmp' | sed "s/=.*/=$path/"
echo "exit=$?"
echo 'log.dir=/tmp' | sed "s|=.*|=$path|"
```

### 6. Rewrite a config in place — the safe workflow (required)

```bash
cd /tmp/day7
cp app.properties app.properties.orig
echo "--- 1. preview"
sed -E 's/^server\.port=.*/server.port=9090/; s|//localhost:|//db.internal:|; s/^#?spring\.profiles\.active=.*/spring.profiles.active=prod/' app.properties
echo "--- 2. edit for real, keeping a backup"
sed -i.bak -E 's/^server\.port=.*/server.port=9090/; s|//localhost:|//db.internal:|; s/^#?spring\.profiles\.active=.*/spring.profiles.active=prod/' app.properties
echo "--- 3. confirm exactly what changed"
diff app.properties.bak app.properties
echo "--- 4. run it AGAIN — idempotent means the second diff is empty:"
cp app.properties before-rerun
sed -i -E 's/^server\.port=.*/server.port=9090/; s|//localhost:|//db.internal:|; s/^#?spring\.profiles\.active=.*/spring.profiles.active=prod/' app.properties
diff before-rerun app.properties && echo "(no changes — idempotent)"
```

Three edits, one `sed`, separated by `;`. The profile edit uses `#?` so it works
whether the line is commented out or not.

### 7. Prove `-i` replaces the file (new inode, broken hard link)

```bash
cd /tmp/day7
cp app.properties.orig demo.properties
ln demo.properties hardlink.properties
echo "--- before: both names share one inode"
ls -i demo.properties hardlink.properties
sed -i 's/^feature\.newCheckout=.*/feature.newCheckout=true/' demo.properties
echo "--- after: demo.properties has a NEW inode"
ls -i demo.properties hardlink.properties
echo "--- and the hard link still has the old content:"
grep newCheckout demo.properties hardlink.properties
```

### 8. The upsert idiom

```bash
cd /tmp/day7
set_prop() {
  if grep -q "^$1=" "$3"; then sed -i "s|^$1=.*|$1=$2|" "$3"; else echo "$1=$2" >> "$3"; fi
}
set_prop logging.level.root DEBUG app.properties     # exists — rewritten
set_prop management.port 9091 app.properties         # missing — appended
set_prop management.port 9091 app.properties         # run again — no duplicate
grep -E '^(logging|management)' app.properties
```

(A key containing `.` is used as a regex here, where `.` matches anything —
harmless for this config, but it's why production tools escape keys first.)

### 9. awk fields — and why `cut` fails

```bash
cd /tmp/day7
echo "--- ip, status, response time:"; awk '{print $1, $9, $NF}' access.log | head -5
echo "--- \$13 is NOT always the response time (spaces in the user agent):"
awk '{print NF, $13}' access.log | head -4
echo "--- disk usage of / as a single number:"; df -h / | awk 'NR == 2 {print $5}'
echo "--- username and shell from /etc/passwd:"; awk -F: '$7 ~ /bash$/ {print $1, $7}' /etc/passwd
echo "--- the double-quote bug:"; echo "alpha beta" | awk "{print $1}"
```

### 10. Filters, BEGIN/END, and the traps

```bash
cd /tmp/day7
echo "--- 5xx responses:"; awk '$9 >= 500 {print $1, $7, $9, $NF}' access.log
echo "--- slower than 1 second:"; awk '$NF > 1 {print $7, $NF}' access.log
echo "--- total bytes served:"; awk '{sum += $10} END {print sum}' access.log
echo "--- error rate:"; awk '$9 >= 500 {e++} END {printf "%d of %d = %.1f%%\n", e, NR, 100 * e / NR}' access.log
echo "--- average response time:"; awk '{s += $NF} END {if (NR) printf "%.3f s\n", s / NR}' access.log
echo "--- string vs number: responses larger than 1000 bytes"
echo "numeric  \$10 > 1000:   $(awk '$10 > 1000' access.log | wc -l) lines"
echo "string   \$10 > \"1000\": $(awk '$10 > "1000"' access.log | wc -l) lines   <- 812, 256, 17... counted as bigger"
echo "--- unguarded average on an empty file:"
awk '{s += $1} END {print s / NR}' /dev/null; echo "exit=$?"
```

### 11. Total bytes per status code (required)

```bash
cd /tmp/day7
awk '{ bytes[$9] += $10; hits[$9]++ }
     END { for (s in bytes) printf "%s  %4d hits  %12d bytes  %10.1f KB\n", s, hits[s], bytes[s], bytes[s] / 1024 }' access.log | sort
```

The `304` line has `-` for bytes, which awk silently turned into `0`. Here
that's right. In general it's a reminder to check what your "numbers" contain.

### 12. Top 5 client IPs by request count (required)

```bash
cd /tmp/day7
awk '{ n[$1]++ } END { for (ip in n) print n[ip], ip }' access.log | sort -rn | head -5
```

Remove `| sort -rn | head -5` and run it again to see that `for (ip in n)` has
no order of its own.

### 13. Five tools versus one awk

```bash
cd /tmp/day7
echo "--- grep | cut | sort | uniq | sort, requests per endpoint for 5xx:"
grep -E '" 5[0-9]{2} ' access.log | cut -d' ' -f7 | sort | uniq -c | sort -rn
echo "--- one awk + sort, testing the actual status field:"
awk '$9 ~ /^5/ { n[$7]++ } END { for (p in n) print n[p], p }' access.log | sort -rn
echo "--- average response time per endpoint (group by with two arrays):"
awk '{ s[$7] += $NF; c[$7]++ } END { for (p in s) printf "%-20s %7.3f s  (%d req)\n", p, s[p] / c[p], c[p] }' access.log | sort -k2 -rn
```

### 14. The two-file join

```bash
cd /tmp/day7
cat > owners.txt <<'EOF'
10.0.3.14 checkout-service
172.16.4.2 reporting-ui
192.168.10.7 kubelet
EOF
awk 'NR == FNR { owner[$1] = $2; next }
     { who = ($1 in owner) ? owner[$1] : "UNKNOWN"; n[who]++ }
     END { for (w in n) print n[w], w }' owners.txt access.log | sort -rn
```

### 15. The same group-by in Java

awk's array is a `HashMap` that starts every key at zero. Here's what
`n[$1]++` and `bytes[$9] += $10` look like when you have to write them out
(Phase 3 redoes this with streams):

```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        // a text block (Java 15+): a multi-line string literal
        String log = """
            10.0.3.14 - - [11/Sep/2026:08:14:02 +0000] "GET /api/orders HTTP/1.1" 200 5123 "-" "curl/8.5.0" 0.012
            10.0.3.14 - - [11/Sep/2026:08:14:09 +0000] "GET /api/orders HTTP/1.1" 500 1024 "-" "curl/8.5.0" 2.043
            10.0.3.91 - - [11/Sep/2026:08:16:02 +0000] "GET /static/app.js HTTP/1.1" 304 - "-" "Mozilla/5.0 (X11; Linux x86_64)" 0.002
            172.16.4.2 - - [11/Sep/2026:08:15:11 +0000] "GET /api/users HTTP/1.1" 200 20480 "-" "Mozilla/5.0 (Windows NT 10.0)" 0.087
            """;

        Map<String, Integer> hitsPerIp = new HashMap<>();
        Map<String, Long> bytesPerStatus = new TreeMap<>();   // TreeMap = sorted keys, the "| sort"

        for (String line : log.strip().split("\n")) {
            String[] f = line.trim().split("\\s+");   // awk's default FS: runs of whitespace
            String ip = f[0];                           // $1  — Java arrays start at 0
            String status = f[8];                       // $9
            String bytes = f[9];                        // $10
            String responseTime = f[f.length - 1];      // $NF

            hitsPerIp.put(ip, hitsPerIp.getOrDefault(ip, 0) + 1);   // n[$1]++

            // awk silently turns "-" into 0. Java throws NumberFormatException — you must decide.
            long b = bytes.equals("-") ? 0 : Long.parseLong(bytes);
            bytesPerStatus.put(status, bytesPerStatus.getOrDefault(status, 0L) + b);   // bytes[$9] += $10

            System.out.println("last field ($NF) = " + responseTime);
        }
        System.out.println("hits per IP:      " + hitsPerIp);
        System.out.println("bytes per status: " + bytesPerStatus);
    }
}
```

Two differences worth noticing: awk's `$1` is Java's `f[0]`, and awk forgives
non-numbers (`"-"` becomes 0) where Java makes you decide. awk's forgiveness is
why it's fast to write, and also why it hides bad data.

### 16. Clean up

```bash
rm -rf /tmp/day7
```

---

## Gotchas

1. **`sed` without `-i` never changes the file.** It's a preview on stdout.
2. **`s///` without `g` replaces only the first match on each line.**
3. **Plain `sed` is BRE** — `+ ? ( ) { }` need backslashes. Use `sed -E`.
4. **`&` in the replacement means "the whole match"**; escape it as `\&`.
5. **A `/` inside a spliced shell variable breaks `s///`.** Change the
   delimiter (`s|…|…|`) — or use `awk -v` for arbitrary values.
6. **A range with an end pattern that never matches runs to end of file.**
7. **`sed -i` writes a new file and renames it** — new inode, broken hard links,
   replaced symlinks, `Device or resource busy` on Docker bind mounts.
8. **`sed -i` differs on macOS** (`-i ''`). Always give a suffix: `-i.bak`.
9. **`sed` silently does nothing if the line isn't there.** Upsert with
   `grep -q … && sed … || echo … >>`.
10. **Unanchored substitutions aren't idempotent.** Anchor to the key:
    `s/^key=.*/key=val/`.
11. **awk in double quotes**: the shell eats `$1`. Single-quote awk programs;
    pass values with `-v`.
12. **`print $1 $2` concatenates**; `print $1, $2` separates with `OFS`.
13. **Setting `OFS` doesn't reformat `$0`** until you assign a field (`$1=$1`).
14. **`$10 > "1000"` is a string comparison.** Never quote numbers.
15. **Non-numeric fields count as 0** in arithmetic — `-` bytes, `N/A`, typos.
16. **`for (k in arr)` is unordered.** Pipe to `sort`.
17. **`arr[k] == ""` creates `arr[k]`.** Use `(k in arr)`.
18. **Unguarded `sum / NR` on empty input is fatal** in gawk.
19. **`-F,` is not CSV parsing** — quoted commas break it.
20. **Your `awk` is gawk; many servers have mawk or BusyBox awk.** Avoid
    `gensub`, `asort`, `strftime` in scripts that ship.

---

## Recap

- `sed` streams each line through the **pattern space**, runs the script, and
  prints — **non-destructive by default**.
- `s/regex/replacement/flags`: `g` for every match, `N` for the Nth, `I` to
  ignore case; `&` and `\1` reuse the match; the delimiter can be any character.
  Use `-E`.
- **Addresses** target lines: `N`, `$`, `/re/`, `start,end`, `!`. With `-n` and
  `p`, `sed -n '/08:15/,/08:17/p'` prints a time window.
- **`-i` is rename-over, not in-place**: new inode, broken hard links. Preview →
  `-i.bak` → `diff`. Anchor edits so they're **idempotent**; upsert when the key
  might be missing.
- awk: `BEGIN { } pattern { action } … END { }`, run per **record**, fields
  `$1`…`$NF`, default `FS` = runs of whitespace (why it beats `cut -d' '`).
- Patterns: regex, `$N ~ /re/`, numeric comparisons (**never quote numbers**),
  `&&`/`||`, `NR > 1`.
- **Associative arrays** make `count[$1]++` / `sum[$9] += $10` a `GROUP BY`;
  iteration is unordered, so `| sort -rn`. `NR==FNR` joins two files.
- Pick by job: **grep finds, sed edits, awk computes.** Test fields, not text.

---

## Say it out loud

About 30 seconds each, out loud, using the real words:

1. Explain what `sed` does to a file when you run it without `-i` — using the
   words "stream" and "pattern space".
2. Your `sed "s/=.*/=$DIR/"` fails with `unknown option to s`. What happened, and
   what are two fixes?
3. Explain why `sed -i` breaks a hard link and fails on a Docker bind-mounted
   file. Use the word "inode".
4. What does "idempotent" mean for a config edit, and how do you write a `sed`
   substitution that has that property?
5. Describe awk's execution model in one breath: records, fields, pattern-action
   rules, `BEGIN`, `END`.
6. Write out loud — then type it — the awk one-liner for "requests per IP, top
   5", and explain why `sort` has to stay in the pipeline.
7. Why is `awk '$10 > "1000"'` a bug, and why might it pass your tests anyway?

---

## Q&A
