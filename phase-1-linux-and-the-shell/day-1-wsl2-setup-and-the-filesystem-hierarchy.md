# Day 1 — WSL2 Setup & the Filesystem Hierarchy

_Why this matters:_ Every server you will ever deploy to runs Linux. Docker
images are Linux. Kubernetes nodes are Linux. When your Spring Boot app dies at
3am, you will be reading Linux logs on a Linux box. Today is about making the
machine stop being a black box.

---

## Before you read: two questions

Answer these in your head first. They tell you where your gaps actually are.

1. On a Linux server, where would you look for the configuration file of a
   service you just installed? And where would you look for its logs?
2. `ps` tells you which processes are running. Where does `ps` get that
   information from? It has to come from somewhere.

If both were instant, skim to the Hands-on. If not, that's exactly what today
covers.

---

## Part 1 — What WSL2 actually is

There were two generations of WSL, and the difference matters.

- **WSL1** was a *translation layer*. Linux programs made Linux system calls,
  and Windows translated each one into a Windows call. No Linux kernel existed.
  Anything that needed real kernel behaviour broke.
- **WSL2** runs a **real Linux kernel** in a lightweight virtual machine. Full
  system-call compatibility. Real `/proc`, real containers, real `systemd`.

You can prove which you have — this is a genuine Linux kernel version string,
built by Microsoft:

```bash
uname -r
grep PRETTY_NAME /etc/os-release
```

On your machine:

```
5.15.133.1-microsoft-standard-WSL2
PRETTY_NAME="Ubuntu 22.04.3 LTS"
```

### Your setup is already done

Worth stating plainly so you don't go looking for work that isn't there. Most
WSL guides tell you to enable `systemd` because it's off by default. **Yours is
already on.**

```bash
cat /etc/wsl.conf
ps -p 1 -o comm=
systemctl is-system-running
```

Gives:

```
[boot]
systemd=true

systemd
running
```

**`systemd` is the init system** — process ID 1, the first process the kernel
starts, and the ancestor of everything else. It starts services at boot, restarts
them when they crash, and manages their logs. `systemctl` is how you talk to it.
Without it, PID 1 would just be your shell and `systemctl` would fail — which is
what most WSL setups look like, and why so many tutorials don't work there.

You'll use this properly on Day 4.

---

## Part 2 — The two filesystems, and why one is slow

This is the single most practical WSL fact.

Your Ubuntu has **two very different kinds of storage**:

| Path | What it is | Speed |
|------|-----------|-------|
| `/`, `/home/shailesh_parigi` | Real **ext4**, inside a virtual disk (`ext4.vhdx`) | Native Linux speed |
| `/mnt/c`, `/mnt/d`, `/mnt/e` | Your **Windows drives**, exposed across the VM boundary | Much slower |

Linux and Windows are two different operating systems here. When Linux touches
`/mnt/d`, every single file operation becomes a message passed to Windows and
back. For one big file that's fine. For **many small files** — which is exactly
what a source tree, `node_modules`, or a Maven build is — the per-file cost
dominates.

Measure it rather than believe me:

```bash
timeit() {
  local dir="$1" label="$2"
  rm -rf "$dir"; mkdir -p "$dir"
  local t0=$(date +%s%N)
  for i in $(seq 1 500); do echo x > "$dir/f$i"; done
  local t1=$(date +%s%N)
  echo "$label $(( (t1 - t0) / 1000000 )) ms"
  rm -rf "$dir"
}

timeit "$HOME/.perftest" "ext4  (~)      :"
timeit "/mnt/d/perftest" "9p    (/mnt/d) :"
```

Measured on your machine:

```
ext4  (~)      : 302 ms
9p    (/mnt/d) : 1781 ms
```

**Nearly 6× slower**, for 500 trivial files.

### The rule that follows

> Linux work belongs in the Linux filesystem (`~`). Windows work belongs on
> `C:`/`D:`.

Your `D:\Learn Daily` project is on the Windows side, which is correct — you
edit it with Windows VS Code and run Windows Node and Java against it. But when
a Linux exercise says "make some files and experiment," do it in `~`, not
`/mnt/d`. Crossing the boundary repeatedly is the mistake.

---

## Part 3 — The filesystem hierarchy

### The organizing principle

Here's the idea that makes the whole layout click, and it is genuinely
different from Windows.

**Windows organizes by application.** Everything for one program lives under
`C:\Program Files\ThatProgram\`.

**Linux organizes by _what kind of data it is_.** A single program is
deliberately scattered:

| The program's… | goes in |
|---|---|
| executable | `/usr/bin/nginx` |
| configuration | `/etc/nginx/` |
| logs and changing data | `/var/log/nginx/` |
| documentation, static assets | `/usr/share/nginx/` |

That looks like chaos until you ask *why*: it groups files by **how they behave**,
so each group can be treated differently.

- `/usr` is program code that only changes when you install or upgrade — it can
  be mounted **read-only**, or shared between machines.
- `/etc` is small, precious, and hand-edited — **back this up**.
- `/var` grows without bound — put it on its own disk so a runaway log file
  fills that partition instead of taking down the whole system.
- `/tmp` is disposable — wipe it on boot.

"A log file filled the disk and took down the server" is a real, common outage.
This layout is the defence against it. You'll meet the same reasoning again when
you write Dockerfiles (Phase 4) and mount volumes in Kubernetes (Phase 17).

### The directories worth knowing

```bash
ls -1 /
```

| Directory | What lives there |
|-----------|------------------|
| `/etc` | System-wide **configuration**, always plain text. Never binaries. |
| `/var` | **Variable** data: `/var/log` (logs), caches, spools, databases |
| `/tmp` | Scratch space, world-writable, cleared on reboot |
| `/usr` | The bulk of the OS: `/usr/bin`, `/usr/lib`, `/usr/share` |
| `/home` | User home directories — yours is `/home/shailesh_parigi` |
| `/root` | **root's** home. Not `/home/root` |
| `/opt` | Self-contained third-party software |
| `/proc` | **Virtual** — live kernel and process state (Part 4) |
| `/sys` | **Virtual** — kernel view of devices and drivers |
| `/dev` | Device files (`/dev/null`, `/dev/sda`) |
| `/boot` | Kernel and bootloader — **nearly empty on WSL**, since Windows boots you |
| `/mnt` | Mount points — where your Windows drives appear |
| `/bin`, `/sbin`, `/lib` | Symlinks into `/usr` on modern Ubuntu (the "usr merge") |

Prove that last one rather than taking it on faith:

```bash
ls -ld /bin /sbin /lib
```

They're symlinks to `/usr/bin`, `/usr/sbin`, `/usr/lib`. Historically `/bin` held
the essentials needed before `/usr` was mounted; that distinction stopped being
useful, so Ubuntu merged them and left symlinks for compatibility.

**WSL oddities you'll see in `/`:** a `Docker` directory (Docker Desktop's
integration) and several `wslXXXXXX` directories (temporary mount points WSL
manages). Neither exists on a normal Ubuntu server.

---

## Part 4 — `/proc`, and why it isn't a real filesystem

This answers diagnostic question 2, and it's the most important idea today.

### First pass: the simple version

`/proc` looks like a directory full of files. **None of those files exist on
disk.** When you read one, the kernel generates the answer *at that instant* out
of its own memory.

It's a live window into the kernel, disguised as files — so that ordinary tools
like `cat`, `grep` and `less` can inspect a running system with no special API.

### Second pass: what's actually there

`/proc` is a **virtual filesystem** (`procfs`) mounted at `/proc`. It contains:

- **One numbered directory per running process** — `/proc/1234/` is PID 1234
- **System-wide files** — `/proc/cpuinfo`, `/proc/meminfo`, `/proc/uptime`
- **`/proc/self`** — a magic symlink that always resolves to *the PID of whoever
  is reading it*

Inside a process's directory:

| File | Contents |
|------|----------|
| `status` | Human-readable: name, PID, parent PID, memory, thread count |
| `cmdline` | The exact command line, arguments separated by NUL bytes |
| `environ` | Its environment variables, also NUL-separated |
| `exe` | Symlink to the actual binary on disk |
| `cwd` | Symlink to its current working directory |
| `fd/` | One entry per open file descriptor |

**So: `ps`, `top`, `free` and `htop` are all just readers of `/proc`.** There is
no secret API. That's the answer to question 2.

### Proving it's generated, not stored

```bash
ls -l /proc/uptime
cat /proc/uptime
sleep 1
cat /proc/uptime
```

Output:

```
-r--r--r-- 1 root root 0 Sep  5 15:23 /proc/uptime
9676.51 115222.64
9677.53 115234.71
```

**The file reports size 0**, because the kernel has no idea how long the answer
will be until you ask. Yet reading it twice gives different values. No file on
disk behaves like that.

### Why you will care later

- **Phase 4 (Docker):** a container is largely a process with a *different view
  of `/proc`*. Isolation is implemented right here.
- **Phase 4, Day 45:** the classic JVM-in-a-container bug is the JVM reading
  the host's memory from `/proc/meminfo` instead of its container limit, sizing
  its heap for a machine it isn't on, and getting killed.
- **Phase 5, Day 69:** diagnosing a stuck production process means reading its
  `/proc` entry — what it's blocked on, what files it has open.

---

## Part 5 — Home, `~`, and dotfiles

Your home directory is `/home/shailesh_parigi`. The shell expands `~` to it, and
`$HOME` holds it.

```bash
echo "$HOME"
cd ~
pwd
```

**A file whose name starts with `.` is hidden** — that's a pure `ls` convention,
not a permission. `ls -a` shows them. Configuration in your home directory uses
this so your home isn't cluttered.

```bash
ls -a ~ | head -20
```

The two that matter now:

- **`~/.bashrc`** — runs for every *interactive non-login* shell. Aliases and
  prompt settings go here. This is the one you'll edit most.
- **`~/.profile`** — runs for *login* shells. Environment variables like `PATH`
  belong here.

The login/non-login distinction confuses everyone; you'll pin it down on Day 5.
For now: if you add something to `.bashrc` and a new shell doesn't see it, you
either need a new shell or `source ~/.bashrc`.

---

## Hands-on

Run each of these. Use the **Ubuntu (WSL)** tab, or hit **Run** on the block —
both land in the same Ubuntu.

### 1. Confirm what you're running on

```bash
uname -r
grep PRETTY_NAME /etc/os-release
ps -p 1 -o comm=
systemctl is-system-running
```

### 2. Tour the top level

```bash
ls -1 /
echo "--- /etc: config, all text ---"
ls /etc | head -10
echo "--- /var/log: where logs live ---"
ls /var/log | head -10
echo "--- /bin is a symlink now ---"
ls -ld /bin /sbin /lib
```

### 3. Inspect your own process through `/proc`

`$$` is the shell's own PID.

```bash
echo "my pid: $$"
grep -E '^(Name|Pid|PPid|VmRSS|Threads)' /proc/$$/status
echo "cmdline: $(tr '\0' ' ' < /proc/$$/cmdline)"
echo "exe -> $(readlink /proc/$$/exe)"
echo "cwd -> $(readlink /proc/$$/cwd)"
```

`cmdline` and `environ` use **NUL bytes** as separators, which terminals don't
display — that's why `tr '\0' ' '` is needed to make it readable.

### 4. Prove `/proc` is live

```bash
ls -l /proc/uptime
cat /proc/uptime
sleep 1
cat /proc/uptime
```

### 5. Feel the `/mnt` penalty

```bash
timeit() {
  local dir="$1" label="$2"
  rm -rf "$dir"; mkdir -p "$dir"
  local t0=$(date +%s%N)
  for i in $(seq 1 500); do echo x > "$dir/f$i"; done
  local t1=$(date +%s%N)
  echo "$label $(( (t1 - t0) / 1000000 )) ms"
  rm -rf "$dir"
}

timeit "$HOME/.perftest" "ext4  (~)      :"
timeit "/mnt/d/perftest" "9p    (/mnt/d) :"
```

### 6. Write it down

Create your own reference — you'll remember what you wrote far better than what
you read:

```bash
mkdir -p ~/notes
cat > ~/notes/fs-hierarchy.md <<'EOF'
# Top-level directories

/etc   -
/var   -
/tmp   -
/usr   -
/home  -
/opt   -
/proc  -
/sys   -
/dev   -
EOF
echo "created ~/notes/fs-hierarchy.md"
```

Then fill in one line each **from memory**, and only check afterwards. The
`<<'EOF'` bit is a heredoc — you'll cover it properly on Day 5.

---

## Gotchas

- **`/mnt/d` is slow.** Do Linux exercises in `~`. Keep the Java project on `D:`
  where Windows tools reach it.
- **`/proc` files report size 0.** They aren't empty. Never trust file size there.
- **`/root` is not `/home/root`.** Root's home is its own top-level directory.
- **`/boot` is nearly empty on WSL** and `/sys` is partially populated. WSL uses
  a Microsoft-built kernel and doesn't boot itself. On a real server both are full.
- **Editing Windows files from Linux and vice versa** works but is a good way to
  create permission and line-ending confusion. Pick a side per project.
- **Harmless noise:** running bash through the Lab prints
  `your 131072x1 screen size is bogus` on stderr. That's WSL noticing there's no
  real terminal attached. It is not your script failing.

---

## Recap

- WSL2 is a **real Linux kernel in a VM** — not a translation layer. Your setup
  already has `systemd` enabled, which most don't.
- Two filesystems: **ext4 (`~`) is fast, `/mnt/*` is ~6× slower** because every
  operation crosses into Windows.
- Linux organizes files **by kind of data, not by application** — so `/usr` can
  be read-only, `/etc` backed up, `/var` given its own disk, `/tmp` wiped.
- **`/proc` is not on disk.** It's kernel state rendered as files, generated on
  read. `ps`, `top` and `free` are just readers of it.
- Dotfiles configure your shell; `~/.bashrc` for interactive shells.

**Tomorrow (Day 2):** navigating and finding things — `find`, globbing, and the
crucial detail that the *shell* expands `*` before your command ever sees it.

---

## Q&A

**Q:** In Part 2 you say "for **many small files** — which is exactly what a
source tree, `node_modules`, or a Maven build is — the per-file cost dominates."
Explain that in depth, from first principles.

### First, what a "file operation" actually is

A program never touches a disk. It asks the kernel to, via a **syscall** — a
system call, the one mechanism a user program has for asking the kernel to do
something on its behalf. `open`, `read`, `write`, `close`, `stat` are all
syscalls.

Trace `cat` reading one six-byte file:

```
openat(AT_FDCWD, ".../f1", O_RDONLY)                 = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=6}) = 0
read(3, "hello\n", 131072)                           = 6
read(3, "", 131072)                                  = 0     <- the EOF read
close(3)                                             = 0
```

**Five syscalls to read one file.** Note the second `read`: the program doesn't
know it's at the end of the file until a read returns zero bytes. And
`newfstatat` — a pure metadata question, "how big is this, what are its
permissions", moving no file data at all.

Across 200 files, `cat` issued **1,270 syscalls** — about 6 per file. Hold onto
that ratio.

### What each syscall costs on ext4

`~` is ext4 inside a virtual disk, managed by the Linux kernel running in your
WSL2 VM. A syscall there is a mode switch into the kernel in the same VM — a few
microseconds. And most of the time the answer is already in the **page cache**
(the kernel's in-RAM copy of recently used file data and metadata), so no disk
is touched at all.

Order of magnitude: **~10 microseconds**, often less.

### What each syscall costs on `/mnt/d`

`/mnt/d` is not a filesystem. It's a **network filesystem client wearing a
filesystem's clothes.** Linux mounts it with the **9p** protocol (from Plan 9;
WSL calls the driver `drvfs`). The sequence for a single `stat`:

```
your program        stat("/mnt/d/x/f1")
   |  syscall
Linux kernel        9p client: build a Twalk/Tgetattr message
   |  write to a virtual socket (hvsocket) - crosses the VM boundary
Windows             a user-mode 9p server process wakes up
   |
Windows             GetFileAttributesEx() -> NTFS
   |                translate: NTFS ACLs -> Unix mode bits, FILETIME -> Unix time
   |  reply message back across the boundary
Linux kernel        hand result to your program
```

Two process wakeups, two VM-boundary crossings, one Win32 call, and a metadata
translation — **for one question.** There is no shared page cache across that
boundary either: Linux can't safely cache Windows metadata, because Windows may
change it behind Linux's back.

Order of magnitude: **~2 milliseconds.** That is roughly **200x the ext4 cost,
per operation.**

### The distinction that explains everything: latency vs throughput

- **Throughput (bandwidth)** — bytes per second once data is moving.
- **Latency** — the fixed delay to complete one request, regardless of size.

A courier crossing a border: customs takes 20 minutes whether the truck carries
one envelope or ten tonnes. One big shipment amortises the crossing. Ten
thousand envelopes, each in its own truck, pay it ten thousand times.

The cost model:

```
total time  ~=  (number of operations x latency)  +  (bytes / bandwidth)
                        ^^^ per-op term                ^^^ bulk term
```

On ext4, latency is so small the per-op term nearly vanishes and the bulk term
dominates. On 9p, latency is ~200x larger, so for any workload with many
operations the **per-op term swallows everything else**. That is what "the
per-file cost dominates" means — literally: one term of that sum becomes so much
larger than the other that the other stops mattering.

### Proof, measured on your machine

```
                             ext4 (~)        9p (/mnt/d)      ratio
copy one 100MB file          3027 ms         1468 ms          9p FASTER
create 2000 tiny files        573 ms         6963 ms          12x slower
stat 2000 files (0 bytes!)     16 ms         3888 ms          243x slower
```

Read the first row again. Moving **100 megabytes** across the boundary was
*faster* than writing it to ext4. (The ext4 run includes a flush into the
`.vhdx`; the 9p one likely benefits from Windows write caching — so don't read
it as "9p is faster at bulk", read it as: **bulk transfer is not where the gap
is.** Bandwidth is fine.)

The third row is the whole argument. Two thousand pure metadata questions,
**zero bytes of file data**, and it's 243x slower. Per operation: 8 microseconds
vs 1.94 milliseconds.

### Now the arithmetic on real workloads

Multiply that ~2ms by what these tools actually do:

- **`node_modules`** — routinely 30,000+ files. `npm install` creates every one,
  then Node's module resolver `stat`s its way up the directory tree for each
  `require`. At 6 syscalls per file: ~180,000 operations x 2ms is roughly
  **6 minutes of pure boundary-crossing.**
- **A Maven build** — `mvn compile` stats every `.java` under `src/`, resolves
  and opens hundreds of jars in `~/.m2`, and writes a `.class` per class.
  Thousands of files, each 5-6 syscalls.
- **`git status`** — `stat`s *every tracked file* to compare mtime against the
  index. On a 5,000-file repo that's 5,000 pure-metadata ops. This is why
  `git status` feels instant in `~` and takes seconds on `/mnt/d`.
- **`ls -R`** — one `getdents` plus a `stat` per entry. That's the Day 1 practice
  exercise, and now you know why it splits the way it does.

### Why *builds* are the worst case specifically

Incremental builds are built on one question, asked over and over: **"is this
output older than its input?"** That's `stat` — timestamp comparison. Not
reading files, not writing them. The single operation that measured **243x
slower** is the one a build system is made of.

A build isn't data-heavy. It's **metadata-heavy and chatty.** That's precisely
the profile 9p is worst at.

### The shape to remember — you'll meet it three more times

Chatty, latency-bound work across a boundary. Same shape, different boundary:

| Where | The chatty pattern | The fix |
|---|---|---|
| Here | 30k files x one round trip each | Keep the work on one side of the boundary |
| **Phase 7 (JPA)** | N+1 queries — 1 query for a list, then 1 per row | `JOIN FETCH` / batch fetch |
| **Phase 11 (system design)** | Microservice calls one service per item in a loop | Batch endpoint |
| **Phase 5 (concurrency)** | Serial blocking I/O calls | Parallelise so latencies overlap |

The fix is always the same idea: **reduce the number of crossings, or overlap
them.** Never "make the crossing faster" — you can't.

> **Interview note:** "why is this slow?" is answered wrong when you reach for
> bandwidth. The first question is always *how many round trips*, and the
> giveaway is that total time scales with the **number of items**, not their
> size. That's the signature of a latency-bound workload.

### Rules that follow

- Source trees, `node_modules`, `target/`, `.git` — **live on the side that
  builds them.** Windows Maven/Node on `D:`; Linux builds in `~`.
- Never build in `/mnt/*` from Linux. Never build in the `\\wsl$\` share from
  Windows — same boundary, opposite direction, same cost.
- One big file across the boundary is fine. Copy a jar, a tarball, a dump — no
  problem.
- Your `D:\Learn Daily` setup is correct: Windows tools, Windows files, no
  crossing.

### Run it yourself

```bash
bench() {
  d="$1"; l="$2"; rm -rf "$d"; mkdir -p "$d"
  head -c 100M /dev/zero > /tmp/src.bin
  t0=$(date +%s%N); cp /tmp/src.bin "$d/big.bin"; t1=$(date +%s%N)
  echo "$l  copy one 100MB file       : $(( (t1-t0)/1000000 )) ms"
  t0=$(date +%s%N); for i in $(seq 1 2000); do echo x > "$d/f$i"; done; t1=$(date +%s%N)
  echo "$l  create 2000 tiny files    : $(( (t1-t0)/1000000 )) ms"
  t0=$(date +%s%N); for i in $(seq 1 2000); do [[ -f "$d/f$i" ]]; done; t1=$(date +%s%N)
  echo "$l  stat 2000 files (0 bytes) : $(( (t1-t0)/1000000 )) ms"
  rm -rf "$d"; rm -f /tmp/src.bin
}
bench "$HOME/.perf3" "ext4"
bench "/mnt/d/perf3" "9p  "
```

`[[ -f ... ]]` is a bash *builtin* — no process is forked, so that row measures
the filesystem and nothing else. Then count the syscalls behind one file read:

```bash
mkdir -p ~/.sysc && echo hello > ~/.sysc/f1
strace -e trace=openat,newfstatat,read,close cat ~/.sysc/f1
```
