# Phase 1 Quiz — Linux & the Shell (Days 1–9)

**How to take it.** Give yourself 60–90 minutes. For every question, answer
**out loud or on paper first**, then open the answer. For the "predict" questions,
write your prediction down, *then* press Run — a prediction you make after
seeing the output doesn't count.

**Scoring.** One point per question, 46 in total. Half a point if you had the
idea but not the words. **37 or more (80%)** — you're ready for Phase 2. Below
that, each answer tells you which day to reread; reread those parts, then retake
the questions you missed a day later.

The predict questions that use `jq` need it installed
(`sudo apt-get install -y jq` in your Ubuntu terminal).

---

## Part A — The machine (Days 1–3)

**A1.** Your Maven build takes 5× longer in `/mnt/d/projects` than in
`~/projects`. What's the mechanism — and is it a bandwidth problem or a latency
problem?

<details>
<summary>Answer</summary>

`/mnt/d` is a **9p** network-filesystem mount of the Windows drive. Every file
operation — every `stat`, `open`, directory read — is a round trip across the VM
boundary into Windows. A build does tens of thousands of tiny operations, so it's
a **latency** problem, not bandwidth: each operation is slow, however few bytes
it moves. Your Day 1 benchmark measured about 6× for 500 small files. Fix: keep
projects in `~` (ext4). → *Day 1*

</details>

**A2.** `ls -l /proc/meminfo` shows a size of 0, but `cat /proc/meminfo` prints
about 50 lines. How?

<details>
<summary>Answer</summary>

`/proc` isn't on any disk. It's kernel state presented as files, and the content
is **generated at the moment you read it** — so there's no stored size. `free`,
`ps` and `top` are just readers of `/proc`. → *Day 1*

</details>

**A3.** Where do these live: system-wide configuration, logs and other changing
data, installed programs, scratch files? Give one operational reason Linux splits
by *kind of data* rather than by application.

<details>
<summary>Answer</summary>

`/etc`, `/var`, `/usr`, `/tmp`. Splitting by kind lets you treat each kind
differently: `/usr` can be read-only, `/etc` is what you back up, `/var` can get
its own disk so runaway logs can't fill the root filesystem, `/tmp` can be wiped
at boot. → *Day 1*

</details>

**A4.** `find . -name *.log` works in one directory but fails with
`paths must precede expression` in another. Why?

<details>
<summary>Answer</summary>

The pattern is unquoted, so **the shell expands the glob before `find` runs**.
Where no `.log` files are in the current directory, bash passes `*.log` through
literally and it works. Where there are several, `find` receives
`-name a.log b.log …` — extra arguments it can't parse. Quote it:
`find . -name '*.log'`. → *Day 2*

</details>

**A5.** `which java` prints `/usr/bin/java`, but typing `java` runs something
else. How is that possible, and what's the better command?

<details>
<summary>Answer</summary>

`which` only searches `PATH`. An **alias**, **shell function** or hashed path
takes precedence over `PATH` and `which` can't see it. `type -a java` (or
`command -v java`) shows what the shell will really run — the usual cause of
"works interactively, fails in a script". → *Day 2*

</details>

**A6.** `df -h` says `/` is 100% full, but `du -sh /*` adds up to 40% of the
disk. What's the most likely cause, and what command finds it?

<details>
<summary>Answer</summary>

A **deleted file that a process still holds open**. `du` walks the directory
tree, so it can't see a file with no name; `df` asks the filesystem, which still
counts its blocks. `lsof +L1` lists open files with zero links. Fix: restart the
holder, or truncate the file through `/proc/PID/fd/N`. Classic cause: someone
`rm`-ed a log that a running app is still writing to. → *Day 2*

</details>

**A7.** `-rw----r--  alice devs  report.txt`. Bob is in the group `devs`. Can bob
read it?

<details>
<summary>Answer</summary>

**No.** The kernel picks **exactly one** triad, first match: owner, else group,
else other. Bob isn't the owner but *is* in `devs`, so the group triad `---`
applies — and the "other" triad's `r` is never consulted, even though strangers
*can* read it. → *Day 3*

</details>

**A8.** `/srv/app/config.yml` is mode `644`, yet `cat` gives
`Permission denied`. What do you check next?

<details>
<summary>Answer</summary>

**Execute (`x`, traverse) permission on every directory in the path** — `/srv`
and `/srv/app`. Reading a file requires `x` on each directory leading to it.
`namei -l /srv/app/config.yml` shows them all at once. → *Day 3*

</details>

**A9.** `/tmp` is world-writable. So why can't you delete another user's file in
it?

<details>
<summary>Answer</summary>

Deleting is a write to the **directory**, not the file — so world-writable would
normally let anyone delete anything. `/tmp` has the **sticky bit** (`drwxrwxrwt`),
which restricts deleting and renaming to the file's owner, the directory's owner,
or root. → *Day 3*

</details>

**A10.** With `umask 022`, what mode does a new file get? A new directory?

<details>
<summary>Answer</summary>

Files start from `666` → **`644`**. Directories from `777` → **`755`**. The umask
removes bits; it never adds them — which is why a new file is never executable
by default. → *Day 3*

</details>

**A11.** A container writes to a bind-mounted directory owned by `shailesh` and
gets `Permission denied`. The container's user is also called `shailesh`.
Explain.

<details>
<summary>Answer</summary>

Ownership is stored as a **UID number**, not a name. The name is just a lookup in
each system's own `/etc/passwd`. Your WSL `shailesh` might be UID 1000 while the
container's `shailesh` is 1001 — same name, different user as far as the kernel
is concerned. → *Day 3*

</details>

---

## Part B — Processes, packages and services (Days 4–5)

**B1.** A JVM shows `VSZ` 11 GB and `RSS` 700 MB. How much RAM is it using, and
what's the other number?

<details>
<summary>Answer</summary>

**RSS — about 700 MB — is real memory** currently in RAM. VSZ is virtual address
space the process has *reserved* (heap reservation, mapped libraries, thread
stacks), most of it never touched. Alarms based on VSZ are almost always wrong.
→ *Day 4*

</details>

**B2.** Load average is 14 on your 12-core laptop, yet CPU usage is only 20%.
How?

<details>
<summary>Answer</summary>

Load average counts processes that are **runnable *or* in uninterruptible sleep
(`D` state)** — usually waiting on disk or network I/O. Lots of processes stuck
in I/O raise the load without using the CPU. Load only means something relative
to core count, and it's inflated by I/O waits. → *Day 4*

</details>

**B3.** Kubernetes stops a pod. What signal is sent first, what happens after how
long, and what exit code do you see in each case?

<details>
<summary>Answer</summary>

`SIGTERM` first — a request the process can handle. If it's still running after
the grace period (**30 s** in Kubernetes, **10 s** for `docker stop`), `SIGKILL`,
which can't be caught. Exit codes are 128 + signal number: **143** for `SIGTERM`,
**137** for `SIGKILL`. Your Spring Boot shutdown has to finish inside the grace
period. → *Days 4 and 9*

</details>

**B4.** How do you get a thread dump from a running JVM without stopping it?

<details>
<summary>Answer</summary>

`kill -3 <pid>` — `SIGQUIT`. The JVM handles it by printing every thread's stack
to its stdout, and keeps running. → *Day 4*

</details>

**B5.** What's the difference between `apt update` and `apt upgrade`? Why does a
Dockerfile write `RUN apt-get update && apt-get install -y …` as one line?

<details>
<summary>Answer</summary>

`update` refreshes the **catalogue** of available versions; `upgrade` actually
installs newer versions. In a Dockerfile, a separate `RUN apt-get update` layer
is cached, so a later `install` step can run against a stale catalogue from
weeks ago. Chaining them in one `RUN` means they're cached together. `apt-get` is
the script-stable interface; `apt` is for humans. → *Day 5*

</details>

**B6.** You `systemctl start` your service, reboot, and it isn't running. Why,
and what does the fix physically create?

<details>
<summary>Answer</summary>

`start` means now; `enable` means at boot. `systemctl enable` creates a
**symlink** to the unit in `/etc/systemd/system/multi-user.target.wants/`, so
it's started when the system reaches that target. `enable --now` does both.
→ *Day 5*

</details>

**B7.** This crontab line never writes anything: `* * * * * date +%F >> /tmp/d.log`.
What's the bug? Name two other classic reasons a cron job works in your shell and
fails in cron.

<details>
<summary>Answer</summary>

**`%` is special in crontab** — cron turns it into a newline and passes the rest
as stdin, so the command is cut off at `%`. Escape it: `date +\%F`. Others: a
minimal **`PATH`** (commands not found); **output discarded** or mailed to a mail
system that doesn't exist, so you never see the error; commands run by
**`/bin/sh`** (dash), not bash; no environment variables from your `.bashrc`.
**Cron is not your shell.** → *Day 5*

</details>

**B8.** A service died at 3am and there's no log file. Where do you look —
give the command with a time filter — and what has to be true for last boot's
logs to still exist?

<details>
<summary>Answer</summary>

The journal: `journalctl -u myapp --since "03:00" --until "04:00"`, with `-p err`
for errors only and `-b -1` for the previous boot. Logs survive reboots only if
the journal is **persistent** — the directory `/var/log/journal` exists —
otherwise they're in `/run` and gone after a reboot. Check before you need it.
→ *Day 5*

</details>

---

## Part C — Finding and transforming text (Days 6–8)

**C1.** Predict the output, then run it:

```bash
printf 'ERROR db down\nINFO a FATAL was logged earlier\nFATAL oom\n' | grep -cE '^ERROR|FATAL'
```

<details>
<summary>Answer</summary>

**3.** `|` has the lowest precedence, so this means "starts with ERROR" **or**
"contains FATAL anywhere" — the INFO line matches too. For "starts with either",
write `^(ERROR|FATAL)`. → *Day 6*

</details>

**C2.** Predict:

```bash
echo 'backend port 8443' | grep -oE '\d+'
```

<details>
<summary>Answer</summary>

**`d`** — from "backend". In GNU grep 3.7, `\d` under `-E` isn't a digit; it's
a literal `d`. And it exits 0: a false positive, not an error. Use `[0-9]+`, or
`grep -P` for PCRE. → *Day 6*

</details>

**C3.** `tail -f app.log | grep ERROR | tee errors.txt` prints nothing for
minutes while errors are being logged. Why, and what's the fix?

<details>
<summary>Answer</summary>

When `grep`'s stdout is a **pipe** rather than a terminal, it **block-buffers**
— it saves output up and flushes in chunks of several KB. Fix:
`grep --line-buffered`. (And use `tail -F` if the log can rotate.) → *Day 6*

</details>

**C4.** `ps aux | grep java` always shows one extra line. What is it? Give two
fixes.

<details>
<summary>Answer</summary>

The `grep java` process itself — its own command line contains "java". Fixes:
`pgrep -a java`, or `ps aux | grep '[j]ava'` — the class `[j]` matches "java" but
the literal text `[j]ava` in grep's own command line doesn't match the pattern.
→ *Day 6*

</details>

**C5.** Predict:

```bash
printf '812\n5000\n17\n' | awk '$1 > "1000"'
```

<details>
<summary>Answer</summary>

**All three lines.** Quoting `"1000"` makes it a *string* comparison, done
character by character: `"812" > "1000"` because `8` > `1`; `"17" > "1000"`
because `7` > `0` at the second character. Numerically only `5000` qualifies:
`awk '$1 > 1000'`. Never quote numbers in awk comparisons. → *Day 7*

</details>

**C6.** Write the pipeline for "top 3 client IPs by request count" from an access
log where the IP is field 1.

<details>
<summary>Answer</summary>

```
awk '{ n[$1]++ } END { for (ip in n) print n[ip], ip }' access.log | sort -rn | head -3
```

`n[$1]++` is a group-by with an associative array. `for (ip in n)` has no order,
so the `sort -rn` stays. The grep-free alternative
`cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -3` also works —
`uniq -c` needs the first `sort`, because it only merges *adjacent* duplicates.
→ *Days 6–7*

</details>

**C7.** You `sed -i` a config file that has a second hard link. What does the
other name show afterwards, and why?

<details>
<summary>Answer</summary>

**The old content.** `sed -i` isn't really in place: it writes a new temp file
and renames it over the original, so the name now points to a **new inode**. The
other hard link still points to the old inode. The same mechanism makes `sed -i`
fail on Docker bind-mounted files and replaces symlinks with regular files.
→ *Day 7*

</details>

**C8.** A CI step runs `sed -i 's/app/app-v2/' deploy.yml`, and the job gets
retried. What goes wrong, and how should it be written?

<details>
<summary>Answer</summary>

It isn't **idempotent**: the second run turns `app-v2` into `app-v2-v2` (and
changes every other "app" in the file). Anchor to the whole line and set the full
value: `sed -i 's/^image: .*/image: app-v2/' deploy.yml` — running it again
changes nothing. → *Day 7*

</details>

**C9.** Predict (needs `jq`):

```bash
echo '{"orderId": 1791234567890123457}' | jq '.orderId'
```

<details>
<summary>Answer</summary>

**`1791234567890123500`.** jq 1.6 stores numbers as 64-bit doubles, exact only
up to 2^53 (about 9 × 10^15); bigger integers are silently rounded. JavaScript
does the same. A Spring API returning `Long` IDs should serialize them as strings
(`@JsonSerialize(using = ToStringSerializer.class)`). → *Day 8*

</details>

**C10.** What's in `config.json` after
`jq '.server.port = 9090' config.json > config.json`? How do you do it correctly?

<details>
<summary>Answer</summary>

**Nothing — it's empty**, and jq exits 0. The shell opens and truncates
`config.json` for the `>` redirect **before** jq runs, so jq reads an empty file.
Correct: `tmp=$(mktemp) && jq '…' config.json > "$tmp" && mv "$tmp" config.json`.
→ *Day 8*

</details>

**C11.** `jq --arg id 5 '.[] | select(.id == $id)' users.json` prints nothing,
although a user with `"id": 5` exists. Why?

<details>
<summary>Answer</summary>

`--arg` always makes a **string**: this compares the number `5` with the string
`"5"`, which are never equal — and there's no error. Use `--argjson id 5`.
→ *Day 8*

</details>

**C12.** A script does `curl -s "$URL" | jq -r '.name'` and later emails
"Dear null". What went wrong, and which flags fix it?

<details>
<summary>Answer</summary>

The API probably returned an error (a `404` with a JSON body like `{}`); without
**`-f`**, curl passed the error body to jq, and `.name` of `{}` is `null`. Fixes:
`curl -fsSL -m 10` so HTTP errors fail; **`set -o pipefail`** so the curl failure
isn't hidden by jq's success; `jq -e` or a check for `null` before using the
value. → *Day 8*

</details>

**C13.** jq says `parse error: Invalid numeric literal at line 1, column 10`.
What has almost certainly happened?

<details>
<summary>Answer</summary>

**You got HTML, not JSON** — a proxy or load-balancer error page, or an SSO login
redirect. Look at the raw response (`curl -sS … | head`) to confirm. → *Day 8*

</details>

**C14.** Give two reasons `xargs` exists, and the flags that make it safe with
odd filenames and with empty input.

<details>
<summary>Answer</summary>

1. Many commands (`rm`, `kill`, `curl URL`) take **arguments, not stdin** —
   `xargs` turns input lines into arguments.
2. The kernel limits command-line size (**`ARG_MAX`**, 2 MB on your WSL), and
   `xargs` splits huge inputs into batches — so `rm *.log` with 300,000 files
   fails but `find … -print0 | xargs -0 rm` works.

Safety: **`-0`** with `find -print0` (any filename, even with spaces or
newlines), **`-r`** so empty input runs nothing. `-P N` runs jobs in parallel.
→ *Day 8*

</details>

---

## Part D — Scripting (Day 9)

**D1.** `./deploy.sh` fails with `/usr/bin/env: 'bash\r': No such file or directory`.
What's the cause? Give three fixes.

<details>
<summary>Answer</summary>

The file has **Windows (CRLF) line endings**, so the shebang line really reads
`#!/usr/bin/env bash\r` and the kernel looks for a program called `bash\r`. Fixes:
`sed -i 's/\r$//' deploy.sh`; switch VS Code's line-ending setting to **LF**; add
`*.sh text eol=lf` to `.gitattributes` so git doesn't convert it on Windows.
→ *Day 9*

</details>

**D2.** Predict:

```bash
set -- "one two" three
for a in $*;   do echo "unquoted: [$a]"; done
for a in "$@"; do echo "quoted:   [$a]"; done
```

<details>
<summary>Answer</summary>

Unquoted: `[one]` `[two]` `[three]` — three words, because unquoted expansion is
word-split. Quoted `"$@"`: `[one two]` `[three]` — each argument intact. `"$@"`
is the only way to pass arguments along safely. → *Day 9*

</details>

**D3.** Predict:

```bash
bash -c 'set -e; f() { false; echo "after false"; }; if f; then echo "f succeeded"; fi; echo "end"'
```

<details>
<summary>Answer</summary>

```
after false
f succeeded
end
```

`set -e` is **suspended** in any condition context — `if`, `while`, `&&`, `||`,
`!` — and that includes *everything inside a function called from there*. `false`
fails, nothing stops, and `f` returns the status of its last command (`echo`, 0).
→ *Day 9*

</details>

**D4.** Predict the output and the exit status:

```bash
bash -c 'set -eo pipefail; seq 1 1000000 | grep -q 5; echo "reached"'; echo "exit=$?"
```

<details>
<summary>Answer</summary>

`exit=141`, and **"reached" is not printed.** `grep -q` finds `5` on the fifth
line and exits; `seq` is still writing, gets `SIGPIPE` (13), and dies with
128 + 13 = 141. With `pipefail`, the pipeline's status is 141, and `-e` kills the
script — because the search *succeeded*. Fix: `grep -q 5 < <(seq 1 1000000)`.
→ *Day 9*

</details>

**D5.** Find at least **five** bugs in this script. Then run the block to see
which one bites first.

```bash
mkdir -p /tmp/quiz && cd /tmp/quiz && rm -f ./*
touch "app 1.log" b.log notes.txt
cat > buggy.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
dir=$1
count=0
for f in $(ls $dir); do
  if [ $f == *.log ]; then
    (( count++ ))
  fi
done
cat "$dir"/*.txt | while read line; do total=$((total + 1)); done
echo "found $count logs, $total notes"
EOF
bash buggy.sh /tmp/quiz; echo "exit=$?"
```

<details>
<summary>Answer</summary>

1. **`dir=$1` without a check.** With no argument, `set -u` stops with
   `$1: unbound variable` — no usage message, no meaningful exit code. Use
   `dir=${1:?usage: buggy.sh DIR}` and check `[[ -d $dir ]]`.
2. **`$(ls $dir)`** — the output is word-split, so `app 1.log` becomes `app` and
   `1.log`. Also `$dir` is unquoted. Use a glob: `for f in "$dir"/*`.
3. **`[ $f == *.log ]`** — `[` doesn't do pattern matching; the unquoted `*.log`
   is expanded by the shell into matching filenames in the *current directory*,
   giving `[: too many arguments`. And `$f` is unquoted. Use `[[ $f == *.log ]]`.
4. **`(( count++ ))`** — when `count` is 0 this returns status 1, and `set -e`
   kills the script on the first match.
5. **`cat … | while`** — the loop runs in a subshell, so `total` never changes in
   the main script — and it was never set, so the final `echo` dies with
   `total: unbound variable`.
6. **`read line`** without `IFS=` and `-r` — mangles whitespace and backslashes.
7. **`"$dir"/*.txt` with no `.txt` files** stays a literal pattern, `cat` fails,
   and with `pipefail` the whole pipeline fails.

When you run it: bug 3 fires first (`too many arguments`, repeatedly — but inside
an `if`, so `set -e` doesn't stop it), which means bug 4 is never even reached;
then bug 5 ends the script with `total: unbound variable`. Bugs hide behind each
other — that's why `shellcheck` beats testing. → *Day 9*

</details>

**D6.** What's the difference between `trap "rm -f $tmp" EXIT` and
`trap 'rm -f "$tmp"' EXIT`? And what can no trap handle?

<details>
<summary>Answer</summary>

Double quotes expand `$tmp` **when the trap is set**; single quotes expand it
**when the trap runs** — normally what you want, and the single-quoted version
also quotes the path properly. No trap runs on **`SIGKILL`** (`kill -9`, the OOM
killer) — it can't be caught, so temp files are left behind. → *Day 9*

</details>

**D7.** One pod keeps restarting with exit code 137; another exited 143; your
script exited 64. What does each tell you?

<details>
<summary>Answer</summary>

**137** = 128 + 9: killed by `SIGKILL` — in Kubernetes usually the **OOM killer**
(check `kubectl describe pod` for `OOMKilled`), or a shutdown that overran the
grace period. **143** = 128 + 15: `SIGTERM` — a normal, requested stop. **64** =
`EX_USAGE`: it was called with the wrong arguments. → *Days 4 and 9*

</details>

**D8.** Why must a variable used in an `EXIT` trap not be declared `local`
inside `main()`?

<details>
<summary>Answer</summary>

The `EXIT` trap runs **after `main` has returned**, when its local variables no
longer exist. Under `set -u`, the cleanup itself crashes with
`unbound variable`, and the temp file leaks. Keep the variables a trap uses
global. → *Day 9*

</details>

**D9.** `next=$(( $(date +%m) + 1 ))` works all year except in two months.
Which ones, why, and what's the fix?

<details>
<summary>Answer</summary>

**August and September.** `date +%m` gives `08` and `09`, and in shell
arithmetic a leading zero means **octal** — where 8 and 9 aren't valid digits:
`value too great for base`. Force base 10: `$(( 10#$(date +%m) + 1 ))`.
→ *Day 9*

</details>

---

## Part E — Put it together

**E1.** 3am: a "disk full" alert on the VM running your Spring Boot service.
Walk through the commands you'd run, in order, and what each one tells you.

<details>
<summary>Answer</summary>

1. `df -h` — which filesystem is full (Day 2).
2. `du -xsh /var/* 2>/dev/null | sort -h` then drill down — where the space is
   (`-x` stays on one filesystem) (Days 2, 7).
3. `find /var/log -xdev -type f -size +500M` — the big files (Day 2).
4. If `df` and `du` disagree: `lsof +L1` — deleted files still held open, often
   a log someone `rm`-ed while the app was writing it (Day 2).
5. `journalctl --disk-usage` — whether the journal is the culprit (Day 5).
6. Fix without making it worse: truncate (`: > file`) rather than delete a file
   in use, or restart the holder; then fix rotation so it doesn't recur (Days 2, 5).

</details>

**E2.** From a JSON Lines log (`app.jsonl`, each line has `level` and `logger`),
produce "number of ERRORs per logger, highest first". Give two ways.

<details>
<summary>Answer</summary>

```
jq -r 'select(.level == "ERROR") | .logger' app.jsonl | sort | uniq -c | sort -rn
jq -s -r 'map(select(.level == "ERROR")) | group_by(.logger)
          | map("\(length) \(.[0].logger)") | .[]' app.jsonl | sort -rn
```

The second needs `-s` (slurp), because `group_by` needs every line in one array.
`grep ERROR | …` would be wrong — it also matches "ERROR" inside messages.
→ *Days 6–8*

</details>

**E3.** Explain to an interviewer, end to end, how
`tail -F app.log | grep --line-buffered ERROR | awk '{ print $5; fflush() }'`
works: the processes, the pipes, the buffering, what happens when the log
rotates, and what happens on Ctrl-C.

<details>
<summary>Answer</summary>

The shell starts **three processes** at once, connected by two kernel pipes;
each process's stdout is the next one's stdin (Day 4). `tail -F` follows the
file **by name**, so when logrotate renames `app.log` and a new one appears, it
reopens the new file (Day 2). `grep` and `awk` would normally **block-buffer**
because their output is a pipe, so `--line-buffered` and `fflush()` make each
line flow immediately (Days 6–7). `$5` is the fifth whitespace-separated field —
the logger in this log format (Day 7). **Ctrl-C** sends `SIGINT` to the whole
foreground process group, so all three die together (Day 4). Kill only one stage
and the others find out when they next read (EOF) or write (`SIGPIPE`).

</details>

**E4.** A teammate wants `set -euo pipefail` at the top of every script "so errors
can't be missed any more". Push back with three specific cases where it won't
catch a failure — or will cause one.

<details>
<summary>Answer</summary>

Any three of:

- **`-e` is off inside `if`/`while`/`&&`/`||`/`!` contexts**, including whole
  functions called from them.
- **`local x=$(cmd)` masks `cmd`'s failure** — the status is `local`'s.
- **`$( )` doesn't inherit `-e`** — commands inside keep running after a failure.
- **`(( i++ ))` from 0 kills the script** even though nothing failed.
- **`grep` with no match exits 1** — a clean log kills the script.
- **`pipefail` + `grep -q`/`head` can give 141** when the search succeeded.

It's a tripwire, not a guarantee: keep it, and still check critical commands
explicitly and run `shellcheck`. → *Day 9*

</details>

---

## Results log

| Date | Score (/46) | Questions missed | Days to reread |
|---|---|---|---|
| | | | |

---

## Q&A
