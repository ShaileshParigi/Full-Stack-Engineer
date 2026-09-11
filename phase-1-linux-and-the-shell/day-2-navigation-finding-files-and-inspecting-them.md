# Day 2 — Navigation, Finding Files & Inspecting Them

_Why this matters:_ On a production box you have no file explorer. Finding the
one config that's wrong, the log line that explains the outage, or the file that
filled the disk is done entirely with these commands. Today also contains the
single most misunderstood thing about the shell — and it causes real bugs.

---

## The one-paragraph version

Moving around Linux is `cd` and `pwd`, and that part is easy. The part that
isn't obvious — and it's today's big idea — is that when you type `*.log`, **the
command you're running never sees those characters.** Your shell looks at the
directory first, replaces `*.log` with the actual matching filenames, and only
then starts the command. That's fine for most commands and catastrophic for
`find`, which needs the pattern itself, which is why `find` patterns must always
be quoted. The rest of today is the toolkit: `find` to search a whole tree by
name, size, age or type; `less` and `tail -F` to read files too big to open;
and `du`/`df` to answer "what filled the disk" — including the case where those
two disagree, which is a real 3am incident with a specific cause.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **shell** | The program that reads your typed commands and runs them | A command interpreter; bash here |
| **argument** | A word you pass to a command after its name | An element of the command's `argv` array |
| **expansion** | The shell rewriting your line before running anything | Substitution the shell performs on the command line pre-execution |
| **glob** | A wildcard pattern matched against real filenames | Pathname expansion — `*`, `?`, `[…]` matched against the filesystem |
| **brace expansion** | Text generation — `{a,b}` becomes `a b`, files or not | Pure textual expansion, performed before globbing, ignores the filesystem |
| **absolute path** | A path starting at `/` — works from anywhere | Fully qualified path from the filesystem root |
| **relative path** | A path starting from where you currently are | Path resolved against the current working directory |
| **working directory** | The folder you're "in" right now | Per-process attribute; `pwd` prints it |
| **builtin** | A command the shell runs itself, with no separate program | A command implemented inside the shell binary |
| **`PATH`** | The list of folders searched for a command's program | Colon-separated directory list searched for executables |
| **symlink** | A shortcut file pointing at another path | Symbolic link |
| **stderr** | The separate output channel errors go to | File descriptor 2, distinct from stdout (fd 1) |
| **`/dev/null`** | A bin — anything written there disappears | A device file discarding all writes |
| **inode** | The record holding a file's real data and metadata | On-disk structure holding metadata and data-block pointers; the name lives separately |
| **mtime / atime / ctime** | Content changed / last read / metadata changed | Modification, access, and inode-change timestamps |
| **magic bytes** | The few bytes at the start that reveal a file's real type | A signature sequence identifying a format regardless of extension |
| **log rotation** | Renaming a full log and starting a fresh one | Archiving the active log and reopening a new file under the original name |
| **file descriptor** | A number the kernel gives a process for an open file | Per-process integer index into the open-file table |

---

## Before you read: two questions

1. You run `find . -name *.log` and get
   `find: paths must precede expression`. But `find . -name "*.log"` works.
   **Why does quoting change anything?** The command is doing the searching
   either way… isn't it?
2. `df` says your disk is 100% full. `du` says the files only add up to half
   that. Both are telling the truth. **How?**

The first is today's core idea. The second is a genuine 3am incident.

---

## Part 1 — Moving around

**In plain words:** At any moment your shell is "sitting in" one directory —
your **working directory**. `pwd` tells you which. `cd` moves you. Everything
you type that doesn't start with `/` is interpreted relative to wherever you're
sitting.

```bash
pwd          # print working directory — where am I?
cd /var/log  # absolute path: starts at /
cd ..        # relative: up one level
cd           # no argument = go home
cd -         # jump back to the previous directory
```

| Symbol | Means |
|--------|-------|
| `/` | Root. A path starting with `/` is **absolute** |
| `.` | Current directory |
| `..` | Parent directory |
| `~` | Your home (`/home/shailesh_parigi`) |
| `-` | The directory you were in *before* (with `cd`) |

`cd -` is the one people don't know and use constantly once they do — it
toggles between two directories, like alt-tab.

**Absolute vs relative matters in scripts.** A relative path means "from
wherever I happen to be", so a script full of them breaks the moment someone
runs it from a different directory — or the moment cron runs it, since cron
starts you in your home directory rather than the script's. In anything you'd
schedule or ship, use absolute paths or `cd` to a known location first.

---

## Part 2 — The shell expands globs, not the command

This is the idea to take away from today.

**In plain words:** You probably assume that when you type `ls *.log`, the `ls`
program receives the text `*.log` and goes looking for matches. It doesn't. Your
**shell** looks in the directory first, finds `a.log` and `b.log`, rewrites your
line to `ls a.log b.log`, and only then runs `ls`. The program never sees a star
in its life. Commands don't understand wildcards — **the shell does the
matching, always, before anything runs.**

Prove it with `echo`, which cannot possibly know what a wildcard is:

```bash
cd /tmp && mkdir -p day2 && cd day2 && touch a.log b.log
echo *.log
```

```
a.log b.log
```

`echo` just prints the arguments it was handed. The shell did all the work
before `echo` started. That's the whole mechanism.

### Why this breaks `find`

`find` is different from `ls`: it walks a whole tree and does its **own**
matching at every level, so it needs the *pattern itself*, not a pre-expanded
list. But the shell gets there first:

```bash
find . -name *.log
```

```
find: paths must precede expression: `b.log'
find: possible unquoted pattern after predicate `-name'?
```

The shell rewrote your command into:

```
find . -name a.log b.log
```

…and `find` has no idea what that trailing `b.log` is meant to be — `-name`
takes one pattern, so the second filename is just garbage sitting in the middle
of the expression.

Quoting stops the shell from expanding, so `find` receives the pattern intact:

```bash
find . -name '*.log'
```

That's question 1: quoting doesn't change what `find` does — it changes what
`find` is *given*.

> **The nasty part:** with exactly **one** matching file, the unquoted version
> works fine (the shell rewrites it to `find . -name a.log`, which is valid).
> So it passes your testing and breaks weeks later when a second file appears.
> Always quote `find` patterns.

### The pattern characters

| Pattern | Matches |
|---------|---------|
| `*` | Zero or more characters (never crosses `/`) |
| `?` | Exactly one character |
| `[ab]` | One character from the set |
| `[a-z]` | One character in the range |
| `[!a]` | One character *not* `a` |

Note these are **not** regular expressions — `*` here means "any characters",
whereas in a regex it means "zero or more of the previous thing". Two different
languages that share symbols. You'll meet real regex on Day 6.

### `{a,b}` is not globbing

Brace expansion looks similar but is a **completely different mechanism**, and
the difference is visible in one line:

```bash
echo {x,y}.zzz    # -> x.zzz y.zzz    (no such files exist!)
echo *.zzz        # -> *.zzz          (stayed literal)
```

- **Brace expansion** is pure text generation. It doesn't look at the filesystem
  at all and doesn't care whether the files exist. That's what makes
  `mkdir -p project/{src,test,docs}` work — you're creating things that don't
  exist yet.
- **Globbing** matches against the filesystem. And in bash's default mode, a
  pattern matching **nothing is left alone** and passed through literally.

That last behaviour causes real bugs and is worth pausing on. `for f in *.log`
in an empty directory doesn't loop zero times — it loops **once**, with `f` set
to the literal seven-character string `*.log`. Your script then tries to process
a file by that name. Guard with `[ -e "$f" ] || continue`.

> **Say this in an interview:** "Globbing is done by the shell, not the command
> — the shell expands the pattern against the filesystem and passes the
> resulting filenames as arguments, so the program never sees the wildcard.
> That's why `find -name` patterns must be quoted: `find` does its own matching
> as it walks, so it needs the literal pattern. And an unmatched glob is passed
> through literally by default, which is why `for f in *.log` runs once with a
> bogus filename in an empty directory."

---

## Part 3 — `ls` flags worth knowing

```bash
ls -l      # long form: permissions, owner, size, mtime
ls -a      # include dotfiles
ls -h      # human-readable sizes (with -l)
ls -t      # sort by modification time, newest first
ls -r      # reverse the sort
ls -d      # the directory itself, not its contents
ls -R      # recurse
```

Two combinations to memorise:

```bash
ls -lah          # the everyday one
ls -ltr          # oldest first, newest LAST — best for log directories
```

**Why `-ltr` and not `-lt`:** `-t` puts the newest at the top, which then
scrolls off the screen as the listing prints. Reversing it (`-r`) puts the
newest file at the **bottom**, right above your prompt, where you're already
looking. That's why it's the reflex on a log directory — no scrolling.

---

## Part 4 — `find`

**In plain words:** `find` walks an entire directory tree, top to bottom,
testing every single file it meets against conditions you give it, and does
something with the ones that match. Three parts: where to start, what to test,
what to do.

```
find <where> <tests…> <action>
```

Multiple tests are combined with an implicit AND — all must be true.

```bash
find . -type f -name '*.log'        # files (not dirs) ending .log
find /etc -type d -name 'ngin*'     # directories
find . -type l                      # symlinks
find . -type f -size +1k            # bigger than 1k  (also M, G)
find . -type f -size -1M            # smaller than 1M
find . -mtime -7                    # modified in the last 7 days
find . -mmin -30                    # modified in the last 30 minutes
find . -maxdepth 1 -type f          # don't recurse
find . -iname '*.LOG'               # case-insensitive
```

**The `+`/`-` convention trips everyone up**, and it's backwards from how it
reads. Think of it as "more than N" and "fewer than N" *of the unit*, not
"newer/older":

- `-mtime -7` → **fewer than** 7 days old → **recent**
- `-mtime +7` → **more than** 7 days old → **old**
- `-mtime 7` → exactly 7

Sizes work identically: `+1M` is larger than, `-1M` is smaller than.

### Actions and `-exec`

**In plain words:** By default `find` just prints what it found. `-exec` lets
you run a command on each result instead — and there are two ways to end that
command, with wildly different performance.

```bash
find . -name '*.log' -exec wc -l {} \;    # one wc per file
find . -name '*.log' -exec wc -l {} +     # ONE wc with all files
```

`{}` is the placeholder that gets replaced with each result.

- `\;` runs the command **once per file**. 1,000 files means starting 1,000
  separate processes, and process startup is expensive.
- `+` batches results into **as few invocations as possible** — usually one.
  Dramatically faster, and as a bonus `wc` gives you a `total` line because it
  saw all the files at once.

Prefer `+` unless the command genuinely handles only one file at a time.

(The `\;` needs the backslash because `;` on its own would end the shell
command — same escaping principle as quoting the glob.)

**Silencing permission noise.** Searching system directories as a normal user
produces a flood of `Permission denied` messages. Those go to **stderr**, a
separate output channel from normal results, so you can throw them away without
losing anything:

```bash
find /var -type f -name '*.log' -mtime -7 2>/dev/null
```

`2>` redirects file descriptor 2 (stderr) and `/dev/null` is a bin that
discards everything written to it. So: "discard the errors, keep the results."
You'll cover descriptors properly later; this idiom is worth using now.

> **Say this in an interview:** "`find` is start directory, tests, action. The
> `-mtime` sign convention is fewer-than for `-` and more-than for `+`, so
> `-mtime -7` is the last week. For actions, `-exec … +` batches results into
> one invocation while `\;` forks a process per file — on a large tree that's
> the difference between a second and a minute. And `2>/dev/null` drops the
> permission-denied noise on stderr without touching the results."

---

## Part 5 — Where does a command come from?

**In plain words:** When you type a word, the shell has to work out what to run.
Some commands aren't programs at all — they're built into the shell itself.
Others are real files somewhere on disk. Knowing which is which explains a class
of "works when I type it, fails in my script" bugs.

```bash
type cd            # cd is a shell builtin
which cd           # (prints nothing, exits 1)
type ls            # ls is /usr/bin/ls
command -v grep    # /usr/bin/grep
```

This is a real distinction, not trivia:

- **`which`** is an *external program*. It can only search `PATH` for files. It
  cannot see shell builtins, functions, or aliases — because those exist inside
  your shell's memory, and `which` is a separate process that can't look in
  there. So `which cd` **fails**, even though `cd` obviously works.
- **`type`** is a shell builtin, so it *is* the shell and knows everything the
  shell knows: builtins, functions, aliases, and PATH executables.
- **`command -v`** is the POSIX-portable version. **Use this one in scripts.**

(And `cd` *has* to be a builtin: changing directory changes the shell's own
state. An external program could only change its own working directory and then
exit, leaving you exactly where you were.)

Why you'll care: "it works when I type it but my script says command not found"
is almost always an alias or shell function that exists in your interactive
shell and not in the script's environment. `type` tells you instantly which
you're dealing with.

`whereis` finds the binary, source and man page. `locate` searches a prebuilt
index — instant, but stale unless `updatedb` has run recently.

---

## Part 6 — Reading files

**In plain words:** `cat` dumps a whole file to the screen, which is fine for
config files and useless for a 2GB log. The skill is picking the right tool for
the size: `head`/`tail` for the ends, `less` to page through interactively,
`tail -F` to watch a file grow live.

```bash
cat file            # dump the whole thing
cat -n file         # with line numbers
head -20 file       # first 20 lines
tail -20 file       # last 20 lines
wc -l file          # count lines  (-w words, -c bytes)
less file           # page through it
```

**`less` is the one to actually learn**, because you cannot `cat` a 2GB log —
it'll flood your terminal and, on a slow connection, take minutes. `less` reads
only what it displays.

| Key | Does |
|-----|------|
| `/text` | Search forward; `n` next match, `N` previous |
| `G` | Jump to end |
| `g` | Jump to start |
| `q` | Quit |
| `-S` (flag) | Don't wrap long lines — invaluable for logs |

`-S` matters more than it sounds: a single JSON log line can be 2,000 characters
and will otherwise fill your whole screen as one entry. With `-S` each log line
stays on one row and you scroll sideways with arrow keys.

### `tail -f` vs `tail -F` — a real production trap

```bash
tail -f  /var/log/syslog     # follow this file handle
tail -F  /var/log/syslog     # follow this NAME, survive rotation
```

**In plain words:** Log files don't grow forever — a tool called logrotate
periodically renames the current one (to `app.log.1`) and starts a fresh empty
`app.log`. Now: `-f` is attached to the *file itself*, not the name. After
rotation it's still faithfully watching the renamed old file, which nothing
writes to any more. Your screen goes quiet, and it looks exactly like the
application stopped logging — during an incident, that's a genuinely dangerous
false signal.

**`-F` follows the filename**, noticing the replacement and reopening it. On
production logs, use `-F`.

You'll run this yourself in exercise 8 and watch it happen.

---

## Part 7 — Inspecting files and disks

```bash
file report.pdf       # what IS this, by content not extension
stat access.log       # size, permissions, owner, timestamps
du -sh ~/projects     # how big is this tree
df -h                 # free space per filesystem
```

**`file`** reads **magic bytes** — the handful of bytes at the start of a file
that identify its real format — so it correctly identifies a JPEG named
`notes.txt`. Genuinely useful when a download saved an HTML error page with a
`.zip` extension and your unzip is failing for no apparent reason.

**`stat`** shows three timestamps, and one of them is universally misread:

- **mtime** — the *contents* changed
- **atime** — the file was *read*
- **ctime** — the *inode* changed: permissions, ownership, rename, link count

People assume `ctime` is "creation time" because of the c. It isn't; Linux
traditionally has **no creation timestamp** at all.

### `du` vs `df` — the 3am question

**In plain words:** These two count disk usage in completely different ways.
`du` walks the folders and adds up the files it can *see*. `df` asks the
filesystem how much space is actually handed out. Normally they agree. When they
don't, something is holding space that no longer has a name.

Here's the mechanism, and it's question 2. On Linux, deleting a file doesn't
delete its data — it removes the **name** from a directory. The data lives until
the last thing holding it open lets go. So if a process has a log file open and
someone deletes it:

- **`du`** walks the directory tree and finds no such file → doesn't count it.
- **`df`** asks the filesystem, whose blocks are still allocated → counts it.

Both are correct. The classic version: the disk fills, someone `rm`s the huge
log to free space, `df` doesn't budge, and everyone is baffled. The application
still has it open, so nothing was actually freed — and worse, the log is now
invisible but still growing.

The fix is to **restart the process**, or truncate the file in place instead of
deleting it:

```bash
> /var/log/huge.log     # truncate to zero bytes, keeps the file and the handle
```

Find the culprits with `lsof +L1` — open files with zero directory links, which
is precisely "deleted but still held."

On your machine right now:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sde       1007G  3.1G  953G   1% /
D:\              26G   20G  5.9G  77% /mnt/d
```

Worth noticing: your Linux filesystem has **953 GB free**, while **`D:` is 77%
full with 5.9 GB left**. Another reason Linux experiments belong in `~`.

> **Say this in an interview:** "`du` sums what it can see by walking the tree;
> `df` asks the filesystem for allocated blocks. They diverge when a file has
> been unlinked but a process still holds it open — the name is gone so `du`
> misses it, but the blocks aren't freed until the last descriptor closes.
> That's why `rm`-ing a log to reclaim space often does nothing. You find it
> with `lsof +L1`, and you fix it by restarting the holder or truncating in
> place with `> file`."

---

## Hands-on

### 1. Prove the shell expands globs

```bash
cd /tmp && rm -rf day2 && mkdir day2 && cd day2
touch a.log b.log notes.txt
echo "shell expands to: $(echo *.log)"
echo "--- unquoted (broken) ---"
find . -name *.log 2>&1 | head -2
echo "--- quoted (correct) ---"
find . -name '*.log'
```

### 2. Glob vs brace

```bash
cd /tmp/day2
echo "?.log    -> $(echo ?.log)"
echo "[ab].log -> $(echo [ab].log)"
echo "{x,y}.zzz -> $(echo {x,y}.zzz)   <- expands, files don't exist"
echo "*.zzz     -> $(echo *.zzz)        <- stays literal, nothing matched"
```

### 3. Brace expansion for real work

```bash
cd /tmp/day2
mkdir -p project/{src,test,docs}/{main,resources}
find project -type d | sort
```

One line, twelve directories. This is the everyday use for braces.

### 4. `find` predicates and `-exec`

```bash
cd /tmp/day2
printf 'line%d\n' $(seq 1 200) > big.log
find . -type f -size +1k
find . -type f -mmin -5
find . -name '*.log' -exec wc -l {} +
```

Notice the `total` line from `wc` — that's the `+` batching all files into one
invocation.

### 5. Builtins vs binaries

```bash
type cd; type ls; type type
which cd; echo "which cd exit code: $?"
command -v grep
```

`which cd` printing nothing and exiting 1 is the whole point of Part 5.

### 6. Practice — logs under `/var` from the last 7 days

```bash
find /var -type f -name '*.log' -mtime -7 2>/dev/null
echo "count: $(find /var -type f -name '*.log' -mtime -7 2>/dev/null | wc -l)"
```

### 7. Practice — the 5 largest files in your home

```bash
find "$HOME" -type f -printf '%s\t%p\n' 2>/dev/null \
  | sort -rn | head -5 \
  | awk -F'\t' '{printf "%8.1f MB  %s\n", $1/1048576, $2}'
```

`-printf '%s\t%p\n'` prints size and path; `sort -rn` sorts numerically (`n`),
descending (`r`). This is the "what filled the disk" command.

### 8. Practice — `tail -f` across two terminals

This one needs **two terminals**, so use the Ubuntu (WSL) tab twice (or the tab
plus a second WSL window).

Terminal 1:

```bash
touch /tmp/day2/watch.log
tail -f /tmp/day2/watch.log
```

Terminal 2:

```bash
echo "first entry $(date +%T)" >> /tmp/day2/watch.log
echo "second entry $(date +%T)" >> /tmp/day2/watch.log
```

Watch them appear live in terminal 1. Then simulate log rotation and watch `-f`
go blind:

```bash
mv /tmp/day2/watch.log /tmp/day2/watch.log.1
echo "after rotation" >> /tmp/day2/watch.log
```

Terminal 1 shows nothing — and crucially, no error. Stop it with `Ctrl+C`,
rerun with `tail -F`, rotate again, and watch it recover. **That difference is
the lesson**, and the silence is why it matters.

### 9. Disk inspection

```bash
df -h / /mnt/d
du -sh ~/.vscode-server 2>/dev/null
file /usr/bin/ls
stat /usr/bin/ls | head -8
```

---

## Gotchas

- **Always quote `find` patterns.** Unquoted works with one match and breaks
  with two.
- **An unmatched glob stays literal.** `for f in *.log` runs once with the
  literal string when nothing matches. Guard with `[ -e "$f" ]`.
- **`-mtime -7` is "less than 7 days ago."** `+7` is older than. Bare `7` is
  exactly 7.
- **`which` can't see builtins.** Use `type` interactively, `command -v` in scripts.
- **`tail -f` dies silently at log rotation.** Use `-F` on production logs.
- **`ctime` is not creation time.** It's inode-change time.
- **`du` and `df` disagreeing** usually means a deleted-but-open file.
- **`-exec … \;` forks per file.** Use `+` unless the command takes one at a time.
- **Always quote `"$variable"` in paths.** Your project path has a space in it
  (`/mnt/d/Learn Daily/…`), so an unquoted variable splits into two arguments.

---

## Recap

- **The shell expands globs before the command runs.** The command receives
  filenames, never the pattern. This is why `find` patterns must be quoted.
- **Brace expansion is not globbing** — it generates text regardless of what
  exists on disk, which is what makes `mkdir {a,b,c}` work.
- **`find`** = where + tests + action. Use `-exec … +` over `\;`, and
  `2>/dev/null` to drop permission noise. `-mtime -7` is recent, `+7` is old.
- **`type`/`command -v` beat `which`** because they see builtins, functions and
  aliases — and that's the usual cause of "works interactively, fails in a
  script."
- **`less -S` for big files, `tail -F` for live logs** that may rotate. `-f`
  fails silently at rotation.
- **`du` walks the tree, `df` asks the filesystem** — a gap between them means
  something deleted is still held open.

---

## Say it out loud

Answer each in about 30 seconds, using the real terms.

1. Walk through exactly what happens between pressing Enter on `ls *.log` and
   `ls` producing output.
2. Why does quoting change the behaviour of `find -name`? Why does the unquoted
   version sometimes work?
3. A script loops `for f in *.csv` and crashes on an empty directory. What
   happened, and how do you guard it?
4. Explain `-exec {} \;` versus `-exec {} +` and when the difference bites.
5. Your `tail -f` on a production log has been silent for 20 minutes. Name two
   possible reasons and how you'd tell them apart.
6. `df` says 100% full, `du` says half. What's going on, and what do you run?

---

**Tomorrow (Day 3):** permissions and ownership — including what "execute" means
on a *directory*, which is not what you'd guess.

---

## Q&A
