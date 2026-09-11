# Day 6 — `grep` & Regular Expressions

_Why this matters:_ Every production incident you debug starts the same way —
a lot of text, and one line in it that explains the outage. `grep` is how you
find that line, and it's the command you'll type most for the rest of your
career: `kubectl logs pod | grep -i timeout`, `grep -rn "@Transactional" src/`,
`docker logs app 2>&1 | grep -c ERROR`. And regular expressions — the pattern
language `grep` speaks — aren't a shell thing. They're in Java
(`Pattern`/`Matcher`), your IDE's find box, Logback filters, Kafka topic
subscriptions and every log-alerting rule. Learn the language once here, use it
everywhere.

> Week 1 was the machine: filesystem, permissions, processes, packages. Week 2 is
> **text**: finding it (today), transforming it (Day 7 — `sed`/`awk`), parsing
> JSON (Day 8 — `jq`) and automating it (Day 9 — scripting).

---

## The one-paragraph version

`grep` reads text one line at a time and prints the lines that match a pattern.
That's all — it is a **filter for lines**. The interesting half is the pattern
language, the **regular expression** (regex): a tiny language where most
characters mean themselves (`error` means the letters e-r-r-o-r) but about a
dozen are **special** and mean things like "any character", "start of line",
"one or more of the previous thing" or "one of these characters". Two things
trip everyone up. First, `grep` matches **anywhere in the line**, not the whole
line — `grep cat` also finds `concatenate`; if you want more than that you have
to say so. Second, regex comes in several dialects, and plain `grep` speaks the
oldest one, where `+`, `?`, `|` and parentheses don't work unless you backslash
them — which is why in practice you **always type `grep -E`**. The rest is
flags: `-i` ignore case, `-v` invert, `-n` line numbers, `-c` count, `-l` just
filenames, `-r` recurse, `-A/-B/-C` show surrounding lines, and `-o` to print
**only the matching text** instead of the whole line, which turns `grep` from a
filter into an extractor. **Filter lines, learn the pattern language, always use
`-E`, and put your pattern in single quotes so the shell keeps its hands off
it.**

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **regex / regular expression** | A pattern describing a *shape* of text, not exact text | A formal language describing a set of strings, compiled to an automaton |
| **pattern** | The thing you hand to `grep` | The regex as a string, before it's compiled |
| **literal** | A character that just means itself | A character with no special meaning in the current dialect |
| **metacharacter** | A character with a special job, like `.` or `*` | A character interpreted as syntax rather than data |
| **escape** | A `\` in front, to change what a character means | Backslash-quoting: makes a metacharacter literal (or, in BRE, the reverse) |
| **anchor** | "Must be at the start / at the end" | Zero-width assertion about position: `^` start of line, `$` end of line |
| **zero-width** | Checks a position without eating any characters | An assertion that consumes no input |
| **character class** | "Any one of these characters" — `[aeiou]` | A bracket expression matching exactly one character from a set |
| **negated class** | "Any one character that is *not* these" — `[^0-9]` | A bracket expression whose first character is `^` |
| **POSIX class** | Named sets like `[[:digit:]]` that work in every dialect | Locale-aware named classes, used *inside* brackets |
| **quantifier** | "How many of the previous thing" — `*`, `+`, `?`, `{2,5}` | A repetition operator binding to the single preceding atom |
| **atom** | The one smallest thing a quantifier applies to | One character, one class, one group, or one escape |
| **greedy** | A quantifier grabs as much as it possibly can | Leftmost-longest match semantics; POSIX regex has no lazy operator |
| **group** | Parentheses bundling things together | A subexpression; also *capturing* — it remembers what it matched |
| **backreference** | "The exact text group 1 matched, again" | `\1`…`\9`, referring back to a capture group |
| **alternation** | "This or that" — `cat` or `dog` | The union operator: bare pipe in ERE, backslash-pipe in BRE |
| **word boundary** | The edge between a word and a non-word character | Zero-width `\b`; word characters are `[A-Za-z0-9_]` |
| **BRE** | Old-style regex — plain `grep` | POSIX **Basic** Regular Expressions: `+ ? ( ) { }` and the pipe need a `\` to work |
| **ERE** | Sane regex — `grep -E` | POSIX **Extended** Regular Expressions: those characters work bare |
| **PCRE** | Perl-style regex — `grep -P`; what Java resembles | Perl Compatible Regular Expressions: adds `\d`, lazy `*?`, lookarounds |
| **fixed string** | "Treat my pattern as plain text, not regex" | `grep -F`: literal substring search, no metacharacters |
| **exit status** | The number a command hands back; 0 = success | `$?`; `grep` returns 0 = matched, 1 = no match, 2 = error |
| **stdin** | Input arriving through a pipe instead of from a file | File descriptor 0; `grep` reads it when you give no filename |
| **glob** | The *shell's* filename wildcards — `*.log` | Pathname expansion, done by the shell **before** `grep` runs |
| **buffering** | Saving up output and sending it in chunks | stdio buffer mode: line-buffered to a terminal, block-buffered to a pipe |

> **Glob is not regex.** `*.log` as a glob means "anything, then `.log`" and
> matches *filenames*, in the shell. As a regex, `.log` means "any one
> character, then `log`" and matches *text*, inside `grep`. Same characters,
> different languages, different programs doing the interpreting.

---

## Before you read: three questions

1. `grep cat animals.txt` — does it print a line containing `concatenate`? Why?
2. What's the difference between `grep "class Foo" Bar.java` and
   `grep class Foo Bar.java`? One of them is not doing what you think.
3. A script starting with `set -e` runs `grep ERROR app.log`. The log is clean —
   no errors at all. What happens to the script?

---

## Part 1 — What `grep` actually does

**In plain words:** picture reading a book one line at a time with a rule in
your head — "does this line mention a timeout?" If yes, copy the whole line onto
a notepad. If no, move on. At the end, hand over the notepad. That's `grep`: it
never sees the file as a whole, only as a stream of lines, and its output is
always a subset of its input lines.

The name comes from the old `ed` editor command `g/re/p` — **g**lobally search
for a **r**egular **e**xpression and **p**rint. That's literally the job.

```
grep [FLAGS] PATTERN [FILE...]
```

- **No FILE** → `grep` reads **stdin**. That's what lets it sit in a pipe.
- **More than one FILE** → each output line is prefixed with its filename.
- The first non-flag argument is the **pattern**. Everything after is a file.

### The rule behind most confusion

`grep` asks: **does the pattern match *somewhere inside* this line?** Not "is
the line equal to the pattern", not "does the line start with it".

So `grep cat` matches `cat`, `concatenate`, `scatter` and `the cat sat`. Whole
words need `-w` (or `\b`). The whole line needs anchors — `^cat$` — or `-x`.

That's the same split as Java's `Matcher.find()` (anywhere inside) versus
`Matcher.matches()` (the whole string). **`grep` is `find()`.** `grep -x` is
`matches()`.

### Exit status — the half nobody teaches

`grep` doesn't just print. It **returns a number**:

| Exit status | Meaning |
|---|---|
| `0` | At least one line matched |
| `1` | No line matched — *not* an error |
| `2` | A real error: missing file, invalid pattern |

That makes `grep` a **test**, not only a printer:

```bash
if grep -q '^root:' /etc/passwd; then echo "root account exists"; else echo "no root"; fi
grep -q 'no-such-user-xyz' /etc/passwd; echo "exit status: $?"
```

`-q` (quiet) prints nothing and stops at the first match. It's the right way to
ask a yes/no question. `grep ... > /dev/null` works but is the long way round;
`grep ... | wc -l` as a yes/no test reads the whole file for no reason.

The trap: status `1` means "nothing found", but `set -e` treats every non-zero
status as failure and kills the script. A clean log becomes a failed deploy.
Day 9 covers the fix properly; the short form is `grep ... || true`.

> **Say this in an interview:** "`grep` is a line-oriented filter: it evaluates
> the regex against each input line and prints the lines that match. Matching is
> unanchored — substring semantics, like `Matcher.find()` rather than
> `matches()`. It also reports through its exit status — 0 matched, 1 no match,
> 2 error — which is why `grep -q` is the idiomatic conditional in shell
> scripts."

---

## Part 2 — Regular expressions, from zero

**In plain words:** a regex describes a *shape*. "Three digits, a dash, four
digits" is a shape — `555-1234` and `867-5309` both have it. You write that
shape as a short string, and the tool checks text against it. Most characters in
the string just mean themselves. A dozen or so are the grammar. Learn those and
you can read any regex.

### 2.1 Literals

`grep -E 'error'` — the letters `e r r o r`, adjacent, in that order, anywhere
in the line. Most of any real pattern is literals.

### 2.2 The metacharacters

In ERE (`grep -E`), these are the characters that are **not** literal:

```
.  ^  $  *  +  ?  (  )  [  ]  {  }  |  \
```

Everything else means itself. To match one of these literally, put `\` in
front: `\.` is a real dot, `\$` is a real dollar sign, `\[` is a real bracket.

**The classic bug:** `grep '192.168.1.1'` looks like an IP address, but `.`
means "any character", so it also matches `192x168y1z1`, `1920168010100` and
`192.168.1.100`. The correct pattern is `192\.168\.1\.1` — and even that
matches inside `192.168.1.10`, which is what `-w` is for.

### 2.3 `.` — any one character

`.` matches exactly one character, whatever it is.

| Pattern | Matches | Doesn't match |
|---|---|---|
| `c.t` | `cat`, `cut`, `c3t`, `c t` | `ct` (needs exactly one char between), `coat` (two) |

One. Not zero, not two. Most off-by-one regex bugs start here.

### 2.4 Anchors — `^` and `$`

Anchors match a **position**, not a character. They're zero-width: they check
where you are and consume nothing.

| Pattern | Means |
|---|---|
| `^ERROR` | Line **begins** with `ERROR` |
| `;$` | Line **ends** with `;` |
| `^$` | An **empty line** — start immediately followed by end |
| `^ERROR.*timeout$` | Begins with `ERROR`, ends with `timeout` |
| `^cat$` | The line is *exactly* `cat` |

`^` means "start of line" only as the first character of the pattern (or of a
group). Inside `[...]` as the first character it means "not" (section 2.5).
Elsewhere, it's a literal caret. One character, three jobs — position decides.

`grep -c '^$' file` counts blank lines. Genuinely useful.

### 2.5 Character classes — `[...]`

"Exactly one character, chosen from this set."

| Pattern | Matches one character that is… |
|---|---|
| `[aeiou]` | a vowel |
| `[0-9]` | a digit |
| `[A-Za-z]` | a letter — two ranges |
| `[A-Za-z0-9_]` | a "word character" |
| `[^0-9]` | **not** a digit |
| `[.]` | a literal dot — inside brackets `.` loses its power |

Rules that bite:

- **Inside brackets, almost everything is literal.** `[.*+?]` is four literal
  characters. You don't escape inside a class.
- **`^` negates only as the very first character.** `[^abc]` = anything but a, b,
  c. `[a^bc]` includes a literal caret.
- **A literal `]` goes first**: `[]abc]`, or `[^]abc]` when negated.
- **A literal `-` goes first or last**: `[-abc]` or `[abc-]`. In the middle it
  makes a range.
- **`[A-z]` is a bug.** Between `Z` and `a` in ASCII sit `[`, `\`, `]`, `^`,
  `_` and the backtick. `[A-z]` quietly matches all of them. Write `[A-Za-z]`.

### 2.6 POSIX named classes — and why `\d` betrays you

In Java you write `\d` for a digit. **In `grep` and `grep -E`, `\d` isn't a
digit.** GNU grep 3.7 — the one on your Ubuntu 22.04 — treats it as a literal
letter `d`. It doesn't fail. It *matches the wrong thing* and exits 0 — a false
positive, which is worse than an error. (grep 3.8+ at least prints a
"stray \ before d" warning.)

POSIX's portable answer is named classes. They go **inside** brackets, so "one
digit" is `[[:digit:]]` — one pair of brackets for the name, one for the set:

| POSIX | Equivalent | Meaning |
|---|---|---|
| `[[:digit:]]` | `[0-9]` | digit |
| `[[:alpha:]]` | `[A-Za-z]` | letter |
| `[[:alnum:]]` | `[A-Za-z0-9]` | letter or digit |
| `[[:space:]]` | space, tab, … | whitespace |
| `[[:upper:]]` / `[[:lower:]]` | `[A-Z]` / `[a-z]` | case |
| `[[:punct:]]` | `!"#$%&'()*+,-./` … | punctuation |
| `[[:xdigit:]]` | `[0-9A-Fa-f]` | hex digit |

They combine with other characters: `[[:digit:]a-f]`, `[^[:space:]]`.

GNU grep *does* understand `\w` (word character), `\s` (whitespace) and `\b`
(word boundary) as **GNU extensions** — they work on Ubuntu but aren't POSIX, so
don't count on them on macOS or inside a minimal container. `\d` is the one that
is simply missing.

```bash
echo 'backend port 8443' | grep -oE '[0-9]+'         # ERE, portable      -> 8443
echo 'backend port 8443' | grep -oE '[[:digit:]]+'   # ERE, POSIX class   -> 8443
echo 'backend port 8443' | grep -oP '\d+'            # PCRE, needs -P     -> 8443
echo 'backend port 8443' | grep -oE '\d+'            # WRONG: the letter d in "backend"
```

Use `[0-9]`. It's short, works in BRE, ERE and PCRE, and never surprises you.

### 2.7 Quantifiers — "how many of the previous thing"

| Quantifier | Means | Dialect note |
|---|---|---|
| `*` | 0 or more | works in plain `grep` too |
| `+` | 1 or more | needs `-E` |
| `?` | 0 or 1 — "optional" | needs `-E` |
| `{3}` | exactly 3 | needs `-E` |
| `{2,}` | 2 or more | needs `-E` |
| `{2,5}` | 2 to 5 | needs `-E` |

**The rule everyone gets wrong: a quantifier applies to the one atom right
before it — not to everything typed so far.**

| Pattern | Reads as | Matches |
|---|---|---|
| `abc*` | `ab`, then zero-or-more `c` | `ab`, `abc`, `abccc` |
| `(abc)*` | zero-or-more copies of `abc` | empty, `abc`, `abcabc` |
| `[0-9]+` | one or more digits | `7`, `2026` |
| `[0-9]{1,3}` | one to three digits | `7`, `192` |
| `colou?r` | `colo`, optional `u`, `r` | `color`, `colour` |
| `.*` | any characters, including none | anything at all |

`.*` is the workhorse and the footgun. `^ERROR.*timeout` means "starts with
ERROR and has `timeout` somewhere after". And because `*` allows zero,
`grep 'x*'` matches **every line** — every line contains zero `x`s.

### 2.8 Greedy matching

**In plain words:** quantifiers are greedy — `.*` eats as much of the line as it
can, and only gives characters back if the rest of the pattern can't match
otherwise.

Precisely: POSIX regex is **leftmost-longest** — of all possible matches it
picks the one starting earliest, and of those, the longest. On
`<a href="x">link</a>`, the pattern `<.*>` matches the **entire line**, not
`<a href="x">`: `.*` swallows through the middle `>` all the way to the final
one.

```bash
echo '<a href="x">link</a>' | grep -oE '<.*>'       # the whole line
echo '<a href="x">link</a>' | grep -oE '<[^>]*>'    # <a href="x">   then   </a>
```

The fix is not "make it lazy" — POSIX ERE **has no lazy quantifier**. `*?` only
exists in PCRE (`-P`) and Java. The fix is **`[^X]*`**: "any run of characters
that aren't the terminator". It's the most useful trick in this lesson:

| Want | Pattern |
|---|---|
| A double-quoted string | `"[^"]*"` |
| A `[bracketed]` log field | `\[[^]]*\]` |
| A `key=value` value up to the next space | `=[^ ]*` |

### 2.9 Grouping and alternation

`|` means "or". Parentheses control how far the "or" reaches.

```bash
printf 'ERROR db down\nFATAL oom\nINFO a FATAL was logged earlier\n' > /tmp/alt.txt
echo "--- 'ERROR|FATAL'   (either word, anywhere):"; grep -E 'ERROR|FATAL' /tmp/alt.txt
echo "--- '^(ERROR|FATAL)' (line STARTS with either):"; grep -E '^(ERROR|FATAL)' /tmp/alt.txt
echo "--- '^ERROR|FATAL'  (the bug):";                grep -E '^ERROR|FATAL' /tmp/alt.txt
rm -f /tmp/alt.txt
```

The last one matches the `INFO` line. `|` has the **lowest precedence** of any
regex operator, so it splits the *entire* pattern: `^ERROR|FATAL` means
"starts with ERROR" **or** "contains FATAL anywhere". The anchor only belongs to
the left side. This bug ships to alerting rules all the time.

Groups also let a quantifier cover several characters: `(ab)+` matches `ab`,
`abab`, `ababab`.

### 2.10 Backreferences

Groups **capture**: `\1` means "the exact text group 1 matched, again".

```bash
printf 'the the cat\nbook\nnothing here\n' | grep -E '\b([a-z]+) \1\b'   # repeated word
printf 'letter\nbook\ncat\n' | grep -E '(.)\1'                          # doubled character
```

Handy for finding "the the" in docs. It's also the one feature that takes regex
beyond what a finite automaton can do — GNU grep has to fall back to a slower
backtracking matcher, so avoid backrefs on huge files.

### 2.11 Word boundaries

`\b` is a zero-width position where a word character (`[A-Za-z0-9_]`) meets a
non-word character or the line's edge.

```bash
printf 'cat\nconcatenate\nthe cat sat\n' | grep -E '\bcat\b'
printf 'cat\nconcatenate\nthe cat sat\n' | grep -w cat          # same result, easier
```

`-w` is what you'll actually type.

> **Say this in an interview:** "A regex is built from atoms — literals,
> character classes, groups — with quantifiers that bind to the single preceding
> atom, anchors that assert position without consuming input, and alternation,
> which has the lowest precedence and needs a group to scope it. POSIX
> quantifiers are greedy with leftmost-longest semantics and there's no lazy
> operator, so the portable way to stop at a delimiter is a negated class like
> `[^"]*` rather than `.*?`."

---

## Part 3 — BRE vs ERE vs PCRE: the dialect trap

**In plain words:** regex got invented, then reinvented, and the old versions
were kept so old scripts wouldn't break. So the *same pattern string* means
different things depending on a flag. Most "why doesn't my regex work" questions
have one answer: you're in the old dialect.

| Feature | **BRE** — `grep` | **ERE** — `grep -E` | **PCRE** — `grep -P` |
|---|---|---|---|
| `.` `*` `^` `$` `[ ]` | work | work | work |
| `+` `?` | **literal** — need `\+` `\?` | work | work |
| `( )` grouping | need `\(` `\)` | work | work |
| `{ }` counts | need `\{` `\}` | work | work |
| Alternation (the pipe) | needs a backslash before it | works bare | works bare |
| Backrefs `\1` | yes | yes | yes |
| `\w` `\s` `\b` | GNU extension | GNU extension | yes |
| `\d` | **no — matches `d`** | **no — matches `d`** | yes |
| Lazy `*?`, lookahead `(?=…)` | no | no | yes |

```bash
echo "aaa" | grep -c    'a+'     # 0 — BRE: the letter a, then a literal plus sign
echo "aaa" | grep -c    'a\+'    # 1 — in BRE the backslash turns + ON
echo "aaa" | grep -cE   'a+'     # 1 — ERE: + just works
```

In BRE the backslash does the **opposite** of what you'd expect — it *gives*
`+` its power instead of taking it away. That inversion is why BRE feels broken.
Don't fight it.

### The rule

**Always type `-E`.** `egrep` is the old alias for it — obsolete, and GNU grep
3.8+ prints a warning when you use it (Ubuntu 22.04 ships 3.7, so you won't see
it here yet). Reach for `-P` only when you need lookarounds or lazy quantifiers,
and remember it isn't everywhere: macOS grep and the BusyBox grep in Alpine
images lack it — and Alpine is exactly what you'll be `kubectl exec`-ed into
during an incident.

### When you want no regex at all: `-F`

```bash
printf 'app.config.timeout=30\nappXconfigYtimeout=30\n' > /tmp/f.txt
echo "regex (dots match anything):"; grep -c  'app.config.timeout=30' /tmp/f.txt
echo "fixed string:";                grep -cF 'app.config.timeout=30' /tmp/f.txt
rm -f /tmp/f.txt
```

`-F` treats the pattern as **plain text**: `.`, `*` and `[` are just characters.
Use it whenever your search text contains punctuation you mean literally, and
whenever the search text comes from a user or a variable. It's also the fastest
mode. (`fgrep` is its obsolete alias.)

### The Java bridge

Java's `java.util.regex` is PCRE-flavoured — much closer to `grep -P` than to
`grep -E`. So `\d`, `*?` and lookaheads all work in Java and fail in `grep -E`.
Java also adds a second layer of escaping, because `\` is a string escape in
Java source before the regex engine ever sees it:

| Concept | `grep -E` | Java string literal |
|---|---|---|
| One digit | `[0-9]` | `"\\d"` or `"[0-9]"` |
| One or more digits | `[0-9]+` | `"\\d+"` |
| Literal dot | `\.` | `"\\."` |
| Word boundary | `\b` | `"\\b"` |
| Anywhere in the line | default | `matcher.find()` |
| Whole line | `-x` or `^…$` | `matcher.matches()` / `String.matches()` |

Hands-on 15 runs this side by side.

> **Say this in an interview:** "POSIX defines two dialects: BRE, where `+`, `?`,
> braces, parentheses and alternation need backslashes to act as operators, and
> ERE, where they're operators by default. `grep` defaults to BRE, `-E` selects
> ERE, and `-P` switches to PCRE, which is roughly the flavour Java's
> `java.util.regex` implements — that's where `\d`, lazy quantifiers and
> lookarounds live. `-F` opts out of regex entirely for literal matching."

---

## Part 4 — The flags that actually matter

**In plain words:** the pattern is half the tool. The other half is switches
that change *what gets printed* — whole lines, just filenames, just a count, or
the lines around each match. A dozen of them cover almost everything.

| Flag | Long form | Does |
|---|---|---|
| `-i` | `--ignore-case` | Case-insensitive match |
| `-v` | `--invert-match` | Print lines that **don't** match |
| `-n` | `--line-number` | Prefix each line with its line number |
| `-c` | `--count` | Print how many **lines** matched, not the lines |
| `-l` | `--files-with-matches` | Print only names of files that contain a match |
| `-L` | `--files-without-match` | Names of files with **no** match |
| `-w` | `--word-regexp` | Whole words only |
| `-x` | `--line-regexp` | The whole line must match |
| `-r` | `--recursive` | Search a directory tree |
| `-o` | `--only-matching` | Print only the matched text |
| `-E` | `--extended-regexp` | ERE |
| `-F` | `--fixed-strings` | Plain text, no regex |
| `-q` | `--quiet` | Print nothing; exit 0 on first match |
| `-m N` | `--max-count=N` | Stop after N matching lines |
| `-h` / `-H` | | Hide / force the filename prefix |
| `-A N` | `--after-context=N` | Also print N lines **after** each match |
| `-B N` | `--before-context=N` | N lines **before** |
| `-C N` | `--context=N` | N lines on both sides |
| `-e PAT` | `--regexp=PAT` | Mark an argument as the pattern (repeatable) |
| `-f FILE` | `--file=FILE` | Read patterns from a file, one per line |

### Context — the flags you use at 3am

A stack trace line is useless without the line before it and the ten after.

```
grep -n -B 2 -A 10 'NullPointerException' app.log
```

Between separate groups of matches GNU grep prints a `--` line. That confuses
scripts parsing the output; `--no-group-separator` removes it.

### `-c` counts lines, not matches

```bash
echo "cat cat cat" | grep -c cat            # 1 — one matching LINE
echo "cat cat cat" | grep -o cat | wc -l    # 3 — three matches
```

Matters the moment the thing you're counting can appear twice on one line.

### `-r` — searching a codebase

```
grep -rn 'TODO' src/                                    # recurse, with line numbers
grep -rn --include='*.java' '@Transactional' .          # only .java files
grep -rln --exclude-dir={target,.git} 'password' .      # just filenames, skip build output
```

- `-r` follows symlinks only when you name them on the command line; `-R`
  follows every symlink it meets, which can drag it into trees you didn't mean
  to search.
- `--include` / `--exclude` take **globs**, not regexes. Quote them so the shell
  doesn't expand them first.
- `-l` stops reading each file at its first match, so on a big tree it's much
  faster than printing every line.
- `-I` skips binary files — useful with `-r` over a repo with jars in it.

For daily code search, `ripgrep` (`rg`) is faster and respects `.gitignore`, but
it won't be on the server. Its flags are copied from these.

### `-v` — everything except

```bash
grep -vE '^\s*(#|$)' /etc/ssh/ssh_config
```

"Drop lines that are, after optional whitespace, a comment or nothing." What's
left is the settings actually in effect. That's how you read any config file on
a box you've just been paged about. (`\s` is a GNU extension; the portable
spelling is `[[:space:]]`.)

> **Say this in an interview:** "The flags I use most: `-n` for line numbers,
> `-r` with `--include` for code search, `-B`/`-A` for context around a stack
> trace, `-v` to strip comments and blanks out of a config, `-l` when I only need
> which files, and `-c` — remembering it counts matching *lines*, so for
> occurrences I use `-o | wc -l`."

---

## Part 5 — `grep -o`: from filter to extractor

**In plain words:** normally `grep` gives you whole lines and you still have to
spot the interesting bit by eye. `-o` throws the line away and prints **only the
part that matched**, one match per output line. That one flag turns `grep` into
a data-extraction tool, and its output is perfectly shaped for counting.

```
grep -oE '[0-9]{1,3}(\.[0-9]{1,3}){3}' access.log     # every IPv4-looking string
grep -oE '"[^"]*"' app.log                             # every quoted string
grep -oE 'took [0-9]+ms' app.log                       # every timing
```

### The idiom you'll use for the rest of your life

```
grep -oE 'PATTERN' file | sort | uniq -c | sort -rn | head
```

Stage by stage:

1. `grep -oE` — extract every match, one per line
2. `sort` — put identical values next to each other
3. `uniq -c` — collapse each run of identical lines into one, prefixed by its count
4. `sort -rn` — sort by that count: `-n` numeric, `-r` descending
5. `head` — top 10

**`uniq` only collapses *adjacent* duplicates.** That's why step 2 isn't
optional. Forget it and you get a column of 1s and no error — a quiet wrong
answer.

Top IPs by request count, top error messages, slowest endpoints: same five
commands every time.

### A warning about that IP pattern

`[0-9]{1,3}(\.[0-9]{1,3}){3}` also matches `999.999.999.999` and the inside of a
version string like `1.2.3.4`. A strictly correct octet is
`(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])`, which is unreadable. For mining
logs the loose version is right. For *validating input* it's a security bug —
"we checked it with a regex" is how malformed data gets past a filter.

> **Say this in an interview:** "`grep -o` changes the unit of output from the
> line to the match, so it becomes an extractor rather than a filter. Piped into
> `sort | uniq -c | sort -rn` you get a frequency distribution — my first move on
> an unfamiliar log. `uniq` only dedupes adjacent lines, so the first `sort` is
> required."

---

## Part 6 — Quoting: the shell reads your command first

**In plain words:** before `grep` starts, the **shell** reads everything you
typed and rewrites parts of it. `*`, `?`, `[`, `$`, spaces and backslashes all
mean something to the shell, and the shell goes first. An unprotected pattern
can reach `grep` as something completely different — with no error, just a
wrong answer.

### The failure first

```bash
mkdir -p /tmp/day6-parts/q && cd /tmp/day6-parts/q && rm -f ./*
touch ERRORS.log
printf 'ERROR one\nERRRR two\nINFO three\n' > ../q.txt
echo "you typed:       grep ERR* ../q.txt"
echo "grep received:  " grep ERR* ../q.txt
echo "--- result unquoted:"; grep ERR* ../q.txt; echo "exit=$?"
echo "--- result quoted:";   grep 'ERR*' ../q.txt
```

The shell saw `ERR*`, treated it as a glob, found `ERRORS.log` in the current
directory, and substituted it. `grep` searched for the text `ERRORS.log`. Run
the same command in a directory without a matching file and it "works" — which
is exactly why this bug survives testing.

### The rule

| Quoting | The shell | Use for |
|---|---|---|
| `'single'` | Does **nothing** — every character reaches `grep` as typed | **Regex patterns. Default.** |
| `"double"` | Expands `$var` and `$(cmd)`; blocks globbing and splitting | A pattern that must include a shell variable |
| bare | Globs, splits on spaces, expands `$` | Never, for a pattern |

```
grep -E '^ERROR.*timeout$' app.log      # correct
grep -E "^$LEVEL.*timeout" app.log      # correct when you need the variable
```

Inside double quotes the shell also looks at your regex's `$` and `\` — usually
harmlessly, occasionally not. Single quotes remove the whole question.

### Two more argument traps

**A pattern with a space must be quoted**, or the shell splits it into two
arguments and `grep` takes the second as a filename:

```
grep class Foo Bar.java      # pattern "class", files "Foo" and "Bar.java"
grep 'class Foo' Bar.java    # pattern "class Foo"
```

**A pattern starting with `-`** looks like a flag. Use `-e` or `--`:

```
grep -e '-v' notes.txt       # search for the text "-v"
grep -- '-v' notes.txt       # same
```

Hands-on 11 runs both.

> **Say this in an interview:** "Shell expansion happens before the command
> runs, so an unquoted regex can be glob-expanded or word-split before `grep`
> ever sees it. I single-quote patterns by default, use double quotes only when I
> need a variable inside the pattern, and use `-e` or `--` when a pattern starts
> with a dash."

---

## Part 7 — `grep` in real pipelines

**In plain words:** `grep` was designed to sit in the middle of a chain — lines
in from the left, fewer lines out to the right. A few shapes come up constantly,
and two of them have famous failure modes.

### Chaining filters

```
journalctl -u myapp | grep ERROR | grep -v 'health-check'
```

Each `grep` narrows the stream further. Where the logic allows, one pass beats
two: `grep -E 'ERROR|FATAL'` rather than two separate greps.

### The `ps aux | grep` self-match

```
ps aux | grep java
```

This almost always shows one extra line: **the `grep` itself**. By the time `ps`
takes its snapshot, `grep java` is already running, and its own command line
contains `java`. Two fixes:

```
ps aux | grep '[j]ava'    # the regex [j]ava matches "java", but grep's own
                          # command line contains "[j]ava", which it doesn't
pgrep -a java             # the right tool: no pipe, never matches itself
```

Interviewers like the `[j]ava` trick because explaining it proves you
understand both the process list (Day 4) and character classes.

### `tail -f | grep | …` prints nothing — the buffering trap

This one costs real minutes during an incident:

```
tail -f app.log | grep ERROR | tee errors.txt       # silence, even as errors arrive
```

When `grep`'s output goes to a **terminal**, it flushes after every line. When
its output goes into a **pipe**, it switches to **block buffering**: it saves
output up and sends it in chunks of several kilobytes. On a quiet log that can
mean minutes of nothing.

```
tail -f app.log | grep --line-buffered ERROR | tee errors.txt
```

`--line-buffered` forces a flush per line. For commands without such a flag,
`stdbuf -oL cmd` does the same. The rule: the **last** command in a pipe is fine;
a `grep` with **anything after it** needs `--line-buffered` if you want to watch
live.

### `grep` as a test in scripts

```
if grep -qE '^spring\.profiles\.active=prod' application.properties; then
  echo "prod profile active"
fi

errors=$(grep -c ERROR app.log || true)   # grep -c prints 0 but EXITS 1 on no match
```

> **Say this in an interview:** "Two pipeline gotchas I always check: `ps aux |
> grep foo` matches the grep process itself, so I use `pgrep` or the `[f]oo`
> bracket trick; and `grep` block-buffers when its stdout is a pipe instead of a
> terminal, so live-tailing into another command needs `--line-buffered` or
> `stdbuf -oL`."

---

## Hands-on

Open the **Ubuntu (WSL)** tab. **Run block 1 first** — the rest use the log it
creates. Blocks 1–7 are the plan's required practice (count ERRORs, context,
filenames, extract IPs). 8–15 are worth doing if you have time.

### 1. Build a log to work on

```bash
mkdir -p /tmp/day6 && cd /tmp/day6
cat > app.log <<'EOF'
2026-09-11 08:14:02 INFO  [http-nio-8080-exec-1] c.a.OrderController - GET /api/orders 200 from 10.0.3.14 in 12ms
2026-09-11 08:14:03 WARN  [http-nio-8080-exec-2] c.a.PaymentClient - retry 1/3 for txn 8821 from 10.0.3.14
2026-09-11 08:14:05 ERROR [http-nio-8080-exec-2] c.a.PaymentClient - connect timeout to 192.168.10.7:8443
2026-09-11 08:14:05 ERROR [http-nio-8080-exec-2] c.a.OrderController - POST /api/orders 500 from 10.0.3.14 in 2043ms
2026-09-11 08:15:11 INFO  [http-nio-8080-exec-3] c.a.OrderController - GET /api/orders 200 from 10.0.3.91 in 9ms
2026-09-11 08:15:12 INFO  [http-nio-8080-exec-3] c.a.UserService - user="alice" action="login" ok
2026-09-11 08:16:40 ERROR [scheduler-1] c.a.ReportJob - java.lang.NullPointerException
2026-09-11 08:16:40 ERROR [scheduler-1] c.a.ReportJob -     at c.a.ReportJob.build(ReportJob.java:87)
2026-09-11 08:16:40 ERROR [scheduler-1] c.a.ReportJob -     at c.a.ReportJob.run(ReportJob.java:41)
2026-09-11 08:17:02 INFO  [http-nio-8080-exec-4] c.a.OrderController - GET /api/health 200 from 127.0.0.1 in 1ms
2026-09-11 08:17:33 WARN  [http-nio-8080-exec-5] c.a.RateLimiter - throttled 10.0.3.14 (11 req/s)
2026-09-11 08:18:04 ERROR [http-nio-8080-exec-5] c.a.OrderController - GET /api/orders 503 from 10.0.3.14 in 30001ms
2026-09-11 08:18:20 INFO  [http-nio-8080-exec-6] c.a.UserService - user="bob" action="logout" ok
2026-09-11 08:19:00 DEBUG [scheduler-2] c.a.CacheWarmer - warming 240 keys
2026-09-11 08:19:59 ERROR [http-nio-8080-exec-7] c.a.PaymentClient - connect timeout to 192.168.10.8:8443
2026-09-11 08:20:15 INFO  [http-nio-8080-exec-8] c.a.OrderController - GET /api/orders 200 from 172.16.4.2 in 15ms

2026-09-11 08:21:00 WARN  [http-nio-8080-exec-9] c.a.PaymentClient - retry 3/3 for txn 8821 from 10.0.3.14
EOF
wc -l app.log
```

### 2. Count ERROR lines — and prove `-c` counts lines

```bash
cd /tmp/day6
echo "matching lines: $(grep -c 'ERROR' app.log)"
echo "total matches:  $(grep -o 'ERROR' app.log | wc -l)"
grep -n 'ERROR' app.log | head -3
```

They agree — one `ERROR` per line. Now make them disagree:

```bash
cd /tmp/day6
echo "cat cat cat" > twice.txt
echo "lines:   $(grep -c cat twice.txt)"
echo "matches: $(grep -o cat twice.txt | wc -l)"
```

### 3. Context around a stack trace

```bash
cd /tmp/day6
grep -n -C 3 'NullPointerException' app.log
echo "=========== now 1 before, 5 after:"
grep -n -B 1 -A 5 'NullPointerException' app.log
```

### 4. Only the filenames containing a pattern

```bash
cd /tmp/day6
cp app.log copy.log
echo "2026-09-11 08:30:00 INFO [main] c.a.Boot - started" > clean.log
echo "--- files that contain ERROR:";        grep -rl 'ERROR' .
echo "--- files that do NOT:";               grep -rL 'ERROR' .
echo "--- only *.log files mentioning timeout:"; grep -rl --include='*.log' 'timeout' .
```

### 5. Extract every IP address

```bash
cd /tmp/day6
grep -oE '[0-9]{1,3}(\.[0-9]{1,3}){3}' app.log
```

Then rank them:

```bash
cd /tmp/day6
grep -oE '[0-9]{1,3}(\.[0-9]{1,3}){3}' app.log | sort | uniq -c | sort -rn
echo "=========== same thing WITHOUT the first sort:"
grep -oE '[0-9]{1,3}(\.[0-9]{1,3}){3}' app.log | uniq -c | sort -rn
```

Compare the two. `10.0.3.14` appears several times in the second list, each with
a small count — `uniq` only merged the copies that happened to be adjacent.

### 6. Anchored vs unanchored — feel the difference

```bash
cd /tmp/day6
echo "contains ERROR anywhere:  $(grep -c 'ERROR' app.log)"
echo "line STARTS with ERROR:   $(grep -c '^ERROR' app.log)"
echo "level field is ERROR:     $(grep -cE '^[0-9-]+ [0-9:]+ ERROR' app.log)"
echo "blank lines:              $(grep -c '^$' app.log)"
```

The second is 0 — every line starts with a date. Getting the position wrong
doesn't error. It returns a confident zero.

### 7. BRE vs ERE vs PCRE, demonstrated

```bash
cd /tmp/day6
echo "BRE, + is literal:      $(grep -c   'exec-[0-9]+' app.log)"
echo "BRE with \\+:            $(grep -c   'exec-[0-9]\+' app.log)"
echo "ERE:                    $(grep -cE  'exec-[0-9]+' app.log)"
echo "--- what \\d+ extracts under -E (you wanted digits):"
grep -oE '\d+' app.log | sort | uniq -c
echo "--- what \\d+ extracts under -P:"
grep -oP '\d+' app.log | head -5
```

Under `-E`, `\d` found the letter `d` in `OrderController`, `build`,
`CacheWarmer`… and reported success. That's why the Java habit of `\d` is so
dangerous here.

### 8. The negated-class idiom

```bash
cd /tmp/day6
echo "--- greedy \".*\" runs from the FIRST quote to the LAST:"
grep -oE '".*"' app.log
echo "--- \"[^\"]*\" stops at the first closing quote:"
grep -oE '"[^"]*"' app.log
echo "--- every thread name, deduplicated:"
grep -oE '\[[^]]+\]' app.log | sort -u
```

### 9. Status codes, ranked

```bash
cd /tmp/day6
grep -oE '/api/[a-z]+ [0-9]{3}' app.log | grep -oE '[0-9]{3}$' | sort | uniq -c | sort -rn
```

Two greps: the first extracts `/api/orders 200` so you don't accidentally catch
`240` from `warming 240 keys`; the second pulls the three digits off the end.

### 10. Read a config the way you will on a live box

```bash
echo "total lines:     $(wc -l < /etc/ssh/ssh_config)"
echo "effective lines: $(grep -cvE '^\s*(#|$)' /etc/ssh/ssh_config)"
grep -vE '^\s*(#|$)' /etc/ssh/ssh_config
```

### 11. The argument traps

```bash
cd /tmp/day6
echo "--- unquoted pattern with a space:"
grep connect timeout app.log; echo "exit=$?"
echo "--- quoted:"
grep 'connect timeout' app.log
echo "--- pattern that starts with a dash:"
echo 'pass -v for verbose output' > flags.txt
grep '-v' flags.txt < /dev/null; echo "exit=$?"
grep -e '-v' flags.txt
```

Read the first result carefully: `grep` complained that `timeout` isn't a file,
and matched `connect` in `app.log`. In the dash case, `grep` took `-v` as the
invert flag and `flags.txt` as the *pattern*, then waited for input on stdin —
in a terminal it would just hang. The `< /dev/null` here stops that.

### 12. `grep -q` as a test

```bash
cd /tmp/day6
if grep -q 'NullPointerException' app.log; then echo "NPE present"; fi
grep -q 'CompletelyAbsentString' app.log; echo "no-match exit status: $?"
grep -q 'x' /no/such/file;              echo "error exit status: $?"
```

### 13. The `ps aux | grep` self-match

```bash
sleep 300 &
pid=$!
echo "--- naive (note the extra grep line):"; ps aux 2>/dev/null | grep 'sleep 300'
echo "--- bracket trick:";                    ps aux 2>/dev/null | grep '[s]leep 300'
echo "--- the right tool:";                   pgrep -a -f 'sleep 300'
kill "$pid"
```

(`2>/dev/null` only hides a harmless `ps` warning about screen size that appears
when there's no real terminal, as in the lab runner.)

### 14. The buffering trap, with timestamps

Each line is written one second apart. Watch *when* it reaches the end of the
pipe.

```bash
cd /tmp/day6
stamp() { while IFS= read -r l; do echo "  $(date +%T) received: $l"; done; }
writer() { for i in 1 2 3 4 5; do echo "line $i ERROR boom" >> "$1"; sleep 1; done; }

: > live1.log; : > live2.log
echo "=== WITHOUT --line-buffered (started $(date +%T)):"
writer live1.log & timeout 7 tail -n +1 -f live1.log | grep ERROR | stamp; wait
echo "=== WITH --line-buffered (started $(date +%T)):"
writer live2.log & timeout 7 tail -n +1 -f live2.log | grep --line-buffered ERROR | stamp; wait
```

In the first run, all five lines arrive with the same timestamp — only when
`timeout` kills `tail` and `grep` flushes on exit. In the second, one per second.

### 15. Java: the same ideas, the other dialect

```java
import java.util.regex.*;

public class Main {
    public static void main(String[] args) {
        String line = "2026-09-11 08:14:05 ERROR [http-nio-8080-exec-2] connect timeout to 192.168.10.7:8443";

        // find() = anywhere in the string — this is grep
        System.out.println("find(\"ERROR\"):     " + Pattern.compile("ERROR").matcher(line).find());
        // matches() = the WHOLE string — this is grep -x
        System.out.println("matches(\"ERROR\"):  " + line.matches("ERROR"));
        System.out.println("matches(\".*ERROR.*\"): " + line.matches(".*ERROR.*"));

        // \d works here (PCRE-flavoured) — but needs \\ because \ is also a Java string escape
        Matcher ip = Pattern.compile("\\b\\d{1,3}(\\.\\d{1,3}){3}\\b").matcher(line);
        while (ip.find()) System.out.println("IP:                " + ip.group());

        // negated class + a capture group: group(1) is what's inside the brackets
        Matcher thread = Pattern.compile("\\[([^\\]]+)\\]").matcher(line);
        if (thread.find()) System.out.println("thread (group 1):  " + thread.group(1));

        // same idiom as grep -oE '"[^"]*"'
        Matcher q = Pattern.compile("\"[^\"]*\"").matcher("user=\"alice\" action=\"login\"");
        while (q.find()) System.out.println("quoted:            " + q.group());
    }
}
```

`find()` is `grep`. `matches()` is `grep -x`. The `\d` that silently lied under
`grep -E` works here — Java's regex is PCRE-flavoured.

### 16. Clean up

```bash
rm -rf /tmp/day6 /tmp/day6-parts
```

---

## Gotchas

1. **`grep` matches substrings, not lines.** `grep cat` finds `concatenate`. Use
   `-w` for words, `-x` or `^…$` for whole lines.
2. **`.` means any character — including the dot you meant.**
   `192.168.1.1` also matches `192x168y1z1`. Escape it: `192\.168\.1\.1`.
3. **`\d` is not a digit in `grep` or `grep -E`.** On grep 3.7 it matches the
   letter `d` and exits 0 — a false positive. Use `[0-9]`, or `-P`.
4. **`+ ? { } ( )` and the pipe are literals in plain `grep`.** A pattern that
   works in Java but not in the shell usually means you forgot `-E`.
5. **`|` has the lowest precedence.** `^ERROR|FATAL` is not `^(ERROR|FATAL)`.
6. **`-c` counts matching lines, not matches.** For occurrences:
   `grep -o … | wc -l`.
7. **`uniq -c` only merges adjacent duplicates.** Always `sort` first.
8. **Unquoted patterns get rewritten by the shell** — globbed, split, expanded.
   Single-quote by default.
9. **A pattern with a space becomes two arguments**, and the second is treated
   as a filename.
10. **A pattern starting with `-` becomes a flag.** Use `-e` or `--`.
11. **`grep` block-buffers into a pipe.** `tail -f | grep X | …` looks dead. Add
    `--line-buffered`.
12. **Exit 1 means "no match", and `set -e` kills the script on it.** Add
    `|| true` when finding nothing is a legitimate outcome.
13. **`ps aux | grep foo` finds itself.** Use `pgrep`, or `[f]oo`.
14. **`[A-z]` is not "all letters"** — it includes `[`, `\`, `]`, `^`, `_` and
    the backtick. Write `[A-Za-z]`.
15. **`.*` is greedy and ERE has no lazy form.** Use `[^X]*` to stop at a
    delimiter.
16. **`x*` matches every line**, because every line contains zero `x`s.
17. **`grep -P` isn't everywhere** — not on macOS, not in Alpine/BusyBox.
18. **Binary files**: `grep` prints `binary file matches` instead of the line.
    `-a` forces text mode; `-I` skips binaries.

---

## Recap

- `grep` is a **line filter**: it tests a regex against each line and prints the
  matches. Matching is **unanchored** — substring semantics, like Java's
  `find()`, not `matches()`.
- It also reports through its **exit status**: 0 matched, 1 no match, 2 error.
  `grep -q` is the shell's yes/no question.
- A regex is **literals** plus about a dozen **metacharacters**: `.` any
  character; `^` `$` anchors; `[...]` classes; `* + ? {n,m}` quantifiers that
  bind to the **one atom before them**; `( )` groups; `|` alternation with the
  **lowest precedence**; `\1` backreferences.
- Quantifiers are **greedy** (leftmost-longest), POSIX has **no lazy form**, and
  the portable way to stop at a delimiter is the **negated class** `[^X]*`.
- Three dialects: **BRE** (plain `grep`; `+ ? ( ) { }` and the pipe need
  backslashes), **ERE** (`-E` — always use it), **PCRE** (`-P`; `\d`, lazy,
  lookarounds; what Java resembles). **`-F`** turns regex off.
- Flags that carry the day: `-i -v -n -c -l -w -r -o -E -F -q` and `-A/-B/-C`.
- `-o` makes `grep` an **extractor**; `grep -oE … | sort | uniq -c | sort -rn`
  is frequency analysis in one line.
- **Single-quote patterns** — the shell expands before `grep` runs.
- Pipeline traps: `ps aux | grep` finding itself, and block buffering silencing
  `tail -f | grep | …` (`--line-buffered`).

---

## Say it out loud

About 30 seconds each, out loud, using the real words:

1. Explain what `grep` does in terms of lines and exit status — and why
   `grep cat` finds `concatenate`.
2. A colleague's `grep '[0-9]+'` finds nothing in a file full of numbers.
   Diagnose it, name both dialects, and give two fixes.
3. Explain why `^ERROR|FATAL` is a bug in an alerting rule and what it actually
   matches.
4. Explain greedy matching using `<.*>` on an HTML line, then explain the
   `[^>]*` fix without using the word "lazy".
5. Walk through `grep -oE … | sort | uniq -c | sort -rn | head` stage by stage,
   and say why the first `sort` is mandatory.
6. "`tail -f app.log | grep ERROR | tee errors.txt` is broken, nothing prints."
   What's actually happening and what's the fix?
7. Why do you single-quote a regex in the shell, and when would you use double
   quotes instead?

---

## Q&A
