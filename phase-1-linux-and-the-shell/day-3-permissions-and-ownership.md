# Day 3 — Permissions & Ownership

_Why this matters:_ Permissions are the reason your deploy script "works on my
machine" and dies in the pipeline, the reason a container can't write its own
log, and the reason a `chmod 777` shows up in a git diff and a security reviewer
blocks the PR. This is also the first day where the **kernel**, not the shell,
is the thing enforcing the rule — and knowing where the check happens is what
lets you debug it.

---

## The one-paragraph version

Every file on Linux records three things: a number for **who owns it**, a number
for **which group it belongs to**, and nine on/off switches — read, write and
execute, for three audiences: the owner, the group, and everyone else. When you
try to touch a file, the kernel picks **exactly one** of those three sets of
switches — the first one that describes you — and that's the answer. Not the
friendliest set: the first. So you can own a file and be locked out of it while
the whole world can edit it. The second big idea is that those same three
switches mean something **completely different on a directory**, because a
directory isn't a container of files — it's a lookup table of names. So "read"
on a directory means you can see the *names*, "write" means you can *delete
things inside it* (regardless of who owns them), and "execute" means you're
allowed to *follow a name through it* to reach what's beyond. Almost every
confusing permission error in your career is one of those two ideas.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **inode** | The record holding a file's real metadata and data pointers | On-disk structure with owner, group, mode, timestamps and block pointers |
| **UID** | The number identifying a user; names are just a lookup | User ID; the actual value stored and compared by the kernel |
| **GID** | The number identifying a group | Group ID |
| **mode** | The permission switches, written as `rwxr-xr-x` or `755` | 12 bits: 9 permission bits plus setuid, setgid and sticky |
| **triad** | One group of three switches — `rwx` for one audience | One of the user/group/other permission fields |
| **u / g / o / a** | user / group / others / all — used by `chmod` | The `who` operand in symbolic mode |
| **octal** | Base 8 — how `755` encodes nine on/off switches | Each digit encodes one triad's three bits |
| **root** | The all-powerful account | UID 0, for which most kernel permission checks are skipped |
| **`sudo`** | Run one command as another user, usually root | Setuid helper that consults `/etc/sudoers` |
| **syscall** | How a program asks the kernel to do something | The controlled entry point into kernel code |
| **`execve()`** | The syscall that runs a program | Replaces the calling process's image with a new program |
| **traverse / search bit** | `x` on a directory — permission to pass *through* it | Permission to resolve a name within the directory to its inode |
| **umask** | The switches automatically removed from every new file | A mask of permission bits cleared at creation time |
| **setuid** | A program that runs as its owner, not as you | Mode bit 4000; `execve()` sets the effective UID to the file's owner |
| **setgid** | On a directory: new files inherit the directory's group | Mode bit 2000 |
| **sticky bit** | In a shared folder, you can only delete your own stuff | Mode bit 1000; restricts unlink/rename to the entry's or directory's owner |
| **capabilities** | Root's powers split into ~40 separate, grantable pieces | Per-thread privilege units replacing all-or-nothing root |
| **service account** | An identity that exists to run software, not a person | A non-login account, typically with `/usr/sbin/nologin` as its shell |
| **shebang** | The `#!/bin/bash` first line naming the interpreter | The two-byte magic `#!` plus interpreter path, read by the kernel |
| **CRLF** | Windows line endings — an invisible extra character per line | Carriage-return + line-feed, versus Unix's bare line-feed |

---

## Before you read: two questions

1. A file is `-r--rw-rw-`. You are the **owner**, and you are also in the
   group. The group bits say `rw-`. **Can you write to it?**
2. A directory is `dr--------`. `ls` on it prints the filenames just fine. But
   `cat dir/file.txt` says `Permission denied`, and so does `cd dir`.
   **How can you see a name you're not allowed to look at?**

Both answers are counter-intuitive, both are today's core mechanics, and you'll
run both yourself at the bottom.

---

## Part 1 — What a permission actually is

**In plain words:** Attached to every file is a tiny record — nothing to do with
its contents — holding *who owns it*, *what group it's in*, and *nine on/off
switches*. That's it. There's no permission system beyond those, no access
control lists by default, no per-user grants. Nine switches and two numbers.

Every file and directory has an **inode** — the filesystem's metadata record for
it. Three parts matter today:

| Stored on the inode | Example |
|---|---|
| **Owner UID** — a number, not a name | `1000` |
| **Group GID** — a number, not a name | `1000` |
| **Mode** — 12 permission bits | `0644` |

**Names never appear on disk.** `ls` prints `shailesh_parigi` because it looks
up UID `1000` in `/etc/passwd` and turns it into a string for your benefit. The
kernel only ever compares numbers.

That matters more than it sounds. Copy a file to another machine where UID 1000
is a *different human*, and the file now belongs to that human — no conversion
happens, because there's nothing to convert. **This is exactly the Docker volume
and NFS "wrong owner" bug:** a UID that means one thing inside the container and
something else on the host.

The check itself happens **inside the kernel**, in the `open()` / `execve()` /
`unlink()` syscall — **not** in `cat`, `bash`, or your JVM. That's why no amount
of reading the application source explains a `Permission denied`: the program
asked, the kernel said no, and the program printed exactly what it was told.

---

## Part 2 — Reading the `ls -l` line

**In plain words:** That wall of letters at the start of an `ls -l` line is
actually four separate fields jammed together with no spaces: one character for
*what kind of thing this is*, then three groups of three switches.

```bash
ls -l /usr/bin/passwd
```

```
-rwsr-xr-x 1 root root 59976 Feb  6  2024 /usr/bin/passwd
│└┬┘└┬┘└┬┘   └─┬┘ └─┬┘
│ │  │  │      │    └── group name (GID → /etc/group)
│ │  │  │      └─────── owner name (UID → /etc/passwd)
│ │  │  └────────────── permissions for OTHERS  (everyone else)
│ │  └───────────────── permissions for GROUP   (members of the file's group)
│ └──────────────────── permissions for USER    (the owner)
└────────────────────── file type
```

**File type character** (position 1 — this is a type, not a permission, and it's
the field people accidentally read as part of the owner's bits):

| Char | Type |
|---|---|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symlink |
| `c` / `b` | Character / block device (e.g. `/dev/null`, `/dev/sda`) |
| `s` / `p` | Socket / named pipe |

**Three triads, three audiences.** Read them as *user / group / others* — often
abbreviated **u / g / o**, with **a** meaning all three. Those letters are
literally what you type into `chmod`.

---

## Part 3 — The kernel picks exactly ONE triad

**In plain words:** This is the bit that surprises everyone. The three triads
are not added together. The kernel asks three questions in order — *are you the
owner? are you in the group? otherwise* — and **stops at the first yes**. It
then uses only that triad and ignores the others completely, even if a later one
would have given you more.

This answers question 1. The decision, in order:

```
if (your UID == file's owner UID)      -> use the USER bits.  Done.
else if (file's GID is in your groups) -> use the GROUP bits. Done.
else                                   -> use the OTHER bits. Done.
```

It is a **first-match**, not a union. The most *specific* triad wins, even when
a later triad is *more permissive*.

So a file that is `-r--rw-rw-`:

- **You, the owner:** matched on the first line → you get `r--`. **You cannot
  write to your own file.**
- Anyone in the group → `rw-`.
- Anyone else on the box → `rw-`.

You are the only person on the entire machine who can't write it. You'll run
this in a minute and see `Permission denied` on a file you own while the world
can edit it.

The design reason: it means you can deliberately restrict yourself — a
protection against your own typos and scripts — and it means "more specific
beats more general", which is the same rule you'll meet in firewall rules and
Kubernetes RBAC.

**Root ignores all of this.** UID 0 bypasses the permission check entirely.
That's *all* "root" means — not a special group, not a flag, just UID 0 with the
kernel checks short-circuited. Which is why `sudo` fixes everything, and why
"just run it as root" is a security smell rather than a solution.

> **Say this in an interview:** "The kernel evaluates the triads first-match, not
> cumulatively — owner, then group, then other — and stops at the first one that
> applies. So an owner with `r--` can't write a file even if the group bits say
> `rw-` and they're in that group. Root is just UID 0, for which the check is
> skipped entirely."

---

## Part 4 — What r / w / x mean on a **file**

| Bit | On a regular file |
|---|---|
| `r` | You may **read the contents** |
| `w` | You may **modify the contents** |
| `x` | You may **execute it** as a program |

These are the obvious ones. But note what `w` on a file does **not** cover:
**deleting the file.** Deleting isn't an operation on the file at all — it's
removing a name from a directory's table. Which brings us to the important half
of today.

---

## Part 5 — What r / w / x mean on a **directory**

**In plain words:** A directory is not a box holding files. It's a **lookup
table**: a list of names, each paired with a pointer to where the real thing
lives. Once you picture that table, the three bits stop being strange — they're
just permissions *on the table*.

| Bit | On a directory | Plain version |
|---|---|---|
| `r` | You may **read the list of names** | you can `ls` it |
| `w` | You may **add/remove/rename rows** | you can create and delete files inside |
| `x` | You may **use a name to reach what it points at** | you can `cd` into it and pass *through* it |

`x` on a directory is called the **search** or **traverse** bit, and it's the
one nobody guesses. It has nothing whatsoever to do with running programs.

Three consequences, all of which bite in production:

**1. `r` without `x` = names but no access.** You can read the table, so you get
the filenames — but you can't follow any of them to the actual file:

```
$ chmod 400 dir
$ ls dir
file.txt                        <- the name is visible
$ cat dir/file.txt
cat: dir/file.txt: Permission denied
$ cd dir
bash: cd: dir: Permission denied
```

That's question 2, and the answer is that a name and its contents live in two
different places. The **name** is a row in the *directory's* table, which you're
allowed to read. The **contents** live in the file's own inode, which you can't
reach without traverse permission on the directory in the path.

**2. `x` without `r` = access if you already know the name.** The classic
"drop-box" pattern — you can put things in and retrieve what you know about, but
you can't browse:

```
$ chmod 100 dir
$ ls dir
ls: cannot open directory 'dir': Permission denied
$ cat dir/file.txt
secret                          <- works! you guessed the name
```

**3. Deleting a file needs `w` on the *directory*, not on the file.** Because
deleting means removing a row from the table. The file's own permissions are
irrelevant:

```
$ chmod 555 wd                  # directory: read + traverse, no write
$ rm wd/a.txt
rm: cannot remove 'wd/a.txt': Permission denied
```

And the inverse — a read-only file in a writable directory deletes fine (`rm`
just asks for confirmation, and `rm -f` doesn't even do that). **You can delete
a file you cannot read.** Sit with that one; it's the reason `/tmp` needs a
special bit, which you'll meet in Part 10.

> **Every directory in the path needs `x`.** Opening
> `/home/shailesh_parigi/app/config.yml` requires traverse on `/`, `/home`,
> `/home/shailesh_parigi`, and `.../app`. One missing `x` five levels up
> produces a `Permission denied` that *looks* like it's about the file — and you
> can waste an hour chmod-ing the file. `namei -l /path/to/file` prints the
> permissions of every component at once and is the fastest way to find the
> guilty directory.

> **Say this in an interview:** "A directory is a name-to-inode table, so the
> bits apply to the table: `r` lists names, `w` adds and removes entries, and
> `x` is traverse — permission to resolve a name through it. That's why deleting
> a file depends on write permission on the *directory*, not the file, and why
> reading a file requires `x` on every directory in its path. `namei -l` shows
> you which component is actually blocking."

---

## Part 6 — `chmod`: numeric and symbolic

### Numeric (octal)

**In plain words:** Three switches can be written as one digit if you give each
switch a value and add them up: read is 4, write is 2, execute is 1. So `rwx` =
7, `rw-` = 6, `r-x` = 5. Three triads, three digits — that's what `755` means.

| | r | w | x |
|---|---|---|---|
| value | 4 | 2 | 1 |

| Octal | Bits | Meaning |
|---|---|---|
| `7` | `rwx` | 4+2+1 |
| `6` | `rw-` | 4+2 |
| `5` | `r-x` | 4+1 |
| `4` | `r--` | 4 |
| `0` | `---` | none |

The ones you'll actually type:

| Mode | Use |
|---|---|
| `644` | Normal file: owner edits, everyone reads |
| `600` | Private file — **secrets, `.env`, private keys** |
| `755` | Script or directory: owner writes, everyone runs/traverses |
| `700` | Private directory |
| `777` | **Never.** World-writable is a finding, not a fix |

```bash
chmod 644 notes.txt
chmod 755 deploy.sh
chmod 600 ~/.ssh/id_ed25519
```

Numeric mode is **absolute** — it sets all nine bits at once, wiping whatever
was there before. That's fine when you know exactly what you want and dangerous
when you don't.

### Symbolic

**In plain words:** Instead of stating the final answer, you describe the
*change*: who, what operation, which bits. `u+x` = "give the user execute,
leave everything else alone."

`[who][op][perm]`, where *who* is `u`/`g`/`o`/`a`, and *op* is `+` (add),
`-` (remove), `=` (set exactly).

```bash
chmod u+x   deploy.sh    # owner can execute
chmod go-w  shared.txt   # take write away from group and others
chmod a+r   readme.md    # everyone can read
chmod u=rw,go=r file     # set precisely: 644
chmod -R g+rX project/   # recurse; capital X = x only on dirs (and things
                         # already executable) — the flag that saves you from
                         # marking every .java file executable
```

That **capital `X`** deserves a pause. `chmod -R 755` or `-R a+x` on a source
tree marks every single `.java`, `.md` and `.png` executable, which is noise in
git and a smell in review. Capital `X` means "execute, but only on directories
and on files that already had it" — which is almost always what you actually
meant.

Symbolic is **relative** — it edits only the bits you name, which makes it the
safer choice in scripts where you don't want to clobber the rest.

> **`chmod +x` with no `who` is filtered by your umask.** With `umask 022` it
> behaves like `a+x`; with `umask 077` it silently only does `u+x`. If you mean
> everyone, write `a+x` explicitly.

---

## Part 7 — `chown` and `chgrp`

**In plain words:** `chmod` changes the switches. `chown` changes *who the file
belongs to*. And there's a rule that catches people: you can't give your files
away, even though they're yours.

```bash
sudo chown appuser file          # change owner
sudo chown appuser:appgroup file # owner and group
sudo chgrp docker file           # group only
chgrp docker file                # allowed without sudo IF you own the file
                                 # AND you're a member of the target group
sudo chown -R appuser:appgroup /opt/myapp
```

**Only root can give a file away.** A non-root user can't `chown` a file to
someone else even if they own it. Two reasons, both practical: you could dodge
a disk quota by donating your files to someone else, and you could plant a
setuid binary owned by another user and trick them into running it.

`ls -n` shows raw numeric UID/GID instead of names — reach for it when a name
doesn't resolve (e.g. a container user that doesn't exist in your
`/etc/passwd`), because seeing a bare number *is* the diagnosis.

---

## Part 8 — `umask`: where default permissions come from

**In plain words:** Nobody sets new files to `644` explicitly — so where does
that come from? Two steps: the program asks for generous permissions, and the
system automatically **subtracts** a fixed set of bits called the umask. It's a
mask of things to *remove*, which is why the number looks backwards.

1. The program asks for a mode. `touch` and editors ask for `666`; `mkdir` asks
   for `777`. **The kernel never grants `x` on a new regular file created via
   `open()`** — deliberately, so a downloaded file can't be immediately
   runnable. That's why `666` is the ceiling for files, not `777`.
2. The kernel subtracts your **umask**.

```
File:      666 - 022 = 644   (-rw-r--r--)
Directory: 777 - 022 = 755   (drwxr-xr-x)
```

Your current umask is `0022`, the Ubuntu default. Change it and watch:

```
$ umask 077
$ touch f2; mkdir m2
-rw-------  f2
drwx------  m2
```

`umask 077` = "remove everything for group and others" — the right setting for a
shell that handles credentials. Set it in `~/.bashrc` to make it stick; a bare
`umask 077` applies only to the current shell and its children.

**umask only applies at creation.** It never changes an existing file, which is
why it can't be your security control — it's a *default*, not a policy.

---

## Part 9 — Users, groups, root and `sudo`

```bash
id                     # your UID, GID and every group you're in
whoami
groups
getent passwd shailesh_parigi
getent group sudo
```

On your box:

```
uid=1000(shailesh_parigi) gid=1000(shailesh_parigi)
groups=1000(shailesh_parigi),27(sudo),1001(docker),...
```

**`/etc/passwd`** — one line per account, world-readable (the name is a
historical lie; passwords moved out decades ago):

```
shailesh_parigi:x:1000:1000:,,,:/home/shailesh_parigi:/bin/bash
    name        │  UID  GID  gecos      home           login shell
                └── 'x' = password hash lives in /etc/shadow (root-only, 0640)
```

A login shell of `/usr/sbin/nologin` is how **service accounts** are stopped
from logging in — the account exists so something can *run as* it, but there's
no way to get a shell. You'll meet that exact idea again in Phase 8 as a service
account in Spring Security: an identity that exists to run software, not to be a
person.

**`/etc/group`** maps group names to GIDs and lists supplementary members.

**`sudo`** runs one command as another user (root by default) after checking
`/etc/sudoers`. On Ubuntu, membership in the `sudo` group is what grants it. Two
things worth knowing now, because both waste an afternoon:

- **Group membership is fixed at login.** `sudo usermod -aG docker $USER`
  edits `/etc/group` but does **not** affect your current shell — your process
  inherited its group list when it started and nothing re-reads it. `id` won't
  show the new group until you start a fresh session (`exec su - $USER`, or in
  WSL: `wsl --terminate Ubuntu-22.04` from PowerShell). "I added myself to the
  docker group and still get permission denied on the socket" is this,
  essentially always.
- **`sudo` resets the environment.** `sudo cmd` doesn't see your `$PATH`
  additions or your exported variables — deliberately, so you can't trick root
  into running your own binary. `sudo -E` preserves the environment; `sudo -i`
  gives you root's full login environment.

---

## Part 10 — The three extra bits: setuid, setgid, sticky

**In plain words:** There are actually 12 switches, not 9. The extra three sit
above the ones you've seen and change *who a program runs as* or *who can delete
what*. They're why `chmod` sometimes shows four digits (`2775`, `1777`).

### setuid (`4000`) — run as the file's owner

```
-rwsr-xr-x 1 root root /usr/bin/passwd
   ↑ 's' where the owner's 'x' would be
```

**The problem it solves:** changing your password means writing `/etc/shadow`,
which only root may write. You are not root. So how does `passwd` work?

Rather than making `/etc/shadow` writable by everyone (catastrophic), the
*binary* is marked **setuid root**: when you execute it, the process runs with
root's privileges regardless of who started it. `sudo` works the same way —
that's how an unprivileged user manages to become root at all.

This is a genuinely dangerous bit. A setuid-root program with a bug is a local
privilege escalation, which is why modern Linux is moving away from it. Look at
`ping` on your machine:

```
-rwxr-xr-x 1 root root /bin/ping      <- no setuid!
$ getcap /bin/ping
/bin/ping cap_net_raw=ep
```

`ping` needs to open raw network sockets — one specific root power, not all of
them. **Capabilities** slice root's authority into roughly 40 separate
privileges and grant only the one needed, so a bug in `ping` gets an attacker
raw sockets rather than the whole machine. Remember this name — it returns in
Phase 17/18 as `securityContext.capabilities` in a Kubernetes pod spec, and
"drop ALL, add only what's needed" is the same principle.

An uppercase `S` (rather than `s`) means setuid is set but `x` is **not** —
almost always a bug.

### setgid (`2000`) — on a directory, inherit the group

On a **directory**, setgid means "every file created in here gets *this
directory's* group, rather than the creator's default group":

```
$ chmod 2775 shared && ls -ld shared
drwxrwsr-x  shared
              ↑ 's' in the group triad
```

This is the standard way to make a shared team directory actually work. Without
it, every person's files land in their own personal group and nobody else can
edit them, so collaboration silently breaks one file at a time. With it (plus
`g+w`) the group can genuinely share. (On a *binary*, setgid means run-as-that-
group — same idea as setuid, much rarer.)

### sticky bit (`1000`) — restricted deletion

```
$ ls -ld /tmp
drwxrwxrwt 11 root root /tmp
         ↑ 't' where other's 'x' would be
```

`/tmp` is `1777`: world-writable, because every program needs scratch space.
But recall Part 5 — **write on a directory means you can delete anything in
it.** Without protection, any user could delete every other user's temp files.
Worse: they could delete one and replace it with their own, moments before a
root process opens it by name.

The sticky bit changes the rule for that directory: **you may only delete an
entry if you own the entry, or you own the directory.** That's its entire
purpose. Any world-writable directory *without* `t` is a security finding.

Setting them:

```bash
chmod 4755 prog     # setuid          (or: chmod u+s prog)
chmod 2775 dir      # setgid          (or: chmod g+s dir)
chmod 1777 dir      # sticky          (or: chmod +t dir)
```

> **Say this in an interview:** "setuid makes a binary run as its owner — that's
> how `passwd` writes `/etc/shadow` without `/etc/shadow` being world-writable.
> It's also a privilege-escalation risk, so modern Linux prefers capabilities,
> which split root into ~40 grantable pieces: `ping` has `cap_net_raw` instead
> of setuid root, and that's the same model as dropping capabilities in a
> Kubernetes securityContext. setgid on a directory makes new files inherit its
> group, which is how shared team directories work. And the sticky bit is what
> stops users deleting each other's files in a world-writable `/tmp`."

---

## Part 11 — The errors you will actually hit

### "Permission denied" running a script

```
$ ./deploy.sh
bash: ./deploy.sh: Permission denied
```

No `x` bit. **Three different fixes**, and the difference between them is the
lesson:

```bash
chmod +x deploy.sh && ./deploy.sh   # 1. grant execute, run it directly
bash deploy.sh                      # 2. no x needed — bash only needs READ.
                                    #    You are running bash; bash is reading
                                    #    a data file. execve() never touches
                                    #    deploy.sh at all.
chmod 755 deploy.sh && ./deploy.sh  # 3. numeric equivalent of #1
```

Fix 2 is the one worth internalising: **the `x` bit is only consulted when the
kernel is asked to execute the file.** `./deploy.sh` asks the kernel to execute
it, so `x` is checked. `bash deploy.sh` executes `/bin/bash` (which has `x`) and
hands it your script as *data to read*. Same file, two completely different
questions.

It's also why `x` on a `.jar` is meaningless — you run `java -jar app.jar`, and
`java` just reads it.

### "Permission denied" that is really about the shebang

```
$ ./script.sh
bash: ./script.sh: /bin/bash^M: bad interpreter: No such file or directory
```

Same-looking failure, completely different cause. The file was saved with
**Windows CRLF line endings**, so the first line isn't `#!/bin/bash` — it's
`#!/bin/bash\r`, and the kernel dutifully goes looking for an interpreter
literally named `/bin/bash^M`, which doesn't exist.

This *will* happen to you on `/mnt/d`. Fix with `dos2unix script.sh`, and
prevent it with a `.gitattributes` containing `*.sh text eol=lf`.

### A JAR that can't write its log

The app runs as `appuser`, the log directory is `drwxr-xr-x root root`. Java
throws:

```
FileNotFoundException: /var/log/myapp/app.log (Permission denied)
```

Note the trap: Java reports this as a **file-not-found** exception even though
the real cause is a **directory** with no `w` for `appuser`. Read the message in
the parentheses, not the class name.

```bash
sudo mkdir -p /var/log/myapp
sudo chown appuser:appuser /var/log/myapp
sudo chmod 755 /var/log/myapp
```

Not `chmod 777`. **The fix for a permission problem is almost always
ownership**, not opening the bits up — `777` fixes your symptom by giving every
process on the box write access to your logs.

### The Docker version of the same bug

```dockerfile
RUN useradd -u 1001 appuser
USER 1001
```

A container running as UID 1001, writing to a bind-mounted host directory owned
by UID 1000, gets `Permission denied`. Remember Part 1: the UID inside the
container and the UID on the host are **the same number space**, and 1001 ≠
1000. There's no translation layer. Phase 4 goes into this properly; today just
learn to recognise the shape.

### Git tracks the execute bit — and only that one

Git stores exactly one permission bit: mode `100644` (not executable) or
`100755` (executable). So `chmod +x` is a real, committable change:

```bash
git update-index --chmod=+x deploy.sh
```

That's why a script committed from Windows arrives non-executable in CI and the
pipeline dies with the exact error from the top of this section. It's also why
`chmod -R 777` shows up as a huge diff of mode changes that a reviewer will
block.

---

## Part 12 — WSL gotcha: `/mnt/d` lies to you

Run this and read carefully:

```
$ ls -ld /mnt/d
drwxrwxrwx 1 shailesh_parigi shailesh_parigi /mnt/d

$ echo hi > /mnt/d/perm-test.tmp
$ chmod 600 /mnt/d/perm-test.tmp
$ ls -l /mnt/d/perm-test.tmp
-rwxrwxrwx 1 shailesh_parigi shailesh_parigi /mnt/d/perm-test.tmp
```

**`chmod` reported no error and did nothing.** DrvFs — the driver that exposes
your Windows drives — doesn't store Linux mode bits unless the `metadata` mount
option is enabled. So everything reports a synthetic `0777` and every `chmod` is
silently discarded. No error, no warning, no effect.

Consequences you care about:

- **Everything on `/mnt/c` and `/mnt/d` appears world-writable** — you cannot
  practise permissions there, and today's exercises would all silently "pass"
  while teaching you nothing.
- `~/.ssh` on `/mnt/*` is unusable: `ssh` refuses to use a private key that
  isn't `600`, and you can't make it `600`.
- Combined with Day 2's finding that `/mnt/d` I/O is slow: **do Linux work in
  `~`.** Use `/mnt/d` for files Windows tools need to open.

All of today's exercises therefore run under `/tmp`.

---

## Hands-on

Ubuntu (WSL) tab. Everything lands in `/tmp/day3`.

### 1. Setup

```bash
rm -rf /tmp/day3 && mkdir -p /tmp/day3 && cd /tmp/day3
id
umask
echo "home dir:"; ls -ld ~
```

### 2. The owner triad wins, even when it's the worst one

```bash
cd /tmp/day3
echo "original" > og.txt
chmod 466 og.txt            # owner r--, group rw-, other rw-
ls -l og.txt
echo "--- you own it, group and world can write it, now you try: ---"
echo "more" >> og.txt 2>&1 || echo "^ denied: first match wins, and you matched USER"
echo "--- but you can still chmod it, because you're the OWNER ---"
chmod 644 og.txt && ls -l og.txt
```

That last step matters: **ownership and mode are separate powers.** The mode
says what you may do with the *contents*; the right to run `chmod`/`chown` comes
from *being the owner*. You can always give yourself permission back — which is
why `chmod` is not a security boundary against a file's own owner.

### 3. Directory `r` vs `x` — the two surprises

```bash
cd /tmp/day3
mkdir -p dir && echo "secret" > dir/file.txt

echo "=== mode 400 (r--): names visible, contents unreachable ==="
chmod 400 dir
ls dir            2>&1
cd dir            2>&1 || echo "cd denied"
cat dir/file.txt  2>&1

echo "=== mode 100 (--x): can't list, but a known name works ==="
chmod 100 dir
ls dir            2>&1
cat dir/file.txt  2>&1

chmod 700 dir
```

This is question 2, running in front of you. Watch the name appear in the first
block and the contents appear in the second.

### 4. Deletion is a directory permission

```bash
cd /tmp/day3
mkdir -p wd && echo a > wd/a.txt

chmod 555 wd                       # no write on the directory
rm wd/a.txt 2>&1 || echo "^ can't delete: no w on the DIRECTORY"

chmod 755 wd
chmod 444 wd/a.txt                 # file itself read-only
rm -f wd/a.txt && echo "deleted a read-only file — the directory decided"
```

### 5. Find the guilty directory in a path

```bash
cd /tmp/day3
mkdir -p a/b/c && echo data > a/b/c/target.txt
chmod 600 a/b                      # remove traverse halfway down
cat a/b/c/target.txt 2>&1
echo "--- namei shows exactly which component is blocking ---"
namei -l /tmp/day3/a/b/c/target.txt
chmod 755 a/b
```

Note the error message names `target.txt` while the actual problem is `b`. This
is the hour-waster, and `namei -l` is the cure.

### 6. Practice — a script that won't run, fixed three ways

```bash
cd /tmp/day3
printf '#!/bin/bash\necho "the script ran"\n' > s.sh
chmod 644 s.sh && ls -l s.sh

echo "--- attempt 1: direct ---"
./s.sh 2>&1

echo "--- fix A: bash reads it (only needs r) ---"
bash s.sh

echo "--- fix B: symbolic ---"
chmod u+x s.sh && ./s.sh && ls -l s.sh

echo "--- fix C: numeric, from scratch ---"
chmod 644 s.sh && chmod 755 s.sh && ./s.sh && ls -l s.sh
```

### 7. Practice — umask

```bash
cd /tmp/day3
umask
touch default.txt && mkdir -p default.d
ls -l default.txt; ls -ld default.d

echo "--- subshell with umask 077 ---"
( umask 077
  touch private.txt && mkdir -p private.d
  ls -l private.txt; ls -ld private.d )

echo "--- and note chmod +x is masked too ---"
( umask 077; touch masked.sh; chmod +x masked.sh; ls -l masked.sh )
echo "^ only u+x, not a+x"
```

The parentheses run those commands in a **subshell** — a child shell whose
`umask` change dies with it, so your main shell is unaffected.

### 8. The special bits, live

```bash
echo "=== setuid: how you're allowed to change your own password ==="
ls -l /usr/bin/passwd /usr/bin/sudo

echo "=== capabilities: the modern replacement ==="
ls -l /bin/ping
getcap /bin/ping

echo "=== sticky bit on /tmp ==="
ls -ld /tmp
rmdir /tmp/.font-unix 2>&1 || echo "^ blocked by the sticky bit: root owns that entry, not you"

echo "=== setgid directory: files inherit the directory's group ==="
cd /tmp/day3 && mkdir -p sg && chmod 2775 sg
ls -ld sg
touch sg/inherited.txt && ls -l sg/inherited.txt
stat -c '%a %A %U %G %n' sg sg/inherited.txt
```

### 9. Prove `/mnt/d` ignores chmod

```bash
echo hi > /mnt/d/perm-test.tmp
chmod 600 /mnt/d/perm-test.tmp
ls -l /mnt/d/perm-test.tmp
echo "^ still rwxrwxrwx — chmod succeeded and changed nothing"
rm -f /mnt/d/perm-test.tmp
```

### 10. Cleanup

```bash
rm -rf /tmp/day3 && echo "clean"
```

---

## Gotchas

- **The triads are first-match, not cumulative.** Owner `r--` beats group `rw-`
  for the owner. Check the *first* triad that applies to you, not the friendliest.
- **`x` on a directory is traverse, not execute.** Every directory in a path
  needs it. `namei -l` finds the one that's missing.
- **`w` on a directory grants deletion of any file inside**, regardless of the
  file's own mode. The sticky bit (`t`) is the only thing that restrains it.
- **`chmod -R 755` on a source tree marks every `.java` file executable.** Use
  `chmod -R g+rX` — capital `X` only touches directories.
- **`chmod +x` without a `who` is filtered by umask.** Write `a+x` if you mean it.
- **`umask` subtracts, and only at creation time.** It never fixes an existing file.
- **Group changes need a new login session.** `id` is your source of truth, not
  `/etc/group`.
- **`chmod 777` is never the fix.** The fix is `chown` to the right user.
- **`/mnt/c` and `/mnt/d` silently ignore `chmod`** — permissions practice and
  SSH keys must live in `~`.
- **CRLF line endings produce a "Permission denied"-shaped error** about
  `/bin/bash^M`. `dos2unix`, plus `*.sh text eol=lf` in `.gitattributes`.
- **Only root can give a file away.** `chown` to another user needs `sudo`.

---

## Recap

- A file carries **UID, GID and 12 mode bits** on its inode. Names are a lookup
  for humans; **UIDs are what actually match** — which is the whole Docker
  volume ownership problem.
- The kernel picks **exactly one triad**, first match: owner, else group, else
  other. Root (UID 0) skips the check entirely.
- On a **file**: `r` read, `w` modify, `x` execute.
- On a **directory**: `r` list names, `w` add/remove entries, **`x` traverse**.
  Reading a file needs `x` on *every* directory in its path.
- **Deleting is a directory-write operation**, which is why `/tmp` needs the
  **sticky bit** to stop users from deleting each other's files.
- **`chmod`** numeric = absolute, symbolic = relative; `-R` wants capital `X`.
  **`chown`** needs root to give a file away.
- **`umask` (yours: `022`)** subtracts bits at creation: `666→644` for files,
  `777→755` for directories. It's a default, not a policy.
- **setuid** (`passwd`, `sudo`) runs as the owner; **capabilities** are the
  modern, narrower replacement and reappear in Kubernetes security contexts.
- The `x` bit is only checked by `execve()` — which is why `bash script.sh`
  works without it, and why it means nothing on a `.jar`.

---

## Say it out loud

Answer each in about 30 seconds, using the real terms.

1. A file is `-r--rw-rw-` and you own it. Can you write to it? Explain the rule,
   not just the answer.
2. What does `x` mean on a directory? Give a concrete example of it biting you.
3. Why can you delete a file you have no read permission on?
4. Explain why `./script.sh` fails but `bash script.sh` works on the same file.
5. Someone fixes a logging failure with `chmod 777 /var/log/myapp`. What would
   you say in the code review, and what's the right fix?
6. What problem does setuid solve, and why is the industry moving to
   capabilities instead?
7. Why does `/tmp` have a `t` at the end of its permissions?

---

**Tomorrow (Day 4):** processes, signals and jobs — PIDs and the process tree,
reading `ps aux` column by column, and `SIGTERM` vs `SIGKILL`, which is exactly
what decides whether your Spring Boot app shuts down gracefully or gets its
connections cut mid-request.

---

## Q&A
