# Detailed Learning Roadmap

Java backend depth first, infrastructure interleaved, AI woven through — all
converging on one capstone system.

---

## Track Framework

Every production system has layers. Learn to think in all of them:

| Track | What it is | Examples |
|-------|-----------|---------|
| **Core** | Foundations that outlive frameworks | Linux, networking, the Java language, containers |
| **Java workload** | The services you write | Spring Boot, JPA, Spring Security, concurrency |
| **AI** | LLMs as a component, in your language | Spring AI, embeddings, RAG, pgvector |
| **Scale** | Surviving load and many tenants | System design, gateways, Kafka, caching |
| **Operate** | Running it for real | AWS, Terraform, Kubernetes, Istio, observability, GitOps |
| **AI systems** | The specialism | Agents, tool calling, MCP, self-hosted inference |

A note on ordering: Docker arrives early (Phase 4) because almost everything
after it — PostgreSQL, Redis, Kafka, Keycloak — is easiest to run as a
container. System design lands mid-roadmap (Phase 11) once you have enough
concrete experience for the abstractions to mean something.

---

# CORE

---

## Phase 1: Linux & the Shell (2 weeks — Days 1–10)

Everything you deploy runs on Linux. You're on Windows, so **WSL2 Ubuntu is
your Linux box** — open the *Ubuntu (WSL)* tab in the lab and stay there for
this whole phase.

### What to Learn

**Filesystem and navigation**
- Hierarchy: `/etc`, `/var`, `/tmp`, `/proc`, `/sys`, `/usr`, `/home`, `/opt`
- Why `/proc` isn't a real filesystem — live kernel and process state
- Absolute vs relative paths, `~`, globbing
- `find`, `locate`, `which`, `tree`

**Permissions and users**
- Permission bits, `chmod` numeric and symbolic, `chown`, `umask`
- Why a JAR failing to start is often a permissions problem
- `sudo`, `/etc/passwd`, `/etc/group`
- setuid/setgid — what they are and why they're a security topic

**Processes**
- `ps aux`, `top`/`htop`, reading load average
- Signals: `SIGTERM` vs `SIGKILL` vs `SIGHUP` — and why your Spring Boot app
  needs to handle `SIGTERM` for graceful shutdown in Kubernetes later
- `kill`, `pkill`, background jobs, `nohup`
- `systemctl` (note: WSL2 needs `systemd=true` in `/etc/wsl.conf`)

**Text processing** — the skill you'll use every single day in production
- `grep` with `-i -r -v -E -A -B -C`
- `sed` substitution and in-place edits
- `awk` field extraction and filtering
- `jq` for JSON — you'll parse API responses constantly
- Pipes, redirection, `tee`, `xargs`

**Shell scripting**
- Shebang, executability, `$?`, exit codes
- Variables and quoting (why `"$var"` and not `$var`)
- `if`, `for`, `while`, test operators, `[[ ]]`
- `set -euo pipefail` and why every script should start with it
- Functions, arguments, `trap` for cleanup

**Files and packages**
- `apt` — install, search, show, where files land
- `tar`, `gzip`, archives
- `cron` and `crontab` syntax

### WSL2 specifics worth knowing

- Windows drives are at `/mnt/c`, `/mnt/d` — and I/O there is **slow**. Keep
  project files in the Linux filesystem (`~/`) for anything build-heavy.
- `localhost` is shared with Windows, so a service on `:8080` in WSL is
  reachable from a Windows browser.
- `explorer.exe .` opens the current Linux directory in Windows Explorer.

### Resources
- [The Linux Command Line (free book)](https://linuxcommand.org/tlcl.php)
- [Julia Evans' zines](https://wizardzines.com/) — bite-sized and excellent
- [ExplainShell](https://explainshell.com/) — paste any command, get it parsed

### Checkpoint Project
A log-watching script: tail a file, filter lines matching a pattern, extract
fields with `awk`, and `curl` a webhook when a threshold is crossed. Must use
`set -euo pipefail`, a `trap` for cleanup, and exit codes that mean something.

---

## Phase 2: Networking Fundamentals (2 weeks — Days 11–20)

Every bug you'll chase in distributed systems is, eventually, a networking
question. This is also the phase that makes Kubernetes and Istio comprehensible
later instead of magical.

### What to Learn

**The model**
- OSI vs TCP/IP — focus hard on L3 (IP), L4 (TCP/UDP), L7 (HTTP)
- Encapsulation: what a packet actually looks like going down the stack

**IP and routing**
- IPv4 addressing, private ranges, why `192.168.x.x` and `10.x.x.x` are special
- Subnets and CIDR — be able to read `10.0.1.0/24` and say the usable range
  without thinking. You'll need this for VPCs and Kubernetes pod CIDRs.
- Routing tables, default gateway, NAT

**TCP and UDP**
- The three-way handshake, and what a `SYN` flood is
- Connection teardown, `TIME_WAIT`, and why it shows up in load tests
- Head-of-line blocking
- Why UDP exists and who uses it
- Connection pooling — the reason your JDBC pool and HTTP client have one

**DNS**
- Resolution order, recursive vs authoritative
- Record types: A, AAAA, CNAME, MX, TXT, SRV
- TTL and caching — the cause of half of all "it works on my machine"
- `dig` and `nslookup`

**HTTP and TLS**
- Methods, status codes, idempotency, safe methods
- Headers that matter: `Content-Type`, `Authorization`, `Cache-Control`,
  `X-Forwarded-For`
- Keep-alive and connection reuse
- HTTP/1.1 vs HTTP/2 vs HTTP/3
- **TLS handshake** — certificates, chain of trust, CA, SNI. This is the
  foundation for mTLS in Phase 18, so don't skim it.
- Self-signed certs, and why browsers complain

**Diagnostics** — build the reflex
- `curl` deeply: `-v`, `-H`, `-X`, `-d`, `-o`, `-w`, `--resolve`
- `ss` / `netstat`, `ping`, `traceroute`, `dig`, `tcpdump` basics
- `nc` for testing whether a port is even open

**Firewalls**
- `iptables` / `ufw` conceptually
- How this maps to AWS Security Groups later

### Resources
- [High Performance Browser Networking (free)](https://hpbn.co/) — chapters 1–4
- [Julia Evans' networking zines](https://jvns.ca/networking-zine/)
- [How HTTPS works (comic)](https://howhttps.works/)

### Checkpoint Project
Run a small HTTP service in WSL. Then: capture its traffic with `tcpdump`,
inspect the TLS handshake of a public site with `openssl s_client`, write a
`curl`-based health-check script that reports timing breakdown with `-w`, and
document the full path of one request from browser to process.

---

## Phase 3: Java Core Rebuild (4 weeks — Days 21–40)

The highest-leverage phase in the roadmap — everything after it is Java. Built
from the ground up, including a **full week on Java 8 functional programming**,
because lambdas and functional interfaces are the foundation that streams,
`CompletableFuture`, and half of Spring's API all stand on.

### What to Learn

**Week 1 — how Java actually works underneath**
- JVM, JRE, JDK; bytecode; what "compiled" means here
- **Stack vs heap** — what lives where, and per-thread stacks (this is the
  mental model concurrency needs in Phase 5)
- References vs objects; what "pass by value" really means in Java
- Primitives vs wrappers, autoboxing costs, the `Integer` cache
- Garbage collection: reachability, generations; how leaks still happen
- `equals`/`hashCode` — the contracts, and what breaks in a `HashMap` without them
- `==` vs `.equals`, the string pool, why `String` is immutable
- Exceptions: checked vs unchecked, try-with-resources, wrapping and causes
- Generics: type erasure, bounded types, wildcards (`? extends` / `? super`, PECS)
- `Optional` used correctly — never `.get()`, never a field, never a parameter

**Week 2 — Java 8: lambdas and functional programming**

This is the week that makes the rest of Java readable.

- The problem lambdas solve: anonymous inner classes, and the ceremony they required
- **What a lambda actually is** — a `@FunctionalInterface` implementation, not
  a "function"; what the compiler generates
- Lambda syntax in all its forms; type inference; effectively-final capture
- **The `java.util.function` toolkit**, one at a time with real uses:
  - `Function<T,R>` and `andThen` / `compose`
  - `Predicate<T>` and `and` / `or` / `negate`
  - `Supplier<T>` — and why lazy evaluation matters (`orElseGet` vs `orElse`)
  - `Consumer<T>` and `andThen`
  - `BiFunction`, `UnaryOperator`, `BinaryOperator`
  - Primitive variants (`IntPredicate`, `ToIntFunction`) and why they exist
- **Method references**: static, instance, arbitrary-object, and constructor —
  and how each maps back to a lambda
- Writing your own functional interface, and when that's better than a generic one
- **Default and static methods on interfaces** — why Java 8 needed them
  (interface evolution), the diamond problem, and how Spring uses them
- Functional composition: building a pipeline of `Predicate`s and `Function`s
- `java.time`: `LocalDate`, `LocalDateTime`, `Instant`, `Duration`, `Period`,
  `ZonedDateTime`, formatting — and why the old `Date`/`Calendar` API is banned
- **UTC vs local time**, and storing `timestamptz` (pays off in Phase 7)

**Week 3 — collections and streams**
- `ArrayList` vs `LinkedList` — and why it's almost always `ArrayList`
- `HashMap` internals: buckets, collisions, resize, treeification
- `TreeMap`, `LinkedHashMap`, LRU caches, when ordering matters
- `Set` implementations, `EnumMap`, `ArrayDeque`
- Big-O of the operations you actually call
- `computeIfAbsent`, `merge`, `getOrDefault`
- `Comparable` vs `Comparator`; comparator chaining
- **Streams built on Week 2's foundation**: source → intermediate → terminal
- Laziness, short-circuiting, statelessness
- `map`, `filter`, `flatMap`, `reduce`
- `Collectors`: `toList`, `toMap` (and its duplicate-key trap), `groupingBy`
  with downstream collectors, `partitioningBy`, `joining`
- Primitive streams and avoiding boxing
- **When a plain loop is better** — and it often is
- Why parallel streams are usually the wrong answer

**Week 4 — modern Java, design and testing**
- Java 21: `record`, sealed interfaces, pattern matching for `switch`, record
  patterns, text blocks, `var`
- Modelling success/failure with sealed records instead of exceptions or nulls
- SOLID — especially Single Responsibility and **Dependency Inversion**,
  because Spring is built on the latter
- Composition over inheritance; immutability as a default; builders
- Package structure: by feature, not by layer
- Testing: JUnit 5, AssertJ, parameterized tests, `@Nested`, hand-rolled fakes
- Maven: lifecycle phases, dependency scopes, the dependency tree, BOMs

### Resources
- [Effective Java, 3rd ed. (Bloch)](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/) — items 10–17, 42–48 (42–44 are exactly the Week 2 material)
- [Java 21 language features](https://docs.oracle.com/en/java/javase/21/language/)
- [Oracle's lambda tutorial](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)
- [Baeldung](https://www.baeldung.com/) — good for targeted lookups

### Checkpoint Project
An in-memory library system with **no framework at all**: records for domain
types, sealed interfaces for results, a `Repository<T, ID>` interface with a
`HashMap` implementation, custom exceptions, stream-based queries, and a full
JUnit 5 suite. Build a validation pipeline by composing `Predicate`s, and a
transformation pipeline by composing `Function`s — written first as anonymous
classes, then rewritten as lambdas and method references so you can see exactly
what Java 8 bought you. Deliberately ship `equals` without `hashCode`, prove the
`HashSet` bug in a test, then fix it.

---

## Phase 4: Docker & Containers (2 weeks — Days 41–50)

Docker arrives now because from here on, every dependency — PostgreSQL, Redis,
Kafka, Keycloak — is a container. Learning it now saves you from installing
things you'll want to throw away.

### What to Learn

**Concepts**
- What a container actually is: namespaces + cgroups, not a VM
- Images vs containers vs registries; layers and the union filesystem
- Why an image layer is immutable and what that means for caching

**Dockerfiles**
- `FROM`, `RUN`, `COPY`, `WORKDIR`, `ENV`, `EXPOSE`, `CMD` vs `ENTRYPOINT`
- Layer caching: ordering instructions so a code change doesn't re-download
  every Maven dependency
- **Multi-stage builds** — build with the JDK, run on the JRE
- `.dockerignore`

**Java in containers specifically**
- Why a JVM in a container used to eat all the host's memory, and how
  `-XX:MaxRAMPercentage` fixes it
- Container-aware JVM ergonomics
- Layered JARs and Spring Boot's `layertools` for better caching
- `jlink` custom runtimes; distroless and Alpine base images
- Choosing a base image: `eclipse-temurin:21-jre-jammy` vs distroless

**Running containers**
- Port mapping, volumes vs bind mounts, env vars, restart policies
- `docker logs`, `exec`, `inspect`, `stats`
- Networks: bridge, host, and how containers resolve each other by name

**Compose**
- Services, networks, volumes, `depends_on`, health checks
- A `docker-compose.yml` with PostgreSQL + Redis you'll reuse for months
- `.env` files

**Practices**
- Non-root users, minimal images, no secrets in layers, one process per container
- Tagging strategy — never deploy `latest`

### Resources
- [Docker docs — Get Started](https://docs.docker.com/get-started/)
- [Spring Boot container images](https://docs.spring.io/spring-boot/reference/packaging/container-images/)
- [Dockerfile best practices](https://docs.docker.com/build/building/best-practices/)

### Checkpoint Project
Containerize the Phase 3 project (wrapped in a trivial HTTP server) with a
multi-stage Dockerfile under 250MB, running as non-root. Write the Compose file
with PostgreSQL and Redis plus health checks. Prove layer caching works: change
one line of Java and show the dependency layer is reused.

---

# JAVA WORKLOAD

---

## Phase 5: Concurrency, Executors & CompletableFuture (4 weeks — Days 51–70)

Four weeks, built from zero. A full week on **executors and thread pools** and
a full week on **`CompletableFuture`**, because these are the two things you'll
actually reach for in Spring services — and the two most commonly used without
being understood.

### What to Learn

**Week 1 — the foundations, slowly**
- What a thread *is*: its own stack, shared heap, scheduled by the OS
- Threads vs processes; context switching and what it costs
- Creating threads the old way (`Thread`, `Runnable`) — and why you won't
- Why concurrency is hard: **the three problems**
  - **Visibility** — one thread can't see another's write
  - **Atomicity** — `count++` is three operations, not one
  - **Ordering** — the compiler and CPU reorder your code
- The **Java Memory Model** and happens-before, explained plainly first
- `volatile`: fixes visibility, does *not* fix atomicity
- Writing a race condition on purpose and watching money vanish
- `synchronized`: intrinsic locks, what object you're locking on, reentrancy
- Lock granularity; the cost of holding a lock too long
- Deadlock, livelock, starvation — and lock ordering as the cure
- Immutability as the concurrency strategy that needs no locks (from Phase 3)

**Week 2 — executors and thread pools**

The mental model: *a pool is a fixed set of worker threads pulling tasks off a
queue.* Everything else is configuration of that sentence.

- Why pools exist: thread creation is expensive, and unbounded threads kill a server
- `Executor` vs `ExecutorService` vs `ScheduledExecutorService`
- The `Executors` factory methods — and **why you should avoid most of them**
  (`newFixedThreadPool` and `newCachedThreadPool` both have unbounded queues or
  unbounded threads)
- **`ThreadPoolExecutor` in full**: core pool size, max pool size, keep-alive,
  the work queue, the thread factory, the rejection handler
- **The non-obvious growth rule**: the pool only grows past core size when the
  queue is *full* — which is why an unbounded queue means max size is never used
- Queue choice: `LinkedBlockingQueue` vs `ArrayBlockingQueue` vs `SynchronousQueue`
- **Rejection policies**: abort, caller-runs, discard — and which gives you backpressure
- **Sizing a pool**: CPU-bound (≈ cores) vs IO-bound (much larger), and the
  formula plus why you should measure instead
- Naming your threads, and why an unnamed pool makes production debugging miserable
- Graceful shutdown: `shutdown` vs `shutdownNow`, `awaitTermination`
- `ScheduledExecutorService` for periodic work
- `Callable`, `Future`, and **why `Future.get()` blocks** — the limitation that
  motivates all of Week 3
- Concurrent collections: `ConcurrentHashMap`, `CopyOnWriteArrayList`,
  `BlockingQueue` and the producer-consumer pattern
- Atomics and CAS: `AtomicInteger`, `AtomicReference`, `LongAdder`
- `ReentrantLock`, `ReadWriteLock`, `Semaphore`, `CountDownLatch` — and when
  each beats `synchronized`
- `ThreadLocal` and why it **leaks in a pooled thread** if you don't clean up

**Week 3 — CompletableFuture**

- The problem it solves: `Future` can only be polled or blocked on
- Creating one: `supplyAsync`, `runAsync`, `completedFuture`, manual `complete`
- **Which thread runs your code** — the common ForkJoinPool by default, and why
  that's a trap for IO-bound or blocking work
- **Always pass your own `Executor`** — the single most important habit here
- Transforming: `thenApply` vs `thenApplyAsync`
- **`thenApply` vs `thenCompose`** — map vs flatMap, and the nested-future bug
  that teaches you the difference
- Combining: `thenCombine`, `thenAcceptBoth`, `runAfterBoth`
- Fan-out/fan-in: `allOf`, `anyOf`, and collecting results from `allOf` properly
- Exception handling: `exceptionally`, `handle`, `whenComplete` — and how an
  exception propagates through a chain
- Timeouts: `orTimeout`, `completeOnTimeout`
- Cancellation and its limits
- Composing a real workflow: three parallel service calls, combined, with a
  timeout and a fallback
- Debugging: why stack traces are unhelpful and what to log instead
- When `CompletableFuture` is overkill and a simple parallel `invokeAll` is fine

**Week 4 — virtual threads and applied concurrency**
- **Virtual threads (Java 21)**: platform vs virtual, carrier threads, why
  blocking suddenly becomes cheap
- What this changes: thread-per-request at 100k concurrent requests
- **Pinning** — `synchronized` blocks and native calls that still block a carrier
- Why you should not pool virtual threads
- Structured concurrency: treating concurrent subtasks as a unit
- Thread-per-request vs reactive vs virtual threads — the honest trade-offs
- Testing concurrent code; stress testing; `jcstress` at awareness level
- **Diagnosing in production**: thread dumps, `jstack`, reading a dump,
  spotting a deadlock, spotting pool exhaustion
- Common production failures: pool starvation, nested pool deadlock, unbounded
  queue memory growth

### Resources
- [Java Concurrency in Practice (Goetz)](https://jcip.net/) — still the reference; chapters 2, 3, 5, 6, 8
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Oracle's virtual threads guide](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
- [CompletableFuture guide (Baeldung)](https://www.baeldung.com/java-completablefuture)

### Checkpoint Project
A bank-transfer simulator, built in stages:

1. Unsynchronized balance under 1,000 concurrent transfers — prove money vanishes
2. Fix it three ways (`synchronized`, `AtomicInteger`, `ReentrantLock`) and benchmark each
3. Deadlock two accounts transferring to each other, capture the thread dump,
   then resolve it with lock ordering
4. Build a `ThreadPoolExecutor` by hand with named threads, a bounded queue and
   a caller-runs rejection policy — then flood it and observe backpressure
5. Prove the growth rule: show max pool size is never reached with an unbounded queue
6. Rewrite the transfer report as a `CompletableFuture` pipeline — three parallel
   lookups combined, on your own executor, with a timeout and a fallback
7. Run the whole thing on platform threads vs virtual threads at 10,000 tasks
   and compare throughput and memory

---

## Phase 6: Spring Boot Core, REST & Async (4 weeks — Days 71–90)

Now build the service that everything else in the roadmap will extend. **This
is where the capstone codebase starts.** Week 3 is dedicated to **threading
inside Spring** — taking Phase 5's executors and `CompletableFuture` and
showing how they actually appear in a real service.

### What to Learn

**Week 1 — the framework**
- The IoC container: what a bean is, and what "the container manages it" means
- Dependency injection — constructor injection only, and why (this is Phase 3's
  Dependency Inversion made concrete)
- Component scanning, stereotypes, `@Bean` vs `@Component`
- Bean scopes and lifecycle; `@PostConstruct`, `@PreDestroy`
- Auto-configuration: how `@ConditionalOn*` decides, and reading the actuator
  `conditions` report to see why a bean did or didn't appear
- Starters and what they actually pull in
- Configuration: `application.yml`, profiles, `@ConfigurationProperties`,
  property precedence
- Actuator: health, info, metrics

**Week 2 — APIs done properly**
- REST design: resources, verbs, status codes, idempotency, pagination
- `@RestController`, request mapping, `@RequestBody`, `@PathVariable`,
  `@RequestParam`
- Bean Validation (`@Valid`, groups, custom validators)
- **Global exception handling** with `@RestControllerAdvice` and RFC 7807
  `ProblemDetail`
- DTOs vs entities — never expose entities, and why
- Mapping strategies (manual, MapStruct)
- Content negotiation, Jackson serialization, custom serializers
- API versioning
- OpenAPI with springdoc
- Layering: controller → service → repository, and what belongs in each

**Week 3 — threading and async inside Spring**

- **How a Spring Boot request is served**: Tomcat's thread pool, one thread per
  request, and what `server.tomcat.threads.max` really controls
- What "the request thread is blocked" means, and why 200 slow requests can
  stall a server with 200 threads
- **`@EnableAsync` and `@Async`** — what the annotation actually does (a proxy),
  and the two classic gotchas: it silently does nothing on a **self-invoked**
  call or a **private** method
- Return types: `void`, `Future`, and `CompletableFuture` from an `@Async` method
- **`ThreadPoolTaskExecutor`** — Spring's wrapper over `ThreadPoolExecutor`:
  core size, max size, queue capacity, thread name prefix
- **Why the default executor is dangerous** and how to define your own named
  bean per workload (one pool for AI calls, one for ingestion — bulkheading)
- Choosing the executor with `@Async("myExecutor")`
- Rejection policies and backpressure in a web app
- **`CompletableFuture` in a service layer** — parallel calls to two
  repositories or two APIs, combined, on your own executor
- Async REST endpoints: returning `CompletableFuture` from a controller and
  what that frees up
- **Context propagation**: `SecurityContext`, MDC correlation IDs and request
  scope do **not** cross into a pool thread by default — how to fix it
  (`DelegatingSecurityContextAsyncTaskExecutor`, `TaskDecorator`)
- `@Scheduled` tasks and their separate, single-threaded-by-default pool
- `ApplicationEventPublisher` and `@TransactionalEventListener` for async
  side effects
- **Virtual threads in Spring Boot** (`spring.threads.virtual.enabled=true`) —
  what changes, what it replaces, and what it doesn't fix
- Monitoring pools: Micrometer executor metrics, and alerting on queue depth

**Week 4 — quality and calling out**
- Testing: `@SpringBootTest` vs slices (`@WebMvcTest`, `@DataJpaTest`), MockMvc
- Testing async code without `Thread.sleep` (Awaitility)
- **Testcontainers** — real PostgreSQL in tests instead of H2 lies
- `RestClient` / `WebClient` for outbound calls; **timeouts on every one**
- Retries, circuit breakers and bulkheads with Resilience4j
- Structured JSON logging, MDC, correlation IDs
- Graceful shutdown (ties back to `SIGTERM` from Phase 1) — and draining an
  executor before exit

### Resources
- [Spring Boot reference docs](https://docs.spring.io/spring-boot/index.html)
- [Spring Framework — task execution and scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)
- [Testcontainers for Java](https://java.testcontainers.org/)

### Checkpoint Project
**Capstone milestone 1.** A document-management API: CRUD for documents and
collections, validation, `ProblemDetail` errors, OpenAPI docs, structured
logging with correlation IDs, Actuator health, and an integration test suite on
Testcontainers. No database yet — an in-memory repository behind an interface.

Plus the async work: a bulk-import endpoint that returns immediately and
processes on a **named `ThreadPoolTaskExecutor` you configured**, an endpoint
that fans out to three sources with `CompletableFuture` and combines them, a
demonstration that `@Async` silently does nothing when self-invoked, and proof
that your correlation ID survives into the pool thread once you add a
`TaskDecorator`.

---

## Phase 7: PostgreSQL & Spring Data JPA (3 weeks — Days 91–105)

Your capstone database. Most "the app is slow" incidents are database problems,
so go deeper here than a typical Spring tutorial does.

### What to Learn

**Week 1 — SQL and PostgreSQL itself**
- Joins, aggregation, subqueries, CTEs, window functions
- Data types that matter: `uuid`, `jsonb`, `timestamptz`, `numeric`, arrays
- Constraints, and why they belong in the database not just the app
- **Indexes**: B-tree, composite (and column order!), partial, covering, GIN
- `EXPLAIN ANALYZE` — read a plan, spot a sequential scan, fix it
- Transactions, ACID, isolation levels, MVCC
- Deadlocks and lock contention in PostgreSQL
- Connection pooling: HikariCP sizing, and why "more connections" is wrong

**Week 2 — JPA and Hibernate**
- Entity mapping, ids and generation strategies
- Relationships: `@OneToMany`, `@ManyToOne`, `@ManyToMany`, owning side
- **Lazy vs eager, and the N+1 problem** — cause it, see it in the logs, fix it
  with fetch joins, `@EntityGraph`, and batch sizes
- The persistence context, dirty checking, flush timing
- `@Transactional`: propagation, rollback rules, and why it silently does
  nothing on a private or self-invoked method
- Spring Data repositories, derived queries, JPQL, native queries, projections
- Pagination and sorting done efficiently (keyset vs offset)
- When to drop JPA and use JDBC directly

**Week 3 — production concerns**
- **Flyway** migrations: versioning, repeatable migrations, rollback strategy
- Optimistic locking with `@Version`; pessimistic locking when you must
- Auditing (`@CreatedDate`, `@LastModifiedBy`)
- Soft deletes and their traps
- Multi-tenancy foundations: discriminator column vs schema per tenant
- **Row-Level Security** in PostgreSQL — the capstone's isolation mechanism
- Bulk operations without loading a million entities
- Read replicas and routing reads

### Resources
- [PostgreSQL documentation](https://www.postgresql.org/docs/current/)
- [Use The Index, Luke](https://use-the-index-luke.com/) — free and superb on indexing
- [High-Performance Java Persistence (Mihalcea)](https://vladmihalcea.com/books/high-performance-java-persistence/)

### Checkpoint Project
**Capstone milestone 2.** Move the document API onto PostgreSQL with Flyway
migrations. Then: seed 100k rows, write a deliberately N+1 endpoint and fix it,
show `EXPLAIN ANALYZE` before and after adding the right composite index, add
optimistic locking, and enable Row-Level Security so a tenant physically cannot
read another's rows even with a raw query.

---

## Phase 8: Spring Security Deep Dive (4 weeks — Days 106–125)

You've said you have no idea about these concepts, so this starts from **what
authentication even means** and does not assume a single term. Four weeks, and
the first is pure foundations before Spring appears at all. Most developers
copy a config they don't understand — the goal here is that you can explain
every filter in the chain and defend every decision.

### What to Learn

**Week 1 — the concepts, before any Spring**

- **Authentication vs authorization** — two different questions:
  *who are you?* and *are you allowed to do this?* Almost every security bug
  is confusing the two.
- The vocabulary, defined properly: **principal**, **credential**, **identity**,
  **role**, **authority**, **scope**, **claim**, **subject**, **audience**,
  **issuer**, **bearer token**
- **How the web remembers you at all** — HTTP is stateless, so something must
  travel with every request
  - Cookies and server-side sessions: the session id, where state lives
  - Tokens: the state travels with the client
  - The real trade-off: sessions are revocable but need shared storage; tokens
    scale but can't easily be un-issued
- HTTP Basic, form login, and why neither survives contact with an API
- Password storage: hashing vs encryption, salts, **bcrypt/argon2**, and why
  SHA-256 is the wrong tool
- **The three kinds of caller** your system will have, and why they authenticate
  differently:
  1. A human in a browser
  2. Another service you own — this is a **service account**
  3. A third-party integration using an API key
- The Spring Security **filter chain** as a concept: a request passing through
  a line of filters, each with one job, before it ever reaches your controller

**Week 2 — Spring Security mechanics and JWT**
- `SecurityFilterChain` and the modern lambda DSL (no `WebSecurityConfigurerAdapter`)
- The actual filter order, and how to print it and read it
- `AuthenticationManager`, `AuthenticationProvider`, `UserDetailsService`,
  `PasswordEncoder` — what each one is responsible for
- `Authentication`, `SecurityContext`, `SecurityContextHolder`
- **`SecurityContext` propagation across threads** — why it's empty inside
  `@Async` and how to fix it (this is Phase 6 Week 3 meeting Phase 8)
- Authorization: `authorizeHttpRequests`, matcher ordering pitfalls,
  **roles vs authorities** and the `ROLE_` prefix that trips everyone up
- Method security: `@PreAuthorize`, `@PostAuthorize`, SpEL, `@PostFilter`
- **JWT from first principles**:
  - The three parts: header, payload, signature — decode one by hand
  - **A JWT is signed, not encrypted** — anyone can read the payload
  - Signing: **HS256 (shared secret) vs RS256 (public/private key)** and why
    RS256 is what you want with an external identity provider
  - JWKS and key rotation
  - What the signature actually proves
  - Standard claims: `sub`, `iss`, `aud`, `exp`, `iat`, `nbf`, `jti`
  - **Validating properly**: signature *and* `exp` *and* `iss` *and* `aud` —
    every time, in one place
  - The classic attacks: `alg: none`, algorithm confusion, missing audience check
- **Revocation** — the genuinely hard problem: short TTL + refresh, denylists,
  and why "just log them out" isn't simple
- Refresh token rotation and reuse detection
- Stateless vs stateful — and when a session is still the right answer
- CSRF: what the attack actually is, when disabling it is correct (and when
  it's a vulnerability)
- CORS: where it belongs, preflight, and why wildcard-with-credentials is a bug

**Week 3 — OAuth2, OIDC and service accounts**
- **Why OAuth2 exists**: delegated access without sharing a password
- The four roles: resource owner, client, authorization server, resource server
- **Authorization Code + PKCE** — walked through request by request, for users
- **Client Credentials — the service account flow.** No user is involved: a
  service authenticates *as itself* with a client id and secret to call another
  service. This is the one you'll use constantly and it's rarely explained well.
  - Client id and secret as machine credentials
  - Getting a token, caching it, refreshing before expiry
  - Scopes as machine permissions
  - Why a service account should be least-privilege and per-service, not one
    shared "admin" account
  - How this compares to mTLS (Phase 18) and IAM roles (Phase 15) — three
    answers to the same question
- Why the implicit flow is dead
- **OIDC on top of OAuth2**: `id_token` (who you are) vs `access_token` (what
  you may do); the userinfo endpoint; discovery documents
- **Keycloak hands-on**: realms, clients (public vs confidential), users,
  roles, groups, client scopes, protocol mappers
- Creating a service account client in Keycloak and calling your API with it
- Spring Security as **OAuth2 Resource Server** (validating tokens) and as
  **OAuth2 Client** (obtaining them)
- Mapping claims to Spring authorities with a custom converter

**Week 4 — hardening, multi-tenancy and proving it**
- Mapping a `tenant_id` claim into a request-scoped tenant context
- Wiring that context into PostgreSQL Row-Level Security from Phase 7
- **API keys**: generation, hashing before storage, prefixes for lookup,
  scoping, rotation, and revocation
- Security headers: HSTS, CSP, `X-Content-Type-Options`, frame options
- OWASP Top 10 in practice: injection, **IDOR** (the one that bites
  multi-tenant apps hardest), mass assignment, SSRF, broken access control
- Secrets management — never in `application.yml`; env, vault, Secrets Manager
- Auditing authentication and authorization events
- **Testing security**: `@WithMockUser`, `spring-security-test`, `jwt()`
  post-processors — and testing that authorization actually **denies**, which
  is the test people forget
- Preview of mTLS (Phase 18): certificates as identity

### Resources
- [Spring Security reference](https://docs.spring.io/spring-security/reference/)
- [OAuth 2.0 Simplified (Aaron Parecki)](https://www.oauth.com/) — read this before the Spring docs
- [jwt.io](https://jwt.io/) — paste a token and see it decoded
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Keycloak documentation](https://www.keycloak.org/documentation)

### Checkpoint Project
**Capstone milestone 3.** Secure the document API:

- Keycloak in Docker as the identity provider
- Authorization code + PKCE for human users
- **A second service that calls your API using the client credentials flow with
  its own service account** — token cached and refreshed, least-privilege scopes
- JWT validation as a resource server against Keycloak's JWKS, with `iss` and
  `aud` verified
- `tenant_id` claim extracted into a tenant context driving RLS
- Method-level authorization on service methods
- Hashed, scoped API keys as a third credential type
- A test suite that proves cross-tenant access is **denied** at controller,
  service and database layers
- Break it on purpose: forge a token with `alg: none`, skip the audience check,
  and confirm your validation catches both

---

# AI

---

## Phase 9: AI Foundations for Java Engineers (2 weeks — Days 126–135)

The pivot point. You use AI daily but haven't built with it. This phase gives
you the mental model and the Java toolchain — no Python required.

### What to Learn

**Week 1 — how these things actually work**
- Tokens, context windows, and why cost and latency scale with them
- Temperature, top-p, stop sequences, max tokens
- What a model can and cannot do; hallucination as a structural property, not
  a bug to be patched
- **Embeddings**: text as vectors, cosine similarity, what "semantically near"
  means
- Chat vs completion vs embedding models
- Prompt engineering that matters in production: system prompts, few-shot,
  structured output, delimiters
- Getting **JSON out reliably** — schemas, structured outputs, validation and
  repair
- Model selection and the cost/latency/quality triangle
- Provider landscape: Anthropic, OpenAI, AWS Bedrock, Azure OpenAI, local models

**Week 2 — Spring AI**
- Spring AI project structure, `ChatClient`, `ChatModel` abstractions
- Configuring a provider; swapping providers without rewriting code
- Prompt templates and `@Value`-style substitution
- **Streaming responses** — SSE from a Spring controller to a client
- Mapping model output to Java records (structured output converters)
- Advisors and the chat memory abstraction
- Token usage tracking and cost accounting
- Timeouts, retries, and backoff for a dependency that's slow and flaky
- Caching identical requests
- Observability: logging prompts and responses without leaking PII
- Guardrails: input validation, output filtering, prompt injection awareness
- Evaluation basics — how you know a prompt change made things better

### Resources
- [Spring AI reference](https://docs.spring.io/spring-ai/reference/)
- [Anthropic docs — prompt engineering](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [LangChain4j docs](https://docs.langchain4j.dev/) — the main alternative; worth knowing
- [What are embeddings? (free book)](https://vickiboykis.com/what_are_embeddings/)

### Checkpoint Project
**Capstone milestone 4.** Add an AI endpoint to the document API: summarize a
document, returning a typed Java record, not a string. Stream the response over
SSE. Record token usage and estimated cost per tenant. Add a deliberately
injectable prompt, exploit it, then defend against it. Write an eval harness
that scores 20 fixed inputs so you can compare two prompt versions.

---

## Phase 10: RAG & Vector Search with pgvector (2 weeks — Days 136–145)

Retrieval-Augmented Generation is the workhorse pattern of enterprise AI, and
pgvector means you can do it in the database you already run.

### What to Learn

**Week 1 — retrieval**
- Why RAG exists: grounding answers in your data instead of the model's memory
- The pipeline: ingest → chunk → embed → store → retrieve → rerank → generate
- **Chunking strategies**: fixed size, sentence, recursive, semantic — and how
  chunk size trades recall against precision
- Overlap and why it matters
- Metadata to store alongside each chunk (tenant, source, page, timestamp)
- **pgvector**: the `vector` type, distance operators (`<->`, `<=>`, `<#>`)
- Indexing vectors: **HNSW vs IVFFlat**, build time vs query time vs recall
- Why an exact scan is fine until it isn't
- Spring AI `VectorStore` with the PostgreSQL implementation

**Week 2 — making it good**
- **Hybrid search**: combining vector similarity with PostgreSQL full-text
  search, and why pure vector search misses exact terms like error codes
- Reciprocal Rank Fusion for merging result sets
- Reranking with a cross-encoder
- Query rewriting and multi-query expansion
- **Tenant isolation in a vector store** — filtering by `tenant_id` before
  similarity, and why post-filtering is both slow and a data leak
- Citations: returning which chunk an answer came from
- Handling "I don't know" — the most under-implemented feature in RAG
- **Evaluating RAG**: retrieval metrics (recall@k, MRR) separately from
  generation quality; building a small golden dataset
- Incremental re-indexing when a document changes
- Cost control: caching embeddings, batching

### Resources
- [pgvector README](https://github.com/pgvector/pgvector) — genuinely the best reference
- [Spring AI — RAG](https://docs.spring.io/spring-ai/reference/api/retrieval-augmented-generation.html)
- [Anthropic — contextual retrieval](https://www.anthropic.com/news/contextual-retrieval)

### Checkpoint Project
**Capstone milestone 5.** Full RAG over tenant documents: ingestion pipeline
with configurable chunking, embeddings in pgvector with an HNSW index, hybrid
search fused with full-text, tenant-filtered retrieval enforced by RLS, answers
returned with citations, and an eval set of 25 questions scoring retrieval and
answer quality. Compare two chunking strategies with numbers, not vibes.

---

# SCALE

---

## Phase 11: System Design Fundamentals (3 weeks — Days 146–160)

You named this as a goal, and now you have enough concrete experience for it to
be more than vocabulary. This phase is as much about *explaining* designs as
choosing them.

### What to Learn

**Week 1 — the primitives**
- Latency vs throughput; the latency numbers every engineer should know
- Percentiles: why P99 matters more than the mean, and tail latency amplification
- Vertical vs horizontal scaling; statelessness as the enabler
- Load balancing algorithms and health checking
- Caching: where (client, CDN, gateway, app, DB), what to cache, TTL strategy
- **Cache invalidation**, stampedes, and the thundering herd
- CAP in practice; PACELC as the more useful framing
- Consistency models: strong, eventual, read-your-writes

**Week 2 — data and distribution**
- Replication: leader-follower, multi-leader, quorum
- **Sharding**: by key, range, hash; the resharding problem; hot partitions
- Denormalization as a deliberate choice
- SQL vs NoSQL — decided by access pattern, not fashion
- Idempotency keys and exactly-once as a fiction
- Distributed transactions: two-phase commit and why you avoid it
- **Saga pattern**: choreography vs orchestration
- **Outbox pattern** for the dual-write problem
- CQRS and event sourcing — what they cost
- Clock skew, and why you don't order events by timestamp

**Week 3 — designing and defending**
- Back-of-envelope estimation: QPS, storage, bandwidth
- Failure modes: cascading failure, retry storms, circuit breakers, bulkheads
- Backpressure and load shedding
- Rate limiting algorithms: token bucket, leaky bucket, sliding window
- Graceful degradation
- **Designing under interview conditions**: requirements → constraints →
  high-level → deep dive → bottlenecks → trade-offs
- Practice designs: URL shortener, rate limiter, news feed, chat, notification
  service, and — most relevant to you — a multi-tenant RAG platform

### Resources
- [Designing Data-Intensive Applications (Kleppmann)](https://dataintensive.net/) — chapters 1, 5, 6, 7, 8, 9, 11
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [ByteByteGo](https://blog.bytebytego.com/)

### Checkpoint Project
Write a full design document for the capstone platform: requirements, capacity
estimates, component diagram, data model, API contracts, failure modes, and
explicit trade-offs with rejected alternatives. Then record yourself explaining
it in 20 minutes. This document becomes the reference for Phases 15–21.

---

## Phase 12: Proxies, Load Balancers & API Gateways (2 weeks — Days 161–170)

Every request from outside hits a proxy before your Java code. Understanding
this layer separates people who write services from people who design how
services connect.

### What to Learn

**Proxy fundamentals**
- Forward vs reverse proxy — the direction is the whole distinction
- What a proxy can do: TLS termination, routing, rate limiting, caching, auth,
  header manipulation, compression
- L4 vs L7 load balancing and when you need each

**Nginx**
- Config structure: `server` and `location` blocks
- `proxy_pass`, upstream blocks, load balancing methods
- TLS termination and HTTP→HTTPS redirect
- `limit_req_zone` rate limiting
- Buffering, timeouts, and slow-client protection
- `X-Forwarded-For` / `X-Forwarded-Proto` — and configuring Spring Boot to
  trust them (`server.forward-headers-strategy`)

**Envoy**
- Listeners, clusters, routes, filters
- Why it matters: it's the data plane of Istio in Phase 18

**API gateways** — what they add over a plain proxy
- Auth enforcement at the edge, per-consumer rate limiting, transformation,
  analytics
- **Kong**: services, routes, plugins, consumers
- **Spring Cloud Gateway** — the Java-native option: routes, predicates,
  filters, and when it's the right choice over Kong
- AWS API Gateway — when serverless integration justifies it

**Patterns**
- Path- and header-based routing
- Canary by header
- WebSocket and SSE proxying (your AI streaming endpoint has to survive this)
- Timeouts at every hop, and why the gateway timeout must exceed the app's

### Resources
- [Nginx beginner's guide](https://nginx.org/en/docs/beginners_guide.html)
- [Spring Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/reference/)
- [Kong getting started](https://docs.konghq.com/gateway/latest/get-started/)

### Checkpoint Project
Put Nginx in front of two instances of your Spring Boot app in Compose: TLS with
a self-signed cert, HTTP→HTTPS redirect, round-robin with health checks, and
per-IP rate limiting. Verify your SSE streaming endpoint still works through the
proxy. Then swap in Spring Cloud Gateway and compare what each layer is good at.

---

## Phase 13: Async Messaging with Kafka (2 weeks — Days 171–180)

Synchronous calls couple services together. Your document ingestion pipeline —
upload, chunk, embed, index — is slow and bursty, which makes it the perfect
thing to make asynchronous.

### What to Learn

**Week 1 — Kafka itself**
- Why async: decoupling, buffering, absorbing bursts
- Sync for queries, async for commands
- Topics, partitions, offsets — Kafka as a durable log, not a queue
- Producers: keys and partition assignment, `acks`, idempotent producers
- Consumers and **consumer groups**: rebalancing, partition assignment
- Offset management: auto vs manual commit, and where duplicates come from
- Delivery semantics: at-most-once, at-least-once, and what "exactly-once"
  really means
- Retention, compaction, and replaying history
- `replication.factor`, `min.insync.replicas`, durability trade-offs

**Week 2 — Kafka with Spring, and patterns**
- Spring for Apache Kafka: `KafkaTemplate`, `@KafkaListener`
- Serialization: JSON vs Avro; schema registry and evolution
- Error handling: retry topics, `DeadLetterPublishingRecoverer`, DLQs
- **Idempotent consumers** — mandatory, since redelivery is guaranteed
- **Transactional outbox**: writing to PostgreSQL and publishing atomically
- Ordering guarantees and partition keys (per-tenant ordering)
- Consumer lag as your primary health metric
- Backpressure and concurrency tuning
- Kafka vs RabbitMQ vs SQS — choosing by requirement
- Testing with Testcontainers

### Resources
- [Kafka documentation](https://kafka.apache.org/documentation/)
- [Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/)
- [DDIA chapter 11](https://dataintensive.net/)

### Checkpoint Project
**Capstone milestone 6.** Make ingestion asynchronous: upload returns a job id
immediately and publishes an event; a consumer chunks and embeds the document
and writes results back; the client polls or subscribes for completion. Use the
outbox pattern so the DB write and the event publish can't diverge. Add a DLQ
with a retry topic, make the consumer idempotent, prove it by replaying the same
event twice, and add an audit-log topic.

---

## Phase 14: Caching, Rate Limiting & Multi-Tenancy (2 weeks — Days 181–190)

The patterns that turn "an app" into "a SaaS product."

### What to Learn

**Week 1 — Redis and caching**
- Redis data structures and what each is actually for
- Spring Cache abstraction: `@Cacheable`, `@CacheEvict`, and its limits
- Cache patterns: cache-aside, read-through, write-through, write-behind
- TTL strategy, eviction policies, memory limits
- **Cache stampede** and how to prevent it (locking, early recompute, jitter)
- Caching AI responses: semantic caching by embedding similarity
- Distributed locks with Redis — and why `SETNX` alone isn't safe
- Session storage; sticky sessions vs shared state
- Redis persistence, replication, cluster mode

**Week 2 — limits and tenancy**
- Rate limiting algorithms implemented for real: token bucket and sliding
  window log in Redis with atomic Lua scripts
- Why `INCR` + `EXPIRE` has a race and Lua fixes it
- Granularity: per-IP, per-user, per-tenant, per-endpoint — layered
- Response headers: `X-RateLimit-*`, `Retry-After`, and returning 429 properly
- Where to enforce: gateway vs application, and doing both
- **Multi-tenancy models**: silo, pool, bridge — cost vs isolation
- Row-Level Security revisited as the enforcement mechanism
- Per-tenant connection pooling and the noisy-neighbour problem
- Quotas: hard vs soft limits, usage metering, billing periods
- Tenant-aware caching (never let a cache key leak across tenants)
- Bulkheads: isolating resource pools per tenant

### Resources
- [Redis documentation](https://redis.io/docs/latest/)
- [Redis rate limiting patterns](https://redis.io/learn/howtos/ratelimiting)
- [AWS SaaS multi-tenancy whitepaper](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/tenant-isolation.html)

### Checkpoint Project
**Capstone milestone 7.** Add Redis: cache document summaries and embeddings
with tenant-scoped keys, implement a sliding-window rate limiter in Lua (100
req/min per tenant, 10/min per user) returning correct 429s and headers, add
semantic caching for AI answers, and load-test to demonstrate one tenant hitting
its quota while another is completely unaffected.

---

# OPERATE

---

## Phase 15: AWS Core Services (3 weeks — Days 191–205)

Where the capstone will actually live. Concepts transfer to other clouds.

### What to Learn

**Week 1 — identity and network**
- **IAM first**: users, groups, roles, policies, trust relationships
- Least privilege; reading and writing a policy document
- Instance profiles and IRSA (how workloads get credentials without keys)
- AWS CLI, profiles, SSO
- **VPC**: subnets public vs private, route tables, IGW vs NAT gateway
- Security groups (stateful) vs NACLs (stateless)
- VPC endpoints; why they save money and improve security

**Week 2 — compute, storage, data**
- EC2: instance families, AMIs, user data
- ECS and Fargate — the simpler container runway before EKS
- Lambda: when serverless genuinely fits
- S3: buckets, policies, presigned URLs, lifecycle, storage classes
- EBS vs EFS
- **RDS PostgreSQL**: Multi-AZ, read replicas, parameter groups, backups,
  and enabling pgvector
- ElastiCache Redis
- MSK if you want managed Kafka

**Week 3 — edge, security, cost**
- ALB: listeners, target groups, path routing, health checks
- Route 53 and ACM certificates
- Secrets Manager and Parameter Store — and wiring them into Spring Boot
- KMS basics
- CloudWatch logs, metrics, alarms
- **Cost**: the free tier, what actually bills you, budgets and alerts,
  and destroying everything when you're done

### Resources
- [AWS Skill Builder (free tier)](https://skillbuilder.aws/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Solutions Architect Associate course](https://www.udemy.com/course/aws-certified-solutions-architect-associate-saa-c03/)

### Checkpoint Project
Deploy the capstone manually — console and CLI, no Terraform yet. VPC with
public and private subnets, ECS Fargate for the app, RDS PostgreSQL with
pgvector in a private subnet, ElastiCache Redis, ALB in public subnets, secrets
in Secrets Manager, image in ECR. Set a billing alarm first. Tear it all down
and confirm the bill returns to zero.

---

## Phase 16: Terraform & Infrastructure as Code (2 weeks — Days 206–215)

Clicking in a console doesn't scale, can't be reviewed, and can't be recreated.

### What to Learn

- What IaC is; declarative vs imperative; idempotency
- HCL: resources, data sources, variables, locals, outputs, expressions
- Providers and version pinning
- `init`, `plan`, `apply`, `destroy`, `fmt`, `validate`
- **State**: what it holds, why it's sensitive, remote state in S3 with
  DynamoDB locking, and never committing it
- `terraform import` for existing resources
- **Modules**: writing one, composing them, the registry
- Environments: workspaces vs directory-per-environment (and why the latter
  usually wins)
- `count` vs `for_each`; dynamic blocks
- Dependencies: implicit and `depends_on`
- Lifecycle rules: `prevent_destroy`, `create_before_destroy`
- Secrets in Terraform — how not to leak them into state
- CI for Terraform: `plan` on PR, `apply` on merge

### Resources
- [HashiCorp Terraform tutorials](https://developer.hashicorp.com/terraform/tutorials)
- [Terraform: Up & Running (Brikman)](https://www.terraformupandrunning.com/)

### Checkpoint Project
Recreate all of Phase 15 in Terraform: remote state in S3 with locking,
reusable modules for network / data / app, separate dev and prod variable
files, and zero diff on a second `plan` after `apply`. Destroy and recreate the
whole environment from scratch to prove it's reproducible.

---

## Phase 17: Kubernetes & Orchestration (4 weeks — Days 216–235)

The longest phase, because Kubernetes is genuinely large. Everything you
learned about Linux, networking, and containers pays off here.

### What to Learn

**Week 1 — core objects**
- Architecture: control plane, etcd, scheduler, kubelet, and the reconciliation
  loop as the central idea
- Pods, and why you rarely create one directly
- ReplicaSets, Deployments, rolling updates and rollbacks
- Services: ClusterIP, NodePort, LoadBalancer, and how DNS resolves them
- ConfigMaps and Secrets (and that Secrets are only base64, not encrypted)
- Namespaces and labels/selectors
- `kubectl`: `apply`, `get`, `describe`, `logs`, `exec`, `port-forward`

**Week 2 — networking, storage, scheduling**
- Cluster networking model, CNI, pod-to-pod communication
- **Ingress** controllers and Ingress vs Gateway API
- PersistentVolumes, PVCs, StorageClasses
- StatefulSets — and why you probably still use RDS instead
- Jobs and CronJobs
- Resource requests and limits; QoS classes; **why a JVM needs both set properly**
- OOMKilled, CPU throttling, and diagnosing both
- Probes: liveness, readiness, startup — and how a wrong liveness probe causes
  a restart loop under load

**Week 3 — production**
- HorizontalPodAutoscaler; custom metrics
- PodDisruptionBudgets, affinity, anti-affinity, taints and tolerations
- RBAC: roles, bindings, service accounts
- NetworkPolicies — default-deny as the goal
- Graceful shutdown: `terminationGracePeriodSeconds`, `preStop`, and Spring
  Boot's graceful shutdown (the `SIGTERM` thread from Phase 1 closes here)
- Security contexts, non-root, read-only root filesystem
- **Helm**: charts, values, templating, upgrade and rollback
- Kustomize as the alternative

**Week 4 — EKS and running it**
- EKS: managed control plane, node groups vs Fargate profiles
- `eksctl` and Terraform for EKS
- **IRSA** — pods assuming IAM roles properly
- AWS Load Balancer Controller, External DNS
- Cluster Autoscaler / Karpenter
- GPU node groups and device plugins (groundwork for Phase 21)
- Debugging: crash loops, pending pods, image pull failures, DNS problems

### Resources
- [Kubernetes documentation](https://kubernetes.io/docs/home/)
- [KillerCoda scenarios](https://killercoda.com/) — free browser labs
- [CKA course (Mumshad)](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- [EKS Workshop](https://www.eksworkshop.com/)

### Checkpoint Project
**Capstone milestone 8.** Deploy the whole platform to EKS: Deployments for the
app and consumers, Services, Ingress with TLS, ConfigMaps and Secrets, correct
resource requests/limits with JVM flags to match, all three probes, an HPA, a
PodDisruptionBudget, RBAC and NetworkPolicies, and a Helm chart wrapping it —
with the cluster itself provisioned by Terraform. Then break things on purpose:
delete pods, exhaust memory, fail a probe, and watch it recover.

---

## Phase 18: Istio Service Mesh & mTLS (2 weeks — Days 236–245)

With several services talking to each other, every pair needs encryption,
retries, timeouts, and observability. A mesh moves all of it out of your
application code.

### What to Learn

**Concepts**
- The problem: N² connections to secure and observe; per-team reinvention of
  retry logic
- **Sidecar pattern**: an Envoy injected into every pod, intercepting all
  traffic; the app is unaware
- Control plane (istiod) vs data plane
- Ambient mode as the emerging sidecar-less alternative
- The cost: latency, resource overhead, operational complexity — and when a
  mesh is *not* worth it

**Traffic management**
- `VirtualService`: routing, weights, retries, timeouts, fault injection
- `DestinationRule`: load balancing policy, connection pools, outlier detection
- `Gateway`: ingress into the mesh
- Canary releases by weight; traffic mirroring
- Circuit breaking, and testing it by injecting 500s

**Security — the part you care about**
- **mTLS**: both sides present certificates; identity, not network location,
  is the trust boundary
- SPIFFE identities and workload certificates
- Automatic certificate issuance and rotation by istiod
- `PeerAuthentication`: PERMISSIVE → STRICT migration
- `AuthorizationPolicy`: service-level RBAC, default-deny, and JWT-based rules
- How mesh mTLS relates to the TLS you learned in Phase 2 and the app auth from
  Phase 8 — three distinct layers, and knowing which one failed

**Observability**
- Metrics for every service pair with no code changes
- Distributed tracing and the headers your app must propagate
- Kiali's service graph

### Resources
- [Istio documentation](https://istio.io/latest/docs/)
- [Istio in Action (Manning)](https://www.manning.com/books/istio-in-action)

### Checkpoint Project
Install Istio on the EKS cluster. Enable mTLS in PERMISSIVE, verify with Kiali
that traffic is encrypted, then move to **STRICT** and prove an unmeshed pod is
refused. Add an `AuthorizationPolicy` so only the gateway may call the API and
only the API may call the ingestion service. Canary 10% of traffic to a v2. Add
a circuit breaker and trigger it with injected faults.

---

## Phase 19: Observability, CI/CD & GitOps (3 weeks — Days 246–260)

A system you can't see is a system you can't operate. This phase also covers
the delivery pipeline that makes shipping boring.

### What to Learn

**Week 1 — observability**
- The three pillars, and why traces tie the other two together
- **Metrics**: Prometheus pull model, metric types, cardinality as the thing
  that kills you
- Micrometer in Spring Boot; custom metrics; Actuator's Prometheus endpoint
- PromQL: rate, histogram quantiles, aggregation
- **RED** (rate, errors, duration) and **USE** methods
- Grafana dashboards that answer a question rather than showing everything
- **Logs**: structured JSON, correlation IDs, Loki, and what never to log
- **Tracing**: OpenTelemetry, spans, context propagation, Tempo/Jaeger
- Instrumenting the AI calls: token counts, cost, latency per model, cache hit rate
- **Alerting**: symptom-based not cause-based, SLIs and SLOs, error budgets,
  and alert fatigue as a real failure mode

**Week 2 — CI**
- GitHub Actions: workflows, triggers, jobs, matrix builds, caching
- A pipeline for the capstone: compile → unit tests → integration tests on
  Testcontainers → static analysis → build image → scan → push to ECR
- Secrets and OIDC federation to AWS (no long-lived keys)
- Branch protection, required checks
- Semantic versioning and image tagging
- Supply chain: SBOMs, dependency and image scanning

**Week 3 — CD and GitOps**
- Push-based vs pull-based deployment
- **GitOps**: Git as the single source of truth for cluster state
- **ArgoCD**: Applications, projects, sync policies, self-healing, drift detection
- Repository layout: app repo vs config repo, and why they're separate
- Image updater patterns
- Progressive delivery: blue/green and canary with Argo Rollouts
- Rollback as a first-class operation
- Runbooks, and writing one per alert
- Post-incident review culture

### Resources
- [Prometheus docs](https://prometheus.io/docs/introduction/overview/)
- [Google SRE Book (free)](https://sre.google/sre-book/table-of-contents/) — chapters 4, 6
- [ArgoCD docs](https://argo-cd.readthedocs.io/)
- [OpenTelemetry Java](https://opentelemetry.io/docs/languages/java/)

### Checkpoint Project
**Capstone milestone 9.** Full pipeline: GitHub Actions CI to ECR with OIDC
auth, ArgoCD syncing the cluster from a config repo with self-healing on,
Prometheus and Grafana with per-tenant RED dashboards plus an AI cost dashboard,
Loki for logs and Tempo for traces with correlation IDs flowing end to end,
alerts on error rate and consumer lag, and a runbook for each alert. Push a bad
commit and demonstrate rollback.

---

# AI SYSTEMS

---

## Phase 20: Agents, Tool Calling & MCP in Java (2 weeks — Days 261–270)

RAG answers questions about documents. Agents *do things*. This is where the AI
half stops being a text box and becomes part of the system.

### What to Learn

**Week 1 — tool calling**
- Function/tool calling: how a model requests an action and how you execute it
- Defining tools in Spring AI; schemas and descriptions (the description *is*
  the prompt — vague descriptions cause wrong calls)
- The execution loop: model → tool request → your code → result → model
- **Safety**: tools that write must be authorized as the *user*, not the app;
  confirmation steps for destructive actions
- Error handling — what to return when a tool fails
- Parallel tool calls
- Cost and latency of multi-turn loops; capping iterations

**Week 2 — agents and MCP**
- Agent patterns: ReAct, plan-and-execute, reflection
- When an agent is the wrong answer (a deterministic workflow usually beats one)
- Memory: conversation history, summarization, and what to persist
- Multi-agent orchestration — and its failure modes
- **Model Context Protocol (MCP)**: what it standardizes and why it matters
- Spring AI's MCP client and server support
- Exposing your own domain operations as an MCP server
- Guardrails: input/output filtering, allow-lists, sandboxing, audit logging of
  every tool invocation
- Evaluating agents: task success rate, not token similarity
- Observability for agent runs — tracing a multi-step loop

### Resources
- [Model Context Protocol docs](https://modelcontextprotocol.io/)
- [Spring AI — tool calling](https://docs.spring.io/spring-ai/reference/api/tools.html)
- [Anthropic — building effective agents](https://www.anthropic.com/engineering/building-effective-agents)

### Checkpoint Project
**Capstone milestone 10.** Give the platform tools: search documents, fetch a
tenant's usage stats, create a summary report. Every tool executes under the
calling user's authorization and is audit-logged. Then expose those same
operations as an **MCP server** so an external client can use them. Add an eval
suite of 20 tasks scoring success rate, and trace a full agent run in Tempo.

---

## Phase 21: Self-Hosted Inference & GPU Serving (2 weeks — Days 271–280)

The last piece of "AI systems engineer": running the model yourself. This is
the one phase with Python in it — deliberately minimal, behind your Java
gateway.

### What to Learn

**Week 1 — serving**
- Why self-host: cost at volume, data residency, latency, no per-token billing
- Why *not*: GPU cost, ops burden, model quality gap
- How LLM inference works: prefill vs decode, KV cache, why the second token is
  cheaper than the first
- Batching: static vs dynamic vs **continuous batching**
- **vLLM**: PagedAttention, the OpenAI-compatible API, deployment in Docker
- Pointing Spring AI at your own endpoint by changing a base URL
- Quantization: what you trade for memory (GPTQ, AWQ, FP8)
- Model selection and sizing; how much VRAM a model actually needs
- Alternatives: TGI, Ollama for local dev, Triton for non-LLM models
- Embedding models served locally — often the highest-value thing to self-host

**Week 2 — running it on Kubernetes**
- GPU nodes on EKS, device plugins, scheduling with resource limits
- Node selectors and tolerations for GPU pools
- Model weights: baked into the image vs pulled from S3 at startup
- Cold start and warm-up; keeping a pod ready
- Autoscaling GPU workloads with KEDA on queue depth
- Spot GPU instances and handling interruption
- Monitoring: GPU utilization, memory, queue depth, tokens/sec
- **Load testing** inference: throughput vs latency vs concurrency curves
- Cost per million tokens, self-hosted vs API — the calculation that decides it
- Fallback: routing to a hosted API when your own service is saturated

### Resources
- [vLLM documentation](https://docs.vllm.ai/)
- [NVIDIA Triton](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/)
- [KEDA](https://keda.sh/docs/latest/)

### Checkpoint Project
Serve an open model with vLLM in Docker, point your Spring AI configuration at
it with no application code change, and verify RAG still works. Deploy it to a
GPU node group on EKS with KEDA autoscaling on queue depth. Load-test to find
the throughput/latency knee. Produce a cost comparison against a hosted API at
three volume levels, and implement automatic fallback to the hosted API.

---

## Capstone: Multi-Tenant AI Knowledge Platform (3 weeks — Days 281–295)

You've built most of this incrementally. These three weeks are for integration,
polish, hardening, and being able to present it.

**Week 1 — integrate and harden.** Bring every milestone together in one
deployable system. Close the gaps you deferred. Run a security pass against the
OWASP Top 10. Verify tenant isolation at every layer — API, service, database
RLS, cache keys, vector filters, and mesh policy — and write a test for each.

**Week 2 — prove it.** Gatling load test: 100 concurrent users across 3
tenants. Report P50/P95/P99 and RPS. Demonstrate one tenant exhausting its quota
with zero impact on the others. Chaos: kill pods, sever the database, saturate
the model, revoke a certificate — and show graceful degradation. Confirm the
dashboards actually told you what was wrong.

**Week 3 — present it.** Architecture document with diagrams and explicit
trade-offs. A README that lets someone else run it. A recorded walkthrough. And
a written answer to the question you'll be asked in every interview: *why did
you build it this way, and what would you change at 100× the scale?*

**Final architecture:**

| Layer | Choice |
|-------|--------|
| API | Spring Boot 3, Java 21, virtual threads |
| Security | Spring Security 6, Keycloak OIDC, JWT with `tenant_id`, hashed API keys |
| Data | PostgreSQL + pgvector, Row-Level Security, Flyway |
| AI | Spring AI — RAG, tool calling, MCP server; vLLM self-hosted with hosted fallback |
| Messaging | Kafka — outbox, async ingestion, audit log, DLQ |
| Cache | Redis — semantic cache, per-tenant sliding-window rate limits |
| Edge | Spring Cloud Gateway / Kong, Nginx or ALB, TLS |
| Runtime | EKS, Helm, HPA, KEDA for GPU |
| Mesh | Istio, strict mTLS, authorization policies, canary |
| IaC | Terraform, remote state, modules per environment |
| Delivery | GitHub Actions → ECR → ArgoCD GitOps |
| Observability | Prometheus, Grafana, Loki, Tempo, per-tenant RED + AI cost |

---

## Quick Reference: Tools by Category

| Category | Tool | Use |
|----------|------|-----|
| Language | Java 21 | Everything |
| Framework | Spring Boot 3 | Services |
| Security | Spring Security 6 + Keycloak | AuthN/AuthZ, OIDC |
| Database | PostgreSQL + pgvector | Relational data, embeddings, RLS |
| Migrations | Flyway | Schema versioning |
| Cache | Redis | Caching, rate limits, locks |
| Messaging | Kafka | Async pipelines, audit, outbox |
| AI framework | Spring AI (LangChain4j alt.) | LLM integration, RAG, tools, MCP |
| Inference | vLLM | Self-hosted serving |
| Build | Maven | Build, dependencies |
| Testing | JUnit 5, AssertJ, Testcontainers, Gatling | Correctness and load |
| Containers | Docker, Compose | Local everything |
| Orchestration | Kubernetes (EKS), Helm | Running at scale |
| Mesh | Istio | mTLS, traffic policy, telemetry |
| Gateway | Spring Cloud Gateway / Kong / Nginx | Edge routing, rate limits, TLS |
| Cloud | AWS | Infrastructure |
| IaC | Terraform | Provisioning |
| CI/CD | GitHub Actions + ArgoCD | Build and GitOps delivery |
| Observability | Prometheus, Grafana, Loki, Tempo, OpenTelemetry | Metrics, logs, traces |

---

## Learning Principles

1. **Build, don't just read.** Every phase has a checkpoint. Do it.
2. **Break things on purpose.** Race a counter. Drop an index. Kill a pod.
   Revoke a cert. Watching it fail is the lesson.
3. **One system.** Each checkpoint extends the same codebase. By Phase 21 you
   have a portfolio piece, not twenty tutorials.
4. **Explain it out loud.** If you can't articulate the trade-off, you don't
   own the knowledge yet.
5. **Read the error.** Java stack traces and Kubernetes events are verbose
   because they're telling you the answer.
6. **Destroy cloud resources.** Set a billing alarm before your first `apply`.
7. **Write down what broke.** The Q&A section of each lesson is the artifact
   you'll actually reread.
