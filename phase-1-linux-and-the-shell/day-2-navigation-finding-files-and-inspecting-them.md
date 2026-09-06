# Day 2 — Navigation, Finding Files & Inspecting Them

_Why this matters:_ On a production box you have no file explorer. Finding the
one config that's wrong, the log line that explains the outage, or the file that
filled the disk is done entirely with these commands. Today also contains the
single most misunderstood thing about the shell — and it causes real bugs.

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
toggles between two directories.

**Absolute vs relative matters in scripts.** A script using relative paths
breaks the moment someone runs it from a different directory. In anything you'd
schedule or ship, use absolute paths or `cd` to a known location first.

---

## Part 2 — The shell expands globs, not the command

This is the idea to take away from today.

When you type `ls *.log`, **`ls` never sees `*.log`.** The shell expands the
pattern into a list of matching filenames *first*, then runs the command with
those filenames as arguments.

```bash
cd /tmp && mkdir -p day2 && cd day2 && touch a.log b.log
echo *.log
```

```
a.log b.log
```

`echo` has no idea what a glob is. It just printed the two arguments it was
handed. The shell did all the work before `echo` started.

### Why this breaks `find`

`find` wants the *pattern itself*, because it does its own matching as it walks
the tree. But the shell gets there first:

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

…and `find` has no idea what that trailing `b.log` is meant to be.

Quoting stops the shell from expanding, so `find` receives the pattern intact:

```bash
find . -name '*.log'
```

> **The nasty part:** with exactly **one** matching file, the unquoted version
> works fine. It passes your testing and breaks later when a second file
> appears. Always quote `find` patterns.

### The pattern characters

| Pattern | Matches |
|---------|---------|
| `*` | Zero or more characters (never crosses `/`) |
| `?` | Exactly one character |
| `[ab]` | One character from the set |
| `[a-z]` | One character in the range |
| `[!a]` | One character *not* `a` |

### `{a,b}` is not globbing

Brace expansion looks similar but is a **different mechanism**, and the
difference is visible:

```bash
echo {x,y}.zzz    # -> x.zzz y.zzz    (no such files exist!)
echo *.zzz        # -> *.zzz          (stayed literal)
```

- **Brace expansion** is pure text generation. It doesn't care whether the files
  exist. Great for `mkdir -p project/{src,test,docs}`.
- **Globbing** matches against the filesystem. In bash's default mode, a pattern
  that matches **nothing is left alone** and passed through literally — which is
  why a script can end up operating on a file literally named `*.zzz`.

That last behaviour causes real bugs. `for f in *.log` runs once with `f`
set to the literal string `*.log` when the directory is empty.

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

`-ltr` puts the newest file at the bottom, right above your prompt, so you don't
have to scroll. That's why it's the reflex on a log directory.

---

## Part 4 — `find`

`find` walks a directory tree and tests every entry. The shape is:

```
find <where> <tests…> <action>
```

Tests are combined with implicit AND.

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

**The `+`/`-` convention trips everyone up:** `-7` means *less than 7 (more
recent)*, `+7` means *more than 7 (older)*, and a bare `7` means *exactly 7*.
For sizes it's the same: `+1M` is larger than, `-1M` is smaller than.

### Actions and `-exec`

```bash
find . -name '*.log' -exec wc -l {} \;    # one wc per file
find . -name '*.log' -exec wc -l {} +     # ONE wc with all files
```

`{}` is the placeholder for each result.

- `\;` runs the command **once per file**. 1,000 files = 1,000 processes.
- `+` batches results into **as few invocations as possible**. Dramatically
  faster, and gives you a `total` line from `wc`.

Prefer `+` unless the command genuinely handles only one file at a time.

**Silencing permission noise.** Searching system directories as a normal user
produces a flood of `Permission denied` on stderr. Send stderr away:

```bash
find /var -type f -name '*.log' -mtime -7 2>/dev/null
```

`2>` redirects file descriptor 2 (stderr). You'll cover descriptors properly on
Day 6 — for now, `2>/dev/null` means "discard the errors, keep the results."

---

## Part 5 — Where does a command come from?

```bash
type cd            # cd is a shell builtin
which cd           # (prints nothing, exits 1)
type ls            # ls is /usr/bin/ls
command -v grep    # /usr/bin/grep
```

This is a real distinction, not trivia:

- **`which`** is an external program. It only searches `PATH` for files. It
  cannot see shell builtins, functions, or aliases — so `which cd` **fails**,
  even though `cd` obviously works.
- **`type`** is a shell builtin, so it knows everything the shell knows:
  builtins, functions, aliases, and PATH executables.
- **`command -v`** is the POSIX-portable version. **Use this one in scripts.**

Why you'll care: "it works when I type it but my script says command not found"
is almost always an alias or shell function that exists in your interactive
shell and not in the script's environment. `type` tells you instantly.

`whereis` finds the binary, source and man page. `locate` searches a prebuilt
index — instant, but stale unless `updatedb` has run.

---

## Part 6 — Reading files

```bash
cat file            # dump the whole thing
cat -n file         # with line numbers
head -20 file       # first 20 lines
tail -20 file       # last 20 lines
wc -l file          # count lines  (-w words, -c bytes)
less file           # page through it
```

**`less` is the one to actually learn**, because you cannot `cat` a 2GB log.
Inside `less`:

| Key | Does |
|-----|------|
| `/text` | Search forward; `n` next match, `N` previous |
| `G` | Jump to end |
| `g` | Jump to start |
| `q` | Quit |
| `-S` (flag) | Don't wrap long lines — invaluable for logs |

### `tail -f` vs `tail -F`

```bash
tail -f  /var/log/syslog     # follow this file handle
tail -F  /var/log/syslog     # follow this NAME, survive rotation
```

`-f` keeps the file *handle* open. When logrotate renames the file and creates a
new one, your `-f` is still watching the old, now-renamed file and goes silent —
looking exactly like "the application stopped logging."

**`-F` follows the filename**, reopening when it's replaced. On production logs,
use `-F`.

---

## Part 7 — Inspecting files and disks

```bash
file report.pdf       # what IS this, by content not extension
stat access.log       # size, permissions, owner, timestamps
du -sh ~/projects     # how big is this tree
df -h                 # free space per filesystem
```

`file` reads **magic bytes**, so it identifies a JPEG named `notes.txt`
correctly. Useful when something downloaded an HTML error page and saved it with
a `.zip` extension.

`stat` shows three timestamps: **mtime** (content changed), **atime** (read),
**ctime** (inode changed — permissions, rename). People assume ctime is
"created". It isn't; Linux traditionally has no creation time.

### `du` vs `df` — the 3am question

They measure different things:

- **`du`** walks the directory tree and adds up the files it can *see*.
- **`df`** asks the filesystem how many blocks are allocated.

They disagree when **a file has been deleted but a process still holds it
open**. The directory entry is gone, so `du` can't see it — but the blocks stay
allocated until the last file descriptor closes, so `df` still counts them.

Classic version: someone `rm`s a huge log to free space, but the application
still has it open, so nothing is actually freed. The fix is to restart the
process or truncate the file in place (`> file`) instead of deleting it.

Find the culprits with `lsof +L1` — open files with zero directory links.

On your machine right now:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sde       1007G  3.1G  953G   1% /
D:\              26G   20G  5.9G  77% /mnt/d
```

Worth noticing: your Linux filesystem has **953 GB free**, while **`D:` is 77%
full with 5.9 GB left**. Another reason Linux experiments belong in `~`.

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

### 4. `find` predicates and `-exec`

```bash
cd /tmp/day2
printf 'line%d\n' $(seq 1 200) > big.log
find . -type f -size +1k
find . -type f -mmin -5
find . -name '*.log' -exec wc -l {} +
```

### 5. Builtins vs binaries

```bash
type cd; type ls; type type
which cd; echo "which cd exit code: $?"
command -v grep
```

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

`-printf '%s\t%p\n'` prints size and path; `sort -rn` sorts numerically,
descending.

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

Watch them appear live in terminal 1. Then simulate log rotation and see `-f`
go blind:

```bash
mv /tmp/day2/watch.log /tmp/day2/watch.log.1
echo "after rotation" >> /tmp/day2/watch.log
```

Terminal 1 shows nothing. Stop it with `Ctrl+C`, rerun with `tail -F`, rotate
again, and watch it recover. **That difference is the lesson.**

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
- **Always quote `"$variable"` in paths.** Your project path has a space in it
  (`/mnt/d/Learn Daily/…`), so unquoted variables will split into two arguments.

---

## Recap

- **The shell expands globs before the command runs.** The command receives
  filenames, never the pattern. This is why `find` patterns must be quoted.
- **Brace expansion is not globbing** — it generates text regardless of what
  exists on disk.
- **`find`** = where + tests + action. Use `-exec … +` over `\;`, and
  `2>/dev/null` to drop permission noise.
- **`type`/`command -v` beat `which`** because they see builtins, functions and
  aliases.
- **`less -S` for big files, `tail -F` for live logs** that may rotate.
- **`du` walks the tree, `df` asks the filesystem** — a gap between them means
  something deleted is still held open.

**Tomorrow (Day 3):** permissions and ownership — including what "execute" means
on a *directory*, which is not what you'd guess.

---

## Q&A
