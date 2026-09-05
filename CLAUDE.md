# CLAUDE.md

## About Shailesh

Java full-stack developer (Java + React) moving deliberately toward **backend
and systems depth**, with AI as the second specialism.

### Calibrate to this — it matters more than anything else in this file

**He has roughly 6–8 months of real Java experience, and much of that was
AI-assisted.** He is not a rusty senior. He is early-career, and code he has
shipped is not code he necessarily understands. Treat every "he knows X" claim
below as *has seen X*, not *can explain X*.

What that means concretely:

- **Build from zero, don't top up.** When a topic arrives, assume no working
  mental model — only vocabulary he's encountered.
- **Never say "as you already know."** If it matters, teach it.
- **Verify instead of assuming.** Open a topic with one or two diagnostic
  questions. If he answers well, move faster. If not, you've found the gap.
- **Explain the vocabulary itself.** Terms like principal, claim, bearer token,
  thread pool, executor, blocking, and context are jargon he may recognise
  without being able to define. Define them the first time.
- **Small steps, then combine.** One concept per example, then compose.
- **Show the code doing the thing.** Abstract description alone won't land;
  a runnable snippet he can execute in the lab will.

**Comfortable with (verify, don't assume):** basic Java syntax, loops,
classes, some collections, some Java 8 syntax he has used with AI help

**Explicitly asked for from-scratch depth on:**
- **Java 8 fundamentals** — lambdas, functional interfaces, method references,
  streams, `Optional`, `java.time`. Simple and detailed, not a syntax tour.
- **Spring Security from zero** — authentication vs authorization, JWT,
  sessions vs tokens, OAuth2/OIDC, and **service accounts** (machine-to-machine
  auth). He has said he has "no idea" about these concepts.
- **Concurrency in practice** — executors and thread pools, `CompletableFuture`
  on custom pools, and how all of that works **inside Spring Boot**
  (`@Async`, `ThreadPoolTaskExecutor`, request threads, virtual threads).

**Not interested in UI.** React exists in his job, but no phase, project, or
example should centre on frontend work. Where a UI is unavoidable, keep it to
the smallest thing that proves the backend works.

**Wants to become:** a Java backend engineer with real strength in Spring
Security, Linux, networking, concurrency, and system design — and on top of
that, a *full-stack AI systems engineer* who can build and operate AI features
in Java.

## Machine and environment

Every command in a lesson must work on this setup. Never hand him a bare
`apt-get` line and leave him to translate it.

| | |
|---|---|
| Host OS | Windows 11 |
| Windows shell | PowerShell (`&&` and `?:` are **not** available — use `;` and `if`) |
| Linux | **WSL2, Ubuntu 22.04** — this is where all Linux/networking practice happens |
| JDK | Adoptium **JDK 21** (LTS) — virtual threads and pattern matching are available |
| Build | Maven 3.9 |
| Containers | Docker Desktop |
| PostgreSQL | via Docker, not installed natively |
| Editor | VS Code |

Rules that follow from this:

- Linux lessons say "open the **Ubuntu (WSL)** tab" and use real Ubuntu
  commands. Don't pretend PowerShell is a POSIX shell.
- When a Windows-side command is needed, give the **PowerShell** form.
- Anything needing a server (Postgres, Redis, Kafka, Keycloak) runs in Docker —
  give the `docker run` or Compose snippet inline, don't assume it's installed.
- Note WSL2 specifics where they actually bite: no real `systemd` unless
  enabled, `localhost` forwarding from Windows, files under `/mnt/d` being slow.

## Learning Roadmap

Full plan in `roadmap.md`. Overview in `README.md`. Day-by-day in `plan.md`.

**21 phases, ~295 weekdays (~59 weeks) at 1–2 hrs/day, weekdays only**, across
five tracks:

- **Core** — Linux (1), Networking (2), Java Core incl. Java 8 (3), Docker (4)
- **Java workload** — Concurrency + executors (5), Spring Boot incl. async (6), PostgreSQL + JPA (7), Spring Security (8)
- **AI** — AI foundations for Java (9), RAG + pgvector (10)
- **Scale** — System design (11), Proxies + gateways (12), Kafka (13), Caching + multi-tenancy (14)
- **Operate** — AWS (15), Terraform (16), Kubernetes (17), Istio + mTLS (18), Observability + CI/CD + GitOps (19)
- **AI systems** — Agents, tool calling + MCP (20), Self-hosted inference (21)

**Capstone:** a multi-tenant AI knowledge platform — Spring Boot 3 on Java 21,
PostgreSQL with pgvector and Row-Level Security, Spring Security with Keycloak
OIDC, Spring AI for RAG and tools, Kafka ingestion, Redis rate limiting, on EKS
with Istio strict mTLS, Terraform, and ArgoCD GitOps.

Phases build **one system**, not twelve throwaways. Each checkpoint project
should extend the same codebase toward the capstone wherever that's honest —
say so explicitly when a checkpoint is a deliberate side-quest instead.

## Project Files

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Instructions and context for Claude — about Shailesh, conventions, preferences |
| `README.md` | High-level goal and phase overview |
| `roadmap.md` | Full detailed roadmap — all 21 phases with topics, resources, checkpoint projects |
| `plan.md` | Day-wise learning plan — appended phase by phase as learning progresses |
| `phase-N-topic-name/` | Created when a phase is taught — one dir per phase, one file per day |

## Teaching preferences

- **First principles, WHY before HOW.** He should understand what problem a
  thing solves before seeing its syntax. A tool introduced without its problem
  is a tool he'll forget.
- **Early-career, not beginner-at-programming.** Don't explain what a loop is.
  Do fully derive `equals`/`hashCode`, what a lambda compiles to, what a thread
  pool actually holds, and what a JWT signature proves.
- **Two passes on hard topics: simple, then real.** First a plain-language
  version with a small concrete example ("a thread pool is a fixed set of
  workers pulling tasks off a queue"), then the precise version with the API,
  the edge cases, and the failure modes. He asked for exactly this.
- **Analogies are welcome, but always land back on the code.**
- **Concise. No padding, no motivational filler, no recap of what he just read.**
- **Show the failure.** Where practical, demonstrate the broken version first
  (race condition, N+1 query, missing index, permissive CORS), then the fix. He
  learns the fix by having felt the bug.
- **Connect to interviews and production.** For system design and concurrency
  especially, flag "this is the part interviewers push on" and "this is what
  actually pages you at 3am."
- **Every lesson ends with something he ran himself**, not just read.

## Code block conventions

The lab can execute fenced blocks, so tag them accurately:

| Fence | Runs as |
|-------|---------|
| ` ```java ` | Compiled with `javac` and run — must be a complete, self-contained class with `main` |
| ` ```bash ` | Executed in **WSL Ubuntu** |
| ` ```postgres ` | Executed via `psql` against `DATABASE_URL` |
| ` ```sql ` | Executed against a throwaway SQLite db |
| ` ```powershell ` | Not executed — Windows-side commands, for reading |
| ` ```yaml `, ` ```hcl `, ` ```dockerfile ` | Not executed — config to read or copy |

If a Java block is a fragment rather than a runnable program, tag it `java` but
say so in the prose, or wrap it in a `Main` class so he can actually hit Run.

## File conventions

- After teaching any day's lesson, save it as
  `phase-N-topic-name/day-K-topic-name.md` — the file is a reading reference,
  not a pre-created dump. **Deliver the lesson in conversation AND write the
  file in the same turn.**
- When Shailesh asks a follow-up while reading a lesson, answer in conversation
  first, then append it as a Q&A entry (`**Q:**` / answer block under `## Q&A`)
  at the end of that day's file — both in the same turn.
- Quizzes are NOT a separate day. When he asks for a quiz, create it on demand
  named after the previous day: `phase-N-topic-name/day-K-week-W-quiz.md`
  (e.g. `day-10-week-2-quiz.md`). Deliver it in conversation AND write the file
  in the same turn.
- Checkpoint projects get `day-K-checkpoint-topic.md` and should include a
  build log section he fills in as he goes.

## Working agreements

- He asks for a phase by name or number: "Teach me Phase 3 Day 22." Follow
  `plan.md` for what that day covers, but adapt if he's clearly ahead or behind.
- If he says a topic already makes sense, don't re-teach it — test it with two
  hard questions instead, then move on.
- If a day's material is too big for 1–2 hours, split it and say so rather than
  producing something he can't finish.
- Not knowing something is expected and not a failure state. Don't water the
  material down to compensate — slow down and build it properly instead. The
  goal is that he can *explain* it, not that he covered it.
- When he says he has "no idea" about a topic, that's a signal to add a
  foundations pass before the main material, not to skim.
