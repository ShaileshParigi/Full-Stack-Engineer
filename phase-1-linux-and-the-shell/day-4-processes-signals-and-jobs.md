# Day 4 — Processes, Signals & Jobs

_Why this matters:_ This is the most directly career-relevant day in Phase 1.
When Kubernetes rolls out a new version of your Spring Boot app, it sends your
running program a **signal** and gives it a few seconds to finish what it's
doing. If your app doesn't understand that signal, every deploy drops live
requests. The mechanism behind graceful shutdown is exactly what today covers.

> Day 3 (permissions) was skipped. Nothing here depends on it, but circle back
> before Phase 4 — "the JAR can't write its log" and "non-root user in a
> container" are both permissions problems.

---

## The one-paragraph version

A running program is called a **process**, and Linux gives each one a number so
you can find it and talk to it. You talk to a process by sending it a small
message called a **signal**. There are two that matter: one that means *"please
stop, finish up first"* and one that means *"you're dead now, no last words."*
The polite one can be caught by the program, so it gets a chance to close its
database connections and finish the requests it's already handling. The brutal
one cannot be caught by anything, ever — the program is killed mid-instruction.
Docker, Kubernetes and systemd all use the exact same pattern: send the polite
one, wait a few seconds, then send the brutal one. **Your job as a backend
developer is to make sure your app finishes its cleanup inside that window.**
Everything else today — reading process lists, understanding memory numbers,
exit codes — is the tooling you need to check whether that's actually happening.

---

## Words you'll meet today

Read this table once. Don't try to memorise it — it's here so no term below
ambushes you.

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **process** | A running program | A program in execution, with its own memory space, file descriptors and kernel bookkeeping |
| **PID** | The process's ID number | Process ID — a unique integer the kernel assigns to each process |
| **PPID** | The ID of whatever started it | Parent Process ID |
| **fork** | A process making a copy of itself | The syscall that creates a child process as a duplicate of the parent |
| **exec** | That copy becoming a different program | The syscall that replaces a process's program image with another |
| **daemon** | A program that runs in the background forever | A long-lived background process, usually detached from any terminal |
| **orphan** | A process whose starter died | A process whose parent exited; it gets re-parented to PID 1 |
| **zombie** | A finished process nobody has cleaned up | A terminated process whose exit status the parent hasn't collected via `wait()` |
| **signal** | A one-word message sent to a process | A software interrupt delivered by the kernel to a process |
| **handler** | Code that runs when a signal arrives | A function registered to execute on signal delivery; `trap` in bash |
| **RSS** | The RAM the process is really using | Resident Set Size — physical memory pages currently held |
| **VSZ** | Address space it *reserved*, mostly unused | Virtual Size — total virtual address space mapped |
| **load average** | How many things are queued up to run | Count of processes runnable or in uninterruptible sleep, averaged over 1/5/15 min |
| **iowait** | CPU sitting idle because the disk is slow | Fraction of time CPUs are idle *while* I/O requests are outstanding |
| **exit code** | The number a program returns when it finishes | Integer status returned to the parent; 0 = success |
| **graceful shutdown** | Stopping without dropping work in progress | Refusing new work, completing in-flight work, releasing resources, then exiting |
| **grace period** | How long you get before you're force-killed | Configured wait between SIGTERM and SIGKILL |
| **job** | A command *your shell* started and tracks | A process group under the shell's job control |
| **foreground / background** | Whether it's holding your prompt | Whether the process group owns the terminal |

---

## Before you read: two questions

1. `kill -9` always works, so why would anyone use plain `kill`?
2. Your container exits with code **137**. What killed it, and how do you know
   from the number alone?

---

## Part 1 — What a process actually is

**In plain words:** A program sitting on disk is just a file — inert. When you
run it, Linux loads it into memory, gives it an ID number, and starts executing
it. *That* running thing is a process. The same program can be running five
times at once; that's five processes, five ID numbers, five separate piles of
memory that can't see each other.

A **process** is a running program plus everything the kernel tracks about it:
its own memory space, a numeric **PID**, a **parent** (`PPID`), the user it runs
as, a working directory, and a table of open file descriptors.

### Everything has a parent

**In plain words:** Nothing starts itself. Every process is started by another
process — you type a command, your shell starts it. So the shell is its parent.
Follow the parents upward and you always end at one process that started
everything else.

That happens in two steps:

1. **`fork()`** — the parent clones itself. Two nearly identical processes now
   exist, differing only in the return value of `fork`.
2. **`exec()`** — the child replaces its own program image with a different one.

Copy yourself, then become something else. It looks wasteful and isn't — the
copy is lazy, sharing memory until one side writes to it.

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
remains, holding a PID. They show as `Z` in `ps`.

> **Careful with the word "zombie."** It sounds like something still running.
> It's the opposite — the process is already dead. What survives is a row in the
> kernel's table holding its exit code, waiting for the parent to read it. You
> **can't kill a zombie**; there's nothing there to kill. You fix the *parent*,
> or let the parent exit so PID 1 cleans up.

> **Say this in an interview:** "A process is created by `fork` then `exec` —
> the parent clones itself and the child replaces its image. Every process has a
> parent, so the system forms a tree rooted at PID 1. If a parent exits first
> its children are re-parented to PID 1. A zombie is a terminated child whose
> parent hasn't called `wait()` to reap its exit status — it holds a PID slot
> but no memory, and you fix it by fixing the parent."

---

## Part 2 — Reading `ps`

**In plain words:** `ps` prints the list of what's currently running. It's the
Task Manager of the terminal. The only genuinely confusing part is the memory
columns, because one of them lies to you.

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

**In plain words:** Think of renting a warehouse. **VSZ** is the floor space you
signed for. **RSS** is the space you've actually stacked boxes in. A JVM signs
for an enormous warehouse and then uses one corner of it — so if you look at
VSZ you'll panic about a memory leak that isn't there.

**`VSZ` is nearly meaningless on its own.** It counts address space the process
has *reserved*, including memory it never touched and shared libraries counted
in full. A JVM routinely shows several gigabytes of VSZ while using a few
hundred MB of real memory.

**`RSS` is the real physical footprint.** When something gets killed for using
too much memory, RSS is what mattered. You'll return to this in Phase 4 Day 45,
where the JVM's heap and the container's memory limit have to be reconciled.

### STAT codes

**In plain words:** A one-or-two letter code for what the process is doing right
now. Mostly it's `S` — asleep, waiting for something to happen. That's normal;
most processes spend most of their life waiting.

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

**A `D`-state process cannot be killed — not even with `-9`.**

Why: "uninterruptible" means the process is *inside* a kernel operation — say,
waiting on a network filesystem to answer — and the kernel has deliberately
switched off signal delivery until that operation returns. It isn't ignoring
you; the message can't reach it. If you ever meet a genuinely unkillable
process, it's almost always `D`, and almost always a storage or network-storage
problem, not a problem with the process.

> **Say this in an interview:** "`VSZ` is reserved virtual address space and is
> nearly useless as a memory metric — the JVM inflates it enormously. `RSS` is
> resident physical memory and is what the OOM killer acts on. A process in `D`
> state is in uninterruptible sleep inside a kernel call, so signals aren't
> delivered — that's why it survives `SIGKILL`, and it almost always points at
> blocked I/O."

---

## Part 3 — `top`, `htop`, and load average

**In plain words:** `top` is a live, continuously refreshing version of `ps`.
The number people misread constantly is **load average** — it looks like a
percentage and isn't.

```bash
top        # press 1 for per-core, M sort by memory, P sort by CPU, q quit
htop       # nicer; installed on your box
free -h    # memory summary
uptime     # load average, quickly
```

### Load average is not a percentage

**In plain words:** Imagine one checkout till at a supermarket. Load average is
*how many people are in the queue on average*, including the one being served.
Load `1` on a one-till shop means perfectly busy, no queue. Load `4` means three
people waiting. Now open twelve tills — load `4` is suddenly nothing at all.
**The number is only meaningful next to how many tills you have.** Your tills
are CPU cores.

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
processes that are runnable or in uninterruptible sleep**. A *count*, not a
percentage.

With **12 cores**:

- load `0.72` → essentially idle (what you have)
- load `12` → fully saturated, everything runnable is running
- load `24` → twice as much work as you have cores; things are queueing

The trend across 1/5/15 matters more than the instant: `10.0 2.0 0.5` is a spike
starting now; `0.5 2.0 10.0` is a spike that's ending.

**Linux counts `D` state in load** — and that's the trap. Remember `D` means
"stuck waiting on disk". So the queue can be enormous while every CPU sits idle,
because nothing in the queue is waiting for a *CPU*; they're all waiting for a
*disk*. The giveaway is **`iowait`** in `top`: CPU idle *because* it's blocked on
I/O. High load + high iowait + low CPU = a storage problem, not a compute one.
Buying more CPUs would fix nothing.

### Memory: read the right column

Your WSL memory:

```
               total        used        free      shared  buff/cache   available
Mem:           3.6Gi       429Mi       2.8Gi       2.0Mi       340Mi       3.0Gi
```

**In plain words:** Linux borrows spare RAM to cache files from disk, because
unused RAM is doing nothing useful. That borrowed RAM shows up as "not free"
even though you can have it back instantly the moment a program asks. So `free`
under-reports what you can actually use, and `available` is the honest number.

**Read the `available` column, not `free`.** "Free memory is wasted memory" —
`buff/cache` is returned to programs on demand.

Note WSL got **3.6 GiB**, not your whole machine — it's a VM with its own budget,
configurable in `.wslconfig`. Worth remembering before you run Kafka, Keycloak
and Postgres in it simultaneously.

> **Say this in an interview:** "Load average is a count of runnable plus
> uninterruptible-sleep processes, not a percentage, so it only means anything
> relative to core count. And because Linux includes `D` state, load can be high
> with idle CPUs — that's an I/O bottleneck, and `iowait` confirms it. For
> memory, `available` is the number that matters; `free` excludes reclaimable
> page cache."

---

## Part 4 — Signals

**In plain words:** A signal is a one-word message you send to a running
process. There's no room for detail — just the word. The program can register a
piece of code to run when a particular word arrives (that's a **handler**), and
do whatever cleanup it wants. Two words are special: for those two, the program
gets no handler and no say at all.

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
deliberate: if a program could catch every signal, a buggy or malicious one
could make itself immortal. The OS keeps two it can always use. Every other
signal is a *request* the program may choose to handle.

### The difference, demonstrated

Here's a script that cleans up when asked politely. `trap` is bash's way of
registering a handler — "when signal TERM arrives, run this function":

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

The lock file stands in for anything a real app must release on the way out: a
database transaction, a connection pool, a registration in a load balancer.

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
process gets no say — which also means no chance to finish writing a file,
commit a transaction, or deregister from a load balancer.

> **Never reach for `-9` first.** Send `SIGTERM`, give it a few seconds, and only
> escalate if it's genuinely stuck.

### Why this decides whether your deploys lose requests

**In plain words:** When a deploy replaces your app, something has to stop the
old copy. Every system does it the same way: ask politely, wait, then force.
Your app has to finish its goodbyes inside the wait.

| System | What it does |
|--------|--------------|
| `docker stop` | `SIGTERM` → wait **10s** → `SIGKILL` |
| Kubernetes pod deletion | `SIGTERM` → wait `terminationGracePeriodSeconds` (**default 30s**) → `SIGKILL` |
| `systemctl stop` | `SIGTERM` → wait `TimeoutStopSec` → `SIGKILL` |

**Spring Boot graceful shutdown** (`server.shutdown=graceful`) is nothing more
than a `SIGTERM` handler: stop accepting new connections, let in-flight requests
finish, close the connection pool, then exit.

Two failure modes you'll meet for real:

- **App ignores `SIGTERM`** → every rolling deploy kills live requests mid-flight.
- **App takes longer than the grace period** → `SIGKILL` at 30s, connections cut
  anyway. So your app's shutdown timeout must be *shorter* than the grace period.

### The JVM trick worth remembering

```bash
kill -3 <java-pid>      # SIGQUIT
```

The JVM catches `SIGQUIT` and dumps **every thread's stack trace** to stdout
**without stopping the process** — so it's safe to run against production. A
*thread dump* is a snapshot of what every thread is doing at that instant; if
forty threads are all parked waiting for a database connection, the dump says so
in black and white. That's how you diagnose a hung Java app, and it's exactly
the tool for Phase 5 Day 69 (deadlocks and pool exhaustion).

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
`java -jar app.jar` — the process *name* is just `java`, so matching on the name
would hit every JVM on the box.

> Be careful with `pkill -f`: a too-broad pattern will match more than you meant.
> Run `pgrep -f` first to see what you're about to hit.

> **Say this in an interview:** "`SIGTERM` is a catchable request — the process
> can install a handler and clean up. `SIGKILL` and `SIGSTOP` can't be caught,
> blocked or ignored, so `kill -9` gives the process no chance to release
> resources. Docker waits 10 seconds after SIGTERM, Kubernetes 30 by default,
> then SIGKILLs. Spring Boot's graceful shutdown is a SIGTERM handler, and its
> timeout has to be shorter than the grace period or you get killed mid-drain."

---

## Part 5 — Exit codes

**In plain words:** When a program finishes it hands back a single number. Zero
means it worked. Anything else means it didn't, and the number hints at why.
Backwards from booleans, where 0 is false — here 0 is the good case, because
there's exactly one way to succeed and many ways to fail.

`$?` holds the last one:

```bash
true;  echo $?     # 0
false; echo $?     # 1
ls /nonexistent 2>/dev/null; echo $?    # 2
```

| Code | Means |
|------|-------|
| `0` | Success |
| `1`–`125` | Application-defined failure |
| `126` | Found, but not executable (a permissions problem) |
| `127` | Command not found |
| **`128+N`** | **Killed by signal N** |

That last row is a convention, and it's the useful one. If a process was killed
by a signal rather than exiting on its own, the shell reports `128 + the signal
number`. So:

- **`143`** = 128 + 15 = killed by **SIGTERM**
- **`137`** = 128 + 9 = killed by **SIGKILL**

That's question 2. **When a Kubernetes pod shows exit code 137, something sent
it SIGKILL** — most often the kernel's OOM killer because the container exceeded
its memory limit, or the grace period expiring during shutdown. You know that
from the number alone, before opening a single log.

> **Say this in an interview:** "Exit code 0 is success; 128+N means the process
> was killed by signal N. So 137 is 128+9, SIGKILL — in Kubernetes that's
> usually the OOM killer hitting the container memory limit, or the termination
> grace period expiring. 143 is 128+15, a normal SIGTERM shutdown."

---

## Part 6 — Job control

**In plain words:** A **job** is just "a command your shell is keeping track
of". Normally a command holds your prompt until it finishes — that's the
*foreground*. Add `&` and it runs while you keep typing — the *background*.
`Ctrl+Z` freezes the foreground one so you can get your prompt back.

```bash
sleep 300 &        # start in the background
jobs               # list them
fg %1              # bring job 1 to the foreground
bg %1              # resume a stopped job in the background
kill %1            # signal by job number
```

The `%1` is a *job* number, not a PID — it only means something to your shell.

| Keystroke | Sends | Effect |
|-----------|-------|--------|
| `Ctrl+C` | `SIGINT` | Interrupt the foreground job |
| `Ctrl+Z` | `SIGTSTP` | Suspend it (resume with `fg`/`bg`) |
| `Ctrl+\` | `SIGQUIT` | Quit, with a core dump |

Notice those keystrokes are just signals with a keyboard shortcut. `Ctrl+C`
isn't special magic — it's `SIGINT`, which is catchable, which is why some
programs ask "are you sure?" instead of dying.

**`Ctrl+Z` leaves the process stopped, not finished.** Suspending a job and
closing the terminal is a classic way to lose work.

### Surviving logout

**In plain words:** When you close a terminal, everything it started normally
dies with it — the kernel sends `SIGHUP` ("your terminal is gone") to the whole
group. `nohup` means "no hangup": ignore that signal and keep going.

```bash
nohup ./long-job.sh > job.log 2>&1 &
disown
```

`nohup` also redirects output, since there's no terminal left to print to.
`disown` removes the job from the shell's table entirely, so the shell won't
even try to notify it.

**In practice, prefer `tmux` for interactive work and a `systemd` unit for
anything that should genuinely keep running.** `nohup` is the emergency option,
not the design — you'll see the proper version tomorrow.

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
second time. **This is the single most important thing you'll run in Phase 1.**

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

`kill -0` sends no signal at all — it only tests whether you *could* signal the
process. It's the standard "is this PID still alive?" check, and you'll see it
in shell scripts everywhere.

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

- **`kill` sends SIGTERM by default**, not SIGKILL. The name is misleading —
  `kill` really means "send a signal".
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
- **You can't kill a zombie.** It's already dead — fix the parent.

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
- `kill -3` on a JVM prints a full thread dump without stopping it. Remember
  this one.

---

## Say it out loud

Answer each in about 30 seconds, using the real terms. If you can only *point*
at the answer, re-read that part — recognising a word isn't the same as owning
it.

1. What actually happens, step by step, when you type `ls` and press Enter?
2. Why does `kill -9` "always work", and what does that cost you?
3. A colleague says "the server load is 8, we're at 80% capacity." What's wrong
   with that sentence?
4. Your pod restarted with exit code 137. Walk through what you'd check and why.
5. Explain to a junior why `VSZ` on a Java process looks alarming and isn't.
6. What's a zombie process, and why can't you kill it?

---

**Next (Day 5):** packages, `systemd` services and `cron` — turning the
`systemd` you already have running into something you can actually operate,
plus the Week 1 review.

---

## Q&A
