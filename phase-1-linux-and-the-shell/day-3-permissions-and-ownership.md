# Day 3 — Permissions & Ownership

_Why this matters:_ Permissions are the reason your deploy script "works on my
machine" and dies in the pipeline, the reason a container can't write its own
log, and the reason a `chmod 777` shows up in a git diff and a security reviewer
blocks the PR. This is also the first day where the **kernel**, not the shell,
is the thing enforcing the rule — and knowing where the check happens is what
lets you debug it.

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

Every file and directory on disk has an **inode** — a small record the
filesystem keeps holding the metadata: size, timestamps, and three things that
matter today:

| Stored on the inode | Example |
|---|---|
| **Owner UID** — a number, not a name | `1000` |
| **Group GID** — a number, not a name | `1000` |
| **Mode** — 12 permission bits | `0644` |

Names never appear on disk. `ls` prints `shailesh_parigi` because it looks up
UID `1000` in `/etc/passwd` and turns it into a string for your benefit. This
matters more than it sounds: copy a file to another machine where UID 1000 is a
different human, and the file now belongs to that human. **This is exactly what
happens with Docker volumes and NFS mounts** — the "wrong owner" bug is a UID
that means something different on the other side.

The check itself happens inside the kernel, in the `open()` / `execve()` /
`unlink()` syscall — **not** in `cat`, `bash`, or your JVM. Which is why no
amount of reading the application code explains a `Permission denied`: the
program asked, the kernel said no, the program printed what it was told.

---

## Part 2 — Reading the `ls -l` line

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

**File type character** (position 1 — this is a type, not a permission):

| Char | Type |
|---|---|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symlink |
| `c` / `b` | Character / block device (e.g. `/dev/null`, `/dev/sda`) |
| `s` / `p` | Socket / named pipe |

**Three triads, three audiences.** Read them as *user / group / others* — often
abbreviated **u / g / o**, with **a** meaning all three.

---

## Part 3 — The kernel picks exactly ONE triad

Here is the part that surprises people, and it answers question 1.

When you touch a file, the kernel runs this decision, **in order, and stops at
the first match**:

```
if (your UID == file's owner UID)      -> use the USER bits.  Done.
else if (file's GID is in your groups) -> use the GROUP bits. Done.
else                                   -> use the OTHER bits. Done.
```

It is a **first-match**, not a union. The most specific triad wins, even when a
later triad is *more* permissive.

So a file that is `-r--rw-rw-`:

- **You, the owner:** matched on the first line → you get `r--`. **You cannot
  write to your own file.**
- Anyone in the group → `rw-`.
- Anyone else on the box → `rw-`.

You are the only person who can't write it. You'll run this in a minute and see
`Permission denied` on a file you own, with the world able to edit it.

**Root ignores all of this.** UID 0 bypasses the permission check entirely
(that's what "root" means — not a special group, just UID 0 with kernel checks
short-circuited). This is why `sudo` fixes everything and why "just run it as
root" is a security smell rather than a solution.

---

## Part 4 — What r / w / x mean on a **file**

| Bit | On a regular file |
|---|---|
| `r` | You may **read the contents** |
| `w` | You may **modify the contents** |
| `x` | You may **execute it** as a program |

Note what `w` on a file does *not* cover: **deleting the file**. Deletion isn't
an operation on the file — it's removing a name from a directory. Which brings
us to the important half of today.

---

## Part 5 — What r / w / x mean on a **directory**

A directory is just a table mapping **names → inode numbers**. Once you see it
that way, the bits stop being weird:

| Bit | On a directory | Plain version |
|---|---|---|
| `r` | You may **list the names** in the table | you can `ls` it |
| `w` | You may **add/remove/rename entries** in the table | you can create and delete files inside |
| `x` | You may **use a name to reach its inode** | you can `cd` into it and traverse *through* it |

`x` on a directory is called the **search** or **traverse** bit, and it is the
one nobody guesses. It has nothing to do with running programs.

Three consequences that all bite in production:

**1. `r` without `x` = names but no access.** You can read the table, so you get
the filenames, but you can't follow any of them to an inode:

```
$ chmod 400 dir
$ ls dir
file.txt                        <- the name is visible
$ cat dir/file.txt
cat: dir/file.txt: Permission denied
$ cd dir
bash: cd: dir: Permission denied
```

That's question 2. The name lives in the *directory's* table, which you can
read. The **contents** live in the file's inode, which you can't reach without
traverse permission on the directory in the path.

**2. `x` without `r` = access if you already know the name.** The classic
"drop-box" pattern:

```
$ chmod 100 dir
$ ls dir
ls: cannot open directory 'dir': Permission denied
$ cat dir/file.txt
secret                          <- works! you guessed the name
```

**3. Deleting a file needs `w` on the *directory*, not on the file.** The file's
own permissions are irrelevant:

```
$ chmod 555 wd                  # directory: read + traverse, no write
$ rm wd/a.txt
rm: cannot remove 'wd/a.txt': Permission denied
```

And the inverse — a read-only file in a writable directory deletes fine (`rm`
just asks for confirmation, and `rm -f` doesn't even do that). **You can delete
a file you cannot read.**

> **Every directory in the path needs `x`.** Opening
> `/home/shailesh_parigi/app/config.yml` requires traverse on `/`, `/home`,
> `/home/shailesh_parigi`, and `.../app`. One missing `x` five levels up produces
> a `Permission denied` that looks like it's about the file. `namei -l /path/to/file`
> prints the permissions of every component at once and is the fastest way to
> find the guilty directory.

---

## Part 6 — `chmod`: numeric and symbolic

### Numeric (octal)

Each triad is 3 bits, so it's one octal digit:

| | r | w | x |
|---|---|---|---|
| value | 4 | 2 | 1 |

Add them up per triad:

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

Numeric mode is **absolute** — it sets all nine bits, wiping whatever was there.

### Symbolic

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

`-R` with lowercase `x` on a source tree is a real and common mistake. Use `X`.

Symbolic is **relative** — it edits the bits that are there, which makes it the
safer choice in scripts where you don't want to clobber the rest.

> **`chmod +x` with no `who` is filtered by your umask.** With `umask 022` it
> behaves like `a+x`; with `umask 077` it silently only does `u+x`. If you mean
> everyone, write `a+x`.

---

## Part 7 — `chown` and `chgrp`

```bash
sudo chown appuser file          # change owner
sudo chown appuser:appgroup file # owner and group
sudo chgrp docker file           # group only
chgrp docker file                # allowed without sudo IF you own the file
                                 # AND you're a member of the target group
sudo chown -R appuser:appgroup /opt/myapp
```

**Only root can give a file away.** A non-root user can't `chown` a file to
someone else even if they own it — otherwise you could dodge disk quotas by
donating your files, and you could plant a setuid binary owned by another user.

`ls -n` shows raw numeric UID/GID instead of names — use it when a name doesn't
resolve (a container user that doesn't exist in your `/etc/passwd`), which is
itself the diagnosis.

---

## Part 8 — `umask`: where default permissions come from

Nothing sets a new file to `644` explicitly. What happens is:

1. The program asks for a mode. `touch` and editors ask for `666`; `mkdir` asks
   for `777`. **The kernel never grants `x` on a new regular file created via
   `open()`** — that's why `666` is the ceiling for files.
2. The kernel subtracts your **umask** — a *mask of bits to remove*.

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

`umask 077` = "nothing for group or others" — the right setting for a shell that
handles credentials. Set it in `~/.bashrc` to make it stick; a bare `umask 077`
only applies to the current shell and its children.

**umask only applies at creation.** It never changes an existing file, which is
why it can't be your security control — it's a default, not a policy.

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

A login shell of `/usr/sbin/nologin` is how service accounts are stopped from
logging in — you'll meet the same idea again in Phase 8 as a *service account*
in Spring Security: an identity that exists to run something, not to be a person.

**`/etc/group`** maps group names to GIDs and lists supplementary members.

**`sudo`** runs one command as another user (root by default) after checking
`/etc/sudoers` — membership in the `sudo` group is what grants it on Ubuntu.
Two things worth knowing now:

- **Group membership is fixed at login.** `sudo usermod -aG docker $USER`
  doesn't affect your current shell — `id` won't show it until you start a new
  session (`exec su - $USER`, or in WSL: `wsl --terminate Ubuntu-22.04` from
  PowerShell). "I added myself to the docker group and still get permission
  denied on the socket" is this, essentially always.
- **`sudo` resets the environment.** `sudo cmd` doesn't see your `$PATH`
  additions or your exported vars. `sudo -E` preserves the environment;
  `sudo -i` gives you root's full login environment.

---

## Part 10 — The three extra bits: setuid, setgid, sticky

There are 12 mode bits, not 9. The high three are why `chmod` sometimes shows
four digits (`2775`, `1777`).

### setuid (`4000`) — run as the file's owner

```
-rwsr-xr-x 1 root root /usr/bin/passwd
   ↑ 's' where the owner's 'x' would be
```

`passwd` must write `/etc/shadow`, which is root-only. Rather than making
`/etc/shadow` writable, the *binary* is marked setuid root: when you execute it,
the process runs with root's privileges regardless of who started it. `sudo`
works the same way — that's how an unprivileged user manages to become root at
all.

This is a genuinely dangerous bit. A setuid-root program with a bug is a local
privilege escalation, which is why modern Linux is moving away from it. Look at
`ping` on your machine:

```
-rwxr-xr-x 1 root root /bin/ping      <- no setuid!
$ getcap /bin/ping
/bin/ping cap_net_raw=ep
```

`ping` needs raw sockets, not full root. **Capabilities** slice root's powers
into ~40 separate privileges and grant only the needed one. Remember this name —
it comes back in Phase 17/18 as `securityContext.capabilities` in a Kubernetes
pod spec.

An uppercase `S` means setuid is set but `x` is **not** — almost always a bug.

### setgid (`2000`) — on a directory, inherit the group

On a **directory**, setgid means "every file created in here gets *this
directory's* group, not the creator's default group":

```
$ chmod 2775 shared && ls -ld shared
drwxrwsr-x  shared
              ↑ 's' in the group triad
```

This is the standard way to make a shared team directory work: everyone's files
land in the shared group automatically. Combine with `chmod g+w` and the group
can collaborate. (On a *binary*, setgid means run-as-that-group — same idea as
setuid, rarer.)

### sticky bit (`1000`) — restricted deletion

```
$ ls -ld /tmp
drwxrwxrwt 11 root root /tmp
         ↑ 't' where other's 'x' would be
```

`/tmp` is `1777`: world-writable, because every program needs scratch space.
But recall Part 5 — **write on a directory means you can delete anything in
it**. Without protection, any user could delete every other user's temp files
(and worse: delete and replace one that a root process is about to open).

The sticky bit changes the rule for that directory: **you may only delete an
entry if you own the entry, or you own the directory.** That's the whole
purpose. Any world-writable directory without `t` is a security finding.

Setting them:

```bash
chmod 4755 prog     # setuid          (or: chmod u+s prog)
chmod 2775 dir      # setgid          (or: chmod g+s dir)
chmod 1777 dir      # sticky          (or: chmod +t dir)
```

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
kernel is asked to execute the file.** Feeding a script to an interpreter is a
read, not an execute. It's also why `x` on a `.jar` is meaningless — you run
`java -jar app.jar`, and `java` just reads it.

### "Permission denied" that is really about the shebang

```
$ ./script.sh
bash: ./script.sh: /bin/bash^M: bad interpreter: No such file or directory
```

Same-looking failure, different cause: the file was saved with **Windows CRLF
line endings**, so the interpreter path literally ends in a carriage return.
This will happen to you on `/mnt/d`. Fix with `dos2unix script.sh`, and prevent
it with a `.gitattributes` containing `*.sh text eol=lf`.

### A JAR that can't write its log

The app runs as `appuser`, the log directory is `drwxr-xr-x root root`. Java
throws `FileNotFoundException: /var/log/myapp/app.log (Permission denied)` —
note that Java reports it as a *file-not-found* class even though the real cause
is a **directory** with no `w` for `appuser`.

```bash
sudo mkdir -p /var/log/myapp
sudo chown appuser:appuser /var/log/myapp
sudo chmod 755 /var/log/myapp
```

Not `chmod 777`. The fix for a permission problem is almost always **ownership**,
not opening the bits up.

### The Docker version of the same bug

```dockerfile
RUN useradd -u 1001 appuser
USER 1001
```

A container running as UID 1001 writing to a bind-mounted host directory owned
by UID 1000 gets `Permission denied` — the UID inside the container and the UID
on the host are the same number space, and 1001 ≠ 1000. Phase 4 goes into this
properly; today just recognise the shape.

### Git tracks the execute bit — and only that one

Git stores one permission bit: mode `100644` or `100755`. So `chmod +x` is a
real, committable change:

```bash
git update-index --chmod=+x deploy.sh
```

That's why a script committed from Windows arrives non-executable in CI. It's
also why `chmod -R 777` shows up as a huge diff of mode changes.

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

`chmod` **reported no error and did nothing.** DrvFs (the Windows drive mount)
doesn't store Linux mode bits unless the `metadata` mount option is enabled, so
everything is a synthetic `0777` and every `chmod` is silently discarded.

Consequences you care about:

- **Everything on `/mnt/c` and `/mnt/d` appears world-writable** — you cannot
  practise permissions there, and today's exercises would all silently pass.
- `~/.ssh` on `/mnt/*` is unusable: `ssh` refuses keys that aren't `600`.
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

Ownership and mode are separate powers: the mode says what you may *do with the
contents*; `chown`/`chmod` rights come from *being the owner*. You can always
give yourself permission back — which is why `chmod` is not a security boundary
against the file's own owner.

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
  `777→755` for directories.
- **setuid** (`passwd`, `sudo`) runs as the owner; **capabilities** are the
  modern, narrower replacement and reappear in Kubernetes security contexts.
- The `x` bit is only checked by `execve()` — which is why `bash script.sh`
  works without it, and why it means nothing on a `.jar`.

**Tomorrow (Day 4):** processes, signals and jobs — PIDs and the process tree,
reading `ps aux` column by column, and `SIGTERM` vs `SIGKILL`, which is exactly
what decides whether your Spring Boot app shuts down gracefully or gets its
connections cut mid-request.

---

## Q&A
