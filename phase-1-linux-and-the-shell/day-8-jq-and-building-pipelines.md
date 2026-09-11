# Day 8 — `jq` & Building Pipelines

_Why this matters:_ Almost everything you'll operate speaks JSON. REST APIs.
Spring Boot's `/actuator/health`. `docker inspect`. `kubectl get pods -o json`.
The `aws` CLI. Terraform state. And production logs — a Spring Boot service
running in Kubernetes normally logs **one JSON object per line**, not the
friendly text you've been grepping. Days 6 and 7 gave you tools for lines and
columns. JSON is neither: it's a tree. `jq` is grep, sed and awk for trees. The
second half of today is the glue — `tee`, `$( )` and `xargs` — that turns single
commands into automation, and it's what Day 9's scripts are made of.

---

## The one-paragraph version

JSON isn't lines and columns — it's a **tree**: objects with named keys, lists,
nested as deep as you like. And the *same* data can be written on one line or
across fifty, with keys in any order, so `grep` and `awk` see different text for
identical data and break. `jq` actually **parses** the JSON and lets you walk
the tree with paths like `.address.city`. Its model: you write a **filter** —
JSON goes in, JSON comes out — and filters chain with `|`, just like shell
commands. `.[]` takes a list apart so the rest of the filter runs once per item;
`select(...)` keeps the items you want; `{...}` builds new objects; `-r` prints
plain text instead of quoted JSON strings; `@csv` turns a list into a CSV row.
The other half is pipeline glue: **`tee`** to watch data *and* save it,
**`$( )`** to capture a command's output into a variable, and **`xargs`** to turn
lines of input into arguments for commands that don't read input at all — and to
run them in parallel.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **JSON** | A text format for structured data | RFC 8259: objects, arrays, strings, numbers, booleans, `null` |
| **object** | Named values in braces — `{"name": "x"}`. Java's `Map` | Unordered collection of key/value pairs; keys are strings |
| **array** | An ordered list — `[1, 2, 3]`. Java's `List` | Ordered sequence of values of any type |
| **key** | The name half of a name/value pair | A string, unique within its object |
| **scalar** | One single value, not a container | A string, number, boolean or `null` |
| **nested** | Objects and arrays inside other objects and arrays | A tree; values are reached by a path from the root |
| **path** | The route to a value — `.address.geo.lat` | A sequence of keys and indexes from the root |
| **filter** | A jq program: JSON in, JSON out | An expression mapping one input to zero or more outputs |
| **identity** | `.` — "the input, unchanged" | The filter that returns its input; `jq .` pretty-prints |
| **stream** (in jq) | Several separate results, one after another | A sequence of zero or more JSON values produced by a filter |
| **iterator** | `.[]` — "each item, one at a time" | Emits every element of an array (or value of an object) as its own output |
| **raw output** | The text itself, without JSON's quote marks | `-r`: strings printed unquoted and unescaped |
| **compact** | One JSON value per line | `-c`: no pretty-printing |
| **JSON Lines** | One JSON object per line — how structured logs look | Newline-delimited JSON, a.k.a. JSONL / NDJSON |
| **slurp** | Read all the input into one big array first | `-s` |
| **structured logging** | Logs written as JSON instead of sentences | Each event serialized as one JSON object per line, e.g. by Logback's JSON encoder |
| **HTTP status code** | The server's 3-digit verdict: 200 OK, 404 missing, 500 broke | The status line of the HTTP response |
| **response body** | The actual content the server sent back | The payload after the headers |
| **argument** | Words typed after a command name | `argv` — what the program receives at startup |
| **stdin** | Data flowing *into* a command, e.g. through a pipe | File descriptor 0 |
| **`xargs`** | Turns lines of input into command arguments | Builds and executes command lines from stdin |
| **command substitution** | Run a command and paste its output into another command | `$(cmd)` — replaced by cmd's stdout, trailing newlines removed |
| **`tee`** | A T-junction: output goes to a file *and* onward | Copies stdin to stdout and to each named file |
| **`ARG_MAX`** | The kernel's size limit for one command line | Max bytes of arguments + environment passed to `exec` — 2 MB here |
| **parallelism** | Several jobs running at the same time | Concurrent processes; `xargs -P N` keeps N running |

---

## Before you read: three questions

1. An API returns `{"status":"UP"}`. Your health-check script does
   `grep -q '"status":"UP"'`. It works for months, then fails after a library
   upgrade — though the service is perfectly healthy. What changed?
2. `jq '.name'` prints `"Leanne Graham"` *with* the quote marks. When does that
   matter?
3. `rm` ignores stdin completely. So how does
   `find . -name '*.tmp' | ??? rm` delete anything?

---

## Part 1 — Why JSON needs its own tool

**In plain words:** Days 6 and 7 think in lines and columns. JSON thinks in a
tree. Worse, the *same* data can be written many ways — on one line, spread
over twenty, keys in any order, spaces or no spaces — and they all mean exactly
the same thing. A line tool sees different text every time, so it matches
sometimes and not others. A JSON tool reads the meaning, not the text.

Here's the same health response, twice:

```
{"status":"UP","components":{"db":{"status":"DOWN"}}}
```

```
{
  "status": "UP",
  "components": { "db": { "status": "DOWN" } }
}
```

Now try line tools on it:

- `grep '"status":"UP"'` matches the first and **not** the second — the second
  has a space after the colon. That's the answer to question 1: a library
  upgrade switched the server to pretty-printing, and the text changed while the
  meaning didn't.
- `grep status` matches **both** the top-level status and the nested database
  status. It can't tell `status` from `components.db.status`.
- `awk -F'"'` breaks the moment a value contains an escaped quote, like
  `"message": "said \"hi\""`.

`jq` doesn't care about any of that: `jq '.status'` gives `"UP"` for both, and
`jq '.components.db.status'` gives `"DOWN"`.

### JSON in one table

| Type | Looks like | Java equivalent |
|---|---|---|
| object | `{"id": 1, "name": "x"}` | `Map<String, Object>`, or a class Jackson maps it to |
| array | `[1, "two", {"three": 3}]` | `List<Object>` |
| string | `"text"` — always double quotes | `String` |
| number | `42`, `-3.5`, `1e9` | `double` — JSON has **one** number type |
| boolean | `true`, `false` | `boolean` |
| null | `null` | `null` |

Keys must be double-quoted strings. No trailing commas. No comments. No single
quotes. jq's error messages when those rules are broken look cryptic, so learn
to recognise them:

```
parse error: Invalid numeric literal at line 1, column 5            ← {'a': 1} — single quotes
parse error: Expected another key-value pair at line 1, column 9    ← {"a": 1,} — trailing comma
```

### The big-number trap

JSON numbers have no size limit in the spec, but most tools — jq 1.6,
JavaScript, many parsers — store them as a 64-bit `double`, which is exact only
up to 2^53 (about 9 × 10^15). Beyond that they're **silently rounded**:

```bash
echo '{"orderId": 1791234567890123457}' | jq '.orderId'
```

A Java `long` goes up to about 9.2 × 10^18. So a Spring Boot service that
returns `Long` IDs — Twitter-style "snowflake" IDs, or big database sequences —
produces JSON that jq 1.6 and every browser quietly corrupt. That's why many
APIs send IDs **as strings** (`"id": "1791234567890123457"`), and in Spring you
do the same with Jackson's `@JsonSerialize(using = ToStringSerializer.class)`.
(jq 1.7 preserves big number literals as long as you don't do arithmetic on
them. Ubuntu 22.04 ships 1.6.)

> **Say this in an interview:** "JSON is a tree, and the same document can be
> serialized many ways — whitespace, key order, pretty or compact — so
> line-oriented tools like grep and awk match text rather than structure and
> break on formatting changes. jq parses the document and addresses values by
> path. One thing to watch: JSON numbers are usually parsed as IEEE doubles, so
> 64-bit IDs above 2^53 get silently rounded — which is why APIs serialize large
> IDs as strings."

---

## Part 2 — jq's model: filters and streams

**In plain words:** a jq program is a **filter**. JSON goes in, JSON comes out.
You chain filters with `|` exactly like shell commands — the output of one is
the input of the next. The twist that makes jq click: a filter can produce
**several** outputs. `.[]` takes a list and emits each item separately, and
everything after the `|` runs **once per item**. It's a for-each loop you never
have to write.

### Navigation

| Filter | Gives you |
|---|---|
| `.` | The whole input (pretty-printed) |
| `.name` | The value of key `name` |
| `.address.city` | A nested value |
| `.[0]`, `.[-1]` | First / last array element |
| `.[2:4]` | A slice: elements 2 and 3 |
| `.[]` | Every element, as separate outputs |
| `.[].name` | The `name` of every element |
| `.name, .email` | Comma = two outputs, one after the other |
| `."content-type"` | A key with a character jq would otherwise parse |

The same "every element's name", written with an explicit pipe — the form you'll
use once filters get longer:

```
jq '.[] | .name'  users.json
jq '.[] | .address.city'  users.json
```

### Missing things are `null`, not errors

- `.nickname` on an object without that key → `null`. No error.
- `.address.zip.plus4` when `zip` doesn't exist → `null` all the way down.
- `.a.b` when `.a` is a **number** → an error: `Cannot index number with "b"`.
  Add `?` to suppress it: `.a.b?`.
- **`//` supplies a default:** `.nickname // "none"`. (It also replaces `false`,
  not only `null`.)

### A trap with hyphenated keys

```
jq '.content-type'      # parsed as  .content - type  — subtraction!
jq '."content-type"'    # correct
jq '.["content-type"]'  # also correct
```

The first gives the baffling error
`null (null) and string ("object") cannot be subtracted` — `type` is a jq
built-in that returned `"object"`. HTTP headers, Kubernetes labels and
Actuator metric names are full of hyphens and dots. Quote them.

### Streams in, streams out

jq also reads a **stream**: the input can be many JSON values back to back — a
JSON Lines log file, or ten `curl` responses concatenated. jq runs your filter
on each value in turn. That's why `jq '.level' app.jsonl` just works on a
10,000-line log.

And a filter that produces a stream can be **collected back into an array** by
wrapping it in `[ ]`:

```
jq '.[] | select(.id > 5)'      # a stream: five separate objects
jq '[.[] | select(.id > 5)]'    # one array of five objects
jq 'map(select(.id > 5))'       # identical to the line above
```

`map(f)` is literally defined as `[.[] | f]`.

> **Say this in an interview:** "A jq program is a filter from JSON to a stream
> of zero or more JSON values, and filters compose with the pipe. `.[]` iterates
> — each element flows through the rest of the pipeline separately — and wrapping
> an expression in brackets collects the stream back into an array, which is all
> `map` is. Missing keys evaluate to null rather than failing, and `//` provides
> a default."

---

## Part 3 — Filtering and transforming

**In plain words:** once you can reach values, you want to keep some, count
them, sort them and reshape them. jq has a built-in for almost every "SQL on a
list" operation you'd want.

### Keeping what you want: `select`

`select(condition)` passes its input through if the condition is true and
produces **nothing** otherwise.

```
jq '.[] | select(.completed)'                            # boolean field is true
jq '.[] | select(.userId == 1 and .completed == false)'  # and / or / not
jq '.[] | select(.address.city | test("^South"))'        # regex match
jq '.[] | select(.level != "DEBUG")'
```

`test("regex"; "i")` uses jq's own regex engine (Oniguruma), which is
PCRE-flavoured, so `\d` works — but inside a jq string you write it `"\\d"`,
exactly like Java (Day 6).

### The toolbox

| Built-in | Does |
|---|---|
| `length` | Characters in a string, elements in an array, keys in an object; `null` → 0 |
| `keys` | An object's keys, **sorted**. `keys_unsorted` keeps the original order |
| `has("k")` | Does the object have key `k`? |
| `map(f)` | Apply `f` to every element, return an array |
| `map_values(f)` | Same, for an object's values — keeps the keys |
| `sort`, `sort_by(.x)`, `reverse` | Ordering |
| `group_by(.x)` | An array of arrays, one per distinct `.x` — `GROUP BY` |
| `unique`, `unique_by(.x)` | Deduplicate |
| `min_by(.x)`, `max_by(.x)` | The whole element with the smallest / largest `.x` |
| `add` | Sum numbers, join strings, merge objects |
| `first`, `last`, `limit(n; f)` | Take from the front or back |
| `to_entries` | `{"a":1}` → `[{"key":"a","value":1}]` — iterate an object like a map |
| `with_entries(f)` | `to_entries`, then `map(f)`, then back to an object |
| `tonumber`, `tostring` | Convert |
| `ascii_downcase`, `split(",")`, `join(",")`, `ltrimstr("x")` | Strings |

`group_by` then `map` is how you count per group:

```
jq -s 'group_by(.level) | map({level: .[0].level, count: length})' app.jsonl
```

`-s` (slurp) first, because `group_by` needs **all** the lines in one array —
on its own, jq sees each log line separately.

### Two traps you already know

- **String sorting:** `sort_by(.v)` on `"10"` and `"9"` gives `"10"` first —
  they're strings, compared character by character. It's Day 7's awk trap again.
  Fix: `sort_by(.v | tonumber)`.
- **`if` needs `else` in jq 1.6:** `if . > 0 then "pos" end` is a syntax error on
  your version. Write `if . > 0 then "pos" else "neg" end`. (1.7 made `else`
  optional.) To drop an item instead of producing a value, use `empty`.

### Version check

Your Ubuntu gets **jq 1.6**. A lot of Stack Overflow answers are written for
1.7, which added functions that don't exist on 1.6 — `pick`, `abs` — and the
optional `else`. If a snippet fails with `xyz/1 is not defined`, that's why.

> **Say this in an interview:** "`select` filters a stream by a predicate,
> `map` transforms an array, and `group_by` plus `length` gives per-group counts
> — but group_by needs the whole input as one array, so for JSON Lines logs I
> slurp with `-s` first. And I watch for strings that look like numbers:
> `sort_by` on them sorts lexically unless I apply `tonumber`."

---

## Part 4 — Output, and getting values into jq

**In plain words:** by default jq prints JSON — pretty, with strings in quotes.
That's right for humans and for other JSON tools, and wrong for shell commands:
a filename with literal quote marks around it isn't the file you meant. A
handful of flags control what comes out, and two control what goes in.

### Output flags

| Flag | Effect | Use when |
|---|---|---|
| (none) | Pretty JSON, strings quoted | Reading it yourself |
| `-r` | **Raw**: strings without quotes | Output feeds a shell command or a file |
| `-c` | **Compact**: one value per line | Producing JSON Lines; feeding `grep` or `while read` |
| `-e` | Exit status 1 if the last output is `false` or `null` | Using jq as a test in a script |
| `-s` | **Slurp**: all input into one array | Aggregating across JSON Lines |
| `-n` | Don't read input; start from `null` | Building JSON from nothing |
| `-S` | Sort keys | Diffing two JSON files |

```
jq -e '.status == "UP"' health.json > /dev/null && echo healthy || echo unhealthy
```

That's a correct health check — it tests the *parsed* value, so it survives any
formatting change the server makes.

### Formatting text: interpolation and `@` formats

```
jq -r '.[] | "\(.name) <\(.email)>"'                  # string interpolation: \( expr )
jq -r '.[] | [.id, .name, .email] | @csv'             # CSV row: quotes strings, escapes quotes
jq -r '.[] | [.id, .name] | @tsv'                     # tab-separated
jq -r '.[] | .name | @sh'                              # shell-quoted — safe to hand to a shell
```

- `@csv` and `@tsv` take an **array of scalars**. Hand them an object and they
  error: `object ({...}) cannot be csv-formatted, only array`.
- `null` becomes an empty CSV field.
- Other formats: `@json`, `@base64`, `@base64d` (decode), `@uri` (URL-encode).

### Getting shell values into jq: `--arg` and `--argjson`

The same rule as awk's `-v` (Day 7): **never splice a shell variable into the
jq program**. Pass it in:

```
jq --arg city "$CITY" '.[] | select(.address.city == $city)'  users.json
jq --argjson id "$ID"  '.[] | select(.id == $id)'              users.json
```

- `--arg` always makes a **string**.
- `--argjson` parses the value as JSON — so `5` becomes the **number** 5.

The trap: `--arg id 5` then `select(.id == $id)` compares the number `5` with
the string `"5"` — never equal — and you get **nothing**, no error. Use
`--argjson` for numbers. (Or `$ENV.NAME` / `env.NAME` to read an environment
variable.)

And as with awk: single-quote the program. In double quotes the shell would
expand `$city` and `$id` before jq ever saw them.

> **Say this in an interview:** "I use `-r` whenever jq's output feeds another
> command, `-c` for JSON Lines, and `-e` to make jq's exit status reflect the
> result so it works as a condition. For CSV there's `@csv` over an array of
> scalars. Shell values go in with `--arg` for strings and `--argjson` for
> numbers — mixing those up makes an equality silently never match."

---

## Part 5 — Building and modifying JSON

**In plain words:** jq doesn't only read JSON; it writes it. You can reshape an
API's response into exactly the fields you need, build a new document from
scratch, or change one value inside a config file — the JSON version of Day 7's
`sed`.

### Constructing objects

```
jq '.[] | {name, email}'                                        # shorthand for {name: .name, email: .email}
jq '.[] | {name, city: .address.city, company: .company.name}'  # flatten nested fields
jq 'map({id, name})'                                             # an array of slimmed-down objects
jq 'map({(.username): .email}) | add'                            # dynamic keys → one lookup object
```

`{(expr): value}` — parentheses around the key mean "compute the key". Combined
with `add` (which merges objects), that turns a list into a map keyed by
username.

### Updating values

```
jq '.server.port = 9090'              # set
jq '.replicas |= . + 1'               # update relative to the current value
jq 'del(.datasource.password)'        # remove a key
jq '. + {"env": "prod"}'              # merge in a key
```

### jq has no `-i` — and the obvious workaround destroys the file

```
jq '.server.port = 9090' config.json > config.json     # DON'T
```

This leaves you with an **empty file** and exit status 0. The shell handles
`> config.json` **before jq starts** — it opens the file for writing, which
truncates it to zero bytes — and then jq reads an empty file and outputs
nothing. The shell reads your command first; you've now seen that rule bite in
three different ways.

The fix is the temp-file-and-rename that Day 7 showed you `sed -i` does behind
the scenes:

```
tmp=$(mktemp) && jq '.server.port = 9090' config.json > "$tmp" && mv "$tmp" config.json
```

The `&&` chain means `mv` only runs if jq succeeded — a bad filter leaves the
original untouched. (`sponge` from the `moreutils` package does this too, but it
isn't installed by default.)

> **Say this in an interview:** "jq constructs objects with `{}` — including
> computed keys — and updates with `=`, `|=` and `del`. It has no in-place flag,
> and redirecting output back onto the input file truncates it before jq reads
> it, so I write to a `mktemp` file and `mv` it over the original only if jq
> succeeded."

---

## Part 6 — `curl` + `jq`

**In plain words:** `curl` fetches a URL; `jq` makes sense of what came back.
Most of the trouble is at the join: `curl` will happily hand `jq` an error page,
an HTML login screen or an empty body, and the combination often fails
**silently**.

### The flags you want, every time

```
curl -fsSL -m 10 -H 'Accept: application/json' "$URL" | jq ...
```

| Flag | Why |
|---|---|
| `-f` (`--fail`) | On HTTP 400 or above, output nothing and exit 22 instead of passing the error body along |
| `-s` | Silent: no progress meter |
| `-S` | …but still print real errors to stderr (`-s` alone hides them too) |
| `-L` | Follow redirects |
| `-m 10` | Give up after 10 seconds. A script without a timeout eventually hangs on a network call |
| `-H 'Accept: application/json'` | Ask for JSON explicitly |
| `-w '%{http_code}'` | Print the status code — for debugging |

### Three failures you'll see in real life

1. **An error that's still JSON.** Many APIs answer a missing resource with a
   `404` and a JSON body like `{}` or `{"error": "..."}`. Without `-f`,
   `jq '.name'` prints `null`, exits 0, and your script carries on with a user
   called "null".
2. **HTML where JSON was expected** — a load balancer's `502` page, a proxy, an
   SSO login redirect. jq says:
   `parse error: Invalid numeric literal at line 1, column 10`.
   **Memorise that message.** It almost always means "this isn't JSON — it's
   HTML".
3. **An empty body.** jq outputs nothing and exits **0**. Silent.

### The pipeline's exit status is the last command's

```
curl -fsS "$URL/does-not-exist" | jq .
echo $?      # 0 — jq succeeded on empty input; curl's 22 is lost
```

A pipeline reports only the **last** command's exit status. `curl` failed and
nobody knows. The fix is `set -o pipefail`, which makes the pipeline fail if
**any** command in it fails — the "`o pipefail`" part of the
`set -euo pipefail` line that Day 9 puts at the top of every script. Hands-on 7
shows both.

> **Say this in an interview:** "When piping curl into jq I use `-fsSL` with a
> timeout: `-f` so HTTP errors produce a non-zero exit instead of an error body,
> `-sS` for quiet-but-not-silent, `-L` for redirects. Then `pipefail`, because a
> pipeline's exit status is the last command's, so curl's failure would
> otherwise be masked. And 'Invalid numeric literal at line 1' from jq almost
> always means I got HTML back."

---

## Part 7 — Pipeline glue: `tee`, `$( )` and `xargs`

**In plain words:** `|` connects one command's output to the next one's input.
Three situations need more than that: you want to **watch** data *and* **save**
it (`tee`); you want a command's output **inside** another command, as an
argument or a variable (`$( )`); or you want to feed lines to a command that
**doesn't read input at all** (`xargs`).

### `tee` — a T-junction

```
curl -fsS "$URL" | tee raw.json | jq '.[] | .name'
```

The raw response is saved for debugging *and* flows on to jq. `tee -a` appends
instead of overwriting.

It also solves a classic permissions puzzle from Day 3:

```
sudo echo 'x' > /etc/myapp.conf        # Permission denied
echo 'x' | sudo tee /etc/myapp.conf    # works
```

The redirect in the first line is done by **your** shell, running as you — `sudo`
only elevates `echo`. In the second, `tee` itself runs as root and does the
writing.

### `$( )` — command substitution

```
count=$(jq 'length' users.json)
echo "we have $count users"
```

The shell runs the command, captures its stdout, **strips trailing newlines**,
and pastes the result in. Rules:

- **Quote it when you use it:** `"$(cmd)"` or `"$var"`. Unquoted, the shell
  splits the output on whitespace — ten names become twenty words, and the
  newlines between them are gone. Hands-on 10 shows this.
- The old form is backticks `` `cmd` ``. Same thing, but they don't nest
  cleanly. Use `$( )`.
- `var=$(cmd)` sets `$?` to `cmd`'s exit status, so you can check it.

### `xargs` — lines in, arguments out

**The problem it solves.** Many commands don't read stdin at all — they only
look at their **arguments**. `rm`, `kill`, `mkdir`, `curl URL`, `docker rm`.
So this does nothing useful:

```
echo junk.txt | rm           # rm: missing operand — rm never looked at stdin
```

`xargs` reads lines from stdin and turns them into arguments:

```
echo junk.txt | xargs rm     # runs: rm junk.txt
```

| Flag | Does |
|---|---|
| (none) | Pack as many arguments as fit onto one command line, run it as few times as possible |
| `-n 1` | One argument per run |
| `-I{}` | Run once per line, putting the line wherever `{}` appears |
| `-P 4` | Run up to 4 at a time, in parallel |
| `-t` | Print each command before running it — use this while learning |
| `-r` | Don't run at all if the input is empty (GNU) |
| `-d '\n'` | Split only on newlines, not on spaces (GNU) |
| `-0` | Split on NUL bytes — pair with `find -print0` |

### Three `xargs` traps

1. **Spaces split names.** By default `xargs` splits on *any* whitespace, so
   `my file.txt` becomes two arguments, `my` and `file.txt`. Use `-d '\n'`, or
   better, `find … -print0 | xargs -0 …`, which works for **every** possible
   filename.
2. **Empty input still runs the command once.** `grep -l nothing *.log | xargs rm`
   runs `rm` with no arguments. Harmless for `rm`, not for every command. Add
   `-r`.
3. **A failed run makes `xargs` exit 123** — check it in scripts.

### Why `xargs` exists at all: `ARG_MAX`

The kernel limits the total size of one command line — `getconf ARG_MAX` says
2,097,152 bytes on your WSL. `rm *.log` in a directory with 300,000 log files
expands to a command line bigger than that and fails with
`Argument list too long`. `xargs` automatically splits the input into as many
command lines as needed. That's its original purpose.

```
find /var/log/myapp -name '*.log' -mtime +30 -print0 | xargs -0 -r rm
```

"Delete logs older than 30 days, whatever their names, and do nothing if there
aren't any." (`find … -delete` does the same with no `xargs`.)

### `xargs -P` — your first worker pool

`-P 4` keeps four commands running at once; as soon as one finishes, the next
input line starts. Eight one-second jobs take eight seconds sequentially and
about two with `-P 4`. That is precisely what a Java **thread pool** is — a
fixed number of workers pulling jobs off a queue — and you'll build exactly that
in Phase 5. Two consequences carry over directly:

- **Output order isn't guaranteed.** Jobs finish when they finish.
- **More workers isn't free.** Against a remote API, `-P 50` is how you get
  rate-limited or banned.

> **Say this in an interview:** "`xargs` exists because many commands take
> arguments rather than stdin, and because the kernel caps command-line size at
> `ARG_MAX`, so it batches input into as many invocations as needed. I use
> `-print0` with `-0` so any filename is safe, `-r` so empty input runs nothing,
> and `-P` for parallelism — which is a worker pool, so ordering isn't preserved.
> `tee` splits a stream to a file and onward, and `$( )` captures output, which
> I always quote to avoid word splitting."

---

## Hands-on

**Step 0 — install `jq`.** It isn't installed on your WSL yet, and installing
needs your password, which the lab can't type for you. Open your **Ubuntu (WSL)
terminal** and run:

```
sudo apt-get update && sudo apt-get install -y jq
```

Then everything below runs in the lab. **Run block 1 first.** Blocks 6 and 12
are the plan's required practice (API → three fields → filter → CSV; IDs →
`xargs` → fetch each). Blocks 6, 7, 12 and 13 need internet access.

### 1. Check jq, and build the files

```bash
jq --version
mkdir -p /tmp/day8 && cd /tmp/day8
cat > health.json <<'EOF'
{
  "status": "DOWN",
  "components": {
    "db": {
      "status": "DOWN",
      "details": { "database": "PostgreSQL", "error": "org.postgresql.util.PSQLException: Connection refused" }
    },
    "diskSpace": {
      "status": "UP",
      "details": { "total": 268435456000, "free": 107374182400, "threshold": 10485760 }
    },
    "ping": { "status": "UP" },
    "redis": { "status": "UP", "details": { "version": "7.2.4" } }
  }
}
EOF
cat > app.jsonl <<'EOF'
{"@timestamp":"2026-09-11T08:14:02.101Z","level":"INFO","logger":"c.a.OrderController","thread":"http-nio-8080-exec-1","message":"GET /api/orders 200","durationMs":12,"traceId":"a1f3"}
{"@timestamp":"2026-09-11T08:14:03.220Z","level":"WARN","logger":"c.a.PaymentClient","thread":"http-nio-8080-exec-2","message":"retry 1/3","durationMs":0,"traceId":"b7c2","orderId":1791234567890123457}
{"@timestamp":"2026-09-11T08:14:05.004Z","level":"ERROR","logger":"c.a.PaymentClient","thread":"http-nio-8080-exec-2","message":"connect timeout to 192.168.10.7:8443","durationMs":2043,"traceId":"b7c2","orderId":1791234567890123457,"exception":{"class":"java.net.SocketTimeoutException","message":"Connect timed out"}}
{"@timestamp":"2026-09-11T08:14:06.310Z","level":"INFO","logger":"c.a.PaymentClient","thread":"http-nio-8080-exec-2","message":"recovered after ERROR from upstream","durationMs":0,"traceId":"b7c2"}
{"@timestamp":"2026-09-11T08:15:11.450Z","level":"INFO","logger":"c.a.OrderController","thread":"http-nio-8080-exec-3","message":"GET /api/orders 200","durationMs":9,"traceId":"c9d0"}
{"@timestamp":"2026-09-11T08:16:40.000Z","level":"ERROR","logger":"c.a.ReportJob","thread":"scheduler-1","message":"report build failed","durationMs":310,"traceId":"d4e5","exception":{"class":"java.lang.NullPointerException","message":"Cannot invoke \"String.length()\" because \"name\" is null"}}
{"@timestamp":"2026-09-11T08:17:33.900Z","level":"WARN","logger":"c.a.RateLimiter","thread":"http-nio-8080-exec-5","message":"throttled 10.0.3.14","durationMs":0,"traceId":"e6f7"}
{"@timestamp":"2026-09-11T08:18:04.321Z","level":"ERROR","logger":"c.a.OrderController","thread":"http-nio-8080-exec-5","message":"GET /api/orders 503","durationMs":30001,"traceId":"f8a9","exception":{"class":"org.springframework.web.client.ResourceAccessException","message":"I/O error on POST request"}}
{"@timestamp":"2026-09-11T08:19:00.010Z","level":"DEBUG","logger":"c.a.CacheWarmer","thread":"scheduler-2","message":"warming 240 keys","durationMs":55,"traceId":"0a1b"}
{"@timestamp":"2026-09-11T08:19:59.777Z","level":"ERROR","logger":"c.a.PaymentClient","thread":"http-nio-8080-exec-7","message":"connect timeout to 192.168.10.8:8443","durationMs":2051,"traceId":"1c2d","orderId":1791234567890123460,"exception":{"class":"java.net.SocketTimeoutException","message":"Connect timed out"}}
{"@timestamp":"2026-09-11T08:20:15.000Z","level":"INFO","logger":"c.a.OrderController","thread":"http-nio-8080-exec-8","message":"GET /api/orders 200","durationMs":15,"traceId":"2e3f"}
EOF
jq -c . health.json > health-compact.json
wc -l health.json health-compact.json app.jsonl
```

`health.json` is shaped like Spring Boot's `/actuator/health`. `app.jsonl` is
what a Spring Boot service's logs look like with a JSON log encoder — one event
per line.

### 2. Watch line tools fail on JSON

```bash
cd /tmp/day8
echo "--- grep '\"status\":\"DOWN\"' — pretty vs compact (same data):"
echo "pretty:  $(grep -c '"status":"DOWN"' health.json) matches"
echo "compact: $(grep -c '"status":"DOWN"' health-compact.json) matches"
echo "--- grep can't tell WHICH status:"
grep -o '"status": *"[A-Z]*"' health.json
echo "--- jq reads the structure, whatever the formatting:"
jq '.status' health.json
jq '.status' health-compact.json
jq '.components.db.status' health.json
echo "--- grep ERROR vs the actual level field:"
echo "grep -c ERROR:        $(grep -c ERROR app.jsonl)"
echo "jq level == ERROR:    $(jq -c 'select(.level == "ERROR")' app.jsonl | wc -l)"
```

`grep` counted the INFO line whose *message* says "recovered after ERROR" — the
Day 7 lesson again: test the field, not the text.

### 3. Navigate the tree

```bash
cd /tmp/day8
jq '.components.db.details.error' health.json
echo "--- -r drops the quotes:"
jq -r '.components.db.details.error' health.json
echo "--- component names, then each status:"
jq -c '.components | keys' health.json
jq -r '.components | to_entries[] | "\(.key)=\(.value.status)"' health.json
echo "--- free disk in GB (arithmetic works on numbers):"
jq '.components.diskSpace.details.free / 1024 / 1024 / 1024' health.json
echo "--- a missing key is null, and // gives a default:"
jq '.components.kafka' health.json
jq -r '.components.kafka.status // "not configured"' health.json
echo "--- the hyphenated-key trap:"
echo '{"content-type": "application/json"}' | jq '.content-type'
echo '{"content-type": "application/json"}' | jq -r '."content-type"'
```

### 4. A health check that survives formatting

```bash
cd /tmp/day8
for f in health.json health-compact.json; do
  if jq -e '.status == "UP"' "$f" > /dev/null; then echo "$f: healthy"; else echo "$f: UNHEALTHY"; fi
done
echo "--- which components are down, and why:"
jq -r '.components | to_entries[] | select(.value.status != "UP") | "\(.key): \(.value.details.error // "no detail")"' health.json
```

### 5. Structured logs: the jq version of Days 6 and 7

```bash
cd /tmp/day8
echo "--- ERROR events as tab-separated columns:"
jq -r 'select(.level == "ERROR") | [."@timestamp", .logger, .exception.class] | @tsv' app.jsonl
echo "--- count per level (slurp first — group_by needs every line at once):"
jq -s -c 'group_by(.level) | map({level: .[0].level, count: length})' app.jsonl
echo "--- the 3 slowest events:"
jq -s -r 'sort_by(.durationMs) | reverse | .[0:3][] | "\(.durationMs) ms  \(.message)"' app.jsonl
echo "--- everything that happened in one trace (the thing you actually do at 3am):"
jq -c 'select(.traceId == "b7c2") | {level, message}' app.jsonl
echo "--- the big-number trap: compare the raw text with what jq prints"
grep -o '"orderId":[0-9]*' app.jsonl
jq 'select(.orderId) | .orderId' app.jsonl
```

Look at the last two outputs. The raw file has two different order IDs, ending
`…457` and `…460`. jq 1.6 prints all three events with the **same** number. If
you were tracing a payment failure, you'd now be investigating the wrong order.

### 6. Required: API → three fields → filter → CSV

```bash
cd /tmp/day8
curl -fsSL -m 10 https://jsonplaceholder.typicode.com/users | tee users.json | jq 'length'
echo "--- the shape of one user:"
jq '.[0] | {id, name, email, address: {city: .address.city}, company: {name: .company.name}}' users.json
echo "--- users whose city starts with 'South', as CSV with a header:"
jq -r '["id","name","email"], (.[] | select(.address.city | test("^South")) | [.id, .name, .email]) | @csv' users.json | tee south.csv
echo "--- companies that are an LLC or Group, as CSV:"
jq -r '.[] | select(.company.name | test("LLC|Group")) | [.id, .name, .company.name] | @csv' users.json
```

`tee users.json` saves the raw response so every command after it works offline
— you don't hit the API again for each experiment.

### 7. curl + jq failure modes

```bash
cd /tmp/day8
echo "--- 1. a 404 that is still JSON, without -f:"
curl -sS -m 10 -w '   <- HTTP %{http_code}\n' https://jsonplaceholder.typicode.com/users/999
curl -sS -m 10 https://jsonplaceholder.typicode.com/users/999 | jq '.name'
echo "--- ...and with -f:"
curl -fsS -m 10 https://jsonplaceholder.typicode.com/users/999 | jq '.name'
echo "--- 2. HTML where JSON was expected:"
curl -fsS -m 10 https://httpbin.org/html | jq '.title'
echo "--- 3. the pipeline's exit status hides curl's failure:"
curl -fsS -m 10 https://httpbin.org/status/500 | jq .
echo "without pipefail: exit=$?"
set -o pipefail
curl -fsS -m 10 https://httpbin.org/status/500 | jq .
echo "with pipefail:    exit=$?"
```

### 8. Reshape, update, and the truncation trap

```bash
cd /tmp/day8
echo "--- slimmed-down objects:"
jq -c 'map({id, name, city: .address.city}) | .[0:3][]' users.json
echo "--- a username -> email lookup table, built with a dynamic key:"
jq 'map({(.username): .email}) | add | with_entries(select(.key | test("^[A-K]")))' users.json
cat > config.json <<'EOF'
{ "server": { "port": 8080 }, "datasource": { "url": "jdbc:postgresql://localhost:5432/orders", "password": "s3cret" }, "replicas": 2 }
EOF
echo "--- the WRONG way to edit in place:"
cp config.json broken.json
jq '.server.port = 9090' broken.json > broken.json
echo "exit=$?   size of broken.json afterwards: $(wc -c < broken.json) bytes"
echo "--- the right way: temp file, then rename"
tmp=$(mktemp) && jq '.server.port = 9090 | .replicas |= . + 1 | del(.datasource.password)' config.json > "$tmp" && mv "$tmp" config.json
jq -c . config.json
```

### 9. `--arg` vs `--argjson`

```bash
cd /tmp/day8
want=3
echo "--arg (string \"3\" vs number 3 — never equal):"
jq -c --arg id "$want" '.[] | select(.id == $id) | {id, name}' users.json
echo "(nothing above — no error either)"
echo "--argjson (number 3):"
jq -c --argjson id "$want" '.[] | select(.id == $id) | {id, name}' users.json
echo "--arg is right for strings:"
jq -r --arg city "Gwenborough" '.[] | select(.address.city == $city) | .name' users.json
```

### 10. `$( )` and quoting

```bash
cd /tmp/day8
count=$(jq 'length' users.json)
echo "users: $count"
names=$(jq -r '.[0:3][].name' users.json)
echo "--- quoted (\"\$names\") keeps the lines:"
echo "$names"
echo "--- unquoted (\$names) — the shell re-splits on every space:"
echo $names
echo "--- trailing newlines are stripped:"
v=$(printf 'abc\n\n\n'); echo "[$v]"
```

### 11. `xargs` from zero

```bash
mkdir -p /tmp/day8/x && cd /tmp/day8/x && rm -f ./*
echo "--- rm ignores stdin:"
touch junk.txt
echo junk.txt | rm 2>&1; ls
echo "--- xargs turns it into an argument (-t shows the command it builds):"
echo junk.txt | xargs -t rm; ls
echo "--- default packs arguments; -n 1 runs once each; -I{} places them:"
printf 'a\nb\nc\n' | xargs -t echo
printf 'a\nb\nc\n' | xargs -t -n 1 echo
printf 'a\nb\nc\n' | xargs -I{} echo "item={}"
echo "--- the space trap:"
touch "my file.txt"
printf 'my file.txt\n' | xargs ls 2>&1
printf 'my file.txt\n' | xargs -d '\n' ls
find . -name '*.txt' -print0 | xargs -0 ls
echo "--- empty input still runs once, unless -r:"
printf '' | xargs echo "ran anyway"
printf '' | xargs -r echo "never printed"; echo "(-r: nothing ran)"
echo "--- ARG_MAX, and xargs splitting a huge input into batches:"
getconf ARG_MAX
/bin/echo $(seq 1 500000) > /dev/null
echo "exit=$?  <- one command line was too big"
echo "xargs ran /bin/echo $(seq 1 500000 | xargs /bin/echo | wc -l) times instead"
```

### 12. Required: IDs → `xargs` → fetch each

```bash
cd /tmp/day8
echo "--- the IDs of LLC/Group companies' users:"
jq -r '.[] | select(.company.name | test("LLC|Group")) | .id' users.json | tee ids.txt
echo "--- fetch each user's todos and summarise:"
xargs -I{} curl -fsS -m 10 "https://jsonplaceholder.typicode.com/users/{}/todos" < ids.txt \
  | jq -r '"user \(.[0].userId): \(map(select(.completed)) | length)/\(length) todos done"'
```

`xargs` ran one `curl` per ID; their responses came out as a stream of JSON
arrays, back to back; jq ran the filter once per array. No loop anywhere.

### 13. `xargs -P`: sequential vs a worker pool

```bash
ms() { echo $(( ($(date +%s%N) - $1) / 1000000 )); }
echo "--- 8 one-second jobs:"
t=$(date +%s%N); seq 1 8 | xargs -I{} sleep 1;        echo "sequential: $(ms $t) ms"
t=$(date +%s%N); seq 1 8 | xargs -P 4 -I{} sleep 1;   echo "-P 4:       $(ms $t) ms"
echo "--- output order is not guaranteed with -P:"
seq 1 6 | xargs -P 6 -I{} sh -c 'sleep 0.$((7 - {})); echo "job {} done"'
```

Same idea against a real API:

```bash
ms() { echo $(( ($(date +%s%N) - $1) / 1000000 )); }
echo "--- 10 API calls:"
t=$(date +%s%N); seq 1 10 | xargs -I{} curl -fsS -m 10 -o /dev/null "https://jsonplaceholder.typicode.com/posts/{}";      echo "sequential: $(ms $t) ms"
t=$(date +%s%N); seq 1 10 | xargs -P 5 -I{} curl -fsS -m 10 -o /dev/null "https://jsonplaceholder.typicode.com/posts/{}"; echo "-P 5:       $(ms $t) ms"
```

The last command starts six jobs at once, with job 1 sleeping the longest. They
finish in reverse order — exactly what happens with tasks on a thread pool.

### 14. Clean up

```bash
rm -rf /tmp/day8
```

---

## Gotchas

1. **`grep` and `awk` match JSON's text, not its structure** — whitespace, key
   order and pretty-printing change the text without changing the data.
2. **Big integers get silently rounded** (above 2^53 in jq 1.6, JavaScript and
   most parsers). Send 64-bit IDs as strings.
3. **Missing keys are `null`, not errors** — a typo in a path fails silently.
4. **`.content-type` is subtraction.** Quote odd keys: `."content-type"`.
5. **Without `-r`, strings keep their quotes** — wrong for filenames, URLs,
   variables.
6. **`@csv` and `@tsv` need an array of scalars**, not objects.
7. **`--arg` is always a string.** Comparing to a number never matches. Use
   `--argjson`.
8. **Single-quote jq programs**, or the shell eats `$var`s.
9. **`group_by`, `sort_by` and friends need one array.** On JSON Lines, slurp
   with `-s`.
10. **Numeric-looking strings sort as text** — `"10"` before `"9"`. Use
    `tonumber`.
11. **jq 1.6 needs `else`**, and lacks `pick`, `abs` and other 1.7 functions.
12. **`jq … f.json > f.json` empties the file** and exits 0. Temp file + `mv`.
13. **curl without `-f` passes error bodies to jq**, and a JSON `404` becomes
    `null`.
14. **`Invalid numeric literal at line 1` means you got HTML**, not JSON.
15. **Empty input gives no output and exit 0.** Silence isn't success.
16. **A pipeline's exit status is the last command's.** `set -o pipefail`.
17. **Unquoted `$(...)` is re-split on whitespace.** Quote it.
18. **`xargs` splits on spaces** — `-d '\n'`, or `-print0 | xargs -0`.
19. **`xargs` runs once on empty input** — add `-r`.
20. **`xargs -P` doesn't preserve order**, and high `-P` against an API gets you
    rate-limited.
21. **`sudo cmd > /root/file` fails** — your shell does the redirect. Use
    `cmd | sudo tee /root/file`.

---

## Recap

- JSON is a **tree**; line tools match its text and break on formatting. `jq`
  parses it.
- A jq program is a **filter**: JSON in, a **stream** of JSON out, composed with
  `|`. `.[]` iterates; `[ … ]` collects; `map(f)` is `[.[] | f]`.
- Paths: `.a.b`, `.[0]`, `.[]`, `."odd-key"`. Missing → `null`; `//` for
  defaults; `?` to suppress errors.
- Toolbox: `select`, `map`, `length`, `keys`, `sort_by`, `group_by`, `to_entries`,
  `test`, `add`. Slurp (`-s`) to aggregate JSON Lines.
- Output: `-r` raw, `-c` compact, `-e` exit status, `@csv`/`@tsv`/`@sh`,
  `"\(interpolation)"`. Input: `--arg` (string) vs `--argjson` (JSON).
- Build with `{name, city: .address.city}` and `{(.k): .v}`; update with `=`,
  `|=`, `del`. **No `-i`** — temp file and `mv`.
- `curl -fsSL -m 10 | jq`, plus **`pipefail`**. Recognise "Invalid numeric
  literal" as HTML.
- Glue: `tee` splits a stream, `"$(cmd)"` captures output, `xargs` turns lines
  into arguments — with `-0`, `-r`, `-I{}` and `-P` for a worker pool.

---

## Say it out loud

About 30 seconds each, out loud, using the real words:

1. Why is `grep '"status":"UP"'` a fragile health check, and what's the jq
   version that isn't?
2. Explain jq's model — filters, streams, `.[]`, and what wrapping in `[ ]`
   does.
3. A Spring API returns `Long` IDs and the frontend shows the wrong order
   numbers. What's happening, and what's the fix?
4. Why does `jq '…' config.json > config.json` leave an empty file, and how do
   you edit JSON in place safely?
5. Your script does `curl "$URL" | jq '.name'` and happily processes a user
   called `null`. Name everything that went wrong and the flags that fix it.
6. Why does `xargs` exist? Mention `ARG_MAX`, and one command that ignores
   stdin.
7. Explain `xargs -P 4` as a worker pool — and the two consequences that come
   with it.

---

## Q&A
