# Day 5 — Packages, Services & Scheduled Jobs (+ Week 1 Review)

_Why this matters:_ Day 4 taught you how to *watch and signal* a running
program. Today is the layer above: how software gets onto a machine, how it
starts **without anyone typing a command**, and how it gets restarted when it
dies. Every `Dockerfile` you write in Phase 4 is a package-manager script. Every
Spring Boot service deployed outside a container is a `systemd` unit. And
`systemd` is where the `SIGTERM` timing from yesterday is actually configured.

> Day 3 (permissions) is still skipped. Today touches `sudo` lightly; that's
> fine. Circle back before Phase 4.

---

## The one-paragraph version

Software doesn't arrive on a Linux box by downloading an installer. There's a
**package manager** — on Ubuntu that's `apt` — which knows about official
servers full of pre-built software, works out what else each thing depends on,
and installs the lot. Once software is installed, something has to *start* it
and *keep* it started, including after a reboot when nobody is logged in. That's
**systemd**: you write a small text file describing how to run your app, and
systemd runs it, restarts it if it crashes, captures everything it prints, and
stops it politely on shutdown — using exactly the SIGTERM-then-SIGKILL pattern
from yesterday. Separately, for things that should run *on a schedule* rather
than continuously — a nightly backup, a cleanup job — there's **cron**, an old
and extremely reliable tool with one famous quirk: it runs your command in a
nearly empty environment, so commands that work fine when you type them fail
silently under cron. **Packages, services, schedules. That's the whole day.**

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **package** | A zip of a program plus instructions on where its files go | An archive with metadata: file list, dependencies, and pre/post-install scripts |
| **`.deb`** | The package file format Ubuntu/Debian use | Debian binary package archive |
| **package manager** | The tool that installs software and its dependencies | Resolves the dependency graph, downloads, and invokes the low-level installer |
| **repository (repo)** | A server holding thousands of packages plus an index | A signed, indexed collection of packages served over HTTP |
| **index** | The catalogue of what's available and at what version | Package lists downloaded to `/var/lib/apt/lists/` |
| **dependency** | Another package this one needs to work | A declared requirement resolved before installation |
| **GPG key / signing** | A tamper-proof seal proving a repo is who it claims | Cryptographic signature over the repo index, verified before trust |
| **init system** | The first program that starts, which starts everything else | The process the kernel launches as PID 1 |
| **service / daemon** | A program meant to run continuously in the background | A long-lived process, usually detached from any terminal |
| **systemd** | Ubuntu's init system and service manager | PID 1; manages units, dependencies, logging and lifecycle |
| **unit** | One thing systemd manages, described by a text file | A configuration object: `.service`, `.timer`, `.socket`, `.target`, `.mount` |
| **unit file** | That text file — the recipe for running your app | INI-format config in `/lib/systemd/system` or `/etc/systemd/system` |
| **target** | A named milestone in booting, e.g. "system is up" | A unit type used to group and order other units |
| **enable** | "Start this automatically at boot" | Symlinks the unit into a target's `.wants` directory |
| **cgroup** | A box the kernel puts a process and all its children in | Control group — kernel mechanism for grouping and limiting processes |
| **drop-in** | A small override file layered on top of a package's unit file | `/etc/systemd/system/<unit>.d/*.conf`, merged over the vendor unit |
| **journal** | systemd's central log store, queried not read | Structured, indexed binary log managed by `systemd-journald` |
| **cron** | A daemon that runs commands on a schedule | Time-based job scheduler reading crontab files |
| **crontab** | The file listing your scheduled jobs | Per-user schedule file in `/var/spool/cron/crontabs/` |
| **MTA** | The thing that would send email, usually absent | Mail Transfer Agent — cron pipes job output to it |
| **`PATH`** | The list of folders the shell searches for a command | Colon-separated directory list searched for executables |

---

## Before you read: three questions

1. You run `apt install nginx`. Nothing else. Reboot the machine. Is nginx
   running?
2. Your cron job runs `docker ps >> /tmp/out.log` and the file stays empty — but
   the identical line works when you paste it into your shell. Why?
3. A service crashes at 3am. There is no log file anywhere on disk for it. Where
   do you look?

---

## Part 1 — Packages: `dpkg` and `apt`

**In plain words:** On Windows you download an `.exe` and run it. On Linux you
don't — you ask a **package manager** for a program by name, and it fetches it
from an official server along with everything else that program needs. Two
different tools do the two halves of that job, and confusing them is the first
hurdle.

### Two layers, not one

**`dpkg`** is the low-level tool. It takes one `.deb` file and unpacks it into
the filesystem. A `.deb` is an archive of files plus metadata: where each file
goes, what other packages it needs, and scripts to run before and after install.

**`dpkg` does not resolve dependencies.** Hand it a `.deb` that needs three
other packages and it stops with an error listing them. That's all it does.

**`apt`** is the layer on top. It knows about **repositories** — servers holding
thousands of `.deb` files plus an index of them. `apt` reads that index, works
out the full chain of dependencies, downloads everything, and calls `dpkg` to
do the actual installing.

```
apt  →  resolves dependencies, downloads from repos
 ↓
dpkg →  unpacks one .deb into the filesystem
```

If it helps: it's the same split as Maven and your local `.m2`. Maven works out
the transitive dependency tree and fetches it; something much dumber writes the
jar to disk. `apt` is Maven, `dpkg` is the file write.

### `apt` vs `apt-get`

Both exist and both work, which is confusing until you know why.

`apt` is the newer, human-facing front end — colours, a progress bar, sane
defaults. `apt-get` (plus `apt-cache` for searching) is the older pair, and
crucially its **command-line behaviour is guaranteed not to change between
releases**.

That distinction is not cosmetic:

```bash
apt list --installed 2>&1 | head -3
```

Pipe `apt` anywhere and it warns you: *"WARNING: apt does not have a stable CLI
interface. Use with caution in scripts."*

**Rule: `apt` when you're typing, `apt-get` when a script is typing.** That's
why every Dockerfile you'll ever read uses `apt-get` — a Dockerfile is a script,
and a script that breaks on an Ubuntu upgrade is a bad Dockerfile.

### The commands that matter

```bash
sudo apt update              # refresh the package INDEX. Installs nothing.
sudo apt upgrade             # install newer versions of what you already have
sudo apt install jq tree     # install
sudo apt remove jq           # remove the program, KEEP its config files
sudo apt purge jq            # remove the program AND its config files
sudo apt autoremove          # drop dependencies nothing needs any more
apt search json              # search names + descriptions
apt show jq                  # detail on one package
apt list --installed         # everything installed
```

### The naming trap: `update` vs `upgrade`

**In plain words:** `update` refreshes the *catalogue*. `upgrade` installs new
*versions*. The words sound like they mean the same thing and they absolutely
don't — this is the single most common Linux beginner confusion.

- `apt update` downloads the *list* of what's available and at what version. It
  changes nothing on your system except a cache in `/var/lib/apt/lists/`.
- `apt upgrade` actually installs newer versions.

**You must `update` before `install`.** Otherwise apt is working from a stale
catalogue and asks the mirror for a version that was replaced weeks ago. The
mirror only keeps current versions, so you get `404 Not Found` — on a build that
worked last week, with a Dockerfile nobody changed.

That's why every Dockerfile has this exact shape:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
      curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

Three things in there worth naming now, since you'll write this for real in
Phase 4:

- **`update && install` in the *same* `RUN`.** Docker caches each instruction as
  a layer it can reuse. Split these into two `RUN` lines and Docker will happily
  reuse a months-old `update` layer with a fresh `install`. Same 404, and it
  looks like a mystery because *your* line didn't change.
- **`-y`** answers the confirmation prompt. A build has no keyboard.
- **`rm -rf /var/lib/apt/lists/*`** deletes the catalogue you just downloaded —
  tens of MB the running container never needs, baked into the image forever if
  you leave it.

### `remove` vs `purge`

```bash
sudo apt remove  redis-server   # binary gone, /etc/redis/redis.conf stays
sudo apt purge   redis-server   # binary and config both gone
```

**In plain words:** `remove` deliberately leaves your settings behind so a
reinstall picks up where you left off. Helpful — until you're trying to clear a
broken setting, reinstall three times, and it's still broken. `purge` is the one
you actually wanted.

### Where did the files go?

**In plain words:** A package doesn't land in one folder. It scatters files
across the tree you learned on Day 1 — the program into `/usr/bin`, its config
into `/etc`, its docs into `/usr/share/doc`. Two commands let you look both
ways: what did this package install, and what installed this file?

```bash
dpkg -L jq            # list every file this package installed
dpkg -S /usr/bin/jq   # which package OWNS this file? (reverse lookup)
dpkg -l | grep jq     # is it installed, and at what version?
dpkg -s jq            # full status record
```

`dpkg -S` is the one that earns its keep. You find a mystery binary or config
file on a server and want to know where it came from — `dpkg -S` names the
package. And if it answers *"no path found matching pattern"*, that's
information too: **the file was not installed by a package.** Someone `curl`ed
it in, or a build produced it. On a server you didn't set up, that tells you a
lot about how it was assembled and what won't get security updates.

> **Say this in an interview:** "`dpkg` installs a single `.deb` and doesn't
> resolve dependencies; `apt` sits on top, reads the repository index, resolves
> the dependency graph and drives `dpkg`. `apt update` refreshes the index while
> `apt upgrade` installs — which is why `apt-get update` and `apt-get install`
> must be in the same Dockerfile `RUN`, or layer caching serves you a stale
> index and the install 404s. And use `apt-get` in scripts, because `apt`
> doesn't guarantee a stable CLI."

---

## Part 2 — Repositories

**In plain words:** `apt` can only install software that's listed somewhere it
knows to look. Those places are **repositories** — servers run by Ubuntu (and by
vendors like Docker) holding the packages and a catalogue of them. The list of
which servers to trust is just a text file.

```bash
grep -v '^#' /etc/apt/sources.list | grep -v '^$'
ls /etc/apt/sources.list.d/
```

Your box:

```
deb http://archive.ubuntu.com/ubuntu/ jammy main restricted
deb http://archive.ubuntu.com/ubuntu/ jammy-updates main restricted
deb http://archive.ubuntu.com/ubuntu/ jammy universe
...
deb http://security.ubuntu.com/ubuntu/ jammy-security main restricted
```

Read one line as four fields:

| Field | Example | Meaning |
|-------|---------|---------|
| type | `deb` | binary packages (`deb-src` = source code instead) |
| URL | `http://archive.ubuntu.com/ubuntu/` | the server |
| **suite** | `jammy` | the Ubuntu release. `jammy` **is** 22.04 |
| **components** | `main restricted universe` | which sections of it to use |

Ubuntu releases have codenames — `jammy` is 22.04, `focal` is 20.04, `noble` is
24.04. You'll see them constantly in install instructions.

### Components — a support split, not a technical one

| Component | Contents |
|-----------|----------|
| `main` | Free software, **officially supported by Canonical** |
| `restricted` | Proprietary but necessary — mostly hardware drivers |
| `universe` | Free software, community maintained, **no Canonical support** |
| `multiverse` | Legally restricted (patents, non-free licences) |

The thing to take away: `main` gets security patches with a support commitment
behind them; `universe` mostly doesn't. That matters when someone asks whether a
dependency is "supported."

### Pockets — the suffix on the suite

| Suite | What it is |
|-------|-----------|
| `jammy` | The release exactly as it shipped, frozen |
| `jammy-security` | **Security fixes only.** Never disable this one |
| `jammy-updates` | Bug fixes and non-security updates |
| `jammy-backports` | Newer versions pulled back from later releases, opt-in |

**In plain words:** this is how an LTS release stays both *stable* and *patched*.
You don't get a new major version of nginx — you get the same version with the
security hole fixed, shipped through `-security`. That's what "Long Term
Support" actually buys.

### Third-party repos, and why the signing key matters

`/etc/apt/sources.list.d/` is empty on your machine right now. It won't be after
Phase 4 — installing Docker Engine on a real Ubuntu server drops a file there.

**In plain words:** adding a repository means "I trust this server to give me
software." Installing a package **runs its setup scripts as root**. So trusting
a repo is trusting whoever controls it with full control of your machine. That's
why every repo signs its catalogue with a cryptographic key, and `apt` refuses
to install anything whose signature doesn't check out.

The modern pattern (you'll type this in Phase 4):

```bash
# 1. fetch the publisher's key into its own keyring file
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 2. add the repo, pinned to THAT key only
echo "deb [signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu jammy stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list
```

`signed-by=` is the important bit: it scopes that key to **one** repository.

The old advice — `apt-key add` — put keys into a **global** trust store instead.
The consequence: any repo you had ever added, for any reason, could then sign an
update for *any* package on your system, including `openssh-server`. Add one
sketchy PPA in 2019 and it could ship you a backdoored SSH daemon today. That is
why `apt-key` is deprecated, and why a tutorial still using it is a stale
tutorial you shouldn't follow.

Two directories worth knowing:

```bash
ls /var/lib/apt/lists/ | head        # the downloaded index (apt update writes here)
ls /var/cache/apt/archives/ | head   # downloaded .deb files
```

> **Say this in an interview:** "A sources line is type, URL, suite and
> components — `jammy` is 22.04, and `main` is Canonical-supported where
> `universe` is community. Security patches arrive through the `-security`
> pocket, which is how an LTS gets fixes without version bumps. Repos are GPG
> signed because installing a package executes maintainer scripts as root; the
> modern pattern scopes a key to one repo with `signed-by=`, whereas the
> deprecated `apt-key` put it in a global trust store where any repo could sign
> for any package."

---

## Part 3 — `systemd`: units and `systemctl`

**In plain words:** Something has to start your app when the machine boots,
notice when it crashes, start it again, and stop it properly on shutdown — with
nobody logged in. On Ubuntu that something is **systemd**. You describe your app
in a small text file, and systemd does the rest.

You already met systemd on Day 4 as **PID 1**, the root of the process tree.
That's half its job. This is the other half.

### Units

**In plain words:** systemd calls everything it manages a **unit**, and the file
extension tells you what kind of thing it is. You will almost always be writing
`.service` files.

| Type | Manages |
|------|---------|
| `.service` | A process — the one you'll write |
| `.socket` | A listening port; starts the service on first connection |
| `.timer` | A schedule — the modern `cron` |
| `.target` | A grouping or milestone, e.g. `multi-user.target` |
| `.mount` | A filesystem mount |

```bash
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service | head -20
```

Note the difference: `list-units` shows what is **loaded right now**;
`list-unit-files` shows everything **installed**, running or not.

`systemctl` is just the command you use to talk to systemd. Every verb below is
`systemctl <verb> <unit>`.

You have a real service to poke at — Redis is running in your WSL:

```bash
systemctl status redis-server
```

```
● redis-server.service - Advanced key-value store
     Loaded: loaded (/lib/systemd/system/redis-server.service; enabled; ...)
     Active: active (running) since ...; 2h ago
   Main PID: 312 (redis-server)
      Tasks: 5 (limit: 4437)
     Memory: 3.1M
     CGroup: /system.slice/redis-server.service
             └─312 "/usr/bin/redis-server 127.0.0.1:6379"
```

Read that carefully — it answers most of what you'd want to know:

- **`Loaded:`** where the unit file lives, and whether it's **`enabled`** (i.e.
  starts at boot — see below, this word has a specific meaning)
- **`Active:`** current state and **how long for**. A short uptime on a service
  that's been deployed for months means it restarted recently — often your first
  clue during an incident
- **`Main PID:`** the PID from Day 4; `Tasks:` is the thread count
- **`CGroup:`** every process systemd considers part of this service

That last line matters more than it looks. Systemd tracks a service as a
**cgroup** — a box the kernel keeps processes in — rather than as a single PID.
So if your app forks child processes, the kernel still knows they belong to this
service. Stop the service and *all* of them stop. This is the thing `nohup` and
hand-rolled PID files get wrong: they lose track of children, which then survive
a restart and hold the port open, and now your service won't start.

### `start` vs `enable` — the distinction people get wrong

```bash
sudo systemctl start   redis-server        # run it NOW. Says nothing about boot.
sudo systemctl enable  redis-server        # run it AT BOOT. Says nothing about now.
sudo systemctl enable --now redis-server   # both
```

**In plain words:** "start" and "enable" sound like the same thing and are
completely independent. Start = right now. Enable = every future boot. You can
have either without the other.

| | enabled | disabled |
|---|---|---|
| **running** | normal, healthy | started by hand; **gone after reboot** |
| **stopped** | comes back at reboot | off |

That's question 1. `apt install nginx` on Ubuntu happens to enable *and* start
it, because the Debian package's post-install script does that for you. On RHEL
it does neither. And for a unit file *you* write, it does neither. **"It worked
fine, then we rebooted and it never came back" is an `enable` someone forgot.**

`enable` isn't magic. It creates a symlink:

```bash
ls -l /etc/systemd/system/multi-user.target.wants/ | head
```

`multi-user.target` is the milestone meaning "the system is up and ready for
normal use". Enabling a service symlinks it into that target's `.wants`
directory, so on the way to reaching the target, systemd pulls your service in.
`disable` deletes the symlink. That's the entire mechanism — no database, no
registry, just symlinks in a directory you can list.

### The rest of the verbs

```bash
sudo systemctl stop    redis-server
sudo systemctl restart redis-server   # stop, then start
sudo systemctl reload  redis-server   # re-read config WITHOUT restarting
systemctl is-active    redis-server   # scriptable: prints active/inactive
systemctl is-enabled   redis-server
systemctl cat          redis-server   # show the unit file(s)
systemctl show         redis-server   # every resolved property, ~200 of them
```

**`reload` vs `restart`:** restart stops the process and starts a new one —
every connection drops. Reload asks the *running* process to re-read its config
file and carry on, so nothing is interrupted. Remember `SIGHUP` from Day 4?
That's usually how it's implemented. Not every service supports it; if it
doesn't, `reload` errors and `reload-or-restart` falls back to a restart.

### What a unit file looks like

This is the Spring Boot deployment you'll do for real. Read it as three blocks —
who I am, how to run me, when to start me:

```ini
[Unit]
Description=Orders API
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/orders
Environment="SPRING_PROFILES_ACTIVE=prod"
EnvironmentFile=-/etc/orders/env
ExecStart=/usr/bin/java -jar /opt/orders/app.jar
Restart=on-failure
RestartSec=5
TimeoutStopSec=45
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target
```

| Directive | Why it's there |
|-----------|----------------|
| `After=` | **Ordering only** — start me after these, *if* they're being started |
| `Wants=` | A *soft* dependency — try to pull it in, but don't fail if it fails |
| `Requires=` | A *hard* dependency — if it fails, I fail too |
| `Type=simple` | `ExecStart` is the process itself; systemd doesn't wait for it to fork |
| `User=` | **Don't run your app as root.** A web app never needs it |
| `EnvironmentFile=-` | The leading `-` means "don't fail if the file is missing" |
| `Restart=on-failure` | Restart on non-zero exit or a signal, but **not** on a clean exit 0 |
| `RestartSec=5` | Wait 5s between attempts, so a crash loop doesn't pin a CPU |
| `WantedBy=` | Which target `enable` should hook this into |

**`After=` is not `Requires=`** — and this is the classic bug. They feel like
one idea and they're two: *ordering* and *dependency*.

`After=postgresql.service` alone means "if Postgres is being started as part of
this boot, start me afterwards." It does **not** mean "make sure Postgres is
started." If Postgres isn't enabled at all, your app starts anyway, immediately
fails to connect, and — with `Restart=on-failure` — crash-loops forever while
systemd cheerfully reports it's doing its job. You almost always want both
`Requires=` and `After=` naming the same unit.

### The Day 4 connection

Look at these three lines again:

```ini
KillSignal=SIGTERM
TimeoutStopSec=45
Restart=on-failure
```

`systemctl stop` sends **`SIGTERM`**, waits `TimeoutStopSec`, then sends
**`SIGKILL`**. That is precisely `docker stop`'s 10 seconds and Kubernetes'
30-second `terminationGracePeriodSeconds` — **one mechanism, three config key
names.** Yesterday you learned the mechanism; today you're seeing the second of
three dialects.

The consequence is concrete: if your Spring Boot shutdown takes 60 seconds and
`TimeoutStopSec=45`, systemd `SIGKILL`s you at 45. Exit code **137** (`128 + 9`)
instead of the clean **143** you'd get from a completed SIGTERM shutdown, and
whatever was still draining is cut off mid-request.

### Editing units correctly

```bash
sudo systemctl edit redis-server        # drop-in override (preferred)
sudo systemctl edit --full redis-server # copy the whole file to /etc and edit
sudo systemctl daemon-reload            # REQUIRED after editing by hand
```

Three directories, in ascending priority:

| Path | Owner |
|------|-------|
| `/lib/systemd/system/` | **The package.** Never edit — `apt upgrade` overwrites it |
| `/etc/systemd/system/` | **You.** Wins over the package |
| `/run/systemd/system/` | Runtime, gone on reboot |

**In plain words:** if you edit the package's file directly, the next `apt
upgrade` silently replaces it and your change vanishes — usually months later,
during an unrelated patching window, at the worst possible time.

`systemctl edit` avoids that by writing a **drop-in**: a small file at
`/etc/systemd/system/<unit>.d/override.conf` containing *only* the lines you
changed, which systemd layers on top of the package's file. Package updates and
your override coexist.

> **`daemon-reload` is the one everyone forgets.** Systemd reads unit files into
> memory once and caches them. Edit a file in `vim`, run `systemctl restart`,
> and it will confidently start the service with the **old** config — no error,
> no warning, and you'll swear your edit didn't save. `systemctl edit` runs the
> reload for you. Editing by hand does not.

> **Say this in an interview:** "systemd is PID 1 and the service manager. A
> unit file declares `ExecStart`, the user, restart policy and dependencies.
> `start` and `enable` are independent — enable just symlinks the unit into
> `multi-user.target.wants` so it comes up at boot. `After=` is ordering only,
> `Requires=` is the actual dependency, and you normally want both. systemd
> tracks the service as a cgroup, so forked children can't escape a stop. And
> `systemctl stop` is SIGTERM, wait `TimeoutStopSec`, then SIGKILL — the same
> contract as `docker stop` and a Kubernetes grace period."

---

## Part 4 — `journalctl`

**In plain words:** This is question 3. On older systems, each service wrote its
own log file into `/var/log/` and you'd `tail` it. Under systemd that's changed:
**whatever a service prints to the screen, systemd captures and stores centrally.**
So your app can log with nothing but `System.out.println` and still be fully
searchable — but you won't find a file to `cat`. You have to *query* the store,
and the tool is `journalctl`.

The store is deliberately not a text file. It's binary and indexed, with metadata
attached to every line — which unit, which PID, which boot, what priority — so
you can filter on any of those instantly instead of grepping gigabytes.

```bash
journalctl -u redis-server            # everything from one unit
journalctl -u redis-server -n 50      # last 50 lines
journalctl -u redis-server -f         # follow, like tail -f
journalctl -u redis-server -e         # jump to the END (most recent)
journalctl -u redis-server --since "10 min ago"
journalctl -u redis-server --since "2026-09-08 09:00" --until "2026-09-08 10:00"
journalctl -u redis-server -p err     # priority err and worse
journalctl -b                         # this boot only
journalctl -b -1                      # the PREVIOUS boot
journalctl -k                         # kernel messages (dmesg)
journalctl -u redis-server -o json-pretty | head -30
```

The flags that pay for themselves:

| Flag | Use |
|------|-----|
| `-u` | Scope to one unit. Without it you get the whole machine |
| `-f` | Live follow — what you watch during a deploy |
| `-e` | Start at the end, not the beginning |
| `--since` / `--until` | Natural language works: `"1 hour ago"`, `"yesterday"` |
| `-p` | `emerg alert crit err warning notice info debug`; `-p err` = err and worse |
| `-b -1` | **Previous boot** |
| `-o json` | Structured output, for piping into `jq` |

**`journalctl -u <unit> -b -1 -p err` is the single most useful incident command
on a systemd box.** Unpack it: errors only (`-p err`), from that one service
(`-u`), from the boot *before* the one you're in (`-b -1`). That last flag is
the point — when a machine hard-crashes and reboots, the evidence is in the
previous boot, and by default you're only looking at the current one.

### Persistence — the trap

```bash
ls -d /var/log/journal 2>/dev/null && echo PERSISTENT || echo "VOLATILE — logs die on reboot"
journalctl --disk-usage
```

**In plain words:** the journal only survives a reboot if a particular directory
exists. If it doesn't, systemd keeps logs **in RAM** — and a reboot wipes them.
So the command above (`-b -1`) returns absolutely nothing, exactly when you need
it most.

If `/var/log/journal/` exists, logs go to disk. If it doesn't, journald falls
back to `/run/log/journal/`, which is tmpfs — a filesystem that lives in memory.

Yours is persistent. Many minimal Debian installs and container images are not,
and discovering that mid-incident is a genuinely bad afternoon. The fix is one
line:

```bash
sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald
```

Size is capped (default around 10% of the filesystem) and old entries are
dropped automatically, so it won't fill your disk:

```bash
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=500M
```

> **Say this in an interview:** "Under systemd a service's stdout and stderr are
> captured by journald into a structured, indexed binary journal rather than a
> flat file, so you query it with `journalctl` — `-u` to scope to a unit, `-p`
> for priority, `-b -1` for the previous boot. The gotcha is persistence: if
> `/var/log/journal` doesn't exist, logs go to tmpfs and are lost on reboot,
> which is exactly when you need the previous boot's errors."

---

## Part 5 — `cron`

**In plain words:** systemd keeps things running *continuously*. `cron` runs
things *on a schedule* — a nightly backup, a cleanup at 3am. It's a daemon that
wakes up every minute and runs whatever's due. Dead simple, extremely reliable,
and it has a handful of traps that make jobs fail *silently* — which is the
entire reason it has a bad reputation.

```bash
crontab -e     # edit YOUR crontab
crontab -l     # list it
crontab -r     # DELETE it. No confirmation. Right next to -e.
```

> **`crontab -r` wipes your entire crontab silently, no "are you sure".** And
> it's one key away from `-e`. Build the habit now:
> `crontab -l > ~/cron.bak` before you ever edit.

### The five fields

Each line is five time fields then the command. Read the diagram top to bottom:

```
┌───────── minute        0–59
│ ┌─────── hour          0–23
│ │ ┌───── day of month  1–31
│ │ │ ┌─── month         1–12
│ │ │ │ ┌─ day of week   0–7   (0 AND 7 are both Sunday)
│ │ │ │ │
* * * * *  command to run
```

`*` in a field means "every value of this field". So `* * * * *` is every minute
of every hour of every day.

| Syntax | Means |
|--------|-------|
| `*` | every value |
| `*/5` | every 5th — `*/5 * * * *` is every 5 minutes |
| `1,15,30` | a list |
| `9-17` | a range |
| `0 2 * * *` | 02:00 every day |
| `0 9 * * 1-5` | 09:00 Monday–Friday |
| `*/15 9-17 * * 1-5` | every 15 min, 9am–5pm, weekdays |

Shorthands, if you'd rather not count fields:

| Shorthand | Equivalent |
|-----------|-----------|
| `@reboot` | once, when cron starts at boot |
| `@hourly` | `0 * * * *` |
| `@daily` / `@midnight` | `0 0 * * *` |
| `@weekly` | `0 0 * * 0` |

**The day-of-month / day-of-week trap:** if you set *both* to something other
than `*`, cron treats them as **OR**, not AND. `0 0 13 * 5` runs on the 13th
**and** on every Friday — not on Friday the 13th. Almost nobody means that.

### The four gotchas that make cron jobs silently fail

**In plain words:** every one of these has the same root cause — **cron is not
your shell.** It doesn't load your settings, doesn't know your `PATH`, doesn't
use bash, and throws away your output. Your command isn't wrong; its
*surroundings* are missing.

**1. `PATH` is nearly empty.** Cron reads no `~/.bashrc`, no `~/.profile`,
nothing. A user crontab runs with roughly `PATH=/usr/bin:/bin`. So `docker`,
`mvn`, `java`, anything in `/usr/local/bin` or installed by `sdkman` — **not
found.** The job runs, fails instantly, and you never see the error.

That's question 2. Two fixes:

```cron
# absolute paths — always correct
*/5 * * * * /usr/bin/docker ps >> /tmp/out.log 2>&1

# or set PATH at the top of the crontab
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**2. Output goes nowhere.** Cron's original design mails stdout and stderr to
the user. On a box with no mail system installed — yours, most servers, every
container — the mail attempt fails and **the output is simply discarded**. Your
job ran, it failed, and the error message evaporated. This is why "cron jobs
fail silently" is a cliché.

**Always redirect:**

```cron
*/5 * * * * /opt/app/backup.sh >> /var/log/backup.log 2>&1
```

`2>&1` means "send stderr to wherever stdout is currently going", so errors land
in the log too. It must come **after** the redirect — `2>&1 >> file` reads left
to right and sends stderr to the *old* destination, which is nowhere.

**3. `%` is a metacharacter.** Inside a crontab an unescaped `%` means "newline
— treat everything after this as input to the command". It detonates on the most
natural line you'd ever write:

```cron
# BROKEN — cron cuts the line at the first %
* * * * * date +%F >> /tmp/t.log

# CORRECT
* * * * * date +\%F >> /tmp/t.log
```

The broken version raises no error anywhere. Cron runs `date +` and feeds
`F >> /tmp/t.log` to it as input. You get nothing and no explanation. Any date
format string in a crontab needs its `%` escaped.

**4. Not a login shell.** `SHELL` defaults to `/bin/sh`, not bash, so bash-only
syntax breaks. No `JAVA_HOME`, no environment your login sets up, and the working
directory is your home rather than wherever you assumed.

The general fix for all four: **make the cron line call a script, and put the
setup inside the script.** One-liner cron entries are where this goes wrong.

### System crontabs

Your personal crontab lives in `/var/spool/cron/crontabs/<user>` and always runs
as you. There's a second, system-wide form:

```bash
cat /etc/crontab
ls /etc/cron.d/ /etc/cron.daily/
```

`/etc/crontab` and files in `/etc/cron.d/` have **six** fields, not five —
there's an extra **user** column, because a system file has to say who to run as:

```cron
# m h dom mon dow  USER   command
  0 3 *   *   *    root   /usr/local/bin/rotate-logs.sh
```

Copying a line between the two formats is a common failure: paste a system line
into your personal crontab and cron reads `root` as the command.

`/etc/cron.daily/` and friends are directories of *scripts*, run by a helper
called `run-parts`. Scripts there must be executable and **must not have a file
extension** — `run-parts` skips any filename containing a dot. So `backup.sh`
sits there being silently ignored while `backup` runs perfectly. Another silent
one, and a genuinely surprising rule.

### Where cron output actually goes

```bash
journalctl -u cron --since "10 min ago"
grep CRON /var/log/syslog | tail
```

Careful here: cron logs *that it ran the job*, not what the job printed. So a
`CRON` line with nothing changed on disk means **your command ran and failed** —
which narrows it straight to gotcha 1 or 3.

### `systemd` timers — what you'll actually use

**In plain words:** systemd can do scheduling too, and on a server you control
it's the better choice — mainly because the output problem disappears entirely.
It's two files instead of one line: a `.service` saying *what* to run, and a
`.timer` saying *when*.

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Nightly backup
[Service]
Type=oneshot
ExecStart=/opt/app/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup nightly
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300
[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now backup.timer
systemctl list-timers --all
```

Why it's better:

- **Logs land in the journal** — `journalctl -u backup.service`. Cron's single
  biggest problem, gone.
- **`Persistent=true`** runs a job that was missed because the machine was off
  when it was due. Cron just skips it and says nothing.
- **`RandomizedDelaySec`** spreads the start time randomly over 5 minutes, so
  200 servers don't all hammer the same database at exactly 02:00:00.
- Full unit semantics: `After=`, `User=`, resource limits, `Type=oneshot`.

Cron still wins on one axis: it's everywhere, including containers and
non-systemd boxes, and its syntax is in everyone's head. **Know both; reach for
the timer on a server you control.**

> **Say this in an interview:** "Cron failures are almost always environmental,
> not logical — it sources no profile, so `PATH` is roughly `/usr/bin:/bin`, and
> it mails output to an MTA that usually doesn't exist, so errors vanish. You
> fix both with absolute paths and `>> log 2>&1`. Unescaped `%` also truncates
> the line. On a machine I control I'd use a systemd timer instead: output goes
> to the journal, `Persistent=true` catches missed runs, and
> `RandomizedDelaySec` avoids a thundering herd."

---

## Hands-on

### 1. Look before you install

```bash
apt show jq 2>/dev/null | head -20
apt-cache depends jq
command -v jq tree || echo "neither installed yet — correct"
```

### 2. Install, then find every file

```bash
sudo apt update
sudo apt install -y jq tree
command -v jq tree
dpkg -L jq
echo "--- where did the binary land? ---"
dpkg -S "$(command -v jq)"
```

Notice `jq` pulled in `libjq1` and `libonig5` — dependencies you never asked
for and didn't know existed. That is the entire point of `apt` over `dpkg`.

### 3. Actually use them (both are worth keeping)

```bash
tree -L 2 /etc/apt
echo '{"name":"orders","replicas":3,"ports":[8080,9090]}' | jq '.'
echo '{"name":"orders","replicas":3,"ports":[8080,9090]}' | jq -r '.ports[0]'
```

`jq` is the tool you'll live in from Phase 15 onward — every AWS CLI and
Kubernetes command returns JSON.

### 4. Reverse lookup on a mystery file

```bash
dpkg -S /usr/bin/ssh
dpkg -S /bin/ls
dpkg -S /etc/hostname 2>&1 || echo "^ not owned by any package"
```

### 5. Read a real service

```bash
systemctl status redis-server
echo "=== is it enabled? ==="
systemctl is-enabled redis-server
systemctl is-active redis-server
echo "=== the unit file ==="
systemctl cat redis-server
echo "=== resolved shutdown settings (the Day 4 connection) ==="
systemctl show redis-server -p KillSignal -p TimeoutStopUSec -p Restart -p MainPID
```

Then use yesterday's tools on systemd's `Main PID`:

```bash
PID=$(systemctl show redis-server -p MainPID --value)
ps -o pid,ppid,stat,rss,comm -p "$PID"
```

Check the `PPID`. It's **1** — systemd. Not your shell. That's what "runs
independently of any login session" looks like in practice.

### 6. `start` vs `enable`, demonstrated

```bash
systemctl is-enabled redis-server
ls -l /etc/systemd/system/multi-user.target.wants/ | grep redis
```

That symlink **is** "enabled". There is nothing else to it.

```bash
echo "=== what runs at boot on this box ==="
systemctl list-unit-files --type=service --state=enabled --no-pager | head -20
```

### 7. Read the journal

```bash
journalctl -u redis-server -n 20 --no-pager
echo "=== errors on this boot, whole system ==="
journalctl -b -p err --no-pager | tail -20
echo "=== is the journal persistent? ==="
ls -d /var/log/journal >/dev/null 2>&1 && echo PERSISTENT || echo VOLATILE
journalctl --disk-usage
```

Now watch a restart happen live. Open a **second** Ubuntu tab:

```bash
journalctl -u redis-server -f
```

Then in the first tab:

```bash
sudo systemctl restart redis-server
```

You'll see the stop and start logged in real time. That's a deploy, in miniature
— and it's exactly what you'll be staring at during a real one.

### 8. The cron job — the important one

```bash
crontab -l > ~/cron.bak 2>/dev/null; echo "backed up (may be empty)"
mkdir -p /tmp/day5
( crontab -l 2>/dev/null; echo '* * * * * /usr/bin/date +\%F_\%T >> /tmp/day5/tick.log 2>&1' ) | crontab -
crontab -l
```

Three deliberate choices in that one line — absolute path `/usr/bin/date`,
escaped `\%`, and `>> ... 2>&1`. Each defuses one of the four gotchas. Now wait:

```bash
sleep 70; cat /tmp/day5/tick.log
sleep 60; cat /tmp/day5/tick.log
```

Two timestamps, a minute apart. Confirm cron agrees it ran:

```bash
journalctl -u cron --since "5 min ago" --no-pager | tail -5
```

### 9. Now break it on purpose

```bash
mkdir -p /tmp/day5
( crontab -l 2>/dev/null
  echo '* * * * * date +%F >> /tmp/day5/broken-percent.log 2>&1'
  echo '* * * * * echo "PATH=$PATH" >> /tmp/day5/path.log 2>&1'
) | crontab -
sleep 70
echo "--- percent gotcha (expect EMPTY or missing) ---"
cat /tmp/day5/broken-percent.log 2>/dev/null || echo "(no file — the line was truncated at %)"
echo "--- what PATH does cron actually give you? ---"
cat /tmp/day5/path.log
echo "--- what YOUR shell has ---"
echo "$PATH"
```

Compare those last two outputs side by side. That gap is the single most common
reason a cron job "doesn't run" — and now you've seen it rather than been told
it.

### 10. Clean up

```bash
crontab -r
crontab -l 2>/dev/null || echo "crontab removed"
rm -rf /tmp/day5 ~/cron.bak
```

---

## Gotchas

- **`apt update` ≠ `apt upgrade`.** `update` refreshes the catalogue; `upgrade`
  installs. `update` first, and in the same Docker `RUN` layer.
- **`remove` keeps config, `purge` doesn't.** Reinstalling won't clear a bad
  setting.
- **`apt` in a script prints an instability warning.** Use `apt-get`.
- **`dpkg` doesn't resolve dependencies.** That's `apt`'s job.
- **`apt-key` is deprecated.** Use `signed-by=` with a keyring in
  `/etc/apt/keyrings/`.
- **`start` ≠ `enable`.** Works now, gone after reboot = you forgot `enable`.
- **`After=` is ordering, `Requires=` is dependency.** You usually need both.
- **`daemon-reload` after hand-editing a unit**, or systemd silently runs the
  old config.
- **Never edit `/lib/systemd/system/`** — `apt upgrade` overwrites it. Use
  `systemctl edit`.
- **The journal may be volatile.** No `/var/log/journal/` = logs die at reboot.
- **Cron's `PATH` is nearly empty** and it sources no profile. Absolute paths.
- **Cron output is mailed, and mail usually doesn't exist.** Always `>> log 2>&1`.
- **Unescaped `%` truncates a cron line.** `date +\%F`, not `date +%F`.
- **`/etc/cron.d` lines have a user field; personal crontabs don't.**
- **`run-parts` skips filenames containing a dot.** `backup.sh` in
  `/etc/cron.daily/` never runs.
- **`crontab -r` deletes everything with no confirmation.** Back up first.

---

## Recap

- **`dpkg` installs one `.deb`; `apt` resolves dependencies and calls `dpkg`.**
  `apt` for humans, `apt-get` for scripts.
- **`update` refreshes the catalogue, `upgrade` installs.** Both in one Docker
  layer, or layer caching hands you a stale index.
- **`dpkg -L` lists a package's files; `dpkg -S` finds a file's package** — and
  "no package owns this" tells you a human put it there.
- Repos are `deb URL suite components`, signed with a GPG key scoped by
  `signed-by=`, because installing runs scripts as root. `-security` is the
  pocket you never disable.
- **systemd manages units; `.service` is the one you'll write.** It tracks a
  service as a **cgroup**, so forked children can't escape a stop.
- **`start` is now, `enable` is at boot** — and enable is literally a symlink
  into `multi-user.target.wants`.
- A unit's `Restart=`, `TimeoutStopSec=` and `KillSignal=` are yesterday's
  signals turned into configuration. **`systemctl stop` = SIGTERM, wait,
  SIGKILL.**
- **`journalctl -u <unit>`** replaces log files; `-f`, `-e`, `--since`, `-p err`
  and `-b -1` are the flags that matter. Check persistence before you need it.
- **Every cron failure is the same failure: cron is not your shell.** Empty
  `PATH`, discarded output, `%` truncation, `/bin/sh`.
- **Prefer a systemd timer** on a server you control — journal logging,
  `Persistent=true`, jitter.

---

## Say it out loud

Answer each in about 30 seconds, using the real terms.

1. Someone asks "what's the difference between `apt` and `dpkg`?" Answer without
   using the word "wrapper".
2. Why must `apt-get update` and `apt-get install` be in the same Dockerfile
   `RUN`? Name the specific failure.
3. A service is `active (running)` but `disabled`. What happens at reboot, and
   what would you run to fix it?
4. Explain `After=` versus `Requires=` and give an example where getting it
   wrong causes a crash loop.
5. Your app takes 60s to shut down and the unit says `TimeoutStopSec=30`. Walk
   through what happens, including the exit code.
6. A cron job's log is empty but `journalctl -u cron` shows it ran every minute.
   Name two causes and how you'd tell them apart.
7. Why would you choose a systemd timer over cron on a production server?

---

## Week 1 Review

Four days, one arc: **where things live → who may touch them → what's running →
how it got there and how it stays there.**

| Day | Core idea | The one thing to carry forward |
|-----|-----------|-------------------------------|
| 1 — Filesystem | One tree from `/`, no drive letters; `/etc` config, `/var` state, `/usr` programs, `/proc` kernel truth | The FHS is why a container image looks the way it does |
| 2 — Navigation | `find` walks, `grep` reads; absolute vs relative paths; globs are the shell's job | `find -exec` and pipes are how you operate at scale |
| 3 — Permissions | *skipped* | Still owed: `chmod`, `umask`, non-root containers |
| 4 — Processes | fork/exec, PID 1, `RSS` not `VSZ`, load is a count, **`SIGTERM` is catchable and `SIGKILL` isn't** | Exit `137` = SIGKILL, `143` = SIGTERM |
| 5 — Packages & services | `apt` over `dpkg`; systemd units; `start` ≠ `enable`; journal not log files; cron's empty environment | A Spring Boot deploy is a unit file plus a grace period |

**The through-line:** Day 4 gave you the *mechanism* — signals and exit codes.
Day 5 gave you the *policy layer* that drives it — `TimeoutStopSec`, `Restart=`.
Docker in Phase 4 and Kubernetes in Phase 17 are those same two ideas a third
and fourth time, with different config key names. **You are not going to learn
graceful shutdown four times.** You learned it on Day 4; today you saw the
second dialect.

**Self-check — the five that matter most:**

1. Why must `apt-get update` and `apt-get install` share one Dockerfile `RUN`?
2. A service is `active (running)` but `disabled`. What happens at reboot?
3. Your app's shutdown takes 60s and `TimeoutStopSec=30`. What exit code, and
   what breaks?
4. A cron job's log file is empty, but `journalctl -u cron` shows it ran every
   minute. Name two causes.
5. `dpkg -S /usr/local/bin/kubectl` reports no matching package. What does that
   tell you about the machine?

**Next (Day 6):** `grep` and regular expressions — the start of Week 2, text
processing. This is where the shell stops being navigation and becomes a data
tool.

> **End of Week 1.** Ask for a quiz if you want to test yourself before Day 6.

---

## Q&A
