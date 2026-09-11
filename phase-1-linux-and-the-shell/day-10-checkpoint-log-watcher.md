# Day 10 — Checkpoint: `logwatch`, a Log-Watching Alerter

_Why this matters:_ This is what Days 1–9 were for. Before Prometheus, Grafana
and Datadog, this script *was* monitoring, and on plenty of the servers you'll
SSH into, something like it still is. It pulls in almost everything you've
learned: `tail -F` (Day 2), processes and signals (Day 4), systemd (Day 5), regex
and `grep` (Day 6), `awk` (Day 7), `curl` and `jq` (Day 8), and strict mode,
`trap` and exit codes (Day 9). Building it end to end is how you find out which
of those you actually understand.

> **Capstone note — this is a deliberate side-quest.** The capstone codebase
> starts with Spring Boot in Phase 6. But three ideas here come back for real:
> the **sliding-window counter** is the same algorithm as the Redis rate limiter
> in Phase 14; **threshold + cooldown** is how Prometheus alert rules and
> Alertmanager work in Phase 19; and a **webhook POST** is the shape of every
> alert integration (Slack, PagerDuty, Teams).

> **How to use this file.** Do Step 0, then build `logwatch.sh` yourself,
> milestone by milestone, and run the test harness after each one. Fill in the
> **Build log** as you go. The reference implementation is at the bottom — open
> it only when you're finished, or after 30 minutes stuck on one milestone. Plan
> on **two sessions**.

---

## The one-paragraph version

You're building a small program that runs forever, watching a log file. Each new
line flows through a pipeline: `tail -F` hands over lines as they're written,
`grep` keeps the ones matching a pattern, and `awk` pulls out the level, logger
and message. A bash loop reads the results and **remembers when each match
happened**. It forgets matches older than the window (say, 60 seconds), and if
the number left reaches the threshold (say, 5), it builds a JSON alert with `jq`
and POSTs it to a webhook with `curl` — then **stays quiet for a cooldown**, so
one incident sends one alert, not a hundred. Settings come from flags or
environment variables; bad settings exit with a specific code. When it's
stopped — Ctrl-C, or `SIGTERM` from systemd or Kubernetes — a `trap` kills the
background `tail` and deletes its temporary files. **Stream, filter, extract,
count in a window, alert once, clean up.**

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **webhook** | A URL you POST to so that another system reacts — Slack posts a message, PagerDuty pages someone | An HTTP endpoint that accepts event payloads, usually JSON |
| **threshold** | The number of matches that means "this is a problem now" | The condition that fires the alert |
| **sliding window** | "The last 60 seconds" — recalculated at every new event | A time window anchored to *now*, moving continuously |
| **fixed window** | "This clock minute" — 10:04:00 to 10:04:59, then reset | Counters reset at boundaries; a burst across a boundary is split in two |
| **cooldown** | A quiet period after an alert | Suppression interval — Alertmanager's `repeat_interval` |
| **alert fatigue** | So many alerts that people stop reading them | The failure mode cooldowns exist to prevent |
| **named pipe (FIFO)** | A pipe with a filename, so separately started commands can connect through it | A special file (type `p` in `ls -l`, Day 2) made by `mkfifo`; data passes through the kernel, never touching disk |
| **orphan** | A child process still running after its parent died (Day 4) | A process re-parented to PID 1 (or the nearest subreaper) |
| **processing time** | When your program *saw* the line | Wall-clock time at the consumer |
| **event time** | When the thing *happened* — the timestamp inside the line | Time recorded by the producer |
| **`getopts`** | Bash's built-in parser for `-x value` style options | POSIX builtin that walks short options in `"$@"` |
| **precedence** | Which setting wins when it's given in more than one place | Here: flag, then environment variable, then default |
| **log rotation** | Renaming the current log and starting a fresh one | `app.log` → `app.log.1`, new `app.log`; `tail -F` follows the *name* |
| **test double** | A fake stand-in for a real dependency, used in tests | Here, a tiny local HTTP server that records every POST it receives |
| **acceptance test** | A check that the program does what the spec says, from the outside | Black-box tests run against the real executable |

---

## Before you start: three questions

1. `tail -f app.log | grep ERROR | awk '{print $5}'` prints nothing for minutes
   while errors are clearly being logged. Name the two culprits. (Days 6 and 7.)
2. Five errors arrive at 10:00:57, :58, :59, 10:01:00 and :01. Your threshold is
   5 per minute. A watcher that counts per *clock minute* sees 3, then 2. Does
   it alert? Should it?
3. systemd stops your watcher with `SIGTERM`. List everything that must happen
   before the process is really gone.

---

## The spec

```
logwatch.sh [-p PATTERN] [-w SECONDS] [-t COUNT] [-c SECONDS] [-u URL] LOGFILE
```

| Setting | Flag | Environment variable | Default |
|---|---|---|---|
| Pattern to match (extended regex) | `-p` | `LOGWATCH_PATTERN` | `ERROR` |
| Window length, seconds | `-w` | `LOGWATCH_WINDOW` | `60` |
| Matches in the window that trigger an alert | `-t` | `LOGWATCH_THRESHOLD` | `5` |
| Minimum seconds between alerts | `-c` | `LOGWATCH_COOLDOWN` | `300` |
| Webhook URL to POST alerts to | `-u` | `LOGWATCH_URL` | empty: print the alert to stdout instead |
| Show help | `-h` | | |

A flag beats the environment variable; the environment variable beats the
default.

**Behaviour.**

1. Only lines written **after** it starts count — no alerting on history.
2. Each new line that matches the pattern is a *match*. Lines look like Day 6's
   `app.log`: `DATE TIME LEVEL [thread] logger - message`. Use `awk` to extract
   the level (`$3`), the logger (`$5`) and the message (everything after the
   first ` - `).
3. When the number of matches in the last `WINDOW` seconds reaches `THRESHOLD`,
   send an alert — unless one was sent less than `COOLDOWN` seconds ago.
4. An alert is one line of JSON, POSTed with `Content-Type: application/json`:

```
{"text":"3 lines matching /ERROR/ in the last 60s on myhost","count":3,"window_seconds":60,
 "threshold":3,"pattern":"ERROR","file":"/home/you/app.log","host":"myhost",
 "last":{"level":"ERROR","logger":"c.a.PaymentClient","message":"connect timeout …"}}
```

   (`text` is there so the same payload works with a Slack incoming webhook.)
5. If the POST fails, log a warning to stderr and **keep watching**.
6. Status messages go to **stderr**. With no URL, alerts go to **stdout**.
7. It survives **log rotation**: the file renamed and replaced.
8. On `SIGINT` or `SIGTERM` it stops the background `tail`, removes its temp
   files, and exits.

**Exit codes.**

| Code | Meaning |
|---|---|
| `0` | Help was shown (`-h`) |
| `64` | Wrong usage: unknown option, missing value, not exactly one log file |
| `66` | The log file doesn't exist or isn't readable |
| `69` | A required command (`tail`, `grep`, `awk`, `curl`, `jq`, `mkfifo`) is missing |
| `74` | The input stream ended unexpectedly — `tail` died |
| `78` | Bad configuration: a non-numeric or zero window/threshold/cooldown, or an invalid regex |
| `130` | Stopped by `SIGINT` (Ctrl-C) |
| `143` | Stopped by `SIGTERM` |

---

## Step 0 — Set up the project

**In plain words:** a real project lives somewhere permanent, not in `/tmp`, and
on the Linux filesystem, not `/mnt/d` (Day 1: speed; Day 9: `chmod` and line
endings). This block creates `~/logwatch` with three helpers you **don't** have
to write: a log generator, a fake webhook to receive alerts, and the test
harness that checks your script against the spec.

```bash
mkdir -p ~/logwatch && cd ~/logwatch

cat > genlog.sh <<'EOF'
#!/usr/bin/env bash
# genlog.sh — append fake application log lines, for testing logwatch.
# Usage: genlog.sh FILE [COUNT] [LEVEL] [DELAY_SECONDS]
# Override the message text with GENLOG_MESSAGE.
set -euo pipefail
file=${1:?usage: genlog.sh FILE [COUNT] [LEVEL] [DELAY_SECONDS]}
count=${2:-1} level=${3:-ERROR} delay=${4:-0}
message=${GENLOG_MESSAGE:-connect timeout to 192.168.10.7:8443}
loggers=(c.a.PaymentClient c.a.OrderController c.a.ReportJob)
for (( i = 1; i <= count; i++ )); do
  printf '%s %-5s [http-nio-8080-exec-%d] %s - %s\n' \
    "$(date '+%F %T')" "$level" "$i" "${loggers[RANDOM % ${#loggers[@]}]}" "$message" >> "$file"
  if (( i < count )); then sleep "$delay"; fi
done
EOF

cat > receiver.py <<'EOF'
#!/usr/bin/env python3
"""Test double for a webhook: accepts POSTs, appends each body as a line to a file."""
import http.server
import sys

port, out = int(sys.argv[1]), sys.argv[2]


class Handler(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        body = self.rfile.read(int(self.headers.get("Content-Length", 0)))
        with open(out, "ab") as f:
            f.write(body + b"\n")
        self.send_response(204)
        self.end_headers()

    def log_message(self, *args):
        pass


http.server.HTTPServer(("127.0.0.1", port), Handler).serve_forever()
EOF

cat > test-logwatch.sh <<'EOF'
#!/usr/bin/env bash
# test-logwatch.sh — acceptance tests for logwatch.sh (Phase 1 checkpoint).
# Usage: ./test-logwatch.sh [path/to/logwatch.sh]        default: ./logwatch.sh
# Exit status: 0 all checks passed, 1 some failed, 2 couldn't run.
set -uo pipefail          # deliberately no -e: a failed check is reported, not fatal

here=$(cd "$(dirname "$0")" && pwd)
lw=$(realpath "${1:-./logwatch.sh}" 2> /dev/null) || lw=${1:-./logwatch.sh}
port=${PORT:-9977}
work=$(mktemp -d)
log="$work/app.log"
touch "$work/started"
export GENLOG_MESSAGE='upstream said "retry later" (cache C:\temp\new)'
pass=0 fail=0 lw_pid="" recv_pid="" stopped_rc=""

cleanup() {
  [[ -n $lw_pid ]] && kill "$lw_pid" 2> /dev/null
  [[ -n $recv_pid ]] && kill "$recv_pid" 2> /dev/null
  pkill -f -- "$work/" 2> /dev/null        # anything still using our work directory
  wait 2> /dev/null
  rm -rf "$work"
}
trap cleanup EXIT

check() {                 # check DESCRIPTION EXPECTED ACTUAL
  if [[ $3 == "$2" ]]; then
    printf '  PASS  %s\n' "$1"; pass=$(( pass + 1 ))
  else
    printf '  FAIL  %s\n          expected: [%s]\n          got:      [%s]\n' "$1" "$2" "$3"
    fail=$(( fail + 1 ))
  fi
}
exit_code_of() {          # run something that should exit immediately; print its status
  timeout 5 "$@" > /dev/null 2>&1 < /dev/null
  echo $?
}
eventually() {            # eventually SECONDS COMMAND... — retry until it succeeds
  local deadline=$(( SECONDS + $1 )); shift
  until "$@"; do
    (( SECONDS < deadline )) || return 1
    sleep 0.1
  done
}
alert_count() { if [[ -f $work/alerts.jsonl ]]; then wc -l < "$work/alerts.jsonl" | tr -d ' '; else echo 0; fi; }
alerts_are()  { [[ $(alert_count) == "$1" ]]; }
start_watcher() {         # start_watcher ARGS... — run the watcher in the background on $log
  "$lw" "$@" "$log" > "$work/stdout.txt" 2> "$work/stderr.txt" &
  lw_pid=$!
  sleep 0.7               # give it time to start tail before anything is written
}
stop_watcher() {          # SIGTERM the watcher and record its exit status
  kill -TERM "$lw_pid" 2> /dev/null
  wait "$lw_pid" 2> /dev/null; stopped_rc=$?
  lw_pid=""
}
gen() { "$here/genlog.sh" "$log" "$@"; }

[[ -x $lw ]] || { echo "cannot run: $lw is missing or not executable (chmod +x it)"; exit 2; }
for cmd in python3 jq curl timeout pkill; do
  command -v "$cmd" > /dev/null || { echo "cannot run: '$cmd' is not installed"; exit 2; }
done
echo "Testing $lw"

echo "1. Configuration and exit codes"
: > "$log"
check "no arguments                    -> 64" 64 "$(exit_code_of "$lw")"
check "two log files                   -> 64" 64 "$(exit_code_of "$lw" "$log" "$log")"
check "unknown option -z               -> 64" 64 "$(exit_code_of "$lw" -z "$log")"
check "-t with no value                -> 64" 64 "$(exit_code_of "$lw" -t)"
check "log file missing                -> 66" 66 "$(exit_code_of "$lw" "$work/missing.log")"
check "threshold 'abc'                 -> 78" 78 "$(exit_code_of "$lw" -t abc "$log")"
check "window 0                        -> 78" 78 "$(exit_code_of "$lw" -w 0 "$log")"
check "invalid regex 'ERROR('          -> 78" 78 "$(exit_code_of "$lw" -p 'ERROR(' "$log")"
check "LOGWATCH_COOLDOWN=soon          -> 78" 78 "$(exit_code_of env LOGWATCH_COOLDOWN=soon "$lw" "$log")"
check "-h                              -> 0"  0  "$(exit_code_of "$lw" -h)"

echo "2. Threshold, JSON payload and cooldown"
python3 "$here/receiver.py" "$port" "$work/alerts.jsonl" > /dev/null 2>&1 &
recv_pid=$!
eventually 3 curl -s -o /dev/null "http://127.0.0.1:$port/" || { echo "cannot run: receiver did not start on port $port"; exit 2; }
start_watcher -p 'ERROR|FATAL' -w 30 -t 3 -c 60 -u "http://127.0.0.1:$port/alert"
gen 2 ERROR
gen 4 INFO
sleep 0.5
check "2 matches, threshold 3          -> no alert" 0 "$(alert_count)"
gen 1 FATAL
eventually 2 alerts_are 1
check "3rd match (FATAL, via ERROR|FATAL) -> 1 alert" 1 "$(alert_count)"
check "payload is JSON with count 3" 3 "$(jq -r '.count' "$work/alerts.jsonl" 2> /dev/null | head -1)"
check "payload has the right level" FATAL "$(jq -r '.last.level' "$work/alerts.jsonl" 2> /dev/null | head -1)"
check "quotes and backslashes survive" "$GENLOG_MESSAGE" "$(jq -r '.last.message' "$work/alerts.jsonl" 2> /dev/null | head -1)"
gen 3 ERROR
sleep 0.7
check "more matches during cooldown    -> still 1 alert" 1 "$(alert_count)"

echo "3. Stopping cleanly"
stop_watcher
check "SIGTERM                         -> exit 143" 143 "$stopped_rc"
sleep 0.3
check "no process left holding the log" "" "$(pgrep -f -- "$log" | tr '\n' ' ' | sed 's/ $//')"
check "no named pipe left under /tmp" "" "$(find /tmp -maxdepth 2 -type p -newer "$work/started" 2> /dev/null | head -1)"

echo "4. Sliding window and log rotation"
: > "$work/alerts.jsonl"
start_watcher -w 2 -t 3 -c 1 -u "http://127.0.0.1:$port/alert"
gen 2 ERROR
sleep 2.5                                  # those two slide out of the 2-second window
gen 2 ERROR
sleep 0.5
check "expired matches don't count     -> no alert" 0 "$(alert_count)"
gen 1 ERROR
eventually 2 alerts_are 1
check "3 matches inside the window     -> alert" 1 "$(alert_count)"
mv "$log" "$log.1"; : > "$log"             # rotate: rename it, start a fresh file
sleep 1.5                                  # tail -F notices; the 1s cooldown passes
gen 3 ERROR
eventually 3 alerts_are 2
check "still alerting after rotation" 2 "$(alert_count)"
stop_watcher

echo "5. Dry run, environment config, dead endpoint"
export LOGWATCH_THRESHOLD=1
start_watcher -w 60                        # no -u: print alerts; threshold from the environment
unset LOGWATCH_THRESHOLD
gen 1 ERROR
eventually 2 grep -q '"count"' "$work/stdout.txt"
check "no URL -> alert JSON on stdout" 1 "$(jq -r '.count' "$work/stdout.txt" 2> /dev/null | head -1)"
stop_watcher
start_watcher -t 1 -u "http://127.0.0.1:9/alert"   # nothing listens on port 9
gen 1 ERROR
sleep 1
check "failed POST -> watcher keeps running" yes "$(kill -0 "$lw_pid" 2> /dev/null && echo yes || echo no)"
stop_watcher

echo
echo "$pass passed, $fail failed"
if (( fail > 0 )) && [[ -s $work/stderr.txt ]]; then
  echo "--- last lines of the watcher's stderr:"; tail -5 "$work/stderr.txt"
fi
(( fail == 0 ))
EOF

chmod +x genlog.sh receiver.py test-logwatch.sh
ls -l
echo "--- the harness, before logwatch.sh exists:"
./test-logwatch.sh; echo "exit=$?"
```

Now create `logwatch.sh` in the same folder. To edit it in VS Code **with the
Linux filesystem and LF line endings**, run `code ~/logwatch` from your Ubuntu
terminal — that opens VS Code in WSL mode.

---

## Milestone 1 — Skeleton, configuration and exit codes

**In plain words:** before watching anything, the script has to work out *what*
to watch and *how*, and refuse to start if something's wrong — with a clear
message and a distinct exit code for each kind of wrong. Settings can come from
three places, so you need a rule for which one wins. That's this milestone: no
logs, no alerts, just a front door that only lets valid configurations through.

### Configuration precedence

Load defaults from the environment first, then let flags override them:

```
WINDOW=${LOGWATCH_WINDOW:-60}      # env var if set, else the default
…                                  # then getopts overwrites it if -w was given
```

This is the same rule Spring Boot uses (Phase 6): command-line arguments beat
environment variables, which beat the defaults in `application.properties`. It
lets one script run with different settings in different environments without
editing it — the idea behind "configuration in the environment" from the
Twelve-Factor App.

### New tool: `getopts`

Day 9 parsed arguments with `case`. For `-x value` style options, bash has a
builtin that does the fiddly parts:

```
while getopts ':p:w:t:c:u:h' opt; do
  case $opt in
    p) PATTERN=$OPTARG ;;
    w) WINDOW=$OPTARG ;;
    h) usage; exit 0 ;;
    :) die 64 "option -$OPTARG needs a value" ;;
    *) die 64 "unknown option -$OPTARG" ;;
  esac
done
shift $(( OPTIND - 1 ))        # drop the options; what's left is the log file
```

- The option string `':p:w:t:c:u:h'` lists the letters. A `:` **after** a letter
  means "takes a value", which arrives in `$OPTARG`.
- The `:` at the **start** switches on silent mode: instead of printing its own
  error, `getopts` sets `opt` to `:` (missing value) or `?` (unknown option) and
  puts the offending letter in `$OPTARG`, so you print a proper message.
- `$OPTIND` is the index of the next argument to process; `shift` by
  `OPTIND - 1` leaves only the non-option arguments.
- `getopts` handles `-t 5`, `-t5` and combined flags. It doesn't do long
  options (`--threshold`).

### Validation

- **Integers:** `[[ $WINDOW =~ ^[1-9][0-9]*$ ]]` — positive, no leading zeros,
  so none of Day 9's octal surprises.
- **The regex:** ask `grep` itself. `grep` exits **2** for an invalid pattern
  and **1** for "valid, no match". Under `set -e` you capture a status without
  dying like this:

  ```
  rc=0
  grep -Eq -- "$PATTERN" /dev/null 2> /dev/null || rc=$?
  (( rc != 2 )) || die 78 "invalid pattern: $PATTERN"
  ```

  The `|| rc=$?` idiom is worth memorising — a failing command on the left of
  `||` doesn't trigger `set -e` (Day 9), and `$?` still holds its status.
- **Dependencies:** `command -v jq > /dev/null || die 69 …` for each tool.
- **The file:** `[[ -f $LOG_FILE && -r $LOG_FILE ]] || die 66 …`.

**Check yourself:** run `./test-logwatch.sh`. Section 1 should be all PASS.
Later sections will fail — that's expected at this point.

> **Say this in an interview:** "I load configuration with a fixed precedence —
> command-line flags over environment variables over defaults — so one artifact
> runs unchanged across environments. I validate everything at startup and fail
> fast with distinct exit codes: 64 for usage, 66 for missing input, 69 for a
> missing dependency, 78 for bad configuration. That way a supervisor like
> systemd can tell a crash worth retrying from a config error that never will
> succeed."

---

## Milestone 2 — The stream: `tail -F` → `grep` → `awk` → your loop

**In plain words:** you need a never-ending stream of new matching lines, already
split into the fields you care about. Three tools you know, chained — but a
pipeline that never ends brings two new problems. Output sits in buffers instead
of flowing. And when your script dies, the `tail` at the front of the pipe
doesn't notice, and keeps running forever.

### The tools, and the flags that matter

```
tail -n0 -F -- "$LOG_FILE"                  # -n0: start at the end; -F: follow the NAME (rotation)
grep --line-buffered -E -- "$PATTERN"       # flush every line (Day 6)
awk -v OFS='\t' '{ …; fflush() }'           # flush every line (Day 7)
```

- **`-n0`** — skip the existing content. Without it, `tail` replays the last 10
  lines and you alert on history.
- **`-F`, not `-f`** (Day 2) — `-f` follows the open file; after rotation it
  keeps watching `app.log.1`, which nobody writes to any more. `-F` notices the
  name now points at a new file and switches.
- **`--`** — "no more options": a pattern or filename starting with `-` is taken
  literally.

### The awk extraction

For `2026-09-11 08:14:05 ERROR [http-nio-8080-exec-2] c.a.PaymentClient - connect timeout`,
`$3` is the level and `$5` the logger. The message is everything after the first
` - `, which may itself contain spaces or dashes, so find its position instead
of splitting:

```
awk -v OFS='\t' '{ i = index($0, " - "); print $3, $5, (i ? substr($0, i + 3) : $0); fflush() }'
```

`index()` returns where ` - ` starts (0 if absent) and `substr()` takes
everything after it. Tabs separate the output fields, so your loop reads them
with `IFS=$'\t' read -r level logger message`.

### Failure 1 — buffering

Run this. The line is written half a second in; watch *when* it arrives.

```bash
mkdir -p /tmp/day10 && cd /tmp/day10 && : > buf.log
stamp() { while IFS= read -r l; do echo "  $(date +%T) received: $l"; done; }
echo "=== without --line-buffered / fflush (started $(date +%T)):"
( sleep 0.5; echo "2026-09-11 08:00:00 ERROR [t] c.a.PaymentClient - boom" >> buf.log ) &
timeout 3 tail -n0 -F buf.log | grep ERROR | awk '{ print $3, $5 }' | stamp
echo "=== with them (started $(date +%T)):"
( sleep 0.5; echo "2026-09-11 08:00:01 ERROR [t] c.a.ReportJob - boom" >> buf.log ) &
timeout 3 tail -n0 -F buf.log | grep --line-buffered ERROR | awk '{ print $3, $5; fflush() }' | stamp
```

Without the flags, the line arrives only when `timeout` kills `tail` three
seconds later and everything flushes on exit. In a watcher that runs forever,
"on exit" means never — which is the answer to question 1.

### Failure 2 — the orphaned pipeline

The obvious structure is `while read …; done < <(tail -n0 -F … | grep … | awk …)`.
It works — until the script is killed. Here's a cut-down watcher (the grep
pattern is just a word unique to this demo, so `pgrep` can find it):

```bash
mkdir -p /tmp/day10 && cd /tmp/day10 && : > orphan.log
bash -c 'while read -r l; do :; done < <(tail -n0 -F orphan.log | grep --line-buffered ORPHAN_DEMO)' &
watcher=$!
sleep 0.5
kill "$watcher"                          # the "watcher" dies...
sleep 1
if kill -0 "$watcher" 2> /dev/null; then echo "--- watcher $watcher: still alive"; else echo "--- watcher $watcher: gone"; fi
echo "--- but these are still running:"
pgrep -af 'orphan.log|ORPHAN_DEMO'
pkill -f 'orphan.log|ORPHAN_DEMO' && echo "--- killed by hand"
```

The watcher is gone, but three processes are left: `tail`, `grep`, and a
`bash -c …` line with a *different* PID — the **subshell** bash forked to run the
pipeline inside `<( )`, which is why it shows the same command line. They're
**orphans** (Day 4): re-parented, and blocked. `grep` waits for `tail`; `tail`
waits for the file to grow; nothing writes, so nothing gets `SIGPIPE`. They'd
survive until the next matching line — on a quiet log, days. Restart the watcher
a few times and you have a pile of them.

(GNU `tail` on its own would have noticed: on Linux it watches for its reader
disappearing and exits. Here its reader is `grep`, which is still alive.)

Your trap must kill the head of the pipeline — so you need **`tail`'s PID**, and
a process substitution doesn't give you a usable one.

### The fix — a named pipe

A **named pipe** (FIFO) is a pipe with a filename. One process writes to the
name, another reads from it, and data goes straight through the kernel:

```bash
cd /tmp/day10 && rm -f demo.fifo && mkfifo demo.fifo
ls -l demo.fifo
( echo "hello through a named pipe" > demo.fifo ) &
cat demo.fifo
rm demo.fifo
```

The `p` at the start of the `ls -l` line is the file type (Day 2). Opening a FIFO
**blocks** until the other end opens too — that's why the writer runs in the
background.

So the structure becomes:

```
workdir=$(mktemp -d)
mkfifo "$workdir/lines"
tail -n0 -F -- "$LOG_FILE" > "$workdir/lines" &      # tail is a direct child…
tail_pid=$!                                          # …so you have its PID

while IFS=$'\t' read -r level logger message; do
  …
done < <(grep --line-buffered -E -- "$PATTERN" < "$workdir/lines" | awk …)
```

Cleanup kills `tail_pid`; `tail` closing the FIFO gives `grep` end-of-file, so
`grep` and `awk` exit on their own. Then `rm -rf "$workdir"` removes the FIFO.

**Check yourself:** make the loop print what it reads. In one Ubuntu terminal run
`./logwatch.sh app.log` (create `app.log` first); in another,
`./genlog.sh app.log 3 ERROR`. Each line should appear within a fraction of a
second, split into level, logger and message.

> **Say this in an interview:** "Tailing a log into a pipeline needs two things
> people miss: every stage has to flush per line — `grep --line-buffered`,
> `fflush()` in awk — because stdio block-buffers into a pipe, and the pipeline
> must be cleaned up explicitly, because its stages only discover their reader
> has gone on their next write, which on a quiet log may never come. I run `tail`
> as a direct child writing into a FIFO so I have its PID for the cleanup trap."

---

## Milestone 3 — Counting in a sliding window

**In plain words:** keep a list of the times you've seen matches. Each time a new
one arrives, add *now* to the end, then throw away entries from the front that
are older than the window. Whatever's left is "matches in the last N seconds".
That's the whole algorithm.

```
hits+=("$now")
while (( hits[0] <= now - WINDOW )); do      # the newest hit is always inside the window,
  hits=("${hits[@]:1}")                      # so this loop can never empty the array
done
count=${#hits[@]}
```

`${hits[@]:1}` means "every element from index 1 on" — dropping the first.
`$EPOCHSECONDS` gives the current time in seconds, without starting a `date`
process for every line.

### Why not count per minute?

Run this — the same five events, counted both ways, threshold 5:

```bash
mkdir -p /tmp/day10
printf '%s\n' 57 58 59 60 61 > /tmp/day10/events.txt      # seconds after 10:00:00
echo "--- fixed windows (count per clock minute):"
awk '{ n[int($1 / 60)]++ } END { for (m in n) printf "  10:%02d  %d events\n", m, n[m] }' /tmp/day10/events.txt | sort
echo "--- sliding 60-second window (count at each event):"
awk '{ t[NR] = $1; c = 0; for (i = 1; i <= NR; i++) if (t[i] > $1 - 60) c++
       printf "  t=%ds  %d in the last 60s%s\n", $1, c, (c >= 5 ? "   <- ALERT" : "") }' /tmp/day10/events.txt
```

The fixed window splits the burst across the minute boundary — 3 and 2 — and
never alerts. The sliding window sees all five. That's question 2.

### The design space — the part interviewers push on

| Algorithm | Stores | Accuracy | Where you'll meet it |
|---|---|---|---|
| **Fixed window** | One counter per period | Misses bursts across boundaries | Simple rate limits |
| **Sliding window log** (yours) | Every event's timestamp in the window | Exact | Redis sorted set + `ZREMRANGEBYSCORE` (Phase 14) |
| **Sliding window counter** | Counters for the current and previous period, weighted | Close, constant memory | High-volume rate limiters |
| **Token bucket** | A token count and a last-refill time | Allows controlled bursts | API gateways (Phase 12) |

Your version stores one timestamp per match inside the window. For errors in a
60-second window that's tiny. For "requests per client per hour" at 10,000
requests a second it isn't — which is why the constant-memory variants exist.

### Processing time vs event time

You timestamp a match when **you read it**, not with the date inside the line.
That's simpler and immune to wrong clocks on the machine that wrote the log. The
cost: if the watcher falls behind — or you replay an old file — timing is
distorted. Choosing between "when it happened" and "when we saw it" is a central
question in stream processing, and it comes back with Kafka in Phase 13.

**Check yourself:** section 4 of the harness tests the window. Also watch your
own stderr: with `-w 5`, matches more than 5 seconds apart should never add up.

> **Say this in an interview:** "A fixed window is cheap but splits bursts across
> boundaries — five events in five seconds can show up as three and two. A
> sliding log keeps each event's timestamp and evicts anything older than the
> window, which is exact but uses memory proportional to the rate; a sliding
> window counter approximates it with two buckets in constant memory, and a
> token bucket allows controlled bursts. In Redis the sliding log is a sorted set
> scored by timestamp, trimmed with `ZREMRANGEBYSCORE`."

---

## Milestone 4 — The alert: JSON with `jq`, POST with `curl`, and a cooldown

**In plain words:** when the count crosses the threshold, turn the situation into
a small JSON document and send it to a URL. Two things go wrong in real life:
the message contains characters that break hand-built JSON, and one incident
keeps crossing the threshold, sending the same alert over and over until people
stop reading them.

### Failure — building JSON by hand

Log messages contain quotes. `printf` pastes them straight into the JSON:

```bash
msg='upstream said "retry later"'
printf '{"count": 3, "message": "%s"}\n' "$msg" > /tmp/bad.json
cat /tmp/bad.json
jq . /tmp/bad.json
echo "exit=$?"
echo "--- jq -n --arg escapes it properly:"
jq -cn --arg message "$msg" --argjson count 3 '{count: $count, message: $message}'
rm -f /tmp/bad.json
```

Any receiver rejects the first one. Build JSON with `jq -n --arg` (strings) and
`--argjson` (numbers) — Day 8 — and escaping is handled for you, including
backslashes, tabs and non-ASCII text.

### Sending it

```
curl -fsS -m 5 -X POST -H 'Content-Type: application/json' --data "$payload" "$URL" > /dev/null
```

The Day 8 flags: `-f` so HTTP errors count as failure, `-sS` quiet but still
showing errors, and `-m 5` so a hanging endpoint can't freeze the watcher.

**Put the `curl` inside an `if`.** Under `set -e`, a failed POST would kill the
watcher (Day 9) — so the day Slack has an outage, your error monitoring dies with
it. Inside `if … then … else log "WARNING …"; fi`, a failure is handled, not
fatal.

Try the test double by hand:

```bash
cd ~/logwatch
python3 receiver.py 9978 /tmp/received.jsonl > /dev/null 2>&1 &
rpid=$!
sleep 0.5
curl -fsS -m 5 -X POST -H 'Content-Type: application/json' \
  --data '{"text":"hello from curl"}' http://127.0.0.1:9978/alert && echo "POST accepted"
echo "--- what the receiver recorded:"; cat /tmp/received.jsonl
echo "--- and when nothing is listening:"
curl -fsS -m 5 -X POST --data '{}' http://127.0.0.1:9/alert; echo "exit=$?"
kill "$rpid"; rm -f /tmp/received.jsonl
```

Exit 7 means curl couldn't connect. That's what your `else` branch will see when
the webhook is down.

### Cooldown

Keep `last_alert` (0 at start). Alert only when **both** are true:

```
(( count >= THRESHOLD && now - last_alert >= COOLDOWN ))
```

then set `last_alert=$now`. Without it, a steady error rate above the threshold
alerts on **every single match**. Alertmanager has the same knobs under other
names — `group_wait`, `group_interval`, `repeat_interval` — for the same reason:
an alert that fires constantly is an alert everyone mutes.

### Dry-run mode

If there's no URL, print the payload on stdout instead of POSTing. You can
develop and test everything without a webhook, and redirect stdout to a file to
keep an alert history.

**Check yourself:** harness sections 2 and 5.

> **Say this in an interview:** "I build JSON with a real serializer — `jq --arg`
> in shell, Jackson in Java — never string formatting, because a quote or
> backslash in the data produces invalid JSON, or worse, lets data inject
> fields. Delivery is best-effort with a timeout and its failure is handled
> rather than fatal, so the monitor doesn't die with the notification channel.
> And every alert has a cooldown, because a condition that stays true would
> otherwise re-fire on every event — that's alert fatigue."

---

## Milestone 5 — Shutdown, signals and rotation

**In plain words:** a program that runs forever has to stop cleanly when asked:
stop its helpers, delete its temporary files, and report *why* it stopped with
its exit code. It also has to survive the log file being swapped out from under
it.

### The traps

```
cleanup() {
  if [[ -n $tail_pid ]]; then kill "$tail_pid" 2> /dev/null || true; fi
  if [[ -n $workdir ]]; then rm -rf "$workdir"; fi
}
trap cleanup EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
```

- **`EXIT`** runs `cleanup` however the script ends (Day 9).
- **The `INT` and `TERM` traps** turn each signal into an ordinary `exit` with
  the conventional code, so the script always leaves through the `EXIT` path
  with a meaningful status.
- **`|| true` after `kill`.** `set -e` still applies inside trap handlers. If
  `tail` has already died, `kill` fails — and without `|| true`, the handler
  would stop right there, before the `rm`.
- **`tail_pid` and `workdir` are globals** — Day 9's lesson: an `EXIT` trap
  can't see a function's `local` variables after the function has returned.

That's question 3's answer: receive `SIGTERM`, run the `TERM` trap, exit 143,
run the `EXIT` trap, kill `tail`, let `grep` and `awk` end on EOF, delete the
FIFO.

### When the stream ends by itself

If the `while` loop ever finishes, the input ended — `tail` died, or someone
killed it. That isn't normal operation, so exit **74** (I/O error). Under systemd
you'd want that restarted; a config error (78) you wouldn't.

### Rotation

With `tail -F` there's nothing to write: when `app.log` is renamed to
`app.log.1` and a new `app.log` appears, `tail` prints
`'app.log' has become inaccessible` / `has appeared; following new file` on
stderr and carries on. Harness section 4 rotates the log mid-test.

### Running it for real — as a systemd service (Day 5)

```
# /etc/systemd/system/logwatch.service
[Unit]
Description=Alert on errors in the order service log
After=network-online.target

[Service]
Environment=LOGWATCH_URL=https://hooks.example.com/alert LOGWATCH_THRESHOLD=10
ExecStart=/home/shailesh/logwatch/logwatch.sh /var/log/myapp/app.log
Restart=on-failure
RestartPreventExitStatus=64 66 69 78
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target
```

`RestartPreventExitStatus` is where meaningful exit codes pay off: systemd
restarts the watcher if `tail` dies (74) but not if the configuration is wrong
(78) — restarting a broken config every few seconds forever achieves nothing.

> **Say this in an interview:** "A long-running script needs a shutdown path: an
> EXIT trap that reaps background children and removes temp state, and INT/TERM
> traps that convert signals into explicit exits with 130 and 143. `set -e` is
> still active inside handlers, so every cleanup step has to tolerate failure.
> Distinct exit codes then let the supervisor decide — in systemd,
> `RestartPreventExitStatus` stops it restarting on configuration errors while
> still restarting on transient ones."

---

## Hands-on — prove it works

### 1. Run the harness against your script

```bash
cd ~/logwatch && ./test-logwatch.sh ./logwatch.sh; echo "harness exit=$?"
```

All checks passing takes about 12 seconds. If your script is still failing
checks, run the harness from your **Ubuntu terminal** instead — failing checks
wait for their timeouts, and the lab stops a run after 20 seconds.

### 2. Watch it live (Ubuntu terminal, three panes or tabs)

```
# pane 1 — the fake webhook, printing each alert as it lands
cd ~/logwatch && python3 receiver.py 9999 alerts.jsonl & tail -F alerts.jsonl | jq .

# pane 2 — the watcher: window 30s, threshold 3, cooldown 20s
cd ~/logwatch && : > app.log && ./logwatch.sh -w 30 -t 3 -c 20 -u http://127.0.0.1:9999/alert app.log

# pane 3 — the incident
cd ~/logwatch && ./genlog.sh app.log 5 ERROR 1
```

Then: send more errors during the cooldown and watch nothing arrive; rotate the
log (`mv app.log app.log.1; : > app.log`) and send more; press Ctrl-C in pane 2
and check `echo $?` says 130; run `pgrep -af tail` to confirm no orphan.

### 3. `shellcheck` it

```bash
cd ~/logwatch
if [[ ! -f logwatch.sh ]]; then echo "write logwatch.sh first"
elif command -v shellcheck > /dev/null; then shellcheck logwatch.sh && echo "clean"
else echo "install it in your Ubuntu terminal: sudo apt-get install -y shellcheck"; fi
```

If `shellcheck` flags something you did on purpose — for example SC2016,
"expressions don't expand in single quotes", on an awk program whose `$1` is
*meant* for awk — silence **that one line** with a
`# shellcheck disable=SC2016` comment directly above it, plus a comment saying
why. Never switch a check off for the whole file.

### 4. Prove the tests test something

A test suite you've never seen fail proves nothing. Take a working
`logwatch.sh` (yours, or the reference) and break it on purpose — then check the
harness notices. In your Ubuntu terminal (a broken copy can take up to about 20
seconds, because failing checks wait for their timeouts):

```
cd ~/logwatch
sed 's/grep --line-buffered/grep/' logwatch.sh > broken-buffer.sh && chmod +x broken-buffer.sh
./test-logwatch.sh ./broken-buffer.sh        # alerts never arrive

sed 's/read -r level/read level/' logwatch.sh > broken-read.sh && chmod +x broken-read.sh
./test-logwatch.sh ./broken-read.sh          # the backslashes in the message get eaten (Day 9)

rm -f broken-*.sh
```

(Adjust the `sed` patterns if your script words those lines differently.)

---

## Stretch goals

Pick any, once the harness is green:

1. **Event time.** Parse the log's own timestamp in awk and use it for the window
   instead of `EPOCHSECONDS`. What breaks when you replay an old file? What if
   two servers' clocks differ?
2. **Per-logger thresholds.** Alert when *one* logger crosses the threshold,
   using a bash associative array keyed by logger (Day 9).
3. **A "resolved" message.** When the count drops back below the threshold,
   send one "recovered" alert. What does that need that the event-driven loop
   doesn't have? (Hint: nothing arrives when things are quiet — look at
   `read -t`.)
4. **Real Slack.** Create a Slack incoming webhook in a test workspace and point
   `-u` at it.
5. **The systemd unit.** Install it, `systemctl start` it, kill `tail` by hand
   and watch systemd restart the watcher; break the config and watch it *not*
   restart.

---

## Gotchas

1. **`grep` and `awk` buffer into pipes** — `--line-buffered` and `fflush()`, or
   the watcher sees nothing.
2. **`tail -f` loses the file at rotation**; `tail -F` follows the name.
3. **Without `-n0`, `tail` replays history** and you alert on old errors.
4. **A pipeline inside `<( )` outlives your script** — run `tail` as a direct
   child and kill it in the trap.
5. **`set -e` applies inside traps** — `kill … || true`.
6. **Traps can't see `local` variables** after their function returns.
7. **`printf` into JSON breaks on quotes and backslashes** — `jq --arg`.
8. **An unguarded `curl` under `set -e`** kills the watcher when the webhook is
   down.
9. **`read` without `-r`** eats backslashes in log messages.
10. **Capturing a status under `set -e`:** `cmd || rc=$?`.
11. **No cooldown means alert fatigue** — one alert per matching line.
12. **A fixed window misses bursts** that straddle a boundary.
13. **Processing time isn't event time** — fine for live tailing, wrong for
    replays.

---

## Recap

- **Configuration:** flag > environment variable > default, parsed with
  `getopts`, validated up front, each failure mapped to its own exit code.
- **The stream:** `tail -n0 -F` into a FIFO, `grep --line-buffered`, awk with
  `fflush()`, read with `IFS=$'\t' read -r`.
- **The window:** a sliding log of timestamps with eviction from the front — exact
  and simple; know the fixed-window, sliding-counter and token-bucket
  alternatives.
- **The alert:** JSON from `jq -n --arg`, `curl -fsS -m 5` inside an `if`, a
  cooldown against alert fatigue, and dry-run mode.
- **Shutdown:** `EXIT` trap kills `tail` and removes the FIFO; `INT`/`TERM` traps
  exit 130/143; exit codes tell a supervisor whether restarting makes sense.

---

## Say it out loud

About 30 seconds each, out loud, using the real words:

1. Walk through what happens to one log line, from being written to the file to
   an alert arriving at the webhook.
2. Why does a `tail | grep | awk` pipeline need `--line-buffered` and
   `fflush()`, and why does `tail` need to be killed explicitly?
3. Compare a fixed window and a sliding window, with the 10:00:57–10:01:01
   example. Then name one constant-memory alternative.
4. Why build the payload with `jq --arg` instead of `printf`?
5. Explain the full shutdown sequence when systemd sends `SIGTERM`, including
   the exit code.
6. Why do distinct exit codes matter to systemd's `Restart=` settings?

---

## Build log

Fill this in as you build. Future you — and anyone reading your GitHub — will
care more about *what broke and why* than about the final script.

| Milestone | Date | Time spent | What I built | What broke, and why | What I learned |
|---|---|---|---|---|---|
| 1 — config & exit codes | | | | | |
| 2 — the stream | | | | | |
| 3 — sliding window | | | | | |
| 4 — alert & cooldown | | | | | |
| 5 — shutdown & rotation | | | | | |

**Harness results over time:**

- First run: __ passed, __ failed
- Final run: __ passed, __ failed

**Questions I still have:**

-

---

## Reference implementation

**Spoiler.** Open this only when your harness is green, or when you've been stuck
on one milestone for 30 minutes. It installs to `~/logwatch/reference/`, so it
won't overwrite your own `logwatch.sh`, and then runs the harness against it.

```bash
mkdir -p ~/logwatch/reference && cd ~/logwatch
cat > reference/logwatch.sh <<'EOF'
#!/usr/bin/env bash
# logwatch.sh — watch a log file and POST an alert when too many lines match a
# pattern within a sliding time window.
#
# Usage: logwatch.sh [-p PATTERN] [-w SECONDS] [-t COUNT] [-c SECONDS] [-u URL] LOGFILE
# Each option can also be set in the environment (LOGWATCH_PATTERN, …); a flag
# beats the environment variable, which beats the default.
#
# Exit codes: 0 help · 64 usage · 66 unreadable log · 69 missing command
#             74 input ended · 78 bad configuration · 130 SIGINT · 143 SIGTERM
set -euo pipefail

readonly EX_USAGE=64 EX_NOINPUT=66 EX_UNAVAILABLE=69 EX_IOERR=74 EX_CONFIG=78
readonly PROG=${0##*/}

# Pull LEVEL, LOGGER and MESSAGE out of lines shaped like
#   2026-09-11 08:14:05 ERROR [thread] c.a.Logger - message text
# The message is everything after the first " - ". fflush() because awk
# block-buffers when its output is a pipe. The single quotes are deliberate:
# $0, $3 and $5 are awk fields, not shell variables.
# shellcheck disable=SC2016
readonly EXTRACT='{ i = index($0, " - "); print $3, $5, (i ? substr($0, i + 3) : $0); fflush() }'

PATTERN=${LOGWATCH_PATTERN:-ERROR}
WINDOW=${LOGWATCH_WINDOW:-60}
THRESHOLD=${LOGWATCH_THRESHOLD:-5}
COOLDOWN=${LOGWATCH_COOLDOWN:-300}
URL=${LOGWATCH_URL:-}
LOG_FILE=""

# Globals on purpose: the EXIT trap runs after watch() has returned.
workdir="" tail_pid=""

log() { printf '%s %s: %s\n' "$(date +%T)" "$PROG" "$*" >&2; }
die() { local code=$1; shift; printf '%s: %s\n' "$PROG" "$*" >&2; exit "$code"; }

usage() {
  cat <<USAGE
usage: $PROG [-p PATTERN] [-w SECONDS] [-t COUNT] [-c SECONDS] [-u URL] LOGFILE
  -p  pattern to match (extended regex)   env LOGWATCH_PATTERN    default ERROR
  -w  sliding window, in seconds          env LOGWATCH_WINDOW     default 60
  -t  matches in the window that alert    env LOGWATCH_THRESHOLD  default 5
  -c  minimum seconds between alerts      env LOGWATCH_COOLDOWN   default 300
  -u  webhook URL to POST alerts to       env LOGWATCH_URL        default: print alerts
USAGE
}

parse_args() {
  local opt
  while getopts ':p:w:t:c:u:h' opt; do
    case $opt in
      p) PATTERN=$OPTARG ;;
      w) WINDOW=$OPTARG ;;
      t) THRESHOLD=$OPTARG ;;
      c) COOLDOWN=$OPTARG ;;
      u) URL=$OPTARG ;;
      h) usage; exit 0 ;;
      :) usage >&2; die "$EX_USAGE" "option -$OPTARG needs a value" ;;
      *) usage >&2; die "$EX_USAGE" "unknown option -$OPTARG" ;;
    esac
  done
  shift $(( OPTIND - 1 ))
  if [[ $# -ne 1 ]]; then usage >&2; exit "$EX_USAGE"; fi
  LOG_FILE=$1
}

positive_int() {    # positive_int NAME VALUE
  [[ $2 =~ ^[1-9][0-9]{0,8}$ ]] || die "$EX_CONFIG" "$1 must be a positive whole number, got '$2'"
}

validate() {
  local cmd rc=0
  for cmd in tail grep awk curl jq mkfifo; do
    command -v "$cmd" > /dev/null || die "$EX_UNAVAILABLE" "required command not found: $cmd"
  done
  positive_int window "$WINDOW"
  positive_int threshold "$THRESHOLD"
  positive_int cooldown "$COOLDOWN"
  # grep exits 2 for an invalid regex and 1 for "valid, but no match".
  grep -Eq -- "$PATTERN" /dev/null 2> /dev/null || rc=$?
  (( rc != 2 )) || die "$EX_CONFIG" "invalid pattern: $PATTERN"
  [[ -f $LOG_FILE && -r $LOG_FILE ]] || die "$EX_NOINPUT" "cannot read log file: $LOG_FILE"
}

send_alert() {      # send_alert COUNT LEVEL LOGGER MESSAGE
  local payload
  payload=$(jq -cn \
    --argjson count "$1" --argjson window "$WINDOW" --argjson threshold "$THRESHOLD" \
    --arg level "$2" --arg logger "$3" --arg message "$4" \
    --arg pattern "$PATTERN" --arg file "$LOG_FILE" --arg host "$HOSTNAME" \
    '{text: "\($count) lines matching /\($pattern)/ in the last \($window)s on \($host)",
      count: $count, window_seconds: $window, threshold: $threshold,
      pattern: $pattern, file: $file, host: $host,
      last: {level: $level, logger: $logger, message: $message}}')

  if [[ -z $URL ]]; then
    printf '%s\n' "$payload"
  elif curl -fsS -m 5 -X POST -H 'Content-Type: application/json' --data "$payload" "$URL" > /dev/null; then
    log "alert sent: $1 matches in ${WINDOW}s"
  else
    log "WARNING: could not POST the alert to $URL; still watching"
  fi
}

cleanup() {
  if [[ -n $tail_pid ]]; then kill "$tail_pid" 2> /dev/null || true; fi
  if [[ -n $workdir ]]; then rm -rf "$workdir"; fi
  log "stopped"
}

watch() {
  local -a hits=()
  local level logger message now count last_alert=0

  workdir=$(mktemp -d)
  mkfifo "$workdir/lines"
  tail -n0 -F -- "$LOG_FILE" > "$workdir/lines" &
  tail_pid=$!
  log "watching $LOG_FILE for /$PATTERN/: alert at $THRESHOLD in ${WINDOW}s, cooldown ${COOLDOWN}s"

  while IFS=$'\t' read -r level logger message; do
    now=$EPOCHSECONDS
    hits+=("$now")
    # Evict from the front. The newest hit is always inside the window, so
    # this loop can never empty the array.
    while (( hits[0] <= now - WINDOW )); do
      hits=("${hits[@]:1}")
    done
    count=${#hits[@]}
    log "$count/$THRESHOLD in the last ${WINDOW}s: [$level] $logger"

    if (( count >= THRESHOLD && now - last_alert >= COOLDOWN )); then
      send_alert "$count" "$level" "$logger" "$message"
      last_alert=$now
    fi
  done < <(grep --line-buffered -E -- "$PATTERN" < "$workdir/lines" | awk -v OFS='\t' "$EXTRACT")

  die "$EX_IOERR" "input ended unexpectedly (did tail die?)"
}

main() {
  parse_args "$@"
  validate
  trap cleanup EXIT
  trap 'exit 130' INT
  trap 'exit 143' TERM
  watch
}

main "$@"
EOF
chmod +x reference/logwatch.sh
./test-logwatch.sh reference/logwatch.sh; echo "harness exit=$?"
```

Compare it with yours line by line. Differences in structure are fine; for each
difference, ask whether yours handles the same failure.

---

## Q&A
