# From Java Developer to Full-Stack AI Systems Engineer

## Goal

Become a Java backend engineer with genuine depth — Spring Security,
concurrency, Linux, networking, system design — and then build and operate
**production AI systems in Java** on top of that foundation.

Not "a Java dev who also knows some Kubernetes." An engineer who can take a
feature from a PostgreSQL schema all the way to a multi-tenant service running
under mTLS on Kubernetes, with an LLM in the request path and dashboards
showing what it costs.

## Starting point

- **~6–8 months of real Java experience**, much of it AI-assisted — so the
  roadmap builds from the ground up rather than assuming a foundation
- **Familiar with:** basic syntax, classes, some collections, some Java 8 syntax
- **Starting from zero on:** Spring Security (JWT, authorization, service
  accounts), executors and `CompletableFuture`, and the functional side of Java 8
- **Deliberately out of scope:** frontend work

Nothing here assumes you already know a term. Where a phase says "deep dive,"
it starts at the beginning of that topic, not the middle.

## What you'll be able to do

- Write Java that holds up under concurrency, not just Java that compiles
- Design and secure REST APIs with Spring Boot and Spring Security — OAuth2,
  OIDC, JWT, method-level authorization, multi-tenancy
- Model, index, and query PostgreSQL well enough to explain a query plan
- Build RAG and agentic features in Java with Spring AI and pgvector
- Reason about a system's design out loud — the thing senior interviews are
  actually testing
- Containerize, orchestrate, and observe services on Kubernetes with Istio mTLS
- Provision it all with Terraform and ship it with GitOps
- Put a self-hosted model behind your own Java gateway and know what it costs

---

## Track framework

| Track | What it covers |
|-------|---------------|
| **Core** | Linux, networking, the Java language, containers — the ground everything stands on |
| **Java workload** | The services you actually write: concurrency, Spring Boot, PostgreSQL, security |
| **AI** | LLMs as a component you build with, in Java |
| **Scale** | Making it survive load: system design, gateways, messaging, caching, multi-tenancy |
| **Operate** | Running it for real: AWS, Terraform, Kubernetes, Istio, observability, GitOps |
| **AI systems** | The specialism: agents, tool calling, MCP, and self-hosted inference |

## Phases

| # | Track | Topic | Weeks | Days |
|---|-------|-------|-------|------|
| 1 | Core | Linux & the Shell (WSL2) | 2 | 1–10 |
| 2 | Core | Networking Fundamentals | 2 | 11–20 |
| 3 | Core | Java Core Rebuild *(incl. a full week on Java 8)* | 4 | 21–40 |
| 4 | Core | Docker & Containers | 2 | 41–50 |
| 5 | Java | Concurrency, Executors & CompletableFuture | 4 | 51–70 |
| 6 | Java | Spring Boot Core, REST & Async | 4 | 71–90 |
| 7 | Java | PostgreSQL & Spring Data JPA | 3 | 91–105 |
| 8 | Java | Spring Security Deep Dive | 4 | 106–125 |
| 9 | AI | AI Foundations for Java Engineers | 2 | 126–135 |
| 10 | AI | RAG & Vector Search with pgvector | 2 | 136–145 |
| 11 | Scale | System Design Fundamentals | 3 | 146–160 |
| 12 | Scale | Proxies, Load Balancers & API Gateways | 2 | 161–170 |
| 13 | Scale | Async Messaging with Kafka | 2 | 171–180 |
| 14 | Scale | Caching, Rate Limiting & Multi-Tenancy | 2 | 181–190 |
| 15 | Operate | AWS Core Services | 3 | 191–205 |
| 16 | Operate | Terraform & Infrastructure as Code | 2 | 206–215 |
| 17 | Operate | Kubernetes & Orchestration | 4 | 216–235 |
| 18 | Operate | Istio Service Mesh & mTLS | 2 | 236–245 |
| 19 | Operate | Observability, CI/CD & GitOps | 3 | 246–260 |
| 20 | AI systems | Agents, Tool Calling & MCP in Java | 2 | 261–270 |
| 21 | AI systems | Self-Hosted Inference & GPU Serving | 2 | 271–280 |
| — | — | **Capstone** | 3 | 281–295 |

**Total: 295 weekdays ≈ 59 weeks** at 1–2 hrs/day.

### Where the depth is

Four phases are deliberately longer than a typical roadmap would make them,
because they're the ones that decide whether you're a Java engineer or someone
who can get Java to compile:

| Phase | Why it's 4 weeks |
|-------|------------------|
| **3 — Java Core** | An entire week on **Java 8**: lambdas, functional interfaces, method references, `java.time`. Streams make no sense until this is solid. |
| **5 — Concurrency** | A week on **executors and thread pools**, and a week on **`CompletableFuture`**. These are what you'll actually use. |
| **6 — Spring Boot** | A week on **threading inside Spring** — `@Async`, `ThreadPoolTaskExecutor`, async controllers, context propagation. |
| **8 — Spring Security** | A week of **pure concepts before any Spring** — authentication vs authorization, sessions vs tokens, JWT, and **service accounts**. |

### Milestones along the way

You don't have to reach Phase 21 to have gained something hireable:

| After | You are |
|-------|---------|
| **Phase 5** (~Day 70, month 3.5) | Genuinely solid on core Java and concurrency — past the level most 2-year devs reach |
| **Phase 8** (~Day 125, month 6) | A real Java backend engineer — Spring Boot, PostgreSQL, Spring Security, async |
| **Phase 10** (~Day 145, month 7) | The above, plus able to ship RAG features in Java — a genuinely scarce combination |
| **Phase 14** (~Day 190, month 9) | Able to design and defend a scalable multi-tenant system |
| **Phase 19** (~Day 260, month 12) | Able to run it in production yourself |
| **Phase 21 + capstone** | Full-stack AI systems engineer, with a portfolio piece that proves it |

---

## Capstone: Multi-Tenant AI Knowledge Platform

Everything converges on one system, built incrementally from Phase 6 onward.

**Application** — a multi-tenant document Q&A platform:

- **Spring Boot 3 on Java 21** — virtual threads, records, modular services
- **PostgreSQL** — pgvector for embeddings, Row-Level Security for tenant
  isolation, Flyway migrations
- **Spring Security 6** — Keycloak OIDC, JWT carrying `tenant_id`, method
  security, hashed API keys
- **Spring AI** — RAG over tenant documents, tool calling, an MCP server
  exposing internal tools
- **Kafka** — async ingestion (upload → chunk → embed → index) and an audit
  log, using the outbox pattern
- **Redis** — response caching and per-tenant sliding-window rate limits

**Infrastructure** — all of it as code:

- EKS with CPU and GPU node groups, Helm charts
- Istio with **strict mTLS**, canary routing, circuit breaking
- RDS PostgreSQL, ElastiCache Redis, S3, ECR
- Terraform with remote state
- **ArgoCD GitOps** — Git is the source of truth for cluster state

**Operations**:

- GitHub Actions CI: test → build → push
- Prometheus, Grafana, Loki, Tempo — RED metrics per tenant, plus token and
  cost dashboards for the AI calls
- Alerts with runbooks
- Gatling load test: 100 concurrent users across 3 tenants, P50/P95/P99
- Demonstrate one tenant hitting its quota while the others are unaffected

---

## How to use this

Say **"Teach me Phase 1 Day 1"** and the lesson gets written into
`phase-1-linux-and-shell/day-1-*.md` as it's taught. Ask for a quiz at any week
boundary. See [`plan.md`](./plan.md) for the day-by-day breakdown and
[`roadmap.md`](./roadmap.md) for the full detail on every phase.

## Principles

1. **Build, don't just read.** Every phase has a checkpoint. Do it.
2. **Break it on purpose.** Race the counter. Drop the index. Kill the pod.
   Revoke the cert. Watch what happens.
3. **One system, not twenty demos.** Each phase extends the same codebase.
4. **Explain it out loud.** If you can't say why a design works, you don't have it.
5. **Destroy cloud resources when done.** AWS bills you while you sleep.
6. **Write down what broke.** The Q&A section of each lesson is the real artifact.
