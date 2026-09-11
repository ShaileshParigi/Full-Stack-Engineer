# Day 9 — Shell Scripting

_Why this matters:_ Shell scripts are the glue of every system you'll operate.
The `entrypoint.sh` in a Docker image. Every `run:` step in a GitHub Actions
pipeline. Deploy scripts, Kubernetes init containers, the cron jobs from Day 5,
and tomorrow's checkpoint log-watcher. A script's job is usually small, but it
runs unattended — at 3am, in CI, inside a container nobody is watching — so the
bar isn't "does it work when I run it", it's "**does it fail loudly and clean up
after itself when something goes wrong**". Nearly every real-world shell bug
comes from the same half-dozen causes: unquoted variables, the holes in
`set -e`, missing cleanup, Windows line endings, the wrong interpreter, and
failures hidden inside pipelines. Today covers all of them.

> **This is the densest day in Phase 1.** If it runs past two hours, split it:
> Parts 1–4 plus hands-on 1–7 today; Parts 5–8 plus hands-on 8–14 next session.
> The required practice is hands-on 11–13.

---

## The one-paragraph version

A shell script is just the commands you'd type, saved in a file, with a first
line (`#!/usr/bin/env bash`) that says which program should run it, and the
execute permission switched on. What makes it *programming* is variables,
conditions, loops and functions — and the shell's version of each has sharp
edges. The biggest one: **when you use a variable unquoted, the shell splits its
value on spaces and expands any wildcards in it**, so a filename like
`my report.txt` turns into two separate words — always write `"$var"`. The
second: scripts don't stop when a command fails; they carry on with the next
line, broken. So every script starts with **`set -euo pipefail`**: stop on any
failure (`-e`), treat a misspelled or unset variable as an error (`-u`), and
don't let a failure hide inside a pipeline (`pipefail`). Those flags have holes
you need to know about. Finally, **`trap … EXIT`** registers cleanup code that
runs however the script ends — success, failure, or being killed — so temp files
never leak. Quote everything, fail loudly, clean up.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **script** | A file of shell commands you can run like a program | A text file executed by an interpreter named in its shebang |
| **shebang** | The `#!` first line saying which program runs the file | `#!interpreter [arg]` — read by the kernel's `execve` |
| **interpreter** | The program that reads and runs the script | Here `bash`; `sh` on Ubuntu is a different, smaller shell (`dash`) |
| **execute bit** | The permission that lets a file be run | The `x` in `rwx` (Day 3); `chmod +x` |
| **`PATH`** | The list of folders searched when you type a command name | Colon-separated directories, searched left to right |
| **sourcing** | Running a script *inside* your current shell instead of a new one | `source file` / `. file` — no child process |
| **CRLF** | Windows line endings: carriage return + line feed | `\r\n`; Linux uses LF (`\n`) only |
| **expansion** | The shell replacing `$var`, `$(cmd)`, `*` with their values | The steps the shell performs on a command line before running it |
| **word splitting** | The shell chopping an unquoted value into separate words at spaces | Splitting unquoted expansion results on the characters in `IFS` |
| **globbing** | The shell turning `*.txt` into a list of matching filenames | Pathname expansion |
| **`IFS`** | The characters the shell splits words on | Internal Field Separator: space, tab, newline by default |
| **exit status** | The 0–255 number a command returns; 0 means success | `$?`; in shell conditions, 0 is **true** |
| **`[ ]` / `test`** | The old way to ask "is this file there?", "are these equal?" | A command (builtin and `/usr/bin/[`) whose last argument must be `]` |
| **`[[ ]]`** | Bash's safer, smarter version of `[ ]` | A shell keyword: no word splitting or globbing inside, adds `=~` and patterns |
| **arithmetic expansion** | Integer maths in the shell | `$(( expr ))`; `(( expr ))` as a condition |
| **positional parameters** | The arguments given to a script or function | `$1`, `$2`, …; `$#` is how many, `"$@"` is all of them |
| **function** | A named block of commands you can call | Runs in the current shell; gets its own `$1`, `$2`, … |
| **`local`** | A variable that exists only inside one function | Dynamically scoped function-local variable |
| **subshell** | A copy of the shell running in a child process | Created by `( )`, `$( )`, and each side of a pipe; changes don't flow back |
| **errexit** | "Stop the script when a command fails" | `set -e` |
| **nounset** | "Using a variable that was never set is an error" | `set -u` |
| **pipefail** | "A pipeline fails if *any* part of it fails" | `set -o pipefail`; default is last-command-only |
| **signal** | A notification the kernel delivers to a process (Day 4) | `SIGTERM` (15), `SIGINT` (2), `SIGKILL` (9, uncatchable) |
| **`trap`** | "When this happens, run that code" | Registers a handler for signals or the pseudo-signal `EXIT` |
| **here-doc** | A multi-line block of text fed to a command as input | `<<EOF … EOF` redirection |
| **process substitution** | Treat a command's output as if it were a file | `<(cmd)` — expands to a path like `/dev/fd/63` |
| **`shellcheck`** | A spell-checker for shell scripts | Static analyser that flags quoting, `set -e` and portability bugs |

---

## Before you read: three questions

1. You write `deploy.sh` in VS Code on Windows, then run `./deploy.sh` in WSL
   and get `/usr/bin/env: 'bash\r': No such file or directory`. The first line
   clearly says `#!/usr/bin/env bash`. What's wrong?
2. `file="my report.txt"; rm $file` — which files does `rm` try to delete?
3. A script starting with `set -e` has `count=0` and then `(( count++ ))`. It
   dies on that line, printing nothing. Why?

---

## Part 1 — What a script is, and how it runs

**In plain words:** a script is a text file of commands. To run it like a real
program, three things must be true: the first line says which program should
read it, the file has execute permission, and you tell the shell *where* it is.
Each of those three fails in its own way, and each failure has a different
error message worth recognising.

```
#!/usr/bin/env bash
echo "hello from $0"
```

### The shebang

When you run `./deploy.sh`, the kernel looks at the first two bytes. If they're
`#!`, it runs the program named on that line and hands it the file. So
`#!/usr/bin/env bash` means "run `env`, which finds `bash` on the `PATH`, and
give it this file".

- `#!/usr/bin/env bash` — finds bash wherever it lives. The portable choice.
- `#!/bin/bash` — a fixed path. Fine on Ubuntu, wrong on systems where bash lives
  elsewhere (macOS with a newer Homebrew bash, NixOS, some containers).
- **`sh script.sh` ignores the shebang.** On Ubuntu, `sh` is **dash** — a
  smaller, faster POSIX shell. Bash features like `[[ ]]` fail in dash:
  `[[: not found`. The nasty part: an `if [[ … ]]` that fails this way just
  counts as "false", so the script can finish with exit status **0** having
  skipped its real work. Run bash scripts as `./script.sh` or `bash script.sh`.

### The execute bit

```
chmod +x deploy.sh     # without it: "Permission denied", exit status 126
```

**WSL trap:** files under `/mnt/c` and `/mnt/d` live on the Windows filesystem,
which has no Unix permission bits. On your setup every file there shows as
`rwxrwxrwx`, and **`chmod` silently does nothing**. Scripts you develop and run
in Linux belong in your Linux home directory (`~`) — that's also much faster
(Day 1).

### Where is it? `PATH`, and why you type `./`

When you type a bare command name, the shell searches the directories in
`$PATH`, left to right. The **current directory is deliberately not on it** —
otherwise anyone who could drop a file named `ls` into a directory you `cd` into
could run code as you. So a script in the current directory needs `./`:

```
hello.sh      # bash: hello.sh: command not found   — exit status 127
./hello.sh    # runs
```

For your own tools, make a `~/bin` directory. Ubuntu's `~/.profile` adds
`~/bin` to `PATH` if it exists — at **login**, so open a new terminal (or run
`source ~/.profile`) after creating it.

### Running vs sourcing

`./script.sh` starts a **new** bash process (a child). Anything the script does
to its own environment — `cd`, setting variables, `export` — dies with that
child. Your terminal is untouched.

`source script.sh` (or `. script.sh`) runs the commands **in your current
shell**. That's how `~/.bashrc` works, and how you load a file of settings
(`source .env`). It's also why a script that does `cd /somewhere` doesn't move
you anywhere — hands-on 2 shows it.

### Windows line endings — the WSL-specific classic

Windows editors end lines with `\r\n` (**CRLF**); Linux expects `\n` (LF). A
script saved with CRLF has a first line that really reads `#!/usr/bin/env bash\r`,
so the kernel goes looking for a program called `bash\r`:

```
/usr/bin/env: 'bash\r': No such file or directory
```

That's the answer to question 1. The same bug inside Docker usually appears as
`exec /entrypoint.sh: no such file or directory` — a message that sends people
looking for a missing file for hours. Fixes:

- **Now:** `sed -i 's/\r$//' deploy.sh` — Day 7's `sed`, stripping the `\r` from
  every line end. `file deploy.sh` tells you if a file has "CRLF line
  terminators".
- **In VS Code:** click **CRLF** in the bottom-right status bar and switch to
  **LF**, or set `"files.eol": "\n"`.
- **In git:** a `.gitattributes` line `*.sh text eol=lf`, so a Windows checkout
  doesn't convert your scripts.

> **Say this in an interview:** "When a file starting with `#!` is executed, the
> kernel runs the named interpreter with the script as an argument — so it needs
> the execute bit, a valid interpreter path, and LF line endings; a CRLF shebang
> becomes `bash\r`, which doesn't exist. Running it as `sh script.sh` bypasses
> the shebang entirely, and on Debian and Ubuntu `sh` is dash, so bashisms break.
> Executing starts a child process; sourcing runs it in the current shell, which
> is the only way a script can change the caller's directory or variables."

---

## Part 2 — Variables and quoting: the one rule

**In plain words:** when the shell sees `$var` **without quotes**, it doesn't
just paste in the value. It then chops that value into separate words wherever
there's a space, and treats any `*` or `?` in it as a filename wildcard. A
variable holding `my report.txt` becomes two arguments: `my` and `report.txt`.
Put the variable in double quotes — `"$var"` — and it stays exactly one argument,
whatever it contains. That's the whole rule, and it prevents more bugs than
anything else in this lesson.

### Setting and using

```
name="Shailesh"          # no spaces around =
echo "hello $name"
greeting = "hi"          # WRONG: runs a command called "greeting" with args "=" and "hi"
```

### What unquoted expansion actually does

After replacing `$file` with its value, the shell applies **word splitting**
(split on the characters in `IFS` — space, tab, newline) and **globbing**
(`*`, `?`, `[…]` become matching filenames) to the result. Neither happens
inside double quotes.

```
file="my report.txt"
rm $file       # rm receives TWO arguments: "my" and "report.txt"
rm "$file"     # rm receives ONE argument: "my report.txt"
```

The answer to question 2: `rm $file` tries to delete a file called `my` and a
file called `report.txt`. If `report.txt` exists — maybe it's the important
one — **it's gone**. Hands-on 4 does exactly this.

### When quotes aren't needed (so you recognise them, not so you skip them)

- On the right-hand side of an assignment: `copy=$file` doesn't split.
- Inside `[[ ]]`: `[[ -f $file ]]` doesn't split (Part 3).
- Inside `(( ))` arithmetic.

Everywhere else — command arguments, `[ ]`, `for` lists, `echo` — quote. When in
doubt, quote. `shellcheck` will tell you when you forgot.

### Single vs double quotes (Day 6, again)

| | Expands `$var`, `$(cmd)` | Splits and globs |
|---|---|---|
| `'single'` | No — completely literal | No |
| `"double"` | Yes | No |
| unquoted | Yes | **Yes** |

### Parameter expansion — string surgery without extra tools

Bash can transform a variable's value as it expands it. You'll use these
constantly; the practice script uses three.

| Expansion | Gives | With `f=/var/log/app.tar.gz` |
|---|---|---|
| `${f##*/}` | Remove the longest match of `*/` from the front — the file name | `app.tar.gz` |
| `${f%/*}` | Remove the shortest match of `/*` from the end — the directory | `/var/log` |
| `${f##*.}` | Everything after the **last** dot — the extension | `gz` |
| `${f%.*}` | Remove the last `.something` | `/var/log/app.tar` |
| `${#f}` | Length in characters | `19` |
| `${f/log/LOG}` | Replace the first match | `/var/LOG/app.tar.gz` |
| `${1:-default}` | The value, or `default` if unset or empty | |
| `${1:?message}` | The value, or **exit** with `message` if unset or empty | |

Memory aid: `#` is on the left of `$` on the keyboard, so it trims from the
**left**; `%` is on the right, so it trims from the **right**. Doubled (`##`,
`%%`) means "as much as possible".

### `printf`, not `echo`, for data

`echo "$var"` misbehaves when `$var` happens to be `-n` or `-e` — `echo`
treats it as an option and prints nothing. `printf '%s\n' "$var"` prints any
value exactly. Use `echo` for fixed messages, `printf` for data.

> **Say this in an interview:** "Unquoted parameter expansion is subject to word
> splitting on `IFS` and to pathname expansion, so a value containing spaces or
> glob characters turns into multiple arguments — `rm $file` with a space in the
> name deletes the wrong files. I double-quote every expansion except where the
> shell doesn't split — assignments, `[[ ]]`, and arithmetic — and use parameter
> expansion like `${f##*/}` and `${var:?}` instead of spawning `basename` or
> writing manual checks."

---

## Part 3 — Arithmetic and conditions

**In plain words:** in the shell, **a condition is just a command**, and
"true" means "the command succeeded" — exit status **0**. That's backwards from
Java, where 0 is false. `if grep -q ERROR log` literally means "if grep
succeeds". The brackets people write in conditions, `[ ]` and `[[ ]]`, are also
just commands that succeed or fail. Maths is integers only, inside `$(( ))`.

### Arithmetic

```
total=$(( 7 + 3 ))        # 10
echo $(( 7 / 2 ))         # 3 — integers only, truncated
echo $(( 7 % 2 ))         # 1
echo $(( 2 ** 10 ))       # 1024
(( total > 5 )) && echo "big"      # (( )) as a condition
count=$(( count + 1 ))    # the safe way to increment — see Part 6 for why not (( count++ ))
```

Two traps:

- **No decimals.** For averages and percentages, use awk (Day 7):
  `awk -v a="$x" -v b="$y" 'BEGIN { printf "%.1f\n", a / b }'`.
- **A leading zero means octal.** `$(( 010 ))` is 8, and `$(( 08 ))` is an
  error: `value too great for base`. Months and days from `date` have leading
  zeros — so date arithmetic that works in July breaks on August 1st. Force base
  10: `$(( 10#$month + 1 ))`.

### `if` runs a command and checks its exit status

```
if grep -q "ERROR" app.log; then
  echo "errors found"
elif [[ -s app.log ]]; then
  echo "log is clean and non-empty"
else
  echo "log is empty"
fi
```

`&&` and `||` are mini-ifs: `mkdir -p out && cd out`, `[[ -d $dir ]] || exit 66`.
But **`a && b || c` is not if-then-else** — if `b` fails, `c` runs too.

### `[ ]` — a command with sharp edges

`[` is literally a command (`type [` says "shell builtin"; `/usr/bin/[` exists
too). Its arguments are the test, and its last argument must be `]`. Because it
is an ordinary command, its arguments go through word splitting and globbing
first — so unquoted variables break it:

```
x=""
[ $x = foo ]        # [: =: unary operator expected    (it saw:  [ = foo ])
y="a b"
[ $y = foo ]        # [: too many arguments            (it saw:  [ a b = foo ])
[ "$a" > "$b" ]     # > is a REDIRECT: creates a file named after $b, returns true
```

### `[[ ]]` — use this in bash

`[[ ]]` is a bash **keyword**, parsed by the shell itself, so none of that
happens:

- No word splitting or globbing inside — `[[ -f $file ]]` is safe even unquoted.
- `&&`, `||` and `!` work inside it.
- `==` does **glob-pattern** matching: `[[ $f == *.log ]]`.
- `=~` does **regex** matching, with the captured groups in `BASH_REMATCH`:
  `[[ $v =~ ^order-([0-9]+)$ ]] && echo "${BASH_REMATCH[1]}"`.
- `<` and `>` compare **as strings**: `[[ 10 < 9 ]]` is **true** ("1" sorts
  before "9"). The Day 7 awk trap, one more time.

**For numbers, use `-lt` style operators or `(( ))`:** `[[ $n -lt 9 ]]` or
`(( n < 9 ))`.

### Test operators

| Operator | True when |
|---|---|
| `-e path` | It exists (anything) |
| `-f path` | It's a regular file |
| `-d path` | It's a directory |
| `-s path` | It exists and isn't empty |
| `-r` / `-w` / `-x path` | Readable / writable / executable by you |
| `-z "$s"` | The string is empty |
| `-n "$s"` | The string is not empty |
| `a == b` / `a != b` | Strings equal / differ (`=` in `[ ]`) |
| `-eq -ne -lt -le -gt -ge` | Integer comparisons |

> **Say this in an interview:** "Shell conditionals test exit status, where 0 is
> true. `[` is an ordinary command, so its operands undergo word splitting and
> globbing — an empty or spaced variable gives 'unary operator expected' or 'too
> many arguments', and `>` becomes a redirection. `[[ ]]` is a keyword that
> suppresses splitting and adds pattern and regex matching, but its `<` is a
> string comparison, so for numbers I use `-lt` or arithmetic `(( ))`."

---

## Part 4 — Loops, `case`, and reading input

**In plain words:** `for` walks over a list, `while` repeats as long as a
command succeeds, and `case` picks a branch by matching a value against
patterns. The most important loop in shell scripting reads a file one line at a
time, and it has a precise, slightly odd, correct form that you should copy
exactly rather than improvise.

### `for`

```
for f in *.log; do          # a glob — safe with spaces in names
  echo "processing $f"
done

for i in {1..5}; do …; done
for (( i = 0; i < 5; i++ )); do …; done
```

- **Never `for f in $(ls)`.** The output of `ls` gets word-split, so
  `my file.txt` becomes two iterations. The glob `*.txt` doesn't have that
  problem.
- **If the glob matches nothing, the loop runs once with the literal pattern**
  — `f` is the string `*.log`. Guard with `[[ -e $f ]] || continue`, or
  `shopt -s nullglob` so a non-matching glob expands to nothing.

### Reading a file line by line — the exact form

```
while IFS= read -r line; do
  printf '%s\n' "$line"
done < input.txt
```

Every piece is there for a reason:

| Piece | Without it |
|---|---|
| `IFS=` | Leading and trailing spaces are stripped from each line |
| `-r` | Backslashes are eaten: `C:\temp\new` becomes `C:tempnew` |
| `done < input.txt` | *(see below)* |

**Don't pipe into `while`.** `cat input.txt | while read …` runs the loop on the
right side of a pipe — in a **subshell**. Any variable the loop changes is
changed in that child process and thrown away when it ends:

```
count=0
cat input.txt | while IFS= read -r line; do count=$((count + 1)); done
echo "$count"     # 0 — the loop incremented a copy
```

Redirect the file into the loop (`done < file`), or, when the input comes from a
command, use **process substitution**: `done < <(find . -name '*.log')`. The
`<(cmd)` part runs `cmd` and presents its output as a readable file, so the loop
stays in your main shell.

One more edge: **a final line with no trailing newline is skipped** by
`while read`. Files written by some editors and tools end that way. The fix is
`while IFS= read -r line || [[ -n $line ]]; do`.

For filenames, which can legally contain newlines, pair `find -print0` with
`read -d ''` (read up to a NUL byte). The practice script does this.

### `case`

```
case "$1" in
  start|up)      echo "starting";;
  stop)          echo "stopping";;
  *.conf)        echo "config file: $1";;
  -h|--help)     usage;;
  *)             echo "unknown: $1" >&2; exit 64;;
esac
```

Patterns are globs, `|` means "or", `;;` ends a branch, `*)` is the default.
It's the idiomatic way to parse sub-commands and options.

### Reading from the user, and here-docs

```
read -r -p "Environment? " env      # prompt and read one line
read -rs -p "Password: " pw         # -s: don't echo what's typed
```

A **here-doc** feeds a block of text to a command's stdin — the standard way to
generate a config file from a script:

```
cat > app.env <<EOF
APP_ENV=$env
GENERATED_AT=$(date +%F)
EOF
```

| Form | Behaviour |
|---|---|
| `<<EOF` | `$vars` and `$(commands)` are expanded |
| `<<'EOF'` | Completely literal — nothing expanded |
| `<<-EOF` | Leading **tabs** (not spaces) are stripped, so you can indent it |
| `<<< "$text"` | A **here-string**: one string as stdin |

To write a root-owned file from a script, combine a here-doc with Day 8's `tee`:
`sudo tee /etc/myapp.conf > /dev/null <<'EOF'`.

> **Say this in an interview:** "I iterate files with globs, never `$(ls)`, and
> guard against a non-matching glob. Line-by-line reading is
> `while IFS= read -r line; do …; done < file` — `IFS=` preserves whitespace,
> `-r` preserves backslashes, and redirecting instead of piping keeps the loop
> out of a subshell so its variable changes survive; for command output I use
> process substitution. `case` handles pattern dispatch."

---

## Part 5 — Functions, arguments and exit codes

**In plain words:** a script receives its arguments as `$1`, `$2` and so on. A
function works the same way — inside it, `$1` is the function's first argument,
not the script's. Functions can't return data the way Java methods do; they
return only a 0–255 success code. To get data *out*, the function prints it and
the caller captures it with `$( )`. And every variable is **global** unless you
say `local`.

### Arguments

| Variable | Means |
|---|---|
| `$0` | The script's name (as it was invoked) |
| `$1` … `$9`, `${10}` | Positional arguments |
| `$#` | How many arguments |
| `"$@"` | All arguments, **each kept as a separate word** — use this |
| `"$*"` | All arguments joined into **one** word |
| `shift` | Drop `$1`; everything moves down one |

`"$@"` is the only correct way to pass arguments along: with arguments
`"one two" three`, `"$@"` gives two words, while `$*` unquoted gives three.

### Functions

```
greet() {
  local name=$1                     # local: invisible outside this function
  printf 'hello, %s\n' "$name"
}
msg=$(greet "Shailesh")             # capture the function's output
```

- **Variables are global by default.** A function that sets `i` or `tmp` without
  `local` overwrites the caller's `i` or `tmp`. Declare them `local`.
- **`return N` sets the function's exit status (0–255).** It isn't a return
  value — `return 256` gives 0, because only the lowest 8 bits survive. Data
  goes out through stdout.
- **`exit N` ends the whole script** — even from inside a function. *Except*
  inside `$( )`: that's a subshell, so `exit` only ends the subshell, and the
  script carries on.
- **Errors go to stderr:** `echo "error: …" >&2`. Then `result=$(myfunc)`
  captures only the real output, and the messages still reach the terminal.

### The shape of a well-behaved script

```
#!/usr/bin/env bash
set -euo pipefail

usage() { echo "usage: ${0##*/} DIR" >&2; exit 64; }
die()   { echo "${0##*/}: $2" >&2; exit "$1"; }

main() {
  [[ $# -eq 1 ]] || usage
  …
}

main "$@"
```

Everything lives in functions; the last line calls `main` with all the script's
arguments. Helpers like `usage` and `die` give every failure a clear message on
stderr **and** a distinct exit code.

### Exit codes that mean something

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | General failure |
| `2` | Misuse of a shell builtin — bash's own convention |
| `64` | Command-line usage error (`EX_USAGE`) |
| `66` | Input file or directory missing (`EX_NOINPUT`) |
| `69` | A required service is unavailable (`EX_UNAVAILABLE`) |
| `75` | Temporary failure — worth retrying (`EX_TEMPFAIL`) |
| `78` | Configuration error (`EX_CONFIG`) |
| `126` | Found, but not executable |
| `127` | Command not found |
| `128 + N` | Killed by signal N: **130** = Ctrl-C, **137** = `SIGKILL`, **143** = `SIGTERM` |

The 64–78 range comes from BSD's `sysexits.h` — use it and your callers can tell
"you called me wrong" from "the input's missing" from "try again later".
**137 is the one you'll see most in production:** a Kubernetes container that
was OOM-killed exits with 137, because the kernel sent it `SIGKILL`.

> **Say this in an interview:** "Positional parameters are `$1` onwards, `$#` is
> the count, and `"$@"` is the only form that preserves each argument intact.
> Variables are global unless declared `local`; functions return a status code,
> not data, so data comes back via stdout and command substitution — and `exit`
> inside `$( )` only exits the subshell. I use distinct exit codes — sysexits
> for usage and missing input, and I know that 128 plus N means death by signal
> N, so 137 is a SIGKILL, usually the OOM killer."

---

## Part 6 — `set -euo pipefail`

**In plain words:** by default, a bash script ignores failures. A command fails,
bash shrugs and runs the next line — with an empty variable, a missing file, a
half-finished state. `set -euo pipefail` switches on three separate safety
checks: stop when a command fails; stop when you use a variable that was never
set; and don't let a failure hide in the middle of a pipeline. It's the first
line after the shebang in every script you write — but it's a tripwire with
known gaps, not a guarantee, and interviewers ask about the gaps.

### `-e` (errexit): stop on the first failure

Without it:

```
cd /nonexistent/dir
rm -rf ./*         # runs anyway — in whatever directory you were already in
```

With `set -e`, the failed `cd` stops the script before the `rm`.

### `-u` (nounset): unset variables are errors

```
rm -rf "$BUILD_DIR/"*     # BUILD_DIR unset → expands to  rm -rf /*
```

With `set -u` that's `BUILD_DIR: unbound variable` and an immediate exit. It
also catches typos — `$nmae` instead of `$name`. For arguments that are allowed
to be missing, give a default: `"${1:-}"` or `"${1:-default}"`.

### `-o pipefail`: failures inside pipelines count

A pipeline's exit status is normally the **last** command's (Day 8). So
`curl -f "$url" | jq .` "succeeds" even when curl failed. With `pipefail`, the
pipeline fails if any command in it fails. `${PIPESTATUS[@]}` holds every
stage's status if you need to know which one.

### Where `-e` does *not* stop you

These are all real, all silent, and all shown in hands-on 9:

1. **Anything tested by `if`, `while`, `&&`, `||` or `!`.** A failure there is
   treated as an answer, not an error — which is correct for `if grep -q …`.
   But it applies to **everything called from that context**: a function called
   inside `if f; then` runs with `-e` switched off, so a failure halfway
   through `f` doesn't stop `f`.
2. **`local var=$(cmd)` hides `cmd`'s failure.** The line's exit status is
   `local`'s, which always succeeds. Split it: `local var; var=$(cmd)`.
3. **Inside `$( )`, `-e` isn't inherited.** Commands in the substitution keep
   running after a failure. (`shopt -s inherit_errexit` changes that.)
4. **`(( count++ ))` when `count` is 0 kills the script.** `(( ))` returns
   failure when the expression's value is 0, and the value of `count++` is the
   *old* value — 0. That's question 3. Write `count=$(( count + 1 ))`.
5. **`grep` finding nothing exits 1**, so `n=$(grep -c ERROR log)` kills the
   script on a clean log. When "no match" is a normal outcome:
   `n=$(grep -c ERROR log || true)`.

### The `pipefail` edge: `SIGPIPE`

```
seq 1 1000000 | grep -q 5      # grep finds 5 immediately — and the pipeline exits 141
```

`grep -q` (or `head -1`) stops reading as soon as it has its answer and exits.
The command still writing into the pipe then gets `SIGPIPE` (signal 13, so
128 + 13 = **141**). With `pipefail`, that makes the whole pipeline "fail" — and
with `-e`, your script dies *because the search succeeded*. This shows up with
big inputs, so tests on small files pass. Fix: avoid the pipe —
`grep -q 5 < <(seq 1 1000000)` — because process substitution's status isn't
checked.

### Seeing what a script is doing: `set -x`

`set -x` (or `bash -x script.sh`) prints every command, fully expanded, before
running it, prefixed with `+`. It's the fastest way to see what the shell
*actually* did with your quotes.

> **Say this in an interview:** "`set -e` exits on an unhandled non-zero
> status, `-u` treats unset variables as errors, and `pipefail` makes a
> pipeline's status the last non-zero one rather than the final command's. The
> gaps in `-e`: it's suspended in any condition context — `if`, `while`, `&&`,
> `||`, `!` — including functions called from there; `local x=$(cmd)` masks the
> failure; command substitutions don't inherit it; and `(( i++ ))` from zero
> returns 1. With pipefail, early-exiting readers like `grep -q` or `head` can
> make the writer die of SIGPIPE, giving 141. So I treat these flags as a
> tripwire and still check the critical commands explicitly."

---

## Part 7 — `trap`: clean up however the script ends

**In plain words:** scripts create temporary files, lock files and background
processes. If the script fails halfway, or someone presses Ctrl-C, or
Kubernetes stops the pod, that cleanup code at the bottom never runs — and the
junk piles up. `trap` lets you register cleanup code **once, near the top**, and
the shell runs it on the way out, however the exit happens.

```
tmp=$(mktemp)                          # a unique, private temp file: /tmp/tmp.XXXXXXXXXX
trap 'rm -f "$tmp"' EXIT               # run this when the script exits — for any reason
```

- **`EXIT`** is a pseudo-signal: the handler runs on a normal finish, on
  `exit N`, on a `set -e` failure, and when the script is killed by `SIGTERM`
  or `SIGINT` (Ctrl-C). Hands-on 10 checks every one of those.
- **`SIGKILL` can't be trapped** (Day 4). `kill -9` or the OOM killer leave your
  temp files behind. Nothing can help with that — it's why `/tmp` gets cleaned
  at boot.
- **The script's exit status survives the trap.** A script that failed with 1
  still exits 1 after the cleanup runs.
- **`mktemp`, never a fixed name** like `/tmp/myscript.tmp`. Two copies running
  at once would share it, and a fixed, predictable name in a world-writable
  directory is a known security hole. `mktemp -d` makes a temp directory.

### Two trap bugs

**Single quotes, not double.** `trap "rm -f $tmp" EXIT` expands `$tmp` *now*,
when the trap is set. `trap 'rm -f "$tmp"' EXIT` expands it when the trap
*runs*. Usually you want the second, and `shellcheck` flags the first (SC2064).

**Don't make the trapped variable `local`.** The `EXIT` trap runs after `main`
has returned, when its local variables no longer exist. With `set -u`, the
cleanup itself crashes with "unbound variable" — and the temp file leaks.
Hands-on 10 shows it. Keep the variables a trap uses global.

### Why this matters in containers

When Kubernetes stops a pod, it sends `SIGTERM` to the container's main process,
waits (30 seconds by default), then sends `SIGKILL`. If your entrypoint script is
that main process, a `trap … TERM` (or `EXIT`) is your only chance to shut down
cleanly. In Phase 4 you'll see why an entrypoint usually ends with
`exec java -jar app.jar` — `exec` *replaces* the shell with the JVM, so the JVM
receives the `SIGTERM` directly instead of a shell that might not pass it on.

> **Say this in an interview:** "I create temp files with `mktemp` and register
> cleanup with `trap … EXIT` immediately after, using single quotes so variables
> expand when the trap fires. The EXIT trap runs on normal exit, explicit exit,
> errexit, and on SIGTERM or SIGINT, and preserves the exit status — but nothing
> runs on SIGKILL. In containers that matters because Kubernetes sends SIGTERM
> and then SIGKILL after the grace period, which is also why entrypoints `exec`
> the real process."

---

## Part 8 — Tools that catch the bugs for you

**In plain words:** most shell bugs are well known and mechanical to spot — an
unquoted variable here, a `local` masking a failure there. So there's a tool
that spots them: `shellcheck`. Run it on every script before you trust it.

```
sudo apt-get install -y shellcheck      # in your Ubuntu terminal — needs your password
shellcheck count-ext.sh
```

It explains each warning and gives it a code (`SC2086`: "Double quote to prevent
globbing and word splitting"), and every code has a page on the ShellCheck wiki
explaining the bug. Most editors, including VS Code, have a ShellCheck extension
that underlines problems as you type.

Two built-in checks as well:

- `bash -n script.sh` — parse the script without running it; catches syntax
  errors like a missing `fi`.
- `bash -x script.sh` — run it, tracing every expanded command.

> **Say this in an interview:** "I run shellcheck on every script — it catches
> unquoted expansions, masked return values, and bashisms under a `sh` shebang —
> and I use `bash -n` for a syntax check and `bash -x` to trace what the shell
> actually executed."

---

## Hands-on

Open the **Ubuntu (WSL)** tab. **Run block 1 first.** **Blocks 11, 12 and 13
are the plan's required practice**: build the extension-counting script, run it
through every exit path, then break it with an unquoted variable.

For block 13 you'll want `shellcheck`. In your Ubuntu terminal (the lab can't
type your password):

```
sudo apt-get install -y shellcheck
```

### 1. Set up a sample directory — with awkward names on purpose

```bash
rm -rf /tmp/day9 && mkdir -p /tmp/day9 && cd /tmp/day9
mkdir -p sample/src/main sample/src/test sample/logs
touch sample/pom.xml sample/README.md sample/Makefile sample/.gitignore \
      sample/notes.txt "sample/my notes.txt" sample/archive.tar.gz \
      sample/src/main/App.java sample/src/main/Order.java sample/src/test/OrderTest.java \
      sample/logs/app.log sample/logs/app.log.1
cp -r sample "my project"
find sample -type f | sort
echo "--- shellcheck available?"; command -v shellcheck || echo "not installed (needed for block 13)"
```

### 2. Make a script run — three failure modes, then success

```bash
cd /tmp/day9
cat > hello.sh <<'EOF'
#!/usr/bin/env bash
if [[ -n "${1:-}" ]]; then echo "hello, $1"; else echo "hello from ${0##*/}"; fi
EOF
echo "--- not on PATH:";        hello.sh; echo "exit=$?"
echo "--- no execute bit:";     ./hello.sh; echo "exit=$?"
chmod +x hello.sh
echo "--- now:";                ./hello.sh Shailesh; echo "exit=$?"
echo "--- via sh (dash) — the [[ ]] is not understood, and the exit status still says success:"
sh hello.sh Shailesh; echo "exit=$?"
echo "--- running vs sourcing a script that does cd:"
printf '#!/usr/bin/env bash\ncd /tmp\n' > gotmp.sh; chmod +x gotmp.sh
./gotmp.sh;      echo "after ./gotmp.sh:     $PWD"
source gotmp.sh; echo "after source gotmp.sh: $PWD"
```

### 3. The Windows line-endings failure, and the fix

```bash
cd /tmp/day9
printf '#!/usr/bin/env bash\r\necho "written on Windows"\r\n' > win.sh
chmod +x win.sh
file win.sh
./win.sh; echo "exit=$?"
sed -i 's/\r$//' win.sh
file win.sh
./win.sh; echo "exit=$?"
```

### 4. Quoting: delete the wrong file, on purpose

```bash
mkdir -p /tmp/day9/q && cd /tmp/day9/q && rm -f ./*
touch "my report.txt" report.txt
echo "before:"; ls -1
file="my report.txt"
echo "--- rm -v \$file (unquoted):"
rm -v $file
echo "after:"; ls -1
echo "^ 'report.txt' is gone, and the file we meant is still here"
echo "--- an unquoted variable holding a * is globbed too:"
pattern="*.txt"
echo $pattern
echo "$pattern"
```

### 5. Parameter expansion and `printf`

```bash
f=/var/log/app.tar.gz
echo "name:      ${f##*/}"
echo "directory: ${f%/*}"
echo "extension: ${f##*.}"
echo "no ext:    ${f%.*}"
echo "length:    ${#f}"
echo "replace:   ${f/log/LOG}"
unset maybe
echo "default:   ${maybe:-fallback}"
( : "${maybe:?maybe must be set}" ); echo "exit=$?   <- \${var:?} stopped that subshell"
echo "--- echo vs printf when the data is '-n':"
data="-n"
echo "$data"; echo "(echo printed nothing — it took -n as an option)"
printf '%s\n' "$data"
```

### 6. Conditions and arithmetic

```bash
cd /tmp/day9
x=""; y="a b"
echo "--- [ ] with an empty variable:";   [ $x = foo ]; echo "exit=$?"
echo "--- [ ] with a spaced variable:";   [ $y = foo ]; echo "exit=$?"
echo "--- [ \"\$a\" > \"\$b\" ] is a redirect:"
a=apple; b=banana
[ "$a" > "$b" ] && echo "said true"; ls -l banana
echo "--- [[ ]] compares as strings, (( )) as numbers:"
[[ 10 < 9 ]] && echo "[[ 10 < 9 ]] is TRUE"
(( 10 < 9 )) || echo "(( 10 < 9 )) is false"
echo "--- regex with capture groups:"
v="order-42"; [[ $v =~ ^order-([0-9]+)$ ]] && echo "order id: ${BASH_REMATCH[1]}"
echo "--- file tests:"
for p in sample sample/pom.xml sample/nope hello.sh; do
  if   [[ -d $p ]]; then echo "$p: directory"
  elif [[ -x $p ]]; then echo "$p: executable file"
  elif [[ -f $p ]]; then echo "$p: regular file"
  else                   echo "$p: does not exist"; fi
done
echo "--- arithmetic: integer division, and the octal trap:"
echo "7/2 = $(( 7 / 2 ))"
month="09"
echo "\$(( month + 1 )):"; echo $(( month + 1 )); echo "exit=$?"
echo "\$(( 10#\$month + 1 )) = $(( 10#$month + 1 ))"
```

### 7. Loops, reading lines, and `case`

```bash
cd /tmp/day9
printf 'alpha\n   indented\nC:\\temp\\new\nlast line, no newline' > lines.txt
echo "--- piping into while: the counter is lost in a subshell"
count=0; cat lines.txt | while IFS= read -r line; do count=$((count + 1)); done; echo "count=$count"
echo "--- redirecting into while: correct, except the last line is missing"
count=0; while IFS= read -r line; do count=$((count + 1)); done < lines.txt; echo "count=$count"
echo "--- with || [[ -n \$line ]]: all four"
count=0; while IFS= read -r line || [[ -n $line ]]; do count=$((count + 1)); printf '  [%s]\n' "$line"; done < lines.txt; echo "count=$count"
echo "--- read without IFS= and -r mangles lines:"
while read line; do printf '  [%s]\n' "$line"; done < lines.txt
echo "--- for over \$(ls) vs a glob:"
for f in $(ls sample/*.txt); do echo "  \$(ls): $f"; done
for f in sample/*.txt; do echo "  glob:  $f"; done
echo "--- a glob that matches nothing:"
for f in sample/*.csv; do echo "  loop ran with: $f"; done
echo "--- case:"
for arg in start stop app.conf --help bogus; do
  case "$arg" in
    start|up)  echo "  $arg -> starting";;
    stop)      echo "  $arg -> stopping";;
    *.conf)    echo "  $arg -> config file";;
    -h|--help) echo "  $arg -> usage";;
    *)         echo "  $arg -> unknown";;
  esac
done
```

### 8. Functions, `local`, `"$@"`, `return` and `exit`

```bash
leaky()  { i=99; }
careful() { local i=99; }
i=1; careful; echo "after careful: i=$i"
i=1; leaky;   echo "after leaky:   i=$i   <- the function overwrote the caller's variable"
show() { echo "  $# args:"; for a in "$@"; do echo "    [$a]"; done; }
echo "--- \"\$@\" keeps arguments intact:"; set -- "one two" three; show "$@"
echo "--- \$* unquoted re-splits them:";      show $*
r() { return 256; }; r; echo "--- return 256 gives: $?"
quit() { exit 3; }
v=$(quit); echo "--- exit inside \$( ) only ended the subshell (status $?) — still running"
```

### 9. `set -euo pipefail`: each flag, then each hole

```bash
cd /tmp/day9
demo() { echo "== $1"; bash -c "$2"; echo "   exit=$?"; }
demo "without -e, a failure is ignored"         'false; echo "   kept going"'
demo "with -e, the script stops"                'set -e; false; echo "   never printed"'
demo "without -u, a typo is an empty string"     'name=x; echo "   hello [$nmae]"'
demo "with -u, it is an error"                   'set -u; name=x; echo "   hello [$nmae]"'
demo "without pipefail, false | true succeeds"   'false | true'
demo "with pipefail, it fails"                   'set -o pipefail; false | true'
echo; echo "---------- the holes in -e ----------"
demo "(( count++ )) from 0 kills the script"     'set -e; count=0; (( count++ )); echo "   never printed"'
demo "count=\$((count + 1)) is safe"             'set -e; count=0; count=$((count + 1)); echo "   count=$count"'
demo "-e is off inside a function called by if" 'set -e; f() { false; echo "   f kept going after false"; }; if f; then echo "   f returned true"; fi'
demo "local v=\$(false) masks the failure"      'set -e; f() { local v=$(false); echo "   carried on"; }; f'
demo "local v; v=\$(false) does not"            'set -e; f() { local v; v=$(false); echo "   never printed"; }; f'
demo "grep -c on a clean log kills the script"   'set -e; n=$(grep -c ERROR /etc/hostname); echo "   never printed"'
demo "... unless || true"                        'set -e; n=$(grep -c ERROR /etc/hostname || true); echo "   n=$n"'
demo "pipefail + grep -q: SIGPIPE, 141"          'set -eo pipefail; seq 1 1000000 | grep -q 5; echo "   never printed"'
demo "process substitution avoids it"            'set -eo pipefail; grep -q 5 < <(seq 1 1000000); echo "   found it"'
```

### 10. `trap`: every way out

```bash
cd /tmp/day9
cat > trapdemo.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
tmp=$(mktemp)
trap 'echo "   cleanup: removing $tmp"; rm -f "$tmp"' EXIT
echo "   working with $tmp"
case "${1:-ok}" in
  ok)    echo "   finished normally";;
  exit)  exit 75;;
  fail)  false;;
  wait)  sleep 5 > /dev/null 2>&1;;
esac
EOF
chmod +x trapdemo.sh
./trapdemo.sh ok;   echo "normal finish:    exit=$?"
./trapdemo.sh exit; echo "explicit exit 75: exit=$?"
./trapdemo.sh fail; echo "set -e failure:   exit=$?"
./trapdemo.sh wait & pid=$!; sleep 0.5; kill -TERM "$pid"; wait "$pid"; echo "SIGTERM:          exit=$?"
echo "leftover temp files from trapdemo: $(grep -c . <(find /tmp -maxdepth 1 -name 'tmp.*' -newer trapdemo.sh) || true)"
echo
echo "--- the local-variable trap bug:"
cat > localtrap.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
main() {
  local tmp
  tmp=$(mktemp)
  echo "$tmp" > leaked-name.txt
  trap 'rm -f "$tmp"' EXIT
  echo "   main done"
}
main
EOF
bash localtrap.sh; echo "exit=$?"
leaked=$(cat leaked-name.txt)
[[ -e $leaked ]] && echo "   $leaked still exists — the cleanup crashed" && rm -f "$leaked"
```

### 11. Required: the extension counter

```bash
cd /tmp/day9
cat > count-ext.sh <<'EOF'
#!/usr/bin/env bash
# count-ext.sh — count the files under a directory, grouped by extension.
# Usage:      count-ext.sh DIR
# Exit codes: 0 success, 64 wrong usage, 66 DIR missing or not a directory
set -euo pipefail

readonly EX_USAGE=64 EX_NOINPUT=66

usage() { echo "usage: ${0##*/} DIR" >&2; exit "$EX_USAGE"; }
die()   { echo "${0##*/}: $2" >&2; exit "$1"; }

# Print a file's extension, or "(none)". A leading dot marks a hidden file,
# not an extension: .gitignore -> (none), app.tar.gz -> gz, Makefile -> (none).
ext_of() {
  local name=${1##*/}
  name=${name#.}
  if [[ $name == *.* ]]; then printf '%s\n' "${name##*.}"; else echo "(none)"; fi
}

# Global on purpose: the EXIT trap runs after main() has returned.
tmp=""

main() {
  [[ $# -eq 1 ]] || usage
  local dir=$1
  [[ -d $dir ]] || die "$EX_NOINPUT" "not a directory: $dir"

  tmp=$(mktemp)
  trap 'rm -f "$tmp"' EXIT

  # Snapshot the file list, NUL-separated so any filename is safe.
  find "$dir" -type f -print0 > "$tmp"

  declare -A count=()
  local f ext total=0
  while IFS= read -r -d '' f; do
    ext=$(ext_of "$f")
    count[$ext]=$(( ${count[$ext]:-0} + 1 ))
    total=$(( total + 1 ))
  done < "$tmp"

  for ext in "${!count[@]}"; do
    printf '%6d  %s\n' "${count[$ext]}" "$ext"
  done | sort -k1,1nr -k2
  printf '%6d  total\n' "$total"
}

main "$@"
EOF
chmod +x count-ext.sh
./count-ext.sh sample; echo "exit=$?"
```

Read the script top to bottom before moving on. Every idea from today is in it:
shebang, strict mode, named exit codes, `usage`/`die` on stderr, parameter
expansion, `[[ ]]`, `local`, a global for the trap, `mktemp` + `trap`,
`find -print0` with `read -d ''`, a loop fed by redirection (not a pipe), a
bash associative array (`declare -A` — the same idea as awk's arrays on Day 7)
and `"$@"`.

### 12. Required: every exit path

```bash
cd /tmp/day9
echo "--- no arguments:";           ./count-ext.sh;                  echo "exit=$?"
echo "--- two arguments:";          ./count-ext.sh a b;              echo "exit=$?"
echo "--- missing directory:";      ./count-ext.sh /nope;            echo "exit=$?"
echo "--- a file, not a directory:"; ./count-ext.sh count-ext.sh;    echo "exit=$?"
echo "--- a directory with a space in its name:"
./count-ext.sh "my project"; echo "exit=$?"
echo "--- an empty directory:"; mkdir -p empty; ./count-ext.sh empty; echo "exit=$?"
echo "--- only stdout captured, errors still visible:"
out=$(./count-ext.sh /nope) || echo "captured='$out' status=$?"
echo "--- no temp files left behind by any run:"
find /tmp -maxdepth 1 -name 'tmp.*' -newer count-ext.sh | wc -l
```

### 13. Required: break it with an unquoted variable

```bash
cd /tmp/day9
sed 's/find "\$dir"/find $dir/' count-ext.sh > broken.sh
chmod +x broken.sh
echo "--- the only difference:"; diff count-ext.sh broken.sh
echo "--- works on a directory without spaces (so the bug survives testing):"
./broken.sh sample | tail -1; echo "exit=$?"
echo "--- fails on one with a space:"
./broken.sh "my project"; echo "exit=$?"
echo "--- and the trap still cleaned up:"
find /tmp -maxdepth 1 -name 'tmp.*' -newer broken.sh | wc -l
echo "--- the same bug with [ ] gives a misleading diagnosis:"
dir="my project"
[ -d $dir ] && echo "is a directory" || echo "'not a directory' — but it is one"
[ -d "$dir" ] && echo "quoted: is a directory"
echo "--- shellcheck finds it before you do:"
if command -v shellcheck > /dev/null; then
  shellcheck broken.sh
  shellcheck count-ext.sh && echo "count-ext.sh: shellcheck clean"
else
  echo "install shellcheck in your Ubuntu terminal: sudo apt-get install -y shellcheck"
fi
```

Look at *how* it failed: `find` complained about `my` and `project` as two
separate paths. That's word splitting — the same thing that deleted the wrong
file in block 4.

### 14. Watch the shell work: `bash -x`

```bash
cd /tmp/day9
mkdir -p tiny && touch tiny/a.java tiny/b.md
bash -x ./count-ext.sh tiny 2>&1 | head -30
```

Every line starting with `+` is a command the shell ran, with every variable
already expanded. When quoting confuses you, this is how you see the truth.

### 15. Clean up

```bash
rm -rf /tmp/day9
```

---

## Gotchas

1. **CRLF line endings** → `'bash\r': No such file or directory`. Fix with
   `sed -i 's/\r$//'`; set VS Code to LF; `*.sh text eol=lf` in `.gitattributes`.
2. **`sh script.sh` runs dash**, not bash — `[[ ]]` fails, often silently.
3. **`chmod` does nothing on `/mnt/c` and `/mnt/d`.** Keep scripts in `~`.
4. **The current directory isn't on `PATH`.** Use `./script.sh`.
5. **A script can't change your terminal's directory** — unless you `source` it.
6. **Unquoted variables are word-split and globbed.** `"$var"`, always.
7. **`x = 1` is a command**, not an assignment. No spaces around `=`.
8. **`echo "$var"` mangles `-n`/`-e` values.** `printf '%s\n' "$var"`.
9. **`[ $x = y ]` breaks on empty or spaced values**, and `[ a > b ]` creates a
   file. Use `[[ ]]`.
10. **`[[ 10 < 9 ]]` is true** — string comparison. Use `-lt` or `(( ))`.
11. **Integer maths only**, and **`08`/`09` are invalid octal**. Use `10#$n`.
12. **`for f in $(ls)` splits names.** Use a glob.
13. **A non-matching glob stays literal.** Check `[[ -e $f ]]` or use `nullglob`.
14. **Piping into `while` loses variable changes.** `done < file` or
    `done < <(cmd)`.
15. **`while read` without `IFS=` and `-r` mangles lines**, and skips a final
    line with no newline without `|| [[ -n $line ]]`.
16. **Variables are global unless `local`.**
17. **`return` is a 0–255 status, not a value.** `return 256` is 0.
18. **`exit` inside `$( )` only exits the subshell.**
19. **`set -e` is suspended in `if`/`while`/`&&`/`||`/`!` contexts** — including
    whole functions called from them.
20. **`local v=$(cmd)` masks the failure.** Declare, then assign.
21. **`(( count++ ))` from 0 kills a `set -e` script.** Use `count=$((count + 1))`.
22. **`grep` with no match exits 1.** `|| true` when that's acceptable.
23. **`pipefail` + `grep -q`/`head` can give 141 (SIGPIPE).** Use `< <(cmd)`.
24. **Double-quoted traps expand too early.** `trap '…' EXIT`.
25. **A trap can't see `local` variables after the function returns.**
26. **Nothing runs on `SIGKILL`** — exit 137, and your temp files stay.

---

## Recap

- A script needs a **shebang** (`#!/usr/bin/env bash`), the **execute bit**
  (not on `/mnt/*`), **LF line endings**, and a path (`./` or `~/bin` on `PATH`).
  `sh` means dash. Executing runs a child; `source` runs in your shell.
- **Quote every expansion.** Unquoted means word splitting and globbing.
  Parameter expansion (`${f##*/}`, `${f##*.}`, `${1:-}`, `${1:?}`) handles most
  string work.
- Conditions test **exit status — 0 is true**. Use `[[ ]]` for strings and
  files, `(( ))` or `-lt` for numbers. Integers only; watch leading zeros.
- `for` over globs, never `$(ls)`. Read lines with
  `while IFS= read -r line; do …; done < file`. `case` for dispatch.
- `"$@"` passes arguments intact. `local` inside functions. Data out via stdout,
  status via `return`. Meaningful exit codes: 64 usage, 66 no input, 128+N
  signals (137 = SIGKILL).
- **`set -euo pipefail`** — and its holes: condition contexts, `local x=$(…)`,
  `$( )`, `(( i++ ))`, `grep` no-match, SIGPIPE 141.
- **`mktemp` + `trap '…' EXIT`** cleans up on success, failure, `exit`,
  SIGTERM and SIGINT — never on SIGKILL.
- **`shellcheck`**, `bash -n`, `bash -x`.

---

## Say it out loud

About 30 seconds each, out loud, using the real words:

1. A teammate's script fails in WSL with `'bash\r': No such file or directory`.
   Explain what the kernel did and give two fixes.
2. Explain word splitting and globbing using `rm $file` as the example, and the
   places where quoting isn't needed.
3. What's the difference between `[ ]` and `[[ ]]`? Mention one way each can
   surprise you.
4. Explain what each part of `set -euo pipefail` does, then name three
   situations where `-e` doesn't stop the script.
5. Why can `cmd | grep -q x` fail with 141 under `pipefail` even when it found
   the match?
6. What does `trap 'rm -f "$tmp"' EXIT` guarantee, what can't it handle, and why
   are the quotes single?
7. A Kubernetes pod keeps restarting with exit code 137. What does that number
   tell you?

---

## Q&A
