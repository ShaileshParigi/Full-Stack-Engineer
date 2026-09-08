# Day 4 — Processes, Signals & Jobs

_Why this matters:_ This is the most directly career-relevant day in Phase 1.
When Kubernetes rolls out a new version of your Spring Boot app, it sends your
process a **signal** and gives it a few seconds to finish what it's doing. If
your app doesn't understand that signal, every deploy drops live requests. The
mechanism behind graceful shutdown is exactly what today covers.

> Day 3 (permissions) was skipped. Nothing here depends on it, but circle back
> before Phase 4 — "the JAR can't write its log" and "non-root user in a
> container" are both permissions problems.

---

## Before you read: two questions

1. `kill -9` always works, so why would anyone use plain `kill`?
2. Your container exits with code **137**. What killed it, and how do you know
   from the number alone?

---

## Part 1 — What a process actually is

A **process** is a running program plus everything the kernel tracks about it:
its own memory space, a numeric **PID**, a **parent** (`PPID`), the user it runs
as, a working directory, and a table of open file descriptors.

### Everything has a parent

A process is created by an existing process, via two steps:

1. **`fork()`** — the parent clones itself. Two nearly identical processes now
   exist, differing only in the return value of `fork`.
2. **`exec()`** — the child replaces its own program image with a different one.

That's why every process has a parent, and why the whole system is a tree with
**PID 1 at the root**. On your machine that's `systemd` (from Day 1):

```bash
pstree -sp $$
```

```
systemd(1)---init-systemd(Ub(2)---SessionLeader(401)---Relay(403)(402)---bash(403)
```

Read it right to left: your shell's parent is WSL's relay, whose ancestor is
`systemd`. The `SessionLeader`/`Relay` links are WSL-specific plumbing; on a
normal server you'd see `systemd → sshd → bash`.

**Orphans get adopted.** If a parent dies first, its children are re-parented to
PID 1 rather than being killed. This is how `nohup` and daemons survive you
logging out.

**Zombies** are the opposite case: a child has finished, but its parent hasn't
collected the exit code yet. The process is gone; only an entry in the table
remains, holding a PID. They show as `Z` in `ps`. You can't kill a zombie — it's
already dead. You fix the *parent*, or the parent exits and PID 1 cleans up.

---

## Part 2 — Reading `ps`

`ps` has two historical syntaxes and both survive:

```bash
ps aux     # BSD style — no dash
ps -ef     # UNIX style — with dash
ps -o pid,ppid,stat,rss,comm -p $$     # pick your own columns
```

`ps aux` columns:

| Column | Meaning |
|--------|---------|
| `USER` | Who it runs as |
| `PID` / `PPID` | Its id / its parent's id |
| `%CPU` | CPU share. **Can exceed 100%** — 100% means one full core |
| `%MEM` | Share of physical RAM |
| `VSZ` | **Virtual** size — address space reserved |
| `RSS` | **Resident** set size — actual physical RAM in use |
| `STAT` | State code (below) |
| `COMMAND` | The command line |

### VSZ vs RSS — the one that matters

**`VSZ` is nearly meaningless on its own.** It counts address space the process
has *reserved*, including memory it never touched and shared libraries counted
in full. A JVM routinely shows several gigabytes of VSZ while using a few
hundred MB of real memory.

**`RSS` is the real physical footprint.** When something gets OOM-killed, RSS is
what mattered. You'll return to this in Phase 4 Day 45, where the JVM's heap and
the container's memory limit have to be reconciled.

### STAT codes

| Code | State |
|------|-------|
| `R` | Running or runnable |
| `S` | Sleeping — interruptible, waiting for something |
| `D` | **Uninterruptible sleep** — stuck in a kernel call, usually disk I/O |
| `Z` | Zombie |
| `T` | Stopped (suspended) |
| `+` | In the foreground process group |
| `s` | Session leader |
| `l` | Multi-threaded |

**A `D`-state process cannot be killed — not even with `-9`.** It isn't ignoring
you; it's inside a kernel operation where signals aren't delivered. If you ever
meet an unkillable process, it's almost always `D`, and almost always a storage
or network-filesystem problem.

---

## Part 3 — `top`, `htop`, and load average

```bash
top        # press 1 for per-core, M sort by memory, P sort by CPU, q quit
htop       # nicer; installed on your box
free -h    # memory summary
uptime     # load average, quickly
```

### Load average is not a percentage

```bash
cat /proc/loadavg
nproc
```

On your machine:

```
0.72 0.15 0.05 6/304 443
cores: 12
```

Those three numbers are the **1, 5 and 15-minute averages of the number of
processes that are runnable or in uninterruptible sleep**. It's a *count*, not a
percentage — so it only means something next to your core count.

With **12 cores**:

- load `0.72` → essentially idle (what you have)
- load `12` → fully saturated, everything runnable is running
- load `24` → twice as much work as you have cores; things are queueing

The trend across 1/5/15 matters more than the instant: `10.0 2.0 0.5` is a spike
starting now; `0.5 2.0 10.0` is a spike that's ending.

**Linux counts `D` state in load.** This is the trap: load can be high with idle
CPUs because everything is blocked on disk. `iowait` in `top` is the giveaway —
CPU sitting idle *because* it's waiting on I/O. High load + high iowait + low
CPU means a storage problem, not a compute one.

Your WSL memory:

```
               total        used        free      shared  buff/cache   available
Mem:           3.6Gi       429Mi       2.8Gi       2.0Mi       340Mi       3.0Gi
```

**Read the `available` column, not `free`.** Linux deliberately uses spare RAM
for disk cache (`buff/cache`) and hands it back instantly when a program needs
it. "Free memory is wasted memory." Only `available` tells you what you can
actually allocate.

Note WSL got **3.6 GiB**, not your whole machine — it's a VM with its own budget,
configurable in `.wslconfig`. Worth remembering before you run Kafka, Keycloak
and Postgres in it simultaneously.

---

## Part 4 — Signals

**A signal is a small message the kernel delivers to a process.** Some can be
caught and handled; two cannot.

```bash
kill -l      # list them all
```

The ones that matter:

| Signal | № | Default action | Catchable? | Meaning |
|--------|---|----------------|-----------|---------|
| `SIGHUP` | 1 | terminate | yes | Terminal closed. Now conventionally **"reload config"** |
| `SIGINT` | 2 | terminate | yes | **Ctrl+C** |
| `SIGQUIT` | 3 | terminate + core | yes | Ctrl+\ . **On the JVM: prints a thread dump** |
| `SIGKILL` | 9 | terminate | **NO** | Destroy immediately |
| `SIGTERM` | 15 | terminate | yes | **"Please stop."** The default of `kill` |
| `SIGSTOP` | 19 | suspend | **NO** | Pause |
| `SIGCONT` | 18 | resume | yes | Continue |

**`SIGKILL` and `SIGSTOP` cannot be caught, blocked, or ignored.** That's
deliberate — the OS must always retain the ability to stop a process. Every
other signal is a request the program may handle.

### The difference, demonstrated

Here's a script that cleans up when asked politely:

```bash
mkdir -p /tmp/day4 && cd /tmp/day4
cat > graceful.sh <<'EOF'
#!/usr/bin/env bash
cleanup() {
  echo "  [handler] SIGTERM caught — removing lock"
  rm -f /tmp/day4/work.lock
  exit 0
}
trap cleanup TERM
touch /tmp/day4/work.lock
echo "  [worker] started, pid $$, lock created"
while true; do sleep 0.2; done
EOF
chmod +x graceful.sh
```

**Polite:**

```bash
./graceful.sh &
sleep 1
kill -TERM $!        # no -9
wait $!; echo "exit code: $?"
ls work.lock 2>/dev/null || echo "lock cleaned up"
```

```
  [worker] started, pid 455, lock created
  [handler] SIGTERM caught — removing lock
exit code: 0
lock cleaned up
```

**Brutal — the identical script:**

```bash
./graceful.sh &
sleep 1
kill -KILL $!        # -9
wait $! 2>/dev/null; echo "exit code: $?"
ls work.lock 2>/dev/null && echo "LOCK LEAKED — handler never ran"
```

```
  [worker] started, pid 482, lock created
exit code: 137
work.lock
LOCK LEAKED — handler never ran
```

Same program. With `SIGTERM` the handler ran and the lock was released. With
`SIGKILL` **not one further instruction executed** — no cleanup, no flush, no
closing connections. The lock file is now stale forever.

That is the answer to question 1: `kill -9` always works precisely *because* the
process gets no say — which also means no chance to finish writing a file, commit
a transaction, or deregister from a load balancer.

> **Never reach for `-9` first.** Send `SIGTERM`, give it a few seconds, and only
> escalate if it's genuinely stuck.

### Why this decides whether your deploys lose requests

This is the part that connects to everything later:

| System | What it does |
|--------|--------------|
| `docker stop` | `SIGTERM` → wait **10s** → `SIGKILL` |
| Kubernetes pod deletion | `SIGTERM` → wait `terminationGracePeriodSeconds` (**default 30s**) → `SIGKILL` |
| `systemctl stop` | `SIGTERM` → wait `TimeoutStopSec` → `SIGKILL` |

**Spring Boot graceful shutdown** (`server.shutdown=graceful`) is a `SIGTERM`
handler: stop accepting new connections, let in-flight requests finish, close
the connection pool, then exit.

Two failure modes you'll meet for real:

- **App ignores `SIGTERM`** → every rolling deploy kills live requests mid-flight.
- **App takes longer than the grace period** → `SIGKILL` at 30s, connections cut
  anyway. So the shutdown timeout must be *shorter* than the grace period.

### The JVM trick worth remembering

```bash
kill -3 <java-pid>      # SIGQUIT
```

The JVM catches `SIGQUIT` and dumps **every thread's stack trace** to stdout
without stopping the process. That's how you diagnose a hung Java app in
production — and it's exactly the tool for Phase 5 Day 69 (deadlocks and pool
exhaustion).

### Sending signals

```bash
kill 1234            # SIGTERM — the default
kill -TERM 1234      # the same thing, explicit
kill -9 1234         # SIGKILL
kill -HUP 1234       # reload config, for daemons that support it

pgrep -f 'java.*myapp'      # find PIDs by command line
pkill -f 'java.*myapp'      # signal them by command line
killall firefox             # by exact process name
```

`pkill -f` matches against the **full command line**, which is what you need for
`java -jar app.jar` — the process name is just `java`.

> Be careful with `pkill -f`: a too-broad pattern will match more than you meant.
> Run `pgrep -f` first to see what you're about to hit.

---

## Part 5 — Exit codes

Every process returns a number when it ends. `$?` holds the last one.

```bash
true;  echo $?     # 0
false; echo $?     # 1
ls /nonexistent 2>/dev/null; echo $?    # 2
```

| Code | Means |
|------|-------|
| `0` | Success. **Zero is success** — the opposite of a boolean |
| `1`–`125` | Application-defined failure |
| `126` | Found, but not executable (a permissions problem) |
| `127` | Command not found |
| **`128+N`** | **Killed by signal N** |

That last row answers question 2:

- **`143`** = 128 + 15 = killed by **SIGTERM**
- **`137`** = 128 + 9 = killed by **SIGKILL**

**When a Kubernetes pod shows exit code 137, something sent it SIGKILL** — most
often the kernel's OOM killer because the container exceeded its memory limit,
or the grace period expiring during shutdown. You now know that from the number
alone.

---

## Part 6 — Job control

Your shell manages *jobs* — commands it started.

```bash
sleep 300 &        # start in the background
jobs               # list them
fg %1              # bring job 1 to the foreground
bg %1              # resume a stopped job in the background
kill %1            # signal by job number
```

| Keystroke | Sends | Effect |
|-----------|-------|--------|
| `Ctrl+C` | `SIGINT` | Interrupt the foreground job |
| `Ctrl+Z` | `SIGTSTP` | Suspend it (resume with `fg`/`bg`) |
| `Ctrl+\` | `SIGQUIT` | Quit, with a core dump |

**`Ctrl+Z` leaves the process stopped, not finished.** Suspending a job and
closing the terminal is a classic way to lose work.

### Surviving logout

```bash
nohup ./long-job.sh > job.log 2>&1 &
disown
```

`nohup` makes the process ignore `SIGHUP` (the "terminal closed" signal) and
redirects output, since there's no terminal to write to. `disown` removes it from
the shell's job table entirely.

**In practice, prefer `tmux` for interactive work and a `systemd` unit for
anything that should genuinely keep running.** `nohup` is the emergency option,
not the design.

---

## Hands-on

### 1. Locate yourself in the tree

```bash
echo "pid=$$"
ps -o pid,ppid,stat,rss,comm -p $$
pstree -sp $$
```

### 2. Read `ps` properly

```bash
ps aux | head -5
echo "--- top 5 by memory (RSS) ---"
ps -eo pid,rss,comm --sort=-rss | head -6
echo "--- any D-state processes? ---"
ps -eo pid,stat,comm | awk '$2 ~ /^D/ && $3 != "awk"' || true
echo "(empty is normal and healthy)"
```

Note the `$3 != "awk"` filter: without it the scan usually reports **itself**,
because `awk` sits in `D` for an instant reading from the pipe. A brief `D` is
ordinary — it's a process *stuck* there for seconds that indicates trouble.

### 3. Load and memory

```bash
cat /proc/loadavg
nproc
uptime
free -h
```

Say out loud what the load number means **relative to 12 cores** before moving on.

### 4. The graceful shutdown demo — the important one

```bash
mkdir -p /tmp/day4 && cd /tmp/day4
cat > graceful.sh <<'EOF'
#!/usr/bin/env bash
cleanup() {
  echo "  [handler] SIGTERM caught — removing lock"
  rm -f /tmp/day4/work.lock
  exit 0
}
trap cleanup TERM
touch /tmp/day4/work.lock
echo "  [worker] started, pid $$, lock created"
while true; do sleep 0.2; done
EOF
chmod +x graceful.sh

echo "=== SIGTERM ==="
./graceful.sh &
sleep 1; kill -TERM $!; wait $!; echo "exit=$?"
[ -f work.lock ] && echo "lock: STILL THERE" || echo "lock: cleaned up"

echo "=== SIGKILL ==="
./graceful.sh &
sleep 1; kill -KILL $!; wait $! 2>/dev/null; echo "exit=$?"
[ -f work.lock ] && echo "lock: LEAKED" || echo "lock: cleaned up"
rm -f work.lock
```

Confirm you get exit `0` then exit `137`, and that the lock leaks only the
second time.

### 5. Ignore SIGTERM, then escalate

```bash
cd /tmp/day4
cat > stubborn.sh <<'EOF'
#!/usr/bin/env bash
trap 'echo "  [stubborn] ignoring SIGTERM"' TERM
echo "  [stubborn] pid $$"
while true; do sleep 0.2; done
EOF
chmod +x stubborn.sh

./stubborn.sh &
P=$!
sleep 1
kill -TERM $P; sleep 1
kill -0 $P 2>/dev/null && echo "still alive after SIGTERM"
kill -KILL $P; wait $P 2>/dev/null
echo "exit=$?  (137 = SIGKILL)"
```

`kill -0` sends no signal — it only tests whether you *could* signal the
process. It's the standard "is this PID alive?" check.

### 6. Exit codes

```bash
true;  echo "true          -> $?"
false; echo "false         -> $?"
bash -c 'exit 42'; echo "exit 42       -> $?"
bash -c 'kill -TERM $$'; echo "self SIGTERM  -> $? (expect 143)"
bash -c 'kill -KILL $$'; echo "self SIGKILL  -> $? (expect 137)"
```

### 7. Job control (interactive — use the Ubuntu tab)

```bash
sleep 300
```

Press **Ctrl+Z**, then:

```bash
jobs
bg %1
jobs
kill %1
jobs
```

### 8. Find and signal by pattern

```bash
sleep 400 &
pgrep -f 'sleep 400'
pkill -f 'sleep 400'
pgrep -f 'sleep 400' || echo "gone"
```

Always run `pgrep -f` before `pkill -f`.

---

## Gotchas

- **`kill` sends SIGTERM by default**, not SIGKILL. The name is misleading.
- **`kill -9` skips all cleanup.** Stale locks, half-written files, unclosed
  connections. Escalate to it, don't start there.
- **`SIGKILL` and `SIGSTOP` can't be caught.** Nothing you write handles them.
- **A `D`-state process ignores even `-9`** — it's in a kernel call, usually I/O.
- **Exit code 137 = SIGKILL, 143 = SIGTERM.** In Kubernetes, 137 usually means
  the OOM killer or an expired grace period.
- **`VSZ` is not memory usage.** Use `RSS`.
- **Read `available`, not `free`,** in `free -h`.
- **Load average is a count, not a percentage.** Compare it to `nproc`.
- **`pkill -f` with a loose pattern kills more than you meant.** `pgrep` first.
- **`Ctrl+Z` suspends, it doesn't stop.** The job is still there, frozen.

---

## Recap

- A process is created by `fork` + `exec`, always has a parent, and the tree
  roots at PID 1 (`systemd`). Orphans get re-parented; zombies are dead already.
- **`RSS` is real memory, `VSZ` is reserved address space.** `D` state means
  stuck in the kernel and unkillable.
- **Load average is a count of runnable + uninterruptible processes** — only
  meaningful against your 12 cores, and inflated by I/O waits.
- **`SIGTERM` is a request the process can handle; `SIGKILL` is not.** That one
  distinction is graceful shutdown versus lost requests.
- **Docker gives you 10s, Kubernetes 30s**, then `SIGKILL`. Your Spring Boot
  shutdown must finish inside that window.
- **`128+N` encodes the signal in the exit code** — `137` is SIGKILL, `143` is
  SIGTERM.
- `kill -3` on a JVM prints a full thread dump. Remember this one.

**Next (Day 5):** packages, `systemd` services and `cron` — turning the
`systemd` you already have running into something you can actually operate,
plus the Week 1 review.

---

## Q&A
