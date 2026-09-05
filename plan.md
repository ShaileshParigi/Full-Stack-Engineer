# Day-Wise Learning Plan

**How to use this:**
- Each day = 1–2 hrs, **weekdays only** (5 days/week)
- When you're ready for a day: say **"Teach me Phase 1 Day 3"** — the lesson,
  examples and snippets get written into `phase-N-topic/day-K-topic.md`
- Ask for a quiz at any week boundary or end of phase
- Linux work happens in the **Ubuntu (WSL)** tab; Windows-side commands are
  given in PowerShell
- Phases 9–21 are outlined at the bottom and get expanded here as you reach them

---

## Phase 1: Linux & the Shell

**Duration:** 2 weeks — Days 1–10
**Environment:** WSL2 Ubuntu
**Goal:** Be genuinely comfortable on a Linux box — navigate, inspect, control
processes, and write a script you'd be willing to run in production.

---

### Week 1: Living in the Shell

---

#### Day 1 — WSL2 Setup & the Filesystem Hierarchy
**Covers:**
- Confirming your WSL2 Ubuntu setup; `wsl --version`, `/etc/wsl.conf`, enabling `systemd`
- Where Windows drives appear (`/mnt/c`, `/mnt/d`) and why I/O there is slow
- What each top-level directory is for: `/etc`, `/var`, `/tmp`, `/proc`, `/sys`, `/usr`, `/home`, `/opt`
- Why `/proc` is not a real filesystem — live kernel and process state
- Where configs, logs and runtime data actually live
- Home directory, `~`, and the dotfiles that configure your shell

**Practice:** Explore each top-level directory and write one line on what it
holds. Find your shell's PID, then read `/proc/<pid>/status`, `cmdline` and
`environ`. Compare how long `ls -R` takes in `~` versus `/mnt/d`.

---

#### Day 2 — Navigation, Finding Files & Inspecting Them
**Covers:**
- `cd`, `pwd`, absolute vs relative paths, `-` to jump back
- Globbing: `*`, `?`, `[]`, `{}`, and how the *shell* expands them before the command runs
- `ls` flags worth knowing: `-la`, `-lh`, `-lt`, `-R`
- `find`: by name, type, size, mtime, and `-exec`
- `which`, `type`, `whereis`, `locate`
- Reading files: `cat`, `less`, `head`, `tail -f`, `wc`
- `file`, `stat`, `du -sh`, `df -h`

**Practice:** Find every `.log` file under `/var` modified in the last 7 days.
Find the 5 largest files in your home directory. Use `tail -f` on a file while
appending to it from a second terminal.

---

#### Day 3 — Permissions & Ownership
**Covers:**
- The permission triad: user / group / others, read / write / execute
- What execute means on a *directory* (and why it's not what you'd guess)
- `chmod` numeric and symbolic; `chown`, `chgrp`
- `umask` and default permissions
- `sudo`, root, `/etc/passwd`, `/etc/group`
- setuid / setgid / sticky bit — what they are and why `/tmp` has one
- The permissions errors you'll actually hit: a script that won't run, a JAR
  that can't write its log

**Practice:** Create a script, remove execute permission, watch it fail, fix it
three different ways. Create a directory with `r--` and discover you can't `cd`
into it. Set `umask 077` and observe what new files look like.

---

#### Day 4 — Processes, Signals & Jobs
**Covers:**
- What a process is; PID, PPID, the process tree (`pstree`)
- `ps aux` and `ps -ef` — reading every column
- `top` / `htop`: load average, `%CPU` vs `%MEM`, what iowait means
- **Signals**: `SIGTERM` vs `SIGKILL` vs `SIGHUP` vs `SIGINT`
- Why `SIGTERM` matters — your Spring Boot app will need to handle it for
  graceful shutdown in Kubernetes (Phase 17)
- `kill`, `kill -9`, `pkill`, `killall`
- Foreground/background: `&`, `Ctrl+Z`, `jobs`, `fg`, `bg`, `nohup`
- Exit codes and `$?`

**Practice:** Start `sleep 999` in the background, find it with `ps`, send
`SIGTERM`, then repeat with a process that ignores it and escalate to
`SIGKILL`. Write a tiny script that traps `SIGTERM` and cleans up before
exiting — then prove `kill -9` gives it no such chance.

---

#### Day 5 — Packages, Services & Scheduled Jobs + Week 1 Review
**Covers:**
- `apt`: `update` vs `upgrade`, `install`, `remove`, `purge`, `search`, `show`
- Where an installed package puts its files (`dpkg -L`)
- Repositories and `/etc/apt/sources.list.d`
- `systemctl`: `status`, `start`, `stop`, `enable`, `restart` (needs `systemd=true`)
- Reading logs with `journalctl -u <unit>`
- `cron` and crontab syntax; `@reboot`; where cron output goes
- Week 1 recap: filesystem → permissions → processes → packages

**Practice:** Install `jq` and `tree` with apt and find where they landed.
Write a cron job that appends a timestamp to a file every minute, watch it run
for three minutes, then remove it.

**End of Week 1 — ask for a quiz if you want to test yourself.**

---

### Week 2: Text Processing & Scripting

---

#### Day 6 — grep & Regular Expressions
**Covers:**
- Basic vs extended regex; when you need `-E`
- Anchors, character classes, quantifiers, groups, alternation
- `grep` flags that matter: `-i`, `-r`, `-v`, `-n`, `-c`, `-l`, `-w`, `-A/-B/-C`
- `grep -o` for extracting rather than matching
- Combining with pipes; why quoting your pattern matters

**Practice:** Take a real log file (generate one if needed) and: count ERROR
lines, show 3 lines of context around each, list only filenames containing a
pattern, and extract all IP addresses with `-oE`.

---

#### Day 7 — sed & awk
**Covers:**
- `sed`: `s/find/replace/`, the `g` flag, delimiters other than `/`
- Deleting lines, printing ranges, `-i` for in-place edits (and backing up first)
- `awk` mental model: pattern → action, run per line
- Fields (`$1`, `$NF`), `NR`, `NF`, `-F` for a custom separator
- Filtering by field value; `BEGIN` and `END` blocks; summing a column
- When to reach for awk instead of a pipeline of five tools

**Practice:** From an access log, use awk to compute total bytes per status
code and print the top 5 client IPs by request count. Use `sed` to rewrite a
config value in place.

---

#### Day 8 — jq & Building Pipelines
**Covers:**
- Why JSON needs a dedicated tool
- `jq` basics: `.field`, `.[]`, `.[] | .name`, nested access
- `select()`, `map()`, `length`, `keys`
- Output control: `-r` for raw strings, `-c` for compact
- Constructing new objects
- Combining `curl | jq`
- Pipes, `tee`, `xargs`, and command substitution `$( )`

**Practice:** Hit a public JSON API with `curl`, extract three fields, filter by
a condition, and reformat into CSV. Then pipe a list of IDs into `xargs` to
fetch each one.

---

#### Day 9 — Shell Scripting
**Covers:**
- Shebang, making a script executable, `$PATH`
- Variables, quoting rules — why `"$var"` and never bare `$var`
- Command substitution, arithmetic
- `if`/`elif`/`else`, test operators (`-f`, `-d`, `-z`, `-n`), `[ ]` vs `[[ ]]`
- `for`, `while`, `case`
- Functions, arguments (`$1`, `$@`, `$#`), `return` vs `exit`
- **`set -euo pipefail`** — what each flag does and why every script starts with it
- `trap` for cleanup on exit
- Reading input, here-docs

**Practice:** Write a script that takes a directory as an argument, validates
it exists, counts files by extension, and exits with a meaningful code. Add
`set -euo pipefail` and a `trap` that removes a temp file. Deliberately break
it with an unquoted variable containing a space.

---

#### Day 10 — Checkpoint Project + Phase 1 Quiz
**Build:** A log-watching script that:
- Tails a log file continuously
- Filters lines matching a configurable pattern
- Extracts fields with `awk`
- Counts occurrences in a rolling window
- Sends an HTTP POST with `curl` when a threshold is crossed
- Uses `set -euo pipefail`, a `trap` for cleanup, and meaningful exit codes
- Accepts configuration via arguments or environment variables

**Then:** Phase 1 quiz covering everything from Days 1–9.

---

## Phase 2: Networking Fundamentals

**Duration:** 2 weeks — Days 11–20
**Goal:** Understand how services find and talk to each other, so that
Kubernetes networking and Istio mTLS later are comprehensible rather than magic.

---

### Week 3: Addresses, Transport & Names

---

#### Day 11 — The Layered Model & Encapsulation
**Covers:**
- OSI's 7 layers vs the practical TCP/IP 4-layer view
- Focus on L3 (IP), L4 (TCP/UDP), L7 (HTTP) — and what L1/L2 give you
- Encapsulation: what wraps what as data goes down the stack
- MAC addresses and ARP (local delivery) vs IP (global routing)
- Where each piece of infrastructure sits: switch, router, firewall, load
  balancer, proxy
- Why "it's a networking problem" usually means one specific layer

**Practice:** Run `ip addr`, `ip route`, and `ip neigh` in WSL and explain every
line. Draw the path a request takes from your browser to a server in AWS.

---

#### Day 12 — IP Addressing, Subnets & CIDR
**Covers:**
- IPv4 structure, binary intuition, dotted quads
- Private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — and why
- **CIDR notation**: reading `/24`, `/16`, `/26` fluently
- Network address, broadcast, usable host range
- Subnetting: splitting a range, and how many hosts you get
- Why this matters concretely: VPC subnet planning (Phase 15) and pod CIDRs (Phase 17)
- NAT: why your laptop and 50 others share one public IP
- IPv6 at awareness level

**Practice:** Without a calculator, give the usable range and host count for
`10.0.1.0/24`, `172.31.0.0/20` and `192.168.5.64/26`. Then plan a VPC: one
`/16` split into two public and two private `/20` subnets across two AZs.

---

#### Day 13 — TCP, UDP & Connection State
**Covers:**
- Ports: well-known, ephemeral, and what "listening" means
- TCP guarantees: ordering, reliability, flow control, congestion control
- The **three-way handshake**, and what a SYN flood exploits
- Connection teardown, `FIN`/`RST`, and `TIME_WAIT` — why it appears in load tests
- Connection states you'll see in `ss` output
- Head-of-line blocking
- UDP: no handshake, no ordering — and who wants that (DNS, QUIC, video)
- **Connection pooling** — why HikariCP and your HTTP client both have one, and
  what happens without it

**Practice:** Use `ss -tan` to inspect live connections. Start a listener with
`nc -l 9000`, connect from another terminal, and watch the state change through
the handshake and teardown. Find sockets stuck in `TIME_WAIT`.

---

#### Day 14 — DNS
**Covers:**
- What resolution actually does; the resolver chain
- Recursive vs authoritative servers; root → TLD → authoritative
- Record types: A, AAAA, CNAME, MX, TXT, NS, SRV
- **TTL and caching** — the cause of half of all "but it works on my machine"
- `/etc/hosts` and `/etc/resolv.conf` (and WSL's generated one)
- `dig` in depth: `+short`, `+trace`, querying a specific server
- Split-horizon DNS; internal service discovery as a preview of Kubernetes DNS
- DNS as a failure domain — when DNS is down, everything looks down

**Practice:** `dig +trace` a domain and narrate each step. Compare `dig` results
against different resolvers (`@8.8.8.8`, `@1.1.1.1`). Add an entry to
`/etc/hosts` and prove it takes precedence.

---

#### Day 15 — Diagnostics Toolkit + Week 3 Review
**Covers:**
- `ping` and ICMP — and why a blocked ping doesn't mean a host is down
- `traceroute` / `mtr` and reading hops
- `ss` / `netstat` for sockets, `lsof -i` for what owns a port
- `nc` for testing raw connectivity
- `telnet` as a poor man's protocol client
- A decision tree: is it DNS, routing, the port, TLS, or the app?
- Week 3 recap: addresses → transport → names

**Practice:** Given a "the service is unreachable" scenario, work the decision
tree end to end. Deliberately break each layer (bad DNS entry, closed port,
wrong route) and confirm each tool identifies the right one.

**End of Week 3 — quiz available.**

---

### Week 4: HTTP, TLS & the Edge

---

#### Day 16 — HTTP Deep Dive
**Covers:**
- Request/response anatomy: method, path, version, headers, body
- Methods and their semantics; safe vs idempotent (and why it matters for retries)
- Status codes that carry real meaning: 201, 204, 301 vs 302, 400 vs 422,
  401 vs 403, 409, 429, 502 vs 503 vs 504
- Headers you'll live with: `Content-Type`, `Accept`, `Authorization`,
  `Cache-Control`, `ETag`, `X-Forwarded-For`
- Cookies and sessions
- Keep-alive and connection reuse
- HTTP/1.1 vs HTTP/2 (multiplexing) vs HTTP/3 (QUIC)
- Chunked transfer and streaming — the mechanism behind SSE, which your AI
  endpoint will use in Phase 9

**Practice:** Use `curl -v` to inspect a full exchange. Trigger a 301 and follow
it with `-L`. Send a conditional request with `If-None-Match` and get a 304.

---

#### Day 17 — TLS & Certificates
**Covers:**
- What TLS provides: confidentiality, integrity, authentication
- Symmetric vs asymmetric crypto, and why TLS uses both
- **The handshake**: ClientHello → ServerHello → certificate → key exchange
- Certificates: subject, issuer, validity, SAN
- **Chain of trust**: leaf → intermediate → root CA; the system trust store
- Why self-signed certs fail, and when that's fine
- SNI — how one IP serves many HTTPS sites
- TLS termination at a proxy vs end-to-end encryption
- **mTLS preview**: both sides present certificates — the foundation of Phase 18
- Certificate expiry as a leading cause of outages

**Practice:** Inspect a real site's chain with `openssl s_client -connect`.
Generate a self-signed cert, serve HTTPS with it, watch `curl` reject it, then
accept it with `-k` and by adding the CA. Decode a certificate with
`openssl x509 -text`.

---

#### Day 18 — curl Mastery & Packet Inspection
**Covers:**
- `curl` as your primary API tool: `-v`, `-i`, `-H`, `-X`, `-d`, `--data-binary`
- Auth: `-u`, bearer tokens
- `-o`, `-O`, `--resolve`, `--connect-timeout`, `--max-time`
- **`-w`** for timing breakdown: DNS, connect, TLS, first byte, total — the
  fastest way to answer "which part is slow?"
- Reusable request files and `curl` config
- `tcpdump` basics: capturing, filtering by host and port, reading output
- Why you can't read HTTPS payloads in a capture (and what you can still learn)

**Practice:** Write a `curl -w` format file that prints the full timing
breakdown, and use it to compare a local service against a remote one. Capture
DNS traffic with `tcpdump` while running `dig`.

---

#### Day 19 — Ports, Firewalls & NAT
**Covers:**
- Binding: `0.0.0.0` vs `127.0.0.1` vs a specific interface — and the security
  difference
- Port conflicts and finding the offender
- `ufw` / `iptables` conceptually: chains, rules, default policies
- Stateful vs stateless filtering
- How this maps forward: AWS **Security Groups** (stateful) vs **NACLs**
  (stateless) in Phase 15, and Kubernetes NetworkPolicies in Phase 17
- Port forwarding and SSH tunnels
- WSL2 networking: localhost forwarding from Windows, and its limits

**Practice:** Bind a service to `127.0.0.1` and confirm it's unreachable from
Windows; rebind to `0.0.0.0` and confirm it is. Add a `ufw` rule to block a
port and verify with `nc`.

---

#### Day 20 — Checkpoint Project + Phase 2 Quiz
**Build:** A connectivity diagnostic report for a service you run in WSL:
- Document the full request path from Windows browser to the process
- Capture and explain the TLS handshake with `openssl s_client`
- A `curl`-based health-check script with a full timing breakdown and
  threshold-based exit codes
- A `tcpdump` capture showing the TCP handshake
- A written troubleshooting runbook: symptom → tool → likely cause

**Then:** Phase 2 quiz.

---

## Phase 3: Java Core Rebuild

**Duration:** 4 weeks — Days 21–40
**Goal:** Build a real Java foundation from the ground up, including a full week
on Java 8 functional programming. Everything after this phase is Java, so this
is the one to take slowly.

---

### Week 5: How Java Works Underneath

---

#### Day 21 — The JVM, Memory & References
**Covers:**
- JVM, JRE, JDK; source → bytecode → execution
- **Stack vs heap** — what lives where; each thread gets its own stack (the
  model Phase 5 depends on)
- References vs objects; what "pass by value" actually means in Java
- Primitives vs wrappers; autoboxing and its hidden cost; the `Integer` cache
- Garbage collection: reachability, generations, minor vs major
- How memory leaks still happen in a garbage-collected language
- `OutOfMemoryError` vs `StackOverflowError`

**Practice:** Write one method that mutates an object passed in and one that
reassigns it — explain why only one change is visible to the caller. Cause a
`StackOverflowError` with recursion. Compare `Integer` caching with `==` on 127
and 128 and explain the result.

---

#### Day 22 — Identity, equals/hashCode & String
**Covers:**
- `==` vs `.equals()` — reference identity vs logical equality
- The **`equals` contract**: reflexive, symmetric, transitive, consistent
- The **`hashCode` contract** and why it must agree with `equals`
- What actually breaks inside a `HashMap`/`HashSet` when they disagree
- Writing both correctly with `Objects.equals` and `Objects.hash`
- `String` immutability, the string pool, interning
- Concatenation in a loop vs `StringBuilder`

**Practice:** Write a class with `equals` but no `hashCode`, add the same
logical object to a `HashSet` twice, and watch it duplicate. Fix it. Then mutate
a field of a key already stored in a `HashMap` and watch the entry become
unreachable.

---

#### Day 23 — Exceptions & Resource Management
**Covers:**
- The hierarchy: `Throwable`, `Error`, `Exception`, `RuntimeException`
- Checked vs unchecked — the debate and a defensible position
- `try`/`catch`/`finally` and exactly what `finally` guarantees
- **try-with-resources** and `AutoCloseable`; suppressed exceptions
- Custom exceptions: when they earn their place
- Wrapping and preserving the cause; reading a "caused by" chain
- Anti-patterns: catching `Exception`, empty catch blocks, control flow

**Practice:** Write a resource class that logs on close; use it with and without
try-with-resources and prove the difference when an exception is thrown. Build a
three-level exception chain and read the stack trace top to bottom.

---

#### Day 24 — Generics & Type Erasure
**Covers:**
- Why generics exist: compile-time type safety instead of casting
- Generic classes and methods; bounded type parameters
- **Type erasure** — what the JVM actually sees, and its consequences
- Why `new T[]` and `instanceof List<String>` are impossible
- Wildcards: `? extends` (producer) vs `? super` (consumer) — **PECS**
- Why `List<String>` is not a `List<Object>`
- Raw types and unchecked warnings

**Practice:** Write a generic `Repository<T, ID>` interface with a bounded type
parameter. Write a copy method taking `List<? extends T>` and `List<? super T>`
and explain why each wildcard is needed. Break type safety with a raw type.

---

#### Day 25 — Optional & Null Handling + Week 5 Review
**Covers:**
- Why `null` is a design problem, not just a value
- `Optional`: what it communicates and where it belongs
- `map`, `flatMap`, `filter`, and `orElse` vs `orElseGet` vs `orElseThrow`
- **Anti-patterns**: `.get()`, `Optional` as a field, as a parameter,
  `isPresent()` followed by `get()`
- When plain `null` with a documented contract is fine
- Validating at boundaries; `Objects.requireNonNull`
- Week 5 recap

**Practice:** Refactor a nested null-check chain into `Optional` composition.
Then deliberately write the version that's *worse* with `Optional` and
articulate why. Demonstrate the difference between `orElse` and `orElseGet` with
a logging supplier.

**End of Week 5 — quiz available.**

---

### Week 6: Java 8 — Lambdas & Functional Programming

This is the week that makes the rest of Java readable. Take it slowly.

---

#### Day 26 — From Anonymous Classes to Lambdas
**Covers:**
- The problem: passing behaviour before Java 8 meant an anonymous inner class
- Writing the same `Comparator` three ways — anonymous class, lambda, method
  reference — to see exactly what got shorter
- **What a lambda actually is**: an implementation of a functional interface,
  not a free-floating function
- `@FunctionalInterface` — one abstract method, and why that's the whole rule
- Lambda syntax forms: no params, one param, many, expression vs block body
- Type inference — how the compiler knows the parameter types
- **Effectively final capture** — what a lambda may close over, and why the
  restriction exists
- `this` inside a lambda vs inside an anonymous class (they differ!)

**Practice:** Sort a list with an anonymous `Comparator`, rewrite it as a
lambda, then as a method reference. Try to mutate a local variable inside a
lambda and read the compiler error carefully. Print `this` from both a lambda
and an anonymous class inside the same object.

---

#### Day 27 — The Functional Interface Toolkit
**Covers:**
- `java.util.function`, one interface at a time with a real use for each:
  - **`Function<T,R>`** — transform; `andThen` and `compose`
  - **`Predicate<T>`** — test; `and`, `or`, `negate`
  - **`Supplier<T>`** — produce lazily; why this is what makes `orElseGet` lazy
  - **`Consumer<T>`** — side effect; `andThen`
  - `BiFunction`, `BiPredicate`, `BiConsumer`
  - `UnaryOperator`, `BinaryOperator` and how they relate to `Function`
- Primitive variants (`IntPredicate`, `ToIntFunction`, `IntSupplier`) and the
  boxing cost they exist to avoid
- **Composition**: building a validation rule from several `Predicate`s and a
  transformation from several `Function`s

**Practice:** Build a `Predicate<User>` for each validation rule, then combine
them with `and`/`negate` into one rule. Build a `Function` pipeline that
trims → lowercases → truncates, composed rather than nested. Prove `Supplier`
laziness by putting a print statement inside one.

---

#### Day 28 — Method References & Your Own Functional Interfaces
**Covers:**
- The four kinds of method reference, and the lambda each is shorthand for:
  - Static: `Integer::parseInt`
  - Bound instance: `System.out::println`
  - **Unbound instance: `String::toLowerCase`** — the confusing one, where the
    first parameter becomes the receiver
  - Constructor: `ArrayList::new`
- When a method reference is clearer than a lambda, and when it hides too much
- Writing your own `@FunctionalInterface` — and when that beats a generic one
  (naming, documentation, checked exceptions)
- Handling checked exceptions inside lambdas — the ugly reality and the wrappers

**Practice:** Convert five lambdas into method references, and write out the
equivalent lambda for each unbound reference to prove you understand the
receiver rule. Define a `@FunctionalInterface` called `Validator<T>` and use it.
Try to call a method that throws a checked exception inside a `Function`.

---

#### Day 29 — Default & Static Methods on Interfaces
**Covers:**
- The problem Java 8 had: adding `stream()` to `Collection` would have broken
  every existing implementation
- **`default` methods** — interface evolution without breaking implementors
- `static` methods on interfaces
- The diamond problem when two interfaces provide the same default, and how you
  resolve it (`Interface.super.method()`)
- Resolution rules: class wins over interface, most specific interface wins
- How this shows up everywhere: `Comparator.thenComparing`, `Predicate.and`,
  `List.sort` — all default methods you've already used
- When a default method is good design and when it's a code smell
- Abstract class vs interface now that interfaces have behaviour

**Practice:** Write an interface with a default method, implement it twice, and
override the default in one. Create the diamond problem deliberately with two
interfaces and resolve it. Then read `Comparator`'s source and identify every
default and static method.

---

#### Day 30 — java.time + Week 6 Review
**Covers:**
- Why `Date` and `Calendar` were replaced — mutability, zero-indexed months,
  and thread-unsafety
- **`LocalDate`, `LocalTime`, `LocalDateTime`** — no timezone, and what that means
- **`Instant`** — a point on the timeline, always UTC
- `ZonedDateTime`, `OffsetDateTime` and `ZoneId`
- `Duration` vs `Period`
- Immutability: every operation returns a new object
- Parsing and formatting with `DateTimeFormatter`
- **UTC vs local time** — store UTC, present local; the rule that prevents most
  date bugs (and maps to `timestamptz` in Phase 7)
- Common traps: daylight saving, `LocalDateTime` for an event that needs a zone
- Week 6 recap: lambdas → functional interfaces → method references → defaults

**Practice:** Compute the number of days until a future date, add a business
week to a date, and convert an `Instant` into three different time zones. Then
demonstrate a daylight-saving bug with `LocalDateTime` and fix it with
`ZonedDateTime`.

**End of Week 6 — quiz available. This week's material is load-bearing for
Week 7, so take the quiz seriously.**

---

### Week 7: Collections & Streams

---

#### Day 31 — Lists, Deques & Choosing a Collection
**Covers:**
- The `Collection` hierarchy
- `ArrayList` internals: backing array, growth, amortized cost
- `LinkedList` internals — and why the answer is almost always `ArrayList`
- `ArrayDeque` as the correct stack and queue
- Big-O of what you actually call: get, add, insert, remove, contains
- Iteration and `ConcurrentModificationException`
- `Arrays.asList` vs `List.of` vs `new ArrayList<>()` — which are immutable
- Sizing a collection up front

**Practice:** Benchmark `ArrayList` against `LinkedList` for random access and
for head insertion at 100k elements. Trigger a `ConcurrentModificationException`
and fix it two ways.

---

#### Day 32 — HashMap Internals
**Covers:**
- Buckets, hashing, index calculation
- Collision handling: chaining, and treeification at 8 entries per bucket
- Load factor, capacity, and the cost of a resize
- Why a bad `hashCode` degrades a map toward a linked list
- Iteration order — and that there isn't a guaranteed one
- `LinkedHashMap` for insertion/access order, and LRU caches
- `TreeMap` for sorted access
- `EnumMap`, `WeakHashMap` and their niches
- `computeIfAbsent`, `merge`, `getOrDefault` — the methods that delete boilerplate

**Practice:** Write a class whose `hashCode` always returns 1, insert 10k
entries, and measure the slowdown against a correct implementation. Build an LRU
cache with `LinkedHashMap` in under ten lines. Rewrite a get-check-put block
using `computeIfAbsent`.

---

#### Day 33 — Sets, Sorting & Comparators
**Covers:**
- `HashSet`, `LinkedHashSet`, `TreeSet` — the same trade-offs as their maps
- Set operations: union, intersection, difference
- `Comparable` (natural order) vs `Comparator` (imposed order)
- `Comparator.comparing`, `thenComparing`, `reversed`, `nullsFirst` — all
  built on Week 6's default and static methods
- Sorting stability and when it matters
- The contract violation that causes *"Comparison method violates its general
  contract"* and why it only appears on large lists

**Practice:** Sort a list of records by three fields with mixed directions using
comparator chaining. Write an inconsistent comparator and trigger the contract
violation on a list of 100 elements.

---

#### Day 34 — Streams: The Pipeline
**Covers:**
- A stream is a pipeline, not a collection — nothing is stored
- Source → intermediate operations → terminal operation
- **Laziness**: nothing runs until the terminal operation
- `filter`, `map`, `flatMap`, `distinct`, `sorted`, `limit`, `skip`, `peek`
- Short-circuiting: `findFirst`, `anyMatch`, `limit`
- Why every stream operation takes a functional interface from Week 6
- Statelessness — why a stream must not mutate outside state
- `forEach` vs `collect`, and why `forEach` with side effects is a smell
- Primitive streams (`IntStream`) and avoiding boxing
- **When a plain loop is better** — and it often is
- Why parallel streams are usually the wrong answer

**Practice:** Build a pipeline with `peek` logging at every stage and observe
that elements flow through one at a time, not stage by stage. Rewrite a nested
loop as `flatMap`. Then find a case where the loop is clearly more readable and
keep the loop.

---

#### Day 35 — Collectors & Grouping + Week 7 Review
**Covers:**
- `collect` and the `Collector` abstraction
- `toList`, `toSet`, `toMap` — and `toMap`'s duplicate-key exception
- **`groupingBy`**, including downstream collectors
- `partitioningBy`, `counting`, `summingInt`, `averagingDouble`
- `joining`
- `mapping` and `flatMapping` as downstream collectors
- Multi-level grouping
- `reduce` and when it beats a collector
- Week 7 recap

**Practice:** From a list of order records produce: count per status, total
revenue per customer, a map of customer → list of order ids, and a two-level
grouping of region → status → count. Trigger the `toMap` duplicate-key exception
and fix it with a merge function.

**End of Week 7 — quiz available.**

---

### Week 8: Modern Java, Design & Testing

---

#### Day 36 — Records, Sealed Types & Pattern Matching
**Covers:**
- **`record`**: what the compiler generates for you, and its limits
- Compact constructors for validation
- Records as DTOs — directly relevant to Phase 6
- **Sealed** interfaces and classes: a closed set of implementations
- Pattern matching for `instanceof`
- **Pattern matching for `switch`**, record patterns, exhaustiveness checking
- Modelling success/failure with sealed records instead of exceptions or nulls
- Text blocks for SQL and JSON
- `var`: where it helps and where it hurts

**Practice:** Model an API result as a sealed interface with `Success`,
`NotFound` and `Failure` records; handle it with an exhaustive `switch`. Add a
fourth case and watch the compiler point at every place you now have to handle.

---

#### Day 37 — SOLID & Dependency Inversion
**Covers:**
- **Single Responsibility** — the one that actually pays off daily
- Open/Closed without over-abstracting
- Liskov Substitution and the inheritance traps it warns about
- Interface Segregation
- **Dependency Inversion** — depend on abstractions. Spring is built entirely
  on this, so understanding it now makes Phase 6 obvious instead of magic
- Composition over inheritance
- Coupling and cohesion — what the principles are really about
- Over-engineering: when an interface with one implementation is just noise

**Practice:** Take the repository from Day 24 and invert the dependency so the
service depends only on the interface. Provide an in-memory and a file-backed
implementation and swap them without touching the service.

---

#### Day 38 — Immutability & API Design
**Covers:**
- Why immutable objects are easier to reason about — and **thread-safe for
  free**, which Phase 5 will lean on hard
- Making a class properly immutable: final fields, no setters, defensive copies
- Unmodifiable vs immutable collections
- Builders for objects with many fields
- Static factory methods over constructors
- Method signatures: parameter order, avoiding boolean flags
- Failing fast; validating at construction
- Package structure: **by feature, not by layer**

**Practice:** Write a mutable class, share it between two parts of a program,
and cause a bug by mutating it in one place. Make it immutable and watch the bug
become impossible to write. Add a builder.

---

#### Day 39 — Testing with JUnit 5
**Covers:**
- JUnit 5: `@Test`, lifecycle annotations, `@DisplayName`
- **AssertJ** fluent assertions and why they beat `assertEquals`
- Testing exceptions with `assertThatThrownBy`
- `@ParameterizedTest` with `@ValueSource`, `@CsvSource`, `@MethodSource`
- `@Nested` for organising related cases
- Test names that document behaviour; Arrange–Act–Assert
- What's worth testing; coverage as a diagnostic, not a target
- Test doubles: stub vs mock vs fake; hand-rolled fakes vs Mockito

**Practice:** Write a full suite for the Day 38 immutable class and the Day 37
repository using a hand-written in-memory fake rather than a mock. Collapse
three near-identical tests into one parameterized test.

---

#### Day 40 — Checkpoint Project + Phase 3 Quiz
**Build:** A library management system with **no framework at all**:
- Records for `Book`, `Member`, `Loan`; a sealed interface for operation results
- A `Repository<T, ID>` interface with a `HashMap`-backed implementation
- A service layer depending only on the interface
- A custom exception hierarchy with preserved causes
- **A validation pipeline composed from `Predicate`s** and a transformation
  pipeline composed from `Function`s — written first as anonymous classes, then
  rewritten with lambdas and method references so the difference is visible
- Stream-based queries: books by author, overdue loans, most-borrowed titles
- `java.time` for loan and due dates, stored as `Instant`
- Full JUnit 5 + AssertJ suite with parameterized tests
- **Deliberately** ship `equals` without `hashCode` first, prove the `HashSet`
  bug in a test, then fix it

**Then:** Phase 3 quiz — the big one. Everything after this builds on it.

---

## Phase 4: Docker & Containers

**Duration:** 2 weeks — Days 41–50
**Goal:** Containerize Java properly, and build a reusable local stack so every
later phase can run PostgreSQL, Redis and Kafka without installing anything.

---

### Week 9: Images & the JVM in a Container

---

#### Day 41 — What a Container Actually Is
**Covers:**
- Containers vs VMs — shared kernel, no hypervisor
- **Namespaces**: pid, net, mnt, uts, ipc, user — isolation of *view*
- **cgroups**: cpu, memory, io limits — isolation of *resources*
- Why a container is "just a process" with a different view of the world
- Union filesystems and copy-on-write
- Docker architecture: client, daemon, containerd, runc
- Docker Desktop on Windows: the WSL2 backend, and why your containers really
  run inside a Linux VM
- Images vs containers vs registries

**Practice:** Run a container and find its process on the WSL host with `ps`.
Inspect its namespaces. Set `--memory` and watch a process get OOM-killed when
it exceeds the limit.

---

#### Day 42 — Images, Layers & Registries
**Covers:**
- Image layers, how they stack, content-addressable digests
- `docker pull`, `images`, `history`, `inspect`
- Tags are mutable pointers — why `latest` is a trap
- Registries: Docker Hub, ECR (Phase 15), authentication
- Base image choice: full vs slim vs alpine vs distroless
- Image size: pull time and attack surface
- Scanning images for vulnerabilities

**Practice:** Pull three JDK base images and compare sizes. Use `docker history`
to see what each layer contributed. Retag an image and prove both tags point at
the same digest.

---

#### Day 43 — Writing Dockerfiles
**Covers:**
- `FROM`, `WORKDIR`, `COPY` vs `ADD`, `RUN`, `ENV`, `ARG`, `EXPOSE`, `USER`
- **`CMD` vs `ENTRYPOINT`**, the exec form, and how they combine
- Signal handling and PID 1 — why your Java process must receive `SIGTERM`
  (the thread from Phase 1 Day 4, and it matters again in Phase 17)
- `.dockerignore` and build context size
- Build args vs runtime environment variables
- Never baking secrets into a layer — they persist even if a later layer
  deletes the file

**Practice:** Write a Dockerfile for the Phase 3 project. Run it and verify the
JVM receives `SIGTERM` on `docker stop` rather than being killed after the
10-second grace period.

---

#### Day 44 — Multi-Stage Builds & Layer Caching for Java
**Covers:**
- The problem: a JDK and a Maven cache sitting in your runtime image
- **Multi-stage builds**: build with the JDK, run on the JRE
- Ordering for cache hits — copy `pom.xml` and resolve dependencies *before*
  copying source
- `mvn dependency:go-offline` as a cache layer
- Spring Boot **layered JARs** and `layertools`
- BuildKit cache mounts
- Distroless images and `jlink` custom runtimes

**Practice:** Build a naive Dockerfile and time a rebuild after a one-line Java
change. Convert it to multi-stage with dependency caching and time it again.
Apply layered JARs and time a third. Record all three numbers.

---

#### Day 45 — The JVM Inside a Container + Week 9 Review
**Covers:**
- The historical bug: a JVM seeing host memory instead of the cgroup limit
- Container-aware ergonomics in modern JDKs
- **`-XX:MaxRAMPercentage`** and why you set it instead of `-Xmx`
- Heap is not the whole story: metaspace, thread stacks, direct buffers, code
  cache — why the container limit must exceed the heap
- `OutOfMemoryError` vs being OOM-killed by the kernel — telling them apart
- CPU limits, `availableProcessors()`, and thread pool sizing (forward
  reference to Phase 5)
- CPU throttling and its effect on latency
- Week 9 recap

**Practice:** Run your image with `--memory=512m` and no JVM flags; print what
the JVM thinks it has. Set `MaxRAMPercentage=75` and compare. Then set a heap
larger than the container limit and get it OOM-killed.

**End of Week 9 — quiz available.**

---

### Week 10: Compose, Debugging & Security

---

#### Day 46 — Running Containers: Networking, Volumes, Config
**Covers:**
- Port publishing, and reaching a container from Windows through WSL2
- Bridge networks and DNS resolution by container name
- Host and none network modes
- **Volumes vs bind mounts** — and why bind-mounting Windows paths is slow
- Named volumes for database data
- Environment variables and `--env-file`
- Restart policies; `--memory` and `--cpus`

**Practice:** Run two containers on a user-defined network and have one reach
the other by name. Run PostgreSQL with a named volume, write data, destroy and
recreate the container, and prove the data survived — then repeat without a
volume and prove it didn't.

---

#### Day 47 — Docker Compose
**Covers:**
- Why Compose: reproducible multi-service environments
- File structure: services, networks, volumes
- `build` vs `image`; ports, environment, `env_file`
- **Health checks** and `depends_on` with `condition: service_healthy` — the
  fix for "my app started before the database was ready"
- `docker compose up/down/logs/ps/exec`
- Profiles for optional services

**Practice:** Write a Compose file with your app plus PostgreSQL. Add a health
check and a `depends_on` condition. Remove the condition and watch the app crash
on startup to prove the ordering matters.

---

#### Day 48 — The Local Stack You'll Use for Months
**Covers:**
- Building the reusable dev stack: PostgreSQL (with pgvector), Redis, and
  placeholders for Kafka and Keycloak
- Init scripts for database bootstrapping
- Persisting data across restarts
- Exposing ports without collisions
- Connecting from a Windows IDE to a container running under WSL2
- `docker compose down -v` and when you actually want to wipe volumes

**Practice:** Stand up the full stack. Connect to PostgreSQL from a Windows SQL
client. Enable the `vector` extension and confirm it loads. **Save this Compose
file — Phases 7, 8, 10, 13 and 14 all build on it.**

---

#### Day 49 — Debugging & Container Security
**Covers:**
- `docker logs` with `-f` and `--tail`; log drivers
- `docker exec` for a shell inside a running container
- Debugging a container that won't start; `docker events`
- `docker inspect` and `docker stats`
- Remote JVM debugging into a container from your IDE
- Security: non-root `USER`, read-only root filesystem, dropped capabilities,
  no secrets in images
- Image scanning and keeping base images current
- Why one process per container

**Practice:** Break your image three ways (wrong `CMD`, missing env var, port
conflict) and diagnose each from the logs alone. Add a non-root user and confirm
the app still works.

---

#### Day 50 — Checkpoint Project + Phase 4 Quiz
**Build:** Fully containerize the Phase 3 project wrapped in a minimal HTTP layer:
- Multi-stage Dockerfile, final image under 250MB, running as non-root
- Layered JAR caching with before/after rebuild timings recorded
- Correct JVM memory flags for the container limit
- Graceful shutdown on `SIGTERM`, verified
- Compose stack: app + PostgreSQL + Redis, health checks, named volumes
- A README section documenting every decision

**Then:** Phase 4 quiz.

---

## Phase 5: Concurrency, Executors & CompletableFuture

**Duration:** 4 weeks — Days 51–70
**Goal:** Go from "I know the concepts" to being able to spot a race in review,
configure a thread pool deliberately, and compose async work with
`CompletableFuture`. Built from zero.

---

### Week 11: The Foundations

---

#### Day 51 — What a Thread Actually Is
**Covers:**
- A thread is an independent path of execution: its own **stack**, sharing the
  **heap** (this is Day 21 paying off)
- Threads vs processes; what the OS scheduler does
- Context switching and what it costs
- Creating threads the old way: `Thread`, `Runnable`, `start()` vs `run()`
- Thread states and lifecycle
- `join`, `sleep`, interruption and why `Thread.stop` was removed
- Daemon vs user threads
- Why "just add more threads" stops working

**Practice:** Start three threads that each print their name and a counter, and
observe the interleaving change between runs. Call `run()` instead of `start()`
and explain why nothing is concurrent. Interrupt a sleeping thread and handle it.

---

#### Day 52 — The Three Problems: Visibility, Atomicity, Ordering
**Covers:**
- **Visibility** — a write by one thread may never be seen by another; CPU
  caches and why this isn't a bug in the JVM
- **Atomicity** — `count++` is read, add, write: three operations
- **Ordering** — the compiler and CPU may reorder your statements
- The **Java Memory Model** and happens-before, in plain language first
- `volatile`: fixes visibility and ordering, **does not** fix atomicity
- What `synchronized` guarantees beyond mutual exclusion
- Safe publication — how an object becomes visible to other threads correctly

**Practice:** Write a loop that spins on a non-`volatile` boolean flag set by
another thread and watch it never terminate. Add `volatile` and watch it stop.
Then show that `volatile` does **not** fix a `count++` race.

---

#### Day 53 — Writing a Race Condition on Purpose
**Covers:**
- Check-then-act and read-modify-write as the two classic race shapes
- Why races are intermittent and pass in testing
- Building a shared counter that loses increments under load
- Building a bank balance that loses money
- Lost updates, dirty reads, inconsistent state
- Why adding a `sleep` "fixes" it and why that's not a fix
- Thread-safety as a property of a *class*, not a method

**Practice:** Run 1,000 threads each incrementing a shared counter 1,000 times.
Predict 1,000,000, observe less, and run it repeatedly to see the number change.
Do the same with a transfer between two accounts and prove money is created or
destroyed.

---

#### Day 54 — synchronized & Intrinsic Locks
**Covers:**
- Every object has an intrinsic lock (monitor)
- `synchronized` methods vs blocks — and **what object you're actually locking**
- Locking on `this`, on a class, on a dedicated private lock object (and why
  the last is best)
- Reentrancy — why a synchronized method can call another
- Lock scope: holding a lock too long vs too briefly
- `wait`, `notify`, `notifyAll` and the mandatory `while` loop around `wait`
- Fixing Day 53's counter and balance
- Performance: contention, and why uncontended locks are cheap

**Practice:** Fix the Day 53 counter with `synchronized` and confirm the number
is now exact. Then demonstrate the bug where two methods synchronize on
different objects and provide no mutual exclusion at all.

---

#### Day 55 — Deadlock, Livelock & Starvation + Week 11 Review
**Covers:**
- **Deadlock**: two threads each holding what the other needs
- The four conditions required, and breaking any one of them
- **Lock ordering** as the practical cure
- Livelock and starvation
- Nested locks and why they're a warning sign
- Detecting a deadlock in a thread dump
- Immutability as the strategy that avoids all of this (Day 38)
- Week 11 recap: threads → three problems → locks → deadlock

**Practice:** Write two accounts transferring to each other in opposite
directions until they deadlock. Capture a thread dump with `jstack` and find the
cycle. Fix it by ordering locks on account id.

**End of Week 11 — quiz available.**

---

### Week 12: Executors & Thread Pools

The mental model: **a pool is a fixed set of worker threads pulling tasks off a
queue.** Everything else is configuration of that sentence.

---

#### Day 56 — Why Thread Pools Exist
**Covers:**
- The problem: thread creation is expensive and unbounded threads kill a server
- One thread per task vs a pool — the memory and scheduling maths
- `Executor`, `ExecutorService`, `ScheduledExecutorService`
- Submitting work: `execute` vs `submit`
- The `Executors` factory methods — and **why to avoid most of them**:
  - `newFixedThreadPool` — unbounded queue, so requests pile up invisibly
  - `newCachedThreadPool` — unbounded threads, so load creates thousands
  - `newSingleThreadExecutor` — serialised, useful but often accidental
- **Shutdown**: `shutdown` vs `shutdownNow`, `awaitTermination`, and why a
  forgotten pool keeps the JVM alive

**Practice:** Submit 10,000 tasks to a `newCachedThreadPool` and count how many
threads get created. Repeat with a fixed pool. Forget to shut a pool down and
observe the JVM refusing to exit.

---

#### Day 57 — ThreadPoolExecutor in Full
**Covers:**
- Constructing one by hand and what each parameter means:
  - **corePoolSize** — threads kept alive even when idle
  - **maximumPoolSize** — the ceiling
  - **keepAliveTime** — how long extra threads linger
  - **workQueue** — where tasks wait
  - **threadFactory** — naming and daemon status
  - **rejectedExecutionHandler** — what happens when full
- **The non-obvious growth rule**: the pool only creates threads beyond the core
  size when the **queue is full** — so with an unbounded queue, `maximumPoolSize`
  is never reached
- `allowCoreThreadTimeOut` and `prestartAllCoreThreads`
- Naming threads, and why an unnamed pool makes production debugging miserable
- Monitoring: active count, queue size, completed task count

**Practice:** Build a `ThreadPoolExecutor` with core 2, max 10 and an unbounded
queue; flood it and prove only 2 threads ever run. Switch to a bounded queue of
5 and watch it grow to 10. This is the day's whole point.

---

#### Day 58 — Queues, Rejection & Sizing
**Covers:**
- Queue choice and its consequences:
  - `LinkedBlockingQueue` (unbounded by default — memory risk)
  - `ArrayBlockingQueue` (bounded — real backpressure)
  - `SynchronousQueue` (no capacity — hand-off directly to a thread)
  - `PriorityBlockingQueue`
- **Rejection policies**: `AbortPolicy`, `CallerRunsPolicy`, `DiscardPolicy`,
  `DiscardOldestPolicy` — and why **CallerRuns** is the one that gives you
  natural backpressure
- **Sizing a pool**: CPU-bound ≈ number of cores; IO-bound much larger
- The sizing formula, and why measuring beats the formula
- Separate pools per workload (bulkheading) instead of one shared pool
- The failure mode: pool starvation when tasks submit to their own pool and wait

**Practice:** Configure a bounded pool with `CallerRunsPolicy`, flood it from
one thread, and observe the submitter slowing down — backpressure you can feel.
Then create a deadlock by having a task submit to its own single-threaded pool
and block on the result.

---

#### Day 59 — Callable, Future & Concurrent Collections
**Covers:**
- `Runnable` vs `Callable` — returning a value and throwing checked exceptions
- `Future`: `get()`, `get(timeout)`, `isDone`, `cancel`
- **Why `Future.get()` blocks** — the limitation that motivates Week 13
- `invokeAll` and `invokeAny`
- `ExecutionException` and unwrapping the real cause
- **Concurrent collections**:
  - `ConcurrentHashMap` — and why it's not just a synchronized map
  - Atomic operations on it: `compute`, `merge`, `putIfAbsent`
  - `CopyOnWriteArrayList` and its narrow use case
  - `BlockingQueue` and the producer–consumer pattern
- Why wrapping a `HashMap` in `synchronized` still allows check-then-act races

**Practice:** Submit five slow tasks and collect results with `invokeAll`.
Then submit them and call `get()` on each in turn — measure the difference and
explain it. Build a producer–consumer with a `BlockingQueue`.

---

#### Day 60 — Atomics, Locks & ThreadLocal + Week 12 Review
**Covers:**
- Compare-and-swap: how `AtomicInteger` avoids locking
- `AtomicInteger`, `AtomicLong`, `AtomicReference`, `updateAndGet`
- `LongAdder` for high-contention counters
- The ABA problem (awareness)
- **`ReentrantLock`**: what it gives over `synchronized` — `tryLock`, timeouts,
  interruptibility, fairness — and the mandatory `finally { unlock() }`
- `ReadWriteLock` for read-heavy data
- `Semaphore` for limiting concurrent access
- `CountDownLatch` and `CyclicBarrier` for coordination
- **`ThreadLocal`** — per-thread state, and why it **leaks in a pooled thread**
  if you don't `remove()` (this returns in Phase 6 and Phase 8)
- Week 12 recap

**Practice:** Fix the Day 53 counter three ways — `synchronized`,
`AtomicInteger`, `LongAdder` — and benchmark all three under contention. Use a
`Semaphore` to cap concurrent calls to a slow resource at 3. Set a `ThreadLocal`
in a pooled task and prove it's still there on the next task.

**End of Week 12 — quiz available.**

---

### Week 13: CompletableFuture

---

#### Day 61 — CompletableFuture Basics & Which Thread Runs It
**Covers:**
- The problem: `Future` can only be polled or blocked on — you cannot say
  "when this finishes, do that"
- Creating: `supplyAsync`, `runAsync`, `completedFuture`, and manual `complete`
- `join` vs `get` — checked vs unchecked
- **Which thread actually runs your code**: the common `ForkJoinPool` by default
- Why the common pool is a trap for blocking or IO-bound work (it's sized for
  CPU work and shared with parallel streams)
- **Always pass your own `Executor`** — the single most important habit here
- The `*Async` suffix and what the second overload with an executor means

**Practice:** Run a `supplyAsync` and print the thread name — observe
`ForkJoinPool.commonPool-worker-N`. Pass your own named executor and observe the
name change. Then submit 100 blocking tasks to the common pool and watch them
serialise on a small machine.

---

#### Day 62 — thenApply vs thenCompose
**Covers:**
- `thenApply` — transform the result (this is `map`)
- `thenCompose` — chain another async call (this is `flatMap`)
- **The nested-future bug**: using `thenApply` with a function that returns a
  `CompletableFuture` gives you `CompletableFuture<CompletableFuture<T>>`
- `thenAccept` and `thenRun` for terminal side effects
- Sync vs async variants: `thenApply` vs `thenApplyAsync` — which thread
  continues the chain
- Chaining several dependent calls readably

**Practice:** Write a chain that fetches a user id, then fetches the user, then
fetches their orders — first with `thenApply` and hit the nested type error,
then with `thenCompose`. Print the thread name at every stage to see where each
step ran.

---

#### Day 63 — Combining: thenCombine, allOf & anyOf
**Covers:**
- `thenCombine` — two independent futures, combined when both finish
- `thenAcceptBoth`, `runAfterBoth`
- `applyToEither`, `acceptEither` — first one wins
- **`allOf`** — wait for many; and the awkward `CompletableFuture<Void>` return,
  plus the correct pattern for collecting the results
- **`anyOf`** — first to complete
- Fan-out/fan-in as a pattern: three parallel calls, one combined response
- Sequential vs parallel — measuring the difference

**Practice:** Call three simulated services that each take one second.
Do it sequentially and time it (~3s). Do it with `allOf` and time it (~1s).
Then collect all three typed results properly from the `allOf`.

---

#### Day 64 — Exception Handling & Timeouts
**Covers:**
- How an exception propagates through a chain — later stages are skipped
- **`exceptionally`** — recover with a fallback value
- **`handle`** — see both result and exception
- **`whenComplete`** — observe without changing the outcome
- `CompletionException` and unwrapping the real cause
- Where to place recovery in a chain, and why placement changes behaviour
- **Timeouts**: `orTimeout` and `completeOnTimeout`
- Cancellation and its limits — why the underlying work may keep running
- Logging failures in async chains so they aren't silently swallowed

**Practice:** Build a chain where the second stage throws; confirm the third is
skipped and the exception surfaces at `join`. Add `exceptionally` in three
different positions and observe the different outcomes. Add `orTimeout` to a
slow call and supply a fallback.

---

#### Day 65 — A Real Async Workflow + Week 13 Review
**Covers:**
- Composing everything: parallel fetch → combine → transform → fallback →
  timeout, all on a dedicated executor
- Structuring async code so it stays readable
- When `CompletableFuture` is overkill and `invokeAll` is fine
- Debugging: why async stack traces are unhelpful and what to log instead
- Testing async code without `Thread.sleep`
- Preview: how all of this appears inside a Spring service (Phase 6 Week 17)
- Week 13 recap

**Practice:** Build an "order summary" workflow: fetch customer, orders and
recommendations in parallel on your own pool; combine into one response; fall
back to an empty recommendation list on failure; time out the whole thing at
two seconds. Write tests for the success, failure and timeout paths.

**End of Week 13 — quiz available.**

---

### Week 14: Virtual Threads & Applied Concurrency

---

#### Day 66 — Virtual Threads
**Covers:**
- The problem: platform threads are OS threads, ~1MB of stack each, so you
  cannot have a million
- **Virtual threads (Java 21)**: managed by the JVM, mounted on carrier threads
- Why **blocking becomes cheap** — a blocked virtual thread unmounts
- Creating them: `Thread.ofVirtual()`,
  `Executors.newVirtualThreadPerTaskExecutor()`
- **Why you should not pool virtual threads** — they're the cheap thing now
- What virtual threads do *not* fix: CPU-bound work, and shared mutable state
- Rewriting a thread-per-task workload

**Practice:** Launch 10,000 platform threads that each sleep one second and
measure memory and time; do the same with virtual threads and compare. Try
1,000,000 virtual threads.

---

#### Day 67 — Pinning & Structured Concurrency
**Covers:**
- **Pinning**: when a virtual thread cannot unmount and blocks its carrier
  - `synchronized` blocks holding a lock across a blocking call
  - Native calls
- Using `ReentrantLock` instead of `synchronized` to avoid pinning
- Detecting pinning with `jdk.tracePinnedThreads`
- Carrier pool sizing
- **Structured concurrency**: treating concurrent subtasks as one unit with a
  clear scope, so failures and cancellation propagate sensibly
- `StructuredTaskScope`, shutdown-on-failure and shutdown-on-success
- How this compares to a `CompletableFuture` chain

**Practice:** Write a pinning case with `synchronized` around a sleep, detect it
with the tracing flag, and fix it with `ReentrantLock`. Then rewrite the Day 65
workflow using `StructuredTaskScope` and compare readability.

---

#### Day 68 — Thread-per-Request vs Reactive vs Virtual
**Covers:**
- Thread-per-request: simple, debuggable, limited by thread count
- Reactive (WebFlux, Project Reactor): high concurrency, but a different
  programming model and much harder stack traces
- Virtual threads: thread-per-request simplicity at reactive-like concurrency
- **The honest trade-offs**, and why virtual threads reduce the case for reactive
- Where reactive still wins (streaming, backpressure across a pipeline)
- What this means for your capstone choice
- Blocking vs non-blocking IO at the OS level

**Practice:** Write the same "call three services and combine" endpoint three
ways — blocking with a pool, `CompletableFuture`, and virtual threads — and
compare code clarity and stack traces on failure.

---

#### Day 69 — Diagnosing Concurrency in Production
**Covers:**
- Taking a thread dump: `jstack`, `jcmd`, kill -3
- **Reading a thread dump**: thread states, stack frames, lock ownership
- Finding a deadlock in a dump
- Spotting pool exhaustion: all workers busy, queue growing
- Spotting a thread leak
- JFR and VisualVM basics
- Metrics that matter: active threads, queue depth, task latency, rejection count
- Common production failures revisited: starvation, nested pool deadlock,
  unbounded queue memory growth, `ThreadLocal` leaks

**Practice:** Deliberately exhaust a pool with slow tasks, take a thread dump,
and identify from the dump alone what went wrong. Do the same for a deadlock and
a `ThreadLocal` leak.

---

#### Day 70 — Checkpoint Project + Phase 5 Quiz
**Build:** A bank-transfer simulator, in stages, each one committed separately:

1. Unsynchronized balance under 1,000 concurrent transfers — prove money vanishes
2. Fix three ways (`synchronized`, `AtomicInteger`, `ReentrantLock`), benchmark each
3. Deadlock two accounts, capture the thread dump, resolve by lock ordering
4. Build a `ThreadPoolExecutor` by hand: named threads, bounded queue,
   `CallerRunsPolicy` — flood it and observe backpressure
5. Prove the growth rule: show max pool size is never reached with an unbounded queue
6. Rewrite the reporting path as a `CompletableFuture` pipeline — three parallel
   lookups, combined, on your own executor, with a timeout and a fallback
7. Run the whole thing on platform vs virtual threads at 10,000 tasks and
   compare throughput and memory

**Then:** Phase 5 quiz.

---

## Phase 6: Spring Boot Core, REST & Async

**Duration:** 4 weeks — Days 71–90
**Goal:** Build the service the rest of the roadmap extends — and understand
Spring's threading model rather than guessing at it.
**This is where the capstone codebase starts.**

---

### Week 15: The Framework

---

#### Day 71 — The IoC Container & Dependency Injection
**Covers:**
- The problem: objects creating their own dependencies are untestable and rigid
- **Inversion of Control** — something else builds and wires your objects
- What a "bean" is; the `ApplicationContext`
- **Constructor injection only**, and why field injection is bad (untestable,
  hides required dependencies, breaks immutability)
- This is Day 37's Dependency Inversion made concrete
- `@Component`, `@Service`, `@Repository`, `@Configuration` + `@Bean`
- Component scanning and how Spring finds your classes
- Multiple implementations: `@Qualifier`, `@Primary`
- Circular dependencies — what they mean about your design

**Practice:** Write a service that constructs its own repository, then refactor
to constructor injection and write a unit test with a hand-made fake that was
impossible before. Create a circular dependency and read the error.

---

#### Day 72 — Beans: Scopes, Lifecycle & Conditional Creation
**Covers:**
- Singleton scope (the default) and what that means for **thread safety** —
  a singleton service is shared across all request threads, so instance fields
  are shared mutable state (Phase 5 applies directly)
- Prototype, request and session scopes
- Bean lifecycle: instantiation, population, `@PostConstruct`, destruction,
  `@PreDestroy`
- `InitializingBean` / `DisposableBean`
- Lazy initialisation
- `@Conditional` and conditional beans
- Bean definition overriding and startup ordering

**Practice:** Add a mutable instance field to a singleton `@Service`, hit it
from many concurrent requests, and watch it corrupt. Fix it by making the
service stateless. Add `@PostConstruct` and `@PreDestroy` logging and observe
when each fires.

---

#### Day 73 — Auto-Configuration & Starters
**Covers:**
- What a starter actually is — a curated dependency set, not magic
- How auto-configuration works: `@AutoConfiguration`, `spring.factories` /
  `AutoConfiguration.imports`
- `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`
- **Reading the Actuator `conditions` report** to see exactly why a bean did or
  did not get created — the antidote to "Spring is magic"
- Overriding an auto-configured bean by defining your own
- Excluding auto-configuration
- Debugging startup with `--debug`

**Practice:** Run with `--debug` and read the auto-configuration report. Find
three beans you didn't define and identify which condition created them.
Override one with your own and confirm from the report that yours won.

---

#### Day 74 — External Configuration & Profiles
**Covers:**
- `application.yml` vs `application.properties`
- **Property precedence** — command line, environment, profile-specific files,
  defaults — and why this order matters in containers (Phase 4) and
  Kubernetes (Phase 17)
- `@Value` vs **`@ConfigurationProperties`** (and why the latter is better:
  type-safe, validatable, grouped)
- Relaxed binding, and how `MY_APP_TIMEOUT` maps to `my.app.timeout`
- Profiles: `@Profile`, profile-specific files, activating them
- Validating configuration at startup with `@Validated`
- **Never committing secrets** — a preview of Phase 8

**Practice:** Bind a nested configuration block to a `@ConfigurationProperties`
record with validation; start the app with an invalid value and confirm it fails
fast. Override one property four ways and confirm the precedence order.

---

#### Day 75 — Actuator + Week 15 Review
**Covers:**
- What Actuator gives you: health, info, metrics, env, conditions, mappings
- Health indicators, and writing a custom one
- Liveness vs readiness probes (this is exactly what Kubernetes needs in Phase 17)
- Exposing endpoints safely — and why `/actuator/env` must not be public
- Micrometer basics and the metrics registry
- Week 15 recap: container → beans → auto-config → configuration

**Practice:** Enable Actuator, write a custom health indicator that reports
degraded when a dependency is slow, and expose only health and metrics. Inspect
`/actuator/mappings` to see every route your app serves.

---

### Week 16: REST APIs

---

#### Day 76 — Controllers & REST Design
**Covers:**
- REST resource modelling: nouns not verbs, hierarchy, collection vs item
- Choosing status codes deliberately: 200/201/204/400/404/409/422
- **Idempotency** and why it decides whether a client may retry (Phase 13)
- `@RestController`, `@RequestMapping` and the method shortcuts
- `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader`
- `ResponseEntity` and setting status and headers explicitly
- Layering: controller → service → repository, and what belongs where — the
  controller does HTTP, never business logic

**Practice:** Design and implement the document resource: list, get, create,
update, delete. Return the correct status for each, including 201 with a
`Location` header, and 409 on a duplicate.

---

#### Day 77 — Validation
**Covers:**
- Bean Validation: `@NotNull`, `@NotBlank`, `@Size`, `@Email`, `@Positive`
- `@Valid` on a request body and what happens when it fails
- Nested validation with `@Valid` on fields
- Validation groups for create vs update
- Writing a custom constraint annotation and validator
- Validating path variables and request params (`@Validated` on the class)
- **Where validation belongs**: syntax at the boundary, business rules in the
  service — and why mixing them hurts

**Practice:** Add validation to the create-document request. Submit invalid
input and observe the raw error response. Write a custom `@ValidDocumentType`
constraint. Then move a business rule out of validation into the service and
explain why.

---

#### Day 78 — Global Exception Handling & ProblemDetail
**Covers:**
- Why per-controller try/catch is the wrong answer
- `@RestControllerAdvice` and `@ExceptionHandler`
- **RFC 7807 `ProblemDetail`** — the standard error shape, and Spring's support
- Mapping your domain exceptions (Day 23) to HTTP statuses
- Turning validation failures into a useful field-by-field response
- **Never leaking stack traces or internals** to a client
- Logging the cause server-side while returning something safe
- Consistent error contracts across the API

**Practice:** Build a `@RestControllerAdvice` that maps three domain exceptions
to three statuses and returns `ProblemDetail` with a field-level breakdown for
validation errors. Confirm no stack trace reaches the client but the full cause
is logged.

---

#### Day 79 — DTOs, Mapping & Jackson
**Covers:**
- **Why you never expose entities**: coupling your API to your schema, lazy
  loading blowing up during serialization, and accidental data leaks
- Request DTOs vs response DTOs vs domain models
- Records as DTOs (Day 36)
- Mapping: manual, MapStruct — and the trade-off
- Jackson: `@JsonProperty`, `@JsonIgnore`, `@JsonInclude`, naming strategies
- Serializing `java.time` correctly (Day 30) — and why timestamps should be
  ISO-8601 UTC
- Custom serializers and deserializers
- Handling unknown properties

**Practice:** Expose an entity directly, then demonstrate two concrete problems
it causes. Introduce separate request and response records and a mapper. Verify
an `Instant` serialises as ISO-8601.

---

#### Day 80 — OpenAPI, Versioning & Pagination + Week 16 Review
**Covers:**
- springdoc-openapi: generated docs and Swagger UI
- Annotating for useful documentation
- **API versioning**: URI path, header, content negotiation — trade-offs
- What counts as a breaking change
- Pagination: page/size vs cursor, and why offset paging degrades at scale
  (returns in Phase 7)
- Sorting and filtering parameters
- `Page` and `Slice`
- Week 16 recap

**Practice:** Generate OpenAPI docs and make the document endpoints fully
described. Add pagination to the list endpoint with a total count, then add a
cursor-based variant and compare the responses.

---

### Week 17: Threading & Async in Spring

This week takes Phase 5 and shows you where it actually lives in a real service.

---

#### Day 81 — How Spring Boot Serves a Request
**Covers:**
- Embedded Tomcat and its **thread pool** — one thread per request
- `server.tomcat.threads.max` and `accept-count`: what each really controls
- What "the request thread is blocked" means, concretely
- **Why 200 slow requests can stall a server with 200 threads** — do the maths
- The connection queue, and what a client sees when it's full
- Where your controller code runs, and what shares that thread
  (`SecurityContext`, MDC, request scope — all thread-bound)
- Why a singleton controller must be stateless (Day 72)
- Measuring: active threads, busy threads, request latency under load

**Practice:** Add a controller endpoint that sleeps two seconds. Set
`server.tomcat.threads.max=10`, fire 50 concurrent requests, and watch queueing
and latency. Plot the difference when you raise the pool size.

---

#### Day 82 — @EnableAsync, @Async & Its Two Gotchas
**Covers:**
- What `@Async` actually does: Spring wraps the bean in a **proxy** that submits
  the call to an executor
- `@EnableAsync` and why nothing happens without it
- **Gotcha 1 — self-invocation**: calling an `@Async` method from another method
  in the same class bypasses the proxy and runs synchronously, silently
- **Gotcha 2 — visibility**: `@Async` on a `private` or `final` method does nothing
- Return types: `void` (fire and forget, exceptions vanish), `Future`,
  `CompletableFuture`
- **Exception handling in `@Async void`** — `AsyncUncaughtExceptionHandler`
- When `@Async` is the right tool and when a queue (Phase 13) is better

**Practice:** Write an `@Async` method and prove it runs on another thread by
logging thread names. Then call it from a sibling method in the same class and
watch it run on the request thread instead — the bug that catches everyone. Fix
it by moving the method to another bean. Throw from an `@Async void` and observe
the exception disappear.

---

#### Day 83 — Configuring ThreadPoolTaskExecutor
**Covers:**
- **`ThreadPoolTaskExecutor`** — Spring's wrapper over `ThreadPoolExecutor`
  (Day 57), with the same parameters under different names
- `corePoolSize`, `maxPoolSize`, `queueCapacity`, `threadNamePrefix`,
  `keepAliveSeconds`
- **The same growth rule applies**: `queueCapacity` fills before `maxPoolSize`
  is used — and Spring's default queue capacity is `Integer.MAX_VALUE`
- Why the default `SimpleAsyncTaskExecutor` (a new thread per task, no pooling)
  is dangerous in production
- Defining **multiple named executor beans** — one per workload (AI calls,
  ingestion, notifications) so a slow dependency can't starve everything else:
  bulkheading from Day 58
- Selecting one with `@Async("aiExecutor")`
- Rejection policy and graceful shutdown
  (`setWaitForTasksToCompleteOnShutdown`, `setAwaitTerminationSeconds`)
- Exposing executor metrics through Micrometer

**Practice:** Define two named executors with different sizes and prefixes.
Route two `@Async` methods to different pools and confirm from the logs. Set
`queueCapacity` to 0 and watch `maxPoolSize` finally get used. Then saturate one
pool and prove the other is unaffected.

---

#### Day 84 — CompletableFuture in a Service & Async Controllers
**Covers:**
- Using `CompletableFuture` (Week 13) inside a Spring service, **on your own
  executor bean** rather than the common pool
- Parallelising independent calls: two repositories, or a database read plus an
  HTTP call
- Returning `CompletableFuture<ResponseEntity<T>>` from a controller — the
  request thread is released while the work runs
- `DeferredResult` and `Callable` return types, and how they differ
- **Server-Sent Events** with `SseEmitter` — the mechanism your AI streaming
  endpoint uses in Phase 9
- Timeouts on async requests (`spring.mvc.async.request-timeout`)
- When async controllers actually help, and when they just add complexity

**Practice:** Build an endpoint that fetches document metadata, permissions and
statistics in parallel with `CompletableFuture` on a named executor, combined
into one response. Measure against the sequential version. Then convert it to
return `CompletableFuture` and confirm the Tomcat thread is released.

---

#### Day 85 — Context Propagation & Virtual Threads + Week 17 Review
**Covers:**
- **The problem**: `SecurityContext`, MDC correlation ids and request-scoped
  beans are stored in `ThreadLocal` (Day 60) — so they are **empty inside an
  async task**
- Symptoms: logs losing the correlation id, `SecurityContextHolder` returning
  null in an `@Async` method (this bites hard in Phase 8)
- Fixes: `DelegatingSecurityContextAsyncTaskExecutor`, a custom `TaskDecorator`
  that copies MDC, and Micrometer's context propagation
- Cleaning up `ThreadLocal` afterwards to avoid leaking into the next task
- `@Scheduled` — its own separate, **single-threaded by default** pool, and why
  one slow job delays every other
- `ApplicationEventPublisher` and `@TransactionalEventListener` for side effects
  after commit
- **Virtual threads in Spring Boot**: `spring.threads.virtual.enabled=true` —
  what it changes, what it doesn't, and how it interacts with pools
- Week 17 recap

**Practice:** Log a correlation id from a request thread and from an `@Async`
method — watch it vanish. Add a `TaskDecorator` that copies MDC and watch it
survive. Schedule two `@Scheduled` jobs where one sleeps, and prove the other is
delayed; fix it with a scheduler pool. Then enable virtual threads and re-run
the Day 81 load test.

**End of Week 17 — quiz available.**

---

### Week 18: Quality & Outbound Calls

---

#### Day 86 — Testing: Slices & MockMvc
**Covers:**
- The testing pyramid in a Spring app
- `@SpringBootTest` — full context, slow; when it's justified
- **Slice tests**: `@WebMvcTest`, `@DataJpaTest`, `@JsonTest` — faster and focused
- MockMvc: performing requests, asserting status, headers, JSON body
- `@MockBean` vs constructor-injected fakes, and when each is right
- Testing validation and error handling paths
- Test configuration and profiles
- Why testing the happy path only is how bugs reach production

**Practice:** Write `@WebMvcTest` coverage for the document controller —
success, validation failure, not found, and conflict. Assert the `ProblemDetail`
shape. Compare the runtime against the equivalent `@SpringBootTest`.

---

#### Day 87 — Testcontainers & Testing Async
**Covers:**
- Why H2 lies: different SQL dialect, no real constraints, no PostgreSQL
  features — you'll believe code works that doesn't
- **Testcontainers**: a real PostgreSQL in Docker for every test run
- `@Testcontainers`, `@Container`, `@ServiceConnection`
- Container reuse and keeping the suite fast
- **Testing async code without `Thread.sleep`** — Awaitility, latches, and
  polling assertions
- Testing that something ran on a different thread
- Flaky tests and how concurrency causes them

**Practice:** Convert an integration test to Testcontainers with real
PostgreSQL. Then write a test for the Day 83 `@Async` method using Awaitility
instead of a sleep, and one that asserts it ran on the expected executor.

---

#### Day 88 — Outbound Calls: RestClient & Timeouts
**Covers:**
- `RestClient` (the modern synchronous client), `WebClient`, and the deprecated
  `RestTemplate`
- **Every outbound call needs a timeout** — connect and read, separately.
  A missing timeout is how one slow dependency takes down your whole thread pool
  (Day 81's maths)
- Connection pooling for HTTP clients
- Error handling and mapping remote failures to your own exceptions
- Serialising and deserialising responses
- Logging outbound calls with correlation ids propagated
- Testing with MockWebServer or `@RestClientTest`

**Practice:** Call a deliberately slow endpoint with no timeout from a
thread-limited pool and stall your service. Add connect and read timeouts and
watch it degrade gracefully instead.

---

#### Day 89 — Resilience4j & Structured Logging
**Covers:**
- **Retry** with exponential backoff and jitter — and why retrying a
  non-idempotent call is dangerous (Day 76)
- **Circuit breaker**: closed → open → half-open, and what each state does
- **Bulkhead** — limiting concurrent calls per dependency, the Day 83 idea
  applied to outbound work
- Rate limiter and time limiter
- Combining them, and the correct order
- Fallbacks that degrade rather than fail
- **Structured JSON logging**, MDC, correlation ids end to end
- What never to log: tokens, passwords, PII (this matters in Phase 8 and 9)
- Log levels that mean something

**Practice:** Wrap the Day 88 client in a circuit breaker; make the dependency
fail and watch the breaker open, reject fast, then recover through half-open.
Add structured JSON logging with a correlation id that appears in every log line
of a request, including async ones.

---

#### Day 90 — Checkpoint Project + Phase 6 Quiz
**Build — Capstone milestone 1.** A document-management API:
- CRUD for documents and collections, correct status codes
- Validation with custom constraints
- `@RestControllerAdvice` returning `ProblemDetail`
- Request/response records, never exposing entities
- OpenAPI docs, pagination
- Structured JSON logging with correlation ids
- Actuator health with a custom indicator
- In-memory repository behind an interface (no database yet)
- Integration tests on Testcontainers

**Plus the async work:**
- A bulk-import endpoint that returns `202 Accepted` immediately and processes
  on a **named `ThreadPoolTaskExecutor` you configured** with a bounded queue
- An endpoint that fans out to three sources with `CompletableFuture` and
  combines them, measured against the sequential version
- A demonstration that `@Async` silently does nothing when self-invoked
- Proof that your correlation id survives into the pool thread after adding a
  `TaskDecorator`
- A second executor for a different workload, with a test proving saturating
  one does not affect the other

**Then:** Phase 6 quiz.

---

## Phase 7: PostgreSQL & Spring Data JPA

**Duration:** 3 weeks — Days 91–105
**Goal:** Your capstone database. Most "the app is slow" incidents are database
problems, so go deeper here than a typical Spring tutorial does.

---

### Week 19: SQL & PostgreSQL Itself

---

#### Day 91 — SQL: Joins, Aggregation & CTEs
**Covers:** inner/left/right/full joins and when each is correct; `GROUP BY` and
`HAVING`; subqueries vs joins; CTEs (`WITH`) for readability; window functions
(`ROW_NUMBER`, `RANK`, `SUM OVER`); `DISTINCT ON`; set operations.

**Practice:** Write the same report three ways — subquery, join, CTE — and
compare readability and plans. Use a window function to rank documents per
tenant by size.

---

#### Day 92 — Data Types & Constraints
**Covers:** `uuid` vs `bigserial` as a primary key; `text` vs `varchar`;
`numeric` vs `float` for money; **`timestamptz` vs `timestamp`** (Day 30 pays
off); `jsonb` and when it's appropriate; arrays and enums; `NOT NULL`, `UNIQUE`,
`CHECK`, foreign keys and cascade behaviour; **why constraints belong in the
database**, not only the application.

**Practice:** Model the document schema with correct types and full constraints.
Try to insert violating data and read each error. Store metadata as `jsonb` and
query inside it.

---

#### Day 93 — Indexes
**Covers:** what an index actually is (a B-tree); when an index helps and when
it hurts (write cost, storage); **composite indexes and why column order
matters**; the leftmost-prefix rule; partial indexes; covering indexes and
index-only scans; GIN for `jsonb` and full-text; unique indexes; why an index on
a low-cardinality column is usually useless.

**Practice:** Seed 100k rows. Time a query with no index, then add a
single-column index, then a composite one in both column orders, and record the
timings for each.

---

#### Day 94 — EXPLAIN ANALYZE
**Covers:** reading a query plan bottom-up; scan types (sequential, index, index-only,
bitmap); join strategies (nested loop, hash, merge); **estimated vs actual rows** and
what a bad estimate means; cost units; buffers; spotting the sequential scan on a
large table; when a sequential scan is correct; `ANALYZE` and statistics.

**Practice:** Run `EXPLAIN ANALYZE` on the Day 93 queries before and after each
index and narrate the plan change. Find a query where the planner chooses a
sequential scan despite an index existing, and explain why.

---

#### Day 95 — Transactions, Isolation & MVCC + Week 19 Review
**Covers:** ACID concretely; `BEGIN`/`COMMIT`/`ROLLBACK`; **isolation levels**
(read committed, repeatable read, serializable) and the anomalies each prevents
— dirty read, non-repeatable read, phantom, write skew; PostgreSQL's default and
what it does *not* protect you from; **MVCC** — how readers don't block writers;
row locks, `SELECT FOR UPDATE`; deadlocks in the database; vacuum and bloat.

**Practice:** Open two `psql` sessions and reproduce a non-repeatable read at
read committed, then prevent it at repeatable read. Create a database deadlock
between two sessions and read the error.

**End of Week 19 — quiz available.**

---

### Week 20: JPA & Hibernate

---

#### Day 96 — Entity Mapping & Ids
**Covers:** `@Entity`, `@Table`, `@Column`; id generation strategies (`IDENTITY`,
`SEQUENCE`, UUID) and **why `IDENTITY` disables JDBC batching**; field vs property
access; `@Enumerated` (never `ORDINAL`); `@Embedded` value objects; mapping
`Instant` and `timestamptz`; why entities need a no-arg constructor and cannot be
records; `equals`/`hashCode` on entities (Day 22 — and it's genuinely subtle here).

**Practice:** Map the document schema. Compare generated SQL for `IDENTITY` vs
`SEQUENCE` with batching enabled. Implement entity equality correctly using the
business key.

---

#### Day 97 — Relationships & the Owning Side
**Covers:** `@ManyToOne`, `@OneToMany`, `@OneToOne`, `@ManyToMany`; **the owning
side** and why the wrong side updates nothing; `mappedBy`; `@JoinColumn`; cascade
types and `orphanRemoval`; bidirectional relationships and keeping both sides in
sync; why `@ManyToMany` usually deserves an explicit join entity.

**Practice:** Map documents to collections both ways. Update from the non-owning
side and watch nothing persist. Add a helper method that maintains both sides.

---

#### Day 98 — Lazy vs Eager and the N+1 Problem
**Covers:** lazy loading and proxies; `LazyInitializationException` and why it
happens outside a transaction; **why `FetchType.EAGER` is almost always wrong**;
**the N+1 problem** — 1 query becoming 101; detecting it by logging SQL and
counting statements; fixes: `JOIN FETCH`, `@EntityGraph`, `@BatchSize`,
DTO projections; when to fetch nothing and query separately.

**Practice:** Build an endpoint that triggers N+1, prove it by counting SQL
statements in the log, then fix it three ways and compare the generated SQL and
timings for each.

---

#### Day 99 — The Persistence Context & @Transactional
**Covers:** the persistence context as a first-level cache; managed / detached /
transient states; dirty checking — **why changing a managed entity persists
without calling save**; flush timing; `@Transactional`: propagation
(`REQUIRED`, `REQUIRES_NEW`), read-only, isolation, timeout; **rollback rules —
only unchecked exceptions roll back by default**; the proxy gotchas (self-invocation
and private methods — exactly like `@Async` on Day 82); transaction boundaries
belonging in the service layer; `@Transactional` and `@Async` together.

**Practice:** Modify a managed entity without calling `save` and confirm it
persists. Throw a checked exception and watch the transaction commit anyway;
fix with `rollbackFor`. Call a `@Transactional` method from a sibling method and
prove no transaction started.

---

#### Day 100 — Spring Data Repositories & Queries + Week 20 Review
**Covers:** `JpaRepository` and what you get free; derived query methods and
their limits; `@Query` with JPQL and native SQL; named parameters; **projections**
(interface and DTO) to avoid loading whole entities; `Specification` for dynamic
queries; pagination and sorting; **why `Pageable` with a count query is expensive**
and keyset pagination as the alternative (Day 80); when to drop JPA and use
`JdbcClient` instead; week recap.

**Practice:** Implement search with a `Specification`. Add a DTO projection and
compare the generated SQL with loading full entities. Implement keyset
pagination and compare against offset paging at page 1,000.

**End of Week 20 — quiz available.**

---

### Week 21: Production Concerns

---

#### Day 101 — Flyway Migrations
**Covers:** why `ddl-auto` must never be `update` in production; versioned vs
repeatable migrations; naming and ordering; **migrations are append-only** —
never edit an applied one; the checksum failure and what it means; rollback
strategy (forward-only, and why); baselining an existing database; testing
migrations with Testcontainers; zero-downtime schema changes (expand/contract).

**Practice:** Convert the schema to Flyway migrations. Edit an applied migration
and read the checksum error. Perform an expand/contract rename across two
migrations without downtime.

---

#### Day 102 — Locking: Optimistic & Pessimistic
**Covers:** the lost update problem; **optimistic locking with `@Version`** and
`OptimisticLockException`; when optimistic is the right default; pessimistic
locking (`PESSIMISTIC_READ`, `PESSIMISTIC_WRITE`) and lock timeouts; deadlock
risk; retrying an optimistic failure; connecting back to Phase 5 — the same race
conditions, now across processes rather than threads.

**Practice:** Reproduce a lost update with two concurrent transactions. Add
`@Version` and watch the second fail. Implement a retry. Then solve the same
problem pessimistically and compare throughput.

---

#### Day 103 — Connection Pooling with HikariCP
**Covers:** why connections are expensive and must be pooled; **HikariCP
settings**: `maximumPoolSize`, `minimumIdle`, `connectionTimeout`,
`idleTimeout`, `maxLifetime`; **why a bigger pool is usually slower** and how to
size against PostgreSQL's `max_connections`; pool exhaustion — the symptom and
the thread dump it produces (Day 69); **the interaction with Phase 5**: a
blocking DB call holds both a request thread and a connection; long transactions
holding connections; monitoring pool metrics; PgBouncer at awareness level.

**Practice:** Set the pool to 2 and fire 20 concurrent requests; observe queueing
and the timeout. Take a thread dump during exhaustion and identify it. Then hold
a transaction open across an HTTP call and explain why that is a serious bug.

---

#### Day 104 — Multi-Tenancy & Row-Level Security
**Covers:** multi-tenancy models — silo, pool, bridge — and their cost/isolation
trade-off; the discriminator-column (pool) approach with `tenant_id`; **why
application-level filtering is not enough** — one forgotten `WHERE` clause leaks
data across tenants; **PostgreSQL Row-Level Security**: enabling it, policies,
`current_setting`; setting the tenant per connection or transaction; the
interaction with connection pooling (a pooled connection must be reset!);
testing isolation; performance implications and indexing `tenant_id` first.

**Practice:** Add `tenant_id` and enable RLS. Write a policy that filters by a
session variable. Prove that a raw `SELECT *` with no `WHERE` returns only the
current tenant's rows. Then forget to reset the setting on a pooled connection
and observe the leak — the bug this whole design exists to prevent.

---

#### Day 105 — Checkpoint Project + Phase 7 Quiz
**Build — Capstone milestone 2.** Move the document API onto PostgreSQL:
- Flyway migrations for the full schema with proper types and constraints
- Entities with correct relationships and id strategy
- Repositories with derived queries, one `@Query`, and a DTO projection
- Seed 100k rows
- **Write an N+1 endpoint, prove it with SQL counts, fix it, document the before
  and after**
- **Add the right composite index and show `EXPLAIN ANALYZE` before and after**
- Optimistic locking with `@Version` plus a retry
- Keyset pagination on the list endpoint
- **Row-Level Security so a tenant cannot read another's rows even with raw SQL**
- HikariCP tuned deliberately, with the reasoning written down
- All integration tests on Testcontainers

**Then:** Phase 7 quiz.

---

## Phase 8: Spring Security Deep Dive

**Duration:** 4 weeks — Days 106–125
**Goal:** Start from *what authentication even means* and finish able to explain
every filter in the chain, validate a JWT correctly, and set up a service
account. **Week 22 contains no Spring at all** — concepts first.

---

### Week 22: The Concepts, Before Any Spring

---

#### Day 106 — Authentication vs Authorization & the Vocabulary
**Covers:**
- **Authentication** = *who are you?* **Authorization** = *are you allowed to do
  this?* Two different questions, two different failures (401 vs 403)
- Almost every security bug is confusing the two
- The vocabulary, each defined properly with an example:
  **principal**, **credential**, **identity**, **role**, **authority**,
  **permission**, **scope**, **claim**, **subject**, **audience**, **issuer**,
  **bearer token**
- **Roles vs permissions** — why role-based checks become unmanageable and
  permission-based checks scale
- Authentication factors; MFA at concept level
- Where each check belongs in a request's journey

**Practice:** For ten realistic failure scenarios, decide whether each is a 401
or a 403 and justify it. Write a one-line definition of every term above without
looking, then check yourself.

---

#### Day 107 — How the Web Remembers You: Cookies, Sessions & Tokens
**Covers:**
- **HTTP is stateless** — the server forgets you between requests, so something
  must travel with each one
- Cookies: what they are, `Set-Cookie`, attributes (`HttpOnly`, `Secure`,
  `SameSite`, expiry) and what each attribute defends against
- **Server-side sessions**: a session id in a cookie, state on the server; where
  that state lives, and why it's a problem across multiple instances
- **Tokens**: state travels with the client, server keeps nothing
- **The real trade-off**:
  - Sessions — revocable instantly, but need shared storage (Redis, Phase 14)
  - Tokens — scale statelessly, but you can't easily un-issue one
- Where each is stored client-side, and the XSS vs CSRF implications of each
- Why "just use JWT" is not automatically correct

**Practice:** Log into any site and inspect the cookies in DevTools — identify
the session cookie and its attributes. Then draw the request/response sequence
for both a session-based and a token-based login.

---

#### Day 108 — Passwords: Hashing, Salts & bcrypt
**Covers:**
- **Never store passwords** — hashing vs encryption vs encoding, and why people
  confuse all three
- Why fast hashes (MD5, SHA-256) are exactly wrong for passwords
- Rainbow tables and what a **salt** prevents
- **bcrypt, scrypt, argon2** — deliberately slow, with a tunable work factor
- Choosing a work factor, and re-hashing as hardware improves
- Timing attacks and constant-time comparison
- Password policies that actually help (length over complexity)
- Credential stuffing and why breach reuse is the real threat
- Storing API keys and other secrets — the same rules apply (Day 123)

**Practice:** Hash the same password with SHA-256 twice and with bcrypt twice;
explain why the SHA outputs match and the bcrypt ones don't. Time bcrypt at work
factors 10, 12 and 14 and pick one with reasoning.

---

#### Day 109 — The Three Kinds of Caller & What a Service Account Is
**Covers:**
- Your system will be called by three fundamentally different clients, and they
  authenticate differently:
  1. **A human in a browser** — interactive login, redirects, sessions or
     tokens, MFA possible
  2. **Another service you own** — no human, no browser, no redirect. This is a
     **service account**: the service authenticates *as itself*
  3. **A third-party integration** — an API key, long-lived, scoped, revocable
- **What a service account actually is**: an identity that represents a program,
  not a person. It has credentials (a client id and secret), permissions
  (scopes), and should be **one per service, least-privilege** — never a shared
  "admin" account
- Why you cannot reuse a user login for a background job
- How machine identity is done in different places — the same problem, three
  answers: client credentials (Phase 8), IAM roles (Phase 15), mTLS certificates
  (Phase 18)
- Credential rotation for machines
- Auditing: knowing which service did what

**Practice:** Map out your capstone: list every caller, classify it into one of
the three kinds, and write down what credential it should present and what
scopes it needs. This document drives Week 24.

---

#### Day 110 — The Filter Chain as a Concept + Week 22 Review
**Covers:**
- A servlet filter: code that runs before and after your controller
- **The security filter chain** as a concept: a line of filters, each with one
  job, that a request passes through before reaching your code
- The conceptual order: does this request need auth? → is there a credential? →
  is it valid? → who is this? → are they allowed here? → proceed
- Why security belongs in a filter, not in your controller
- What "the request is authenticated" means by the time your method runs
- Fail-closed vs fail-open, and why the default must be deny
- Defence in depth: gateway, filter, method, database — four layers you'll build
- Week 22 recap

**Practice:** Draw the full journey of an authenticated request through the
conceptual chain, and mark where a 401 and a 403 would each be produced. Then
predict what happens for a request with no credential, a valid credential and
insufficient permission, and an expired credential.

**End of Week 22 — quiz available. This week is concepts only; be sure they're
solid before Spring appears.**

---

### Week 23: Spring Security Mechanics & JWT

---

#### Day 111 — SecurityFilterChain & the Filter Order
**Covers:**
- Adding Spring Security and what changes immediately (everything locked down,
  a generated password) — and why that default is correct
- **`SecurityFilterChain`** as a bean, and the modern lambda DSL
  (no `WebSecurityConfigurerAdapter`)
- **The actual filter list** — printing it and reading it
- Key filters: `SecurityContextPersistenceFilter`, `UsernamePasswordAuthenticationFilter`,
  `BearerTokenAuthenticationFilter`, `ExceptionTranslationFilter`,
  `AuthorizationFilter`
- Multiple filter chains with `securityMatcher` (e.g. API vs actuator)
- Writing a custom filter and placing it correctly
- `Authentication`, `SecurityContext`, `SecurityContextHolder`
- `AuthenticationEntryPoint` (401) vs `AccessDeniedHandler` (403) — Day 106 made
  concrete

**Practice:** Add Spring Security, print the full filter chain at startup and
identify each filter's job. Configure a custom entry point returning
`ProblemDetail` for 401 and a separate handler for 403, and trigger each.

---

#### Day 112 — Authentication: Providers, UserDetailsService & PasswordEncoder
**Covers:**
- The pieces and their responsibilities: `AuthenticationManager` →
  `AuthenticationProvider` → `UserDetailsService` → `PasswordEncoder`
- `UserDetails` and `GrantedAuthority`
- Implementing a `UserDetailsService` against your own users table
- **`DelegatingPasswordEncoder`** and the `{bcrypt}` prefix — how Spring
  supports upgrading algorithms over time (Day 108)
- `DaoAuthenticationProvider`
- Writing a custom `AuthenticationProvider`
- Form login and HTTP Basic — configuring them, and why neither suits an API
- Handling authentication success and failure events
- Account states: locked, disabled, expired

**Practice:** Implement a `UserDetailsService` backed by PostgreSQL with bcrypt
passwords. Log in with HTTP Basic. Then store a password with `{noop}` and
explain the security implication of what you just did.

---

#### Day 113 — Authorization: URL Rules, Roles vs Authorities & Method Security
**Covers:**
- `authorizeHttpRequests` and matchers
- **Matcher ordering** — first match wins, so `anyRequest()` must be last; the
  classic mistake of shadowing a rule
- **`hasRole` vs `hasAuthority`** and the `ROLE_` prefix that trips up everyone
  (`hasRole("ADMIN")` looks for authority `ROLE_ADMIN`)
- `permitAll`, `authenticated`, `denyAll`
- Custom `AuthorizationManager`
- **Method security**: `@EnableMethodSecurity`, `@PreAuthorize`, `@PostAuthorize`,
  `@PreFilter`, `@PostFilter`
- SpEL in security expressions, including `#id == authentication.name`
- **Why method security matters even with URL rules** — defence in depth, and
  protecting service methods called from a queue or a scheduled job where no
  URL exists
- Same proxy gotchas as `@Async` and `@Transactional`: self-invocation

**Practice:** Write URL rules, then deliberately put `anyRequest().permitAll()`
first and watch every other rule become dead. Add `@PreAuthorize` to a service
method and confirm it still applies when called from a scheduled job with no
HTTP request.

---

#### Day 114 — JWT from First Principles: Structure & Signing
**Covers:**
- What a JWT is: three base64url segments separated by dots
- **Decoding one by hand** — header, payload, signature
- **A JWT is signed, not encrypted** — anyone holding it can read the payload,
  so never put secrets in it
- Standard claims: `sub`, `iss`, `aud`, `exp`, `iat`, `nbf`, `jti` — what each
  is for
- Custom claims (`tenant_id`, `roles`) and keeping tokens small
- **HS256 (shared secret)** vs **RS256 (private/public key pair)**
- Why RS256 is what you want with an external identity provider: the provider
  signs with a private key, your service verifies with the public key and never
  holds a signing secret
- **JWKS** — the endpoint publishing public keys, and key rotation via `kid`
- What the signature proves (integrity and issuer) and what it does not
  (that the token hasn't been stolen)

**Practice:** Take a JWT, split it on the dots, and base64-decode the first two
parts by hand. Generate an HS256 token and verify it; then generate an RS256
pair and verify with only the public key. Fetch a real JWKS endpoint and match a
`kid`.

---

#### Day 115 — Validating a JWT Properly + Week 23 Review
**Covers:**
- **The full validation checklist**, all of it, every time:
  signature → `exp` → `nbf` → `iss` → **`aud`** → required claims
- Why skipping the audience check lets a token issued for another service work
  against yours
- Clock skew tolerance
- **The classic attacks**:
  - `alg: none` — the unsigned token
  - Algorithm confusion (RS256 token verified as HS256 using the public key as
    the secret)
  - Not verifying at all and just decoding
- **Revocation — the genuinely hard problem**: a signed token is valid until it
  expires, so "log out" doesn't invalidate it
  - Short-lived access tokens plus refresh tokens
  - Denylists and what they cost you (you're stateful again)
  - `jti` for targeted revocation
- **Refresh token rotation** and reuse detection as theft evidence
- Where tokens should be stored client-side, and the XSS/CSRF trade-off
- Week 23 recap

**Practice:** Forge a token with `alg: none` and confirm your validation rejects
it. Remove the audience check and prove a foreign token is accepted — then put
it back. Implement refresh rotation and detect a reused refresh token.

**End of Week 23 — quiz available.**

---

### Week 24: OAuth2, OIDC & Service Accounts

---

#### Day 116 — Why OAuth2 Exists & Its Four Roles
**Covers:**
- The original problem: giving an app access to your data **without giving it
  your password**
- The four roles, with a concrete example: **resource owner** (you), **client**
  (the app), **authorization server** (issues tokens), **resource server**
  (your API)
- What a **grant** is, and why there are several
- Access tokens and scopes as delegated, limited permission
- **OAuth2 is authorization, not authentication** — the misunderstanding that
  led to OIDC existing
- Front channel vs back channel, and why that distinction drives flow design
- Reading an OAuth2 error response

**Practice:** Map a real "Sign in with Google" flow onto the four roles. Then map
your own capstone onto them and identify which component plays each part.

---

#### Day 117 — Authorization Code + PKCE
**Covers:**
- **The flow request by request**: redirect to authorization server → user
  authenticates → redirect back with a code → client exchanges code for tokens
- Why the code exists at all — keeping the token off the browser URL
- `state` and CSRF protection on the callback
- `redirect_uri` exact matching and why it must be strict
- **PKCE**: `code_verifier` and `code_challenge`, and the attack it prevents
  (a stolen code being redeemed by an attacker on a public client)
- Public vs confidential clients
- Why the **implicit flow is dead**
- Where tokens land and how the session is maintained afterwards

**Practice:** Draw the complete flow with every parameter. Then run it manually
against Keycloak with `curl` and a browser — get a code, exchange it for a
token, and inspect what you receive.

---

#### Day 118 — Client Credentials & Service Accounts
**Covers:**
- **The machine-to-machine flow.** No user, no browser, no redirect: a service
  presents its own **client id and secret** and receives a token representing
  *itself*
- When you need it: scheduled jobs, service-to-service calls, workers consuming
  a queue — everything from Day 109's second category
- The token request and response in full
- **Scopes as machine permissions** — least privilege for programs
- The `sub` of such a token is the service, not a person — and what that means
  for your authorization rules and audit logs
- **Token caching**: fetch once, reuse until near expiry, refresh proactively —
  never fetch a token per request
- Handling token fetch failure and clock skew
- **Credential management**: one client per service, rotation, storage in a
  secret manager (never in `application.yml`)
- How this compares to the alternatives: **IAM roles** (Phase 15) and **mTLS**
  (Phase 18) — three answers to "how does a machine prove who it is"
- Common mistakes: one shared service account for everything, over-broad scopes,
  a secret committed to Git

**Practice:** Create a service-account client in Keycloak with a narrow scope.
Write a second small Spring app that obtains a token via client credentials,
caches it until 30 seconds before expiry, and calls your API. Inspect the token
and confirm the `sub` is the service. Then request an endpoint outside its scope
and confirm the 403.

---

#### Day 119 — OIDC & Keycloak Hands-On
**Covers:**
- **OIDC = OAuth2 + identity.** OAuth2 alone never tells you *who* the user is
- **`id_token` (who you are) vs `access_token` (what you may do)** — the single
  most useful distinction here
- ID token claims and validating them
- The discovery document (`/.well-known/openid-configuration`) and what it gives you
- Userinfo endpoint
- Scopes `openid`, `profile`, `email`
- **Keycloak concretely**: realms, clients (public vs confidential), users,
  **realm roles vs client roles**, groups, client scopes
- **Protocol mappers** — putting `tenant_id` and roles into the token, which is
  exactly what Phase 7's RLS needs
- Running Keycloak in your Compose stack (Day 48)
- Logout, and why it's harder than it sounds with tokens

**Practice:** Stand up Keycloak in Compose. Create a realm, a confidential
client, a user, and roles. Add a protocol mapper that puts `tenant_id` into the
access token. Complete a login and decode both tokens, identifying which claims
come from which.

---

#### Day 120 — Spring as Resource Server & Client + Week 24 Review
**Covers:**
- **Resource server**: validating incoming JWTs
  - `oauth2ResourceServer(jwt())`, `issuer-uri`, automatic JWKS fetching and caching
  - What Spring validates by default and **what you must add** (audience!)
  - Custom `JwtDecoder` with extra validators
- **Mapping claims to authorities** with a `JwtAuthenticationConverter` — turning
  Keycloak's nested `realm_access.roles` into Spring authorities (the fiddly bit
  everyone hits)
- **OAuth2 client**: obtaining tokens
  - `spring-boot-starter-oauth2-client`, registrations and providers
  - `OAuth2AuthorizedClientManager` handling caching and refresh for you
  - Wiring it into `RestClient` for outbound service calls (Day 88)
- Testing: `jwt()` post-processors, `@WithMockUser`
- Week 24 recap

**Practice:** Configure your API as a resource server against Keycloak, add an
audience validator, and map realm roles to authorities so `hasRole` works.
Confirm a token from the wrong audience is rejected. Then configure the client
side so your second app's calls are authenticated automatically.

**End of Week 24 — quiz available.**

---

### Week 25: Hardening, Multi-Tenancy & Proving It

---

#### Day 121 — SecurityContext Propagation, CSRF & CORS
**Covers:**
- **`SecurityContextHolder` is a `ThreadLocal`** (Day 60) — so it is **empty in
  an `@Async` method or a `CompletableFuture` task** (Day 85). The bug: your
  `@PreAuthorize` service method throws or silently sees an anonymous user when
  called asynchronously
- Fixes: `DelegatingSecurityContextAsyncTaskExecutor`,
  `SecurityContextHolder.setStrategyName(MODE_INHERITABLETHREADLOCAL)` and its
  dangers with pooled threads
- Propagating identity into a background job or Kafka consumer (Phase 13) —
  where there is no request at all
- **CSRF**: what the attack actually is, step by step; why cookie-based auth is
  vulnerable and bearer-token auth generally isn't; when disabling it is correct
  and when it's a vulnerability; `SameSite` cookies as a partial defence
- **CORS**: the same-origin policy, preflight requests, and what the headers do;
  why `allowedOrigins("*")` with `allowCredentials(true)` is rejected; where CORS
  config belongs (and that it is *not* a security control for your API)

**Practice:** Call a `@PreAuthorize` service method from an `@Async` method and
watch authorization fail; fix it with the delegating executor. Then disable CSRF
on a cookie-authenticated form endpoint and write the HTML page that exploits it.

---

#### Day 122 — Tenant Context from Claims → Row-Level Security
**Covers:**
- Extracting `tenant_id` from the validated JWT (Day 119's mapper)
- Storing it in a request-scoped bean or `ThreadLocal` tenant context
- **Setting the PostgreSQL session variable per transaction** so RLS (Day 104)
  applies automatically
- The connection-pool hazard: a pooled connection carrying the previous
  tenant's setting — and how to reset it reliably
- Why this belongs in one place (a filter or an aspect), never in each repository
- **Defence in depth**: the token says the tenant, the service enforces it, the
  database enforces it again — so one forgotten check isn't a breach
- **IDOR** — the attack this defends against: changing an id in a URL to read
  another tenant's data
- Auditing tenant access

**Practice:** Wire the `tenant_id` claim through to the RLS session variable.
Then attempt IDOR: authenticate as tenant A and request a document belonging to
tenant B by its id. Confirm it returns 404 (not 403 — think about why) and that
even a raw repository query returns nothing.

---

#### Day 123 — API Keys: Hashing, Scoping & Rotation
**Covers:**
- When an API key is the right credential (third-party integrations, Day 109's
  third category) and when it isn't
- Generating one with sufficient entropy
- **Storing only a hash** — the same rule as passwords (Day 108), because your
  database will eventually be dumped
- The lookup problem: you can't search by hash, so use a **key prefix** for
  lookup plus a hash for verification
- Displaying a key exactly once at creation
- Scoping keys to specific permissions
- Expiry and rotation without downtime (overlapping keys)
- Revocation, and why this is easier than JWT revocation
- Writing a custom `AuthenticationFilter` for API keys alongside JWT auth
- Rate limiting per key (Phase 14)

**Practice:** Implement scoped API keys: generate, show once, store a prefix plus
bcrypt hash, and authenticate with a custom filter that populates the
`SecurityContext`. Support two active keys during a rotation window, then revoke
one and confirm it fails immediately.

---

#### Day 124 — OWASP in Practice & Testing That Access Is Denied
**Covers:**
- **OWASP Top 10** against your own code, not in the abstract:
  - Broken access control and **IDOR** — the top risk for multi-tenant apps
  - Injection: how JPA parameter binding protects you and how native queries and
    dynamic `Sort` can undo it
  - Mass assignment — why binding a request body straight to an entity is
    dangerous (Day 79)
  - SSRF — user-supplied URLs, which matters when you ingest documents (Phase 10)
  - Security misconfiguration, exposed actuator endpoints
  - Sensitive data in logs (Day 89)
- **Security headers**: HSTS, CSP, `X-Content-Type-Options`, frame options
- Secrets management: environment, vault, AWS Secrets Manager (Phase 15)
- Auditing authentication and authorization events
- **Testing security properly**: `spring-security-test`, `@WithMockUser`,
  `jwt()` post-processors — and the crucial point that you must test that access
  is **denied**, which is the test everyone forgets
- Negative tests as first-class: no token, expired token, wrong tenant, wrong
  scope, insufficient role

**Practice:** Write the negative test suite: every protected endpoint tested
with no token, an expired token, a valid token for the wrong tenant, and a token
lacking the required scope. Then add security headers and verify them.

---

#### Day 125 — Checkpoint Project + Phase 8 Quiz
**Build — Capstone milestone 3.** Fully secure the document API:

- **Keycloak** in Docker as the identity provider, configured in a realm you
  built yourself
- **Authorization code + PKCE** for human users
- **A second service calling your API with the client credentials flow using its
  own service account** — narrow scopes, token cached and refreshed proactively,
  secret held outside the codebase
- **Scoped, hashed API keys** as a third credential type, with rotation
- JWT validation as a resource server against JWKS, with `iss` **and `aud`**
  verified and realm roles mapped to authorities
- `tenant_id` claim → tenant context → **PostgreSQL RLS**, reset correctly on
  pooled connections
- Method security on service methods, working when called from a scheduled job
- `SecurityContext` propagated correctly into `@Async` work
- Security headers, CSRF and CORS configured deliberately with the reasoning
  written down
- **A negative test suite proving denial** at controller, service and database
  layers, including an IDOR attempt across tenants
- **Break it on purpose**: forge an `alg: none` token, remove the audience check,
  and attempt IDOR — confirm all three are caught, and write up what each attack
  would have achieved

**Then:** Phase 8 quiz — the security one. Take it seriously; this is the phase
you named as a career priority.

---

# Phases 9–21 — Outline

These get expanded into day-level detail as you reach them. Ask to
**"expand Phase 9 into the plan"** when you finish Phase 8.

| Phase | Topic | Days | Checkpoint |
|-------|-------|------|-----------|
| 9 | AI Foundations for Java Engineers | 126–135 | **Capstone M4** — typed AI summarization endpoint, SSE streaming, token/cost tracking, prompt-injection defence |
| 10 | RAG & Vector Search with pgvector | 136–145 | **Capstone M5** — full RAG with HNSW, hybrid search, citations, 25-question eval set |
| 11 | System Design Fundamentals | 146–160 | Written design doc for the capstone + a recorded 20-minute walkthrough |
| 12 | Proxies, LBs & API Gateways | 161–170 | Nginx with TLS, rate limiting and health checks in front of two app instances; SSE verified through the proxy |
| 13 | Async Messaging with Kafka | 171–180 | **Capstone M6** — async ingestion via outbox pattern, idempotent consumer, DLQ, audit topic |
| 14 | Caching, Rate Limiting & Multi-Tenancy | 181–190 | **Capstone M7** — Redis semantic cache, Lua sliding-window limiter, noisy-neighbour demo |
| 15 | AWS Core Services | 191–205 | Manual deploy: VPC, ECS Fargate, RDS, ElastiCache, ALB, Secrets Manager — then tear down |
| 16 | Terraform & IaC | 206–215 | All of Phase 15 as modules with remote state; destroy and recreate from scratch |
| 17 | Kubernetes & Orchestration | 216–235 | **Capstone M8** — full deploy to EKS with Helm, probes, HPA, RBAC, NetworkPolicies; then break it on purpose |
| 18 | Istio Service Mesh & mTLS | 236–245 | Strict mTLS, authorization policies, 10% canary, circuit breaker triggered by fault injection |
| 19 | Observability, CI/CD & GitOps | 246–260 | **Capstone M9** — Actions → ECR → ArgoCD, per-tenant RED dashboards, AI cost dashboard, alerts with runbooks |
| 20 | Agents, Tool Calling & MCP | 261–270 | **Capstone M10** — authorized, audited tools; your own MCP server; 20-task eval suite |
| 21 | Self-Hosted Inference & GPU Serving | 271–280 | vLLM behind the Java gateway, KEDA autoscaling on GPU nodes, cost comparison, automatic fallback |
| — | **Capstone integration** | 281–295 | Harden, load-test with Gatling, chaos-test, document and present |

---

## Progress Log

Fill this in as you finish each phase — it's the fastest way to see momentum.

| Phase | Topic | Finished | Checkpoint done | Notes |
|-------|-------|----------|-----------------|-------|
| 1 | Linux & the Shell | | | |
| 2 | Networking | | | |
| 3 | Java Core (incl. Java 8) | | | |
| 4 | Docker | | | |
| 5 | Concurrency & Executors | | | |
| 6 | Spring Boot & Async | | | |
| 7 | PostgreSQL & JPA | | | |
| 8 | Spring Security | | | |
