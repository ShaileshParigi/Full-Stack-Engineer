# Day 14 — DNS

_Why this matters:_ Every connection your code makes starts with a name —
`orders-db.internal`, `api.stripe.com`, `orders` inside Kubernetes — and before a
single TCP packet leaves, something has to turn that name into an IP address.
When that step is slow, everything is slow. When it's wrong, you talk to the old
server. When it's down, **everything looks down**: on 20 October 2025 a large
part of AWS `us-east-1` failed because DNS for the DynamoDB endpoint came back
empty — the servers were fine, but nobody could find them. And DNS's caching
layers are the cause of half of all "but it works on my machine": two machines
asking the same question get different answers because they asked different
caches at different times. Today you'll find out that your own machine is
holding answers for **29 times longer** than their owner allows.

---

## The one-paragraph version

DNS is the internet's phone book: you ask for a name, you get back records —
usually an IP address. No single server knows every name. Instead there's a
tree: **root servers** know who runs each top-level domain (`.org`, `.com`);
the **`.org` servers** know who runs `httpbin.org`; and **httpbin.org's own
servers** (its *authoritative* servers) hold the actual answer. Your machine
doesn't walk that tree itself — it asks a **recursive resolver** (your ISP's,
or `8.8.8.8`) to do the walking and hand back the answer. Every answer carries
a **TTL** — "you may remember this for N seconds" — so resolvers, your OS and
your JVM all **cache** answers instead of asking again. That makes DNS fast,
and it's also why a change made to DNS reaches different people at different
times. Before any of that, your machine checks a local file, `/etc/hosts`, which
overrides DNS completely. When debugging, the tool is `dig`: it lets you ask
*any* server *any* question and shows you exactly what came back.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **DNS** | The system that turns names into addresses | Domain Name System: a distributed, hierarchical, cached database |
| **domain name** | A name like `api.github.com` | A sequence of labels, read right to left from the root |
| **FQDN** | A name written in full, all the way to the root | Fully Qualified Domain Name; formally ends with a dot: `httpbin.org.` |
| **root (`.`)** | The top of the tree | 13 named root server identities (`a`–`m.root-servers.net`), each run on hundreds of machines |
| **TLD** | The last part of a name: `.org`, `.com`, `.in` | Top-Level Domain, run by a registry |
| **zone** | The part of the tree one organisation controls | A delegated subtree with its own authoritative servers |
| **authoritative server** | The server that *owns* the answer for a domain | Answers from its own zone data; sets the `aa` flag |
| **recursive resolver** | The server that goes and finds answers for you | Walks root → TLD → authoritative, caches results; sets `ra` |
| **stub resolver** | The small DNS client inside your OS | Sends one question to a recursive resolver and waits |
| **delegation** | "I don't know, but ask *them*" | An NS record pointing to the next zone's servers |
| **record (RR)** | One fact in DNS | Resource record: name, TTL, class, type, data |
| **A / AAAA** | Name → IPv4 / IPv6 address | 32-bit / 128-bit address records |
| **CNAME** | "This name is an alias for that name" | Canonical name; the resolver restarts the lookup at the target |
| **MX** | "Mail for this domain goes to these servers" | Mail exchanger, with a priority (lower = preferred) |
| **TXT** | Free text attached to a name | Used for domain verification, SPF, DKIM |
| **NS** | "These servers are authoritative for this zone" | Name server records — the delegations |
| **SRV** | "This service runs on this host and port" | Priority, weight, port, target |
| **PTR** | IP → name (reverse lookup) | Stored under `in-addr.arpa` |
| **SOA** | The zone's admin record | Start of authority; includes the negative-caching TTL |
| **TTL** | How long an answer may be cached | Time to live, in seconds, set by the zone owner |
| **cache** | Remembered answers | Held by resolvers, the OS, and the JVM |
| **NXDOMAIN** | "That name does not exist" | Response code 3 — a definite, cacheable "no" |
| **SERVFAIL** | "I couldn't get an answer" | Response code 2 — the resolver failed, not the name |
| **negative caching** | Remembering "doesn't exist" | Cached NXDOMAIN, for a TTL derived from the SOA |
| **`/etc/hosts`** | A local list of names that overrides DNS | Consulted first, by `nsswitch.conf` order |
| **`/etc/resolv.conf`** | Which DNS server your Linux machine asks | `nameserver`, `search` and `options` lines |
| **`nsswitch.conf`** | The order to try name sources in | `hosts: files dns` = `/etc/hosts`, then DNS |
| **search domain** | A suffix tried automatically on short names | `search default.svc.cluster.local` turns `orders` into a full name |
| **split-horizon DNS** | The same name gives different answers to different askers | E.g. a private IP inside the VPC, a public IP outside |
| **service discovery** | Finding where a service currently runs, by name | In Kubernetes and Docker Compose, done with DNS |

---

## Before you read: three questions

1. Someone moves `httpbin.org` to a new IP. Its owner set a TTL of 60 seconds.
   How long could *your* laptop keep connecting to the old IP?
2. `dig api.internal.example.com` returns the right IP, but your Java service
   still fails with `UnknownHostException` for the same name. Give two different
   reasons that could happen.
3. Inside a Kubernetes pod, your app calls `http://orders:8080`. Nowhere in the
   world is there a DNS record for a name called just `orders`. How does it
   resolve?

---

## Part 1 — What resolution actually does: the chain on your machine

**In plain words:** when your program asks for `httpbin.org`, the question is
passed along a chain of helpers, and each one either answers from memory or
passes it on. On your machine the chain is unusually long because WSL is a
machine inside a machine: your Linux side asks Windows, Windows asks your ISP,
and only your ISP actually goes out and asks the internet. Every link can cache,
and every link can be wrong.

Here's the full chain for a `curl` in WSL, built from what your machine
actually reported:

```
curl httpbin.org   (in WSL)
  │  glibc getaddrinfo() — reads /etc/nsswitch.conf:  "hosts: files dns"
  ├─▶ 1. /etc/hosts                          no entry for httpbin.org → continue
  └─▶ 2. /etc/resolv.conf → nameserver 172.24.160.1
         = Windows (WSL's DNS proxy)
           │  Windows DNS client + its cache   (Get-DnsClientCache)
           └─▶ 3. Your ISP's recursive resolvers 103.57.86.10 / 103.57.86.36
                  (configured on Windows' Wi-Fi adapter; your home router 192.168.1.1
                   forwards to the same place)
                    │ if not cached, it walks the tree:
                    ├─▶ a root server        "I don't know httpbin.org; .org is run by these"
                    ├─▶ a .org server        "httpbin.org is run by ns-1555.awsdns-02.co.uk & co."
                    └─▶ AWS Route 53         "httpbin.org = 8 A records, TTL 60"   ← authoritative
```

Three kinds of server, three jobs:

| Role | Who, for you | Knows | Caches? |
|------|--------------|-------|---------|
| **Stub resolver** | glibc in WSL; the Windows DNS client | Only who to ask | Windows: yes. glibc: no |
| **Recursive resolver** | Your ISP's `103.57.86.10`; or `8.8.8.8`, `1.1.1.1` | Nothing on its own — it finds out | **Yes**, per TTL |
| **Authoritative** | Root, `.org` registry, AWS Route 53 for httpbin | Its own zone, definitively | Doesn't need to |

### What's in the files

```bash
echo "== /etc/nsswitch.conf (hosts line)"; grep '^hosts' /etc/nsswitch.conf
echo; echo "== /etc/resolv.conf"; grep -v '^#' /etc/resolv.conf; ls -l /etc/resolv.conf
echo; echo "== /etc/hosts"; grep -v '^#' /etc/hosts | sed '/^\s*$/d'
```

- `hosts: files dns` — try `/etc/hosts` first, then DNS. That's why `/etc/hosts`
  wins (Part 6).
- `nameserver 172.24.160.1` — your Windows host, the same IP that's your default
  gateway (Day 11). WSL generates this file at boot (it's a symlink to
  `/mnt/wsl/resolv.conf`; the header comment tells you how to stop it).
- `/etc/hosts` is **also** generated by WSL — copied from Windows'
  `C:\Windows\System32\drivers\etc\hosts`. That's why it contains Docker
  Desktop's `host.docker.internal` lines. More on those in Gotchas.

Notice that `systemd-resolved` is running in your WSL, but `resolv.conf` doesn't
point at it — `resolvectl status` reports `resolv.conf mode: foreign`. WSL's own
setup wins.

> **Say this in an interview:** "An application calls the OS stub resolver,
> which checks `/etc/hosts` per `nsswitch.conf` and then asks the recursive
> resolver in `resolv.conf`. The recursive resolver answers from cache or walks
> the hierarchy — root, TLD, then the domain's authoritative servers — and caches
> each answer for its TTL. Authoritative servers own the data; recursive
> resolvers only find and cache it."

---

## Part 2 — Walking the tree: root → TLD → authoritative

**In plain words:** imagine asking a librarian for a book. The front desk
doesn't have it but says "science is on floor 3". Floor 3 says "physics is
shelf 12". Shelf 12 has the book. Each step doesn't know the answer, only who to
ask next. DNS is that: each level *delegates* to the level below with an NS
record, and the recursive resolver follows the pointers.

`dig +trace` does the walk itself, from the root, and shows each step. On your
machine (`-4` because you have no IPv6 route — Day 12):

```bash
# Resolve httpbin.org from the root down, printing each delegation.
timeout 15 dig -4 +trace +nodnssec httpbin.org | grep -vE '^;; global|^; <<>>|^$'
```

Your output, narrated:

```
.        1800  IN NS  a.root-servers.net.  ... (13 of them)
;; Received 239 bytes from 172.24.160.1#53(172.24.160.1) in 1269 ms
```
**Step 0 — "Who are the roots?"** `dig` asks your configured resolver for the
list of root servers. Thirteen names, `a` to `m`. Each is really hundreds of
machines worldwide sharing one IP (*anycast*), so "13 root servers" is 13
identities, not 13 computers.

```
org.     172800  IN NS  a0.org.afilias-nst.info.  ... (6 of them)
;; Received 484 bytes from 170.247.170.2#53(b.root-servers.net) in 179 ms
```
**Step 1 — Ask a root.** `b.root-servers.net` doesn't know `httpbin.org`. It
answers with a **delegation**: "`.org` is run by these six servers." TTL 172800 =
2 days — TLD delegations almost never change.

```
httpbin.org.  3600  IN NS  ns-1555.awsdns-02.co.uk.  ... (4 of them)
;; Received 221 bytes from 199.19.56.1#53(a0.org.afilias-nst.info) in 9 ms
```
**Step 2 — Ask `.org`.** The `.org` registry's server doesn't know the address
either. Another delegation: "`httpbin.org` is run by these four AWS Route 53
servers." The `awsdns-NN` names in `.com`, `.net`, `.org` and `.co.uk` are a
tell: **the domain's DNS is hosted on Route 53** — spread over four TLDs so that
a problem in one TLD can't take it offline.

```
httpbin.org.  60  IN A  18.233.182.23   ... (8 of them)
;; Received 305 bytes from 205.251.198.19#53(ns-1555.awsdns-02.co.uk) in 19 ms
```
**Step 3 — Ask the authoritative server.** Finally an answer: eight A records,
**TTL 60**. That 60 is the real, owner-set TTL. Remember it for Part 4.

A normal lookup does **not** do this every time. A recursive resolver does the
walk once, caches the root and `.org` delegations for days, and after that only
repeats step 3 — and only when its cached answer expires.

> **Say this in an interview:** "Resolution follows delegations: the root
> servers delegate `.org` to the registry's name servers via NS records, those
> delegate `httpbin.org` to its authoritative servers — here Route 53 — and the
> authoritative server returns the records with a TTL. The recursive resolver
> caches every level, so in steady state it only re-queries the authoritative
> server when the record's TTL expires."

---

## Part 3 — Record types: A, AAAA, CNAME, MX, TXT, NS, SRV

**In plain words:** a DNS name isn't just "an address". It's a label you can
attach different kinds of fact to — its IPv4 address, its IPv6 address, "this is
really another name", "send its email here", "here's a verification code". Each
kind of fact is a *record type*. You ask for one type at a time.

All of these are real answers from your machine:

| Type | Example (yours) | Read it as | Where you'll use it |
|------|-----------------|------------|---------------------|
| **A** | `httpbin.org. 60 IN A 18.233.182.23` (×8) | IPv4 address. **Several A records** = simple load spreading ("round-robin DNS"); clients pick one | Everything |
| **AAAA** | `google.com. IN AAAA 2404:6800:4000:1017::8a` | IPv6 address. You get them, but can't use them — no IPv6 route | Dual-stack services |
| **CNAME** | `www.github.com. IN CNAME github.com.` | "Go look up *that* name instead." The resolver follows it | Pointing your name at a load balancer: `api.mycompany.com CNAME my-alb-123.elb.amazonaws.com` |
| **MX** | `gmail.com. IN MX 5 gmail-smtp-in.l.google.com.` | Mail server, priority 5 (lowest = try first; 10, 20… are backups) | Email |
| **TXT** | `google.com. IN TXT "google-site-verification=…"` | Arbitrary text | Proving you own a domain (for TLS certificates, Day 17), SPF/DKIM email auth |
| **NS** | `amazon.com. IN NS ns-264.awsdns-33.com.` | "These servers are authoritative" | Delegating a domain to Route 53 |
| **SRV** | `_xmpp-server._tcp.jabber.org. IN SRV 30 30 5269 scarlet.jabber.org.` | Service at host *and port*: priority 30, weight 30, **port 5269**, target | Service discovery; Kubernetes creates them for named ports |
| **PTR** | `8.8.8.8.in-addr.arpa. IN PTR dns.google.` | Reverse: IP → name | Logs, mail-server checks |
| **SOA** | `httpbin.org. IN SOA ns-1053.awsdns-03.org. … 86400` | The zone's metadata; the last number feeds negative caching | You see it in NXDOMAIN answers |

Try them:

```bash
# One question per record type, asked of Google's resolver (see Gotchas for why not WSL's).
# +noall +answer prints only the answer lines.
for q in "httpbin.org A" "www.github.com CNAME" "gmail.com MX" "amazon.com NS" "_xmpp-server._tcp.jabber.org SRV"; do
  echo "== $q"
  dig @8.8.8.8 +noall +answer $q | head -3   # $q unquoted on purpose: name and type become two arguments
done
echo "== reverse lookup of 8.8.8.8"; dig @8.8.8.8 +noall +answer -x 8.8.8.8
```

### CNAME rules that bite

- **A CNAME can't coexist with other records on the same name.** If
  `api.example.com` is a CNAME, it can't also have a TXT record.
- So **the zone apex can't be a CNAME** — `example.com` itself must have NS and
  SOA records. You can't do `example.com CNAME my-alb.elb.amazonaws.com`. Route 53
  works around this with **alias records**: an A record that Route 53 fills in
  from the load balancer's current IPs. You'll use one in Phase 15.
- Each CNAME hop is another lookup (if not cached). Chains of three or four
  CNAMEs are common with CDNs and add latency to cold lookups.

> **Say this in an interview:** "A and AAAA map names to IPv4 and IPv6, CNAME
> aliases one name to another, MX routes mail by priority, TXT holds
> verification and SPF data, NS delegates a zone, and SRV gives host and port for
> service discovery. A CNAME can't share a name with other records, so you can't
> put one at the zone apex — Route 53 uses alias records for that."

---

## Part 4 — TTL and caching: "works on my machine"

**In plain words:** every answer comes stamped "good for 60 seconds" (or 300, or
86,400). Everyone who sees it is allowed to remember it for that long without
asking again. That's what makes DNS fast enough to use for every connection. The
catch: when the owner changes the answer, nobody finds out until their
remembered copy expires — and every machine's copy expires at a different time.
And some caches don't play by the rules.

### Three caches, three answers, at the same moment

Run this; it asks five servers for the same name:

```bash
# The same question to five servers. The second column is the TTL each one reports.
for s in ns-1555.awsdns-02.co.uk 8.8.8.8 1.1.1.1 103.57.86.10 172.24.160.1; do
  printf '%-26s ' "$s"
  dig @"$s" +noall +answer +time=3 +tries=1 httpbin.org | awk 'NR == 1 { print "TTL", $2 }'
done
```

What your machine reported (7–8 October):

```
ns-1555.awsdns-02.co.uk    TTL 60       ← authoritative: the owner's TTL
8.8.8.8                    TTL 44       ← Google: cached 16 s ago, 44 s left
1.1.1.1                    TTL 5        ← Cloudflare: cached 55 s ago, 5 s left
103.57.86.10               TTL 1717     ← your ISP's primary resolver (!)
172.24.160.1               TTL 0        ← WSL's proxy to Windows
```

Read them in order:

- **The authoritative server always says 60** — that's the configured value,
  and its answer carries the `aa` (authoritative answer) flag.
- **Recursive resolvers count down.** A cached answer is returned with the time
  *remaining*. 8.8.8.8 and 1.1.1.1 are each many machines (anycast), so two
  queries a few seconds apart can hit different caches and show unrelated
  numbers. That's normal.
- **Your ISP's resolver says 1717 — for a record whose owner allows 60.** It
  enforces a *minimum* TTL of about 30 minutes, ignoring what the owner set. It
  did the same to `api.github.com` (862 vs the real 60), and to "does not exist"
  answers (1800). Its secondary, `103.57.86.36`, honoured the TTL. Your Windows
  machine asks the primary (and so does your home router), and Windows' own cache
  showed httpbin with `TimeToLive 1740`.
- **WSL's proxy says 0** — it hands the question to Windows and doesn't expose
  Windows' cache timing. A TTL of 0 from `172.24.160.1` does *not* mean "not
  cached".

That's question 1: if httpbin moved IPs, you could keep connecting to the old IP
for **up to ~30 minutes**, not 60 seconds — while a colleague on a different
ISP picks up the change in under a minute. Same code, same name, different
answers. Now add the layers *inside* your machine:

| Cache | Default lifetime | How to see / clear it |
|-------|------------------|-----------------------|
| Recursive resolver (ISP, 8.8.8.8) | Record TTL (or its own minimum — yours stretches) | You can't; query a different resolver to compare |
| Windows DNS client | Record TTL | `Get-DnsClientCache`; `ipconfig /flushdns` |
| glibc in WSL | No cache | — |
| **The JVM** | **30 seconds**, *regardless of the record's TTL* | `networkaddress.cache.ttl` (Part 7) |
| **An open TCP connection / connection pool** | **Forever** — it never re-resolves | Recycle connections (`maxLifetime`) |

The last row is the one that surprises people. A Hikari pool that connected to a
database's old IP keeps using those connections until they close. When AWS RDS
fails over, the endpoint's DNS flips to the new primary (with a 5-second TTL),
but your pool's existing connections still point at the old one — you need the
pool to notice failures and reconnect, which is one reason `maxLifetime` exists.

```powershell
Get-DnsClientCache -Entry httpbin.org | Select-Object Entry, TimeToLive, Data
ipconfig /flushdns
```

### Planning a DNS change (what you'll actually do at work)

Moving a service to a new IP or load balancer:

1. **Days before:** lower the TTL (say 3600 → 60). Wait at least the *old* TTL so
   every cache has picked up the short one.
2. **Change** the record. Keep the old target running.
3. **Watch traffic drain** from old to new. Badly behaved caches (like your ISP's)
   and long-lived connections take longer — keep the old target up for a while.
4. Raise the TTL again.

> **Say this in an interview:** "Every record has a TTL set by the zone owner,
> and every layer caches for that long — recursive resolvers, the OS, the JVM —
> so changes propagate unevenly, and some resolvers enforce minimum TTLs. Open
> connections never re-resolve at all. Before a migration I lower the TTL ahead
> of time, wait out the old TTL, make the change, and keep the old target alive
> until traffic drains."

---

## Part 5 — `dig` in depth

**In plain words:** `dig` is how you ask DNS a question and see the *whole*
answer, not just the address. It skips `/etc/hosts` and asks the DNS server
directly — useful precisely because it doesn't behave like your application.

### Anatomy of an answer

```bash
dig httpbin.org
```

Your output, annotated:

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 11001       ← status: NOERROR / NXDOMAIN / SERVFAIL
;; flags: qr rd ad; QUERY: 1, ANSWER: 8, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;httpbin.org.			IN	A                                  ← what was asked (A is the default type)

;; ANSWER SECTION:
httpbin.org.		0	IN	A	18.233.182.23                       ← name  TTL  class  type  data
...

;; Query time: 0 msec
;; SERVER: 172.24.160.1#53(172.24.160.1) (UDP)                   ← who answered, over UDP (Day 13)
```

| Flag | Meaning |
|------|---------|
| `qr` | This is a response (query/response bit) |
| `rd` | Recursion desired — we asked the server to do the walking |
| `ra` | Recursion available — the server is a recursive resolver. **Missing here**, hence the WARNING: the WSL proxy forwards to Windows rather than recursing itself |
| `aa` | Authoritative answer — only from the zone's own servers |
| `ad` | Authenticated data — DNSSEC-validated (a signature check; not covered here) |

### The options you'll use

| Command | What it does |
|---------|--------------|
| `dig +short name` | Just the data — for scripts |
| `dig name MX` | Ask for a specific type |
| `dig @8.8.8.8 name` | Ask a specific server — the way to compare resolvers |
| `dig @<authoritative> name` | Ask the owner: the "true" answer, bypassing every cache |
| `dig +trace name` | Walk from the root yourself (Part 2) |
| `dig -x 8.8.8.8` | Reverse lookup (PTR) |
| `dig +noall +answer name` | Only the answer section |
| `dig +tcp name` | Use TCP instead of UDP |

### `dig` vs what your app sees

`dig` doesn't read `/etc/hosts`, `nsswitch.conf` or the JVM's cache. To see what
an *application* sees on Linux, use **`getent hosts name`** — it goes through the
same `getaddrinfo` path as `curl` and (on Linux) Java. When `dig` and your app
disagree, that difference is the clue (question 2).

### Three ways DNS fails, and how to tell them apart

```bash
# NXDOMAIN (name doesn't exist), SERVFAIL (resolver couldn't answer), timeout (no resolver reachable).
printf '%-44s ' "NXDOMAIN - name doesn't exist:";  dig @1.1.1.1 no-such-name-zz9.httpbin.org | grep -oE 'status: [A-Z]+'
printf '%-44s ' "SERVFAIL - deliberately broken zone:"; dig @8.8.8.8 dnssec-failed.org +time=3 +tries=1 | grep -oE 'status: [A-Z]+'
printf '%-44s ' "timeout - nothing answering:"; dig @10.255.255.1 httpbin.org +time=2 +tries=1 | grep -E 'timed out' | head -1
```

| Result | Meaning | Usual cause | Cached? |
|--------|---------|-------------|---------|
| **NXDOMAIN** | The authoritative server says the name doesn't exist | Typo, record deleted, wrong environment's domain | **Yes** — negative caching, TTL from the SOA |
| **SERVFAIL** | The resolver tried and couldn't get a valid answer | Authoritative servers down or misconfigured, DNSSEC failure (`dnssec-failed.org` is broken on purpose) | Briefly |
| **Timeout** | No answer from the resolver at all | Resolver down, firewall blocking UDP 53, wrong `nameserver` | No |
| **NOERROR, 0 answers** | The name exists but has no record of that type | Asked for AAAA on an IPv4-only name | Yes |

`curl` collapses all of these into one message — `Could not resolve host`
(exit 6, Day 11). `dig` tells you which one it really is.

> **Say this in an interview:** "I use `dig @server` to compare what different
> resolvers and the authoritative server return, `+trace` to follow the
> delegation chain, and `getent hosts` to see what applications see, since `dig`
> bypasses `/etc/hosts`. NXDOMAIN means the name doesn't exist and is cached;
> SERVFAIL means the resolver couldn't get an answer, often an authoritative or
> DNSSEC problem; a timeout means the resolver itself is unreachable."

---

## Part 6 — `/etc/hosts`, `resolv.conf`, and WSL's generated files

**In plain words:** before asking anyone, your machine checks a small local
list. If the name is there, that's the answer — DNS is never asked. It's a
sticky note on the phone book: handy for testing, dangerous when forgotten.

- **`/etc/hosts`** — `IP  name [aliases]`, one per line. Checked first because
  of `hosts: files dns`. Used by `curl`, browsers, Java — not by `dig`.
- **`/etc/resolv.conf`** — `nameserver` (up to 3, tried in order), `search`
  (suffixes for short names, Part 8), `options` (timeouts, `ndots`). Yours has a
  single `nameserver 172.24.160.1` and no search line.

On your machine I added `127.0.0.1 httpbin.org` to `/etc/hosts` (then removed
it):

```
getent: 127.0.0.1       httpbin.org          ← app path: hosts file wins
dig:    34.193.104.167                       ← dig ignores /etc/hosts
curl:   000 exit=7                           ← curl went to 127.0.0.1, nothing listening: refused
```

**WSL overwrites both files.** Your `/etc/hosts` starts with "This file was
automatically generated by WSL". On every WSL start it's rebuilt from the
*Windows* hosts file. So:

- An edit to `/etc/hosts` inside WSL **disappears on the next restart.** For a
  permanent change, edit `C:\Windows\System32\drivers\etc\hosts` (as Windows
  admin), or set `generateHosts = false` in `/etc/wsl.conf`.
- Entries from Windows leak in — like Docker Desktop's.

> **Say this in an interview:** "`/etc/hosts` is consulted before DNS because of
> the `files dns` order in `nsswitch.conf`, so it silently overrides DNS for
> anything using the system resolver — but not for `dig`. `resolv.conf` sets the
> recursive resolvers, search domains and options like `ndots`."

---

## Part 7 — DNS inside the JVM

**In plain words:** Java keeps its own memory of lookups, separate from the
operating system's, with its own rules. It remembers successful lookups for 30
seconds no matter what TTL the record had, and failed ones for 10 seconds. That's
usually fine — but it means a Java process can disagree with `dig` on the same
machine.

- Lookups go through `InetAddress.getByName` / `getAllByName` (used by every HTTP
  client and JDBC driver). On Linux they call the OS's `getaddrinfo` (so
  `/etc/hosts` applies); on Windows, the Windows resolver.
- **Positive cache:** security property `networkaddress.cache.ttl`. Unset in your
  JDK 21, which means the default **30 seconds** — the record's TTL is ignored.
- **Negative cache:** `networkaddress.cache.negative.ttl` = **10 seconds** (set
  in your JDK's `conf/security/java.security`). A name that failed once keeps
  failing for 10 s even after it starts existing.
- Set them in code **before the first lookup**:
  `java.security.Security.setProperty("networkaddress.cache.ttl", "10");` or in
  `java.security`. (Very old JVMs running with a security manager cached
  *forever* — the origin of many "restart fixed it" stories, and why AWS SDK docs
  still tell you to set this.)

Hands-on 8 shows all of it on your JDK. That's half of question 2: Java may be
holding a cached failure from 10 seconds ago, or a cached answer, while `dig`
asks fresh. The other half: `dig` ignores `/etc/hosts` and search domains, so
an app and `dig` can be asking different questions entirely.

> **Say this in an interview:** "The JVM caches lookups itself: successful ones
> for 30 seconds by default regardless of the record TTL, failures for 10. It's
> controlled by the `networkaddress.cache.ttl` security properties, set before the
> first lookup. Pooled connections don't re-resolve at all, so DNS-based failover
> also needs connection recycling."

---

## Part 8 — Split horizon, service discovery, and DNS as a failure domain

**In plain words:** the same name can deliberately give different answers
depending on who asks — an office's internal phone list versus the public
directory. Platforms like Docker and Kubernetes run their own small DNS servers so
services can find each other by name. And because everything depends on that
lookup, when DNS breaks it takes everything down with it.

### Split-horizon DNS

The same name, different answers inside and outside:

- **AWS Route 53 private hosted zones:** `db.internal.mycompany.com` resolves
  only inside your VPC. Outside, NXDOMAIN.
- **RDS and other AWS endpoints** resolve to a *private* IP from inside the VPC
  and a public one (or nothing) from outside.
- **Corporate VPNs** push their own resolver, so `jira.corp` works only when
  connected.

The debugging consequence: **always ask "resolved from where?"** `dig` on your
laptop tells you nothing about what a pod in the VPC sees.

The resolver inside every AWS VPC sits at the **VPC CIDR base + 2** — e.g.
`10.0.0.2` for `10.0.0.0/16`. That's one of the five reserved addresses from
Day 12.

### Service discovery by DNS (preview of Phase 4 and 17)

- **Docker Compose:** each service name (`db`, `redis`) resolves to that
  container's IP via Docker's embedded DNS server at `127.0.0.11`.
- **Kubernetes:** every Service gets a name
  `orders.<namespace>.svc.cluster.local`, served by CoreDNS. Pods get a
  `resolv.conf` like:

  ```
  nameserver 10.96.0.10
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5
  ```

  That's question 3. `orders` has no dots, so the resolver tries the **search
  domains**: `orders.default.svc.cluster.local` exists → answer. Nobody needed a
  global record for `orders`.

- **The `ndots:5` trap:** any name with *fewer than 5 dots* is tried with every
  search suffix **first**. `api.stripe.com` (2 dots) becomes
  `api.stripe.com.default.svc.cluster.local` → NXDOMAIN,
  `…svc.cluster.local` → NXDOMAIN, `…cluster.local` → NXDOMAIN, then finally
  `api.stripe.com` — and each try is done for both A and AAAA. Up to 8 queries for
  one external lookup. The fix: a trailing dot (`api.stripe.com.`, which means "this
  is already fully qualified") or lower `ndots` for that pod.

### DNS as a failure domain

When DNS fails, nothing can find anything, so **everything looks down at once**
— and health checks and dashboards that use names fail too, so you can't see
what's happening.

- **4 October 2021, Facebook:** a configuration change withdrew the network
  routes to Facebook's own authoritative DNS servers. Every Facebook name stopped
  resolving worldwide for about six hours, and internal tools that depended on the
  same DNS went down with it.
- **20 October 2025, AWS `us-east-1`:** an automation bug left the DNS record for
  the regional DynamoDB endpoint empty. DynamoDB itself was running, but clients
  couldn't resolve it, and services built on DynamoDB failed in a cascade.

The reflex: when many unrelated things fail at the same time, **check DNS
first** — `dig` a known name against your configured resolver and against
`8.8.8.8`.

> **Say this in an interview:** "Split-horizon DNS returns different answers
> depending on where the query comes from — Route 53 private hosted zones, VPN
> resolvers — so I always check what resolver the failing client uses. Kubernetes
> does service discovery through CoreDNS and search domains, with `ndots:5`
> causing extra lookups for external names unless they're written fully
> qualified. And DNS is a shared failure domain: when it breaks, everything looks
> down at once, which is the classic pattern in outages like Facebook's in 2021."

---

## Hands-on

Blocks marked **lab** run with the Run button. Exercise 6 needs `sudo`, so it
runs in your **Ubuntu (WSL)** terminal.

### 1. Trace the chain on your machine (lab)

Run the Part 1 block. Then answer from its output: which file makes
`/etc/hosts` win over DNS? What is `172.24.160.1`, and where have you seen it
before? Why would an edit to `/etc/hosts` not survive tomorrow?

### 2. Narrate a `+trace` (lab — required practice)

Run the Part 2 block. Then do the same for `www.github.com` and write one line
per step: which server answered, and what it said. Where does the CNAME appear,
and which servers are authoritative for `github.com`?

```bash
timeout 15 dig -4 +trace +nodnssec www.github.com | grep -vE '^;; global|^; <<>>|^$'
```

### 3. Tour the record types (lab)

Run the Part 3 block. Then find, using `dig` only: which servers handle mail for
`outlook.com`, and which is tried first? Who is authoritative for `github.com`
— are they on Route 53? What TXT records does `github.com` publish?

<details>
<summary>Answers (as of 8 October)</summary>

- `dig @8.8.8.8 +short outlook.com MX` → a single MX,
  `5 outlook-com.olc.protection.outlook.com.` — with one record, priority
  doesn't matter.
- `dig @8.8.8.8 +short github.com NS` → **eight** servers: four Route 53
  (`ns-*.awsdns-*`) **and** four NS1 (`dns*.p08.nsone.net`). GitHub uses two
  separate DNS providers, so an outage at either one can't take `github.com`
  offline — a direct lesson from DNS being a failure domain (Part 8).
- `dig @8.8.8.8 github.com TXT` → several records: site-verification tokens for
  various services and an SPF record (`v=spf1 …`) listing who may send email as
  `github.com`.

</details>

### 4. Compare resolvers (lab — required practice)

Run the Part 4 five-server block. Then run it again 10 seconds later.

1. Which numbers went down by about 10? Which didn't, and why?
2. Is your ISP's resolver still stretching the TTL?
3. If `httpbin.org` changed IPs right now, roughly when would each of these see
   the new IP: Google's resolver, your Windows machine, a Java process on
   Windows that looked it up 5 seconds ago, and a pooled HTTP connection that's
   already open?

<details>
<summary>Answers to 3</summary>

- **8.8.8.8:** when its cached copy expires — at most 60 s.
- **Your Windows machine:** it gets answers from `103.57.86.10`, which holds them
  for ~30 minutes, and Windows then caches for the TTL it was given. Worst case
  around **30 minutes**.
- **The Java process:** its own 30 s cache, *then* it asks Windows — which may
  still have the old answer. So 30 s on top of Windows' delay.
- **An open pooled connection:** never, until the connection closes. DNS isn't
  consulted for an existing connection.

</details>

### 5. Three kinds of failure (lab)

Run the Part 5 failure block. For each line, say what you'd check next in a real
incident.

<details>
<summary>Answer</summary>

- **NXDOMAIN:** the name itself — typo, wrong environment, record deleted. Ask the
  authoritative server (`dig @<ns> name`) to rule out a stale negative cache.
- **SERVFAIL:** the domain's authoritative servers — `dig +trace` to see which
  step fails; try another resolver to see if it's resolver-specific.
- **Timeout:** the resolver — is the `nameserver` in `resolv.conf` reachable? Is
  UDP 53 blocked? Does `dig @8.8.8.8` work?

</details>

### 6. Prove `/etc/hosts` wins (Ubuntu terminal, `sudo`)

```
sudo cp /etc/hosts /etc/hosts.bak
echo "127.0.0.1 httpbin.org" | sudo tee -a /etc/hosts
getent hosts httpbin.org                       # 127.0.0.1 — the app's view
dig +short httpbin.org | head -1               # a real AWS IP — dig skips /etc/hosts
curl -s -o /dev/null -w '%{remote_ip}\n' --max-time 3 http://httpbin.org/get; echo "exit $?"
sudo cp /etc/hosts.bak /etc/hosts              # put it back
getent hosts httpbin.org | head -1
```

Explain the `curl` result in Day 13's terms. Then explain why you'd never need
the last two lines if you just restarted WSL (`wsl --shutdown` from PowerShell).

### 7. The Windows side (PowerShell, read-only)

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 -InterfaceAlias Wi-Fi
Get-DnsClientCache | Select-Object -First 10 Entry, TimeToLive, Data
Resolve-DnsName httpbin.org -Type A -Server 8.8.8.8 -DnsOnly | Select-Object -First 1 Name, TTL
Resolve-DnsName httpbin.org -Type A -Server 103.57.86.10 -DnsOnly | Select-Object -First 1 Name, TTL
```

Which resolvers does Windows use? Find an entry in the cache with a suspiciously
large `TimeToLive`.

### 8. The JVM's own DNS cache (lab)

```java
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.security.Security;
import java.util.Arrays;

public class DnsDemo {
    public static void main(String[] args) throws Exception {
        // null = not set -> the JDK's default of 30 seconds, whatever the record's TTL says.
        System.out.println("networkaddress.cache.ttl          = " + Security.getProperty("networkaddress.cache.ttl"));
        System.out.println("networkaddress.cache.negative.ttl = " + Security.getProperty("networkaddress.cache.negative.ttl"));
        System.out.println();

        // Same name three times: the first goes to the OS, the rest come from the JVM's cache.
        for (int i = 1; i <= 3; i++) {
            long start = System.nanoTime();
            InetAddress[] all = InetAddress.getAllByName("httpbin.org");
            System.out.printf("lookup %d: %6.1f ms  %d addresses, first %s%n",
                    i, (System.nanoTime() - start) / 1e6, all.length, all[0].getHostAddress());
        }
        System.out.println();

        // A name that doesn't exist, twice: the failure is cached too (negative caching).
        for (int i = 1; i <= 2; i++) {
            long start = System.nanoTime();
            try {
                InetAddress.getByName("no-such-host.invalid");
            } catch (UnknownHostException e) {
                System.out.printf("failed lookup %d: %6.1f ms  %s%n", i, (System.nanoTime() - start) / 1e6, e.getMessage());
            }
        }
        System.out.println();

        // These come from the Windows hosts file, not DNS (this program runs on Windows).
        System.out.println("localhost            -> " + InetAddress.getByName("localhost").getHostAddress());
        System.out.println("host.docker.internal -> " + InetAddress.getByName("host.docker.internal").getHostAddress());
    }
}
```

What to notice:

- Lookup 1 takes milliseconds (it asks Windows); lookups 2 and 3 take **~0 ms**
  — the JVM's cache.
- The second failed lookup is instant **and its message is different** (`No such
  host is known (…)` from Windows the first time, just the name the second time):
  it never left the JVM.
- `host.docker.internal` resolves although no DNS server has heard of it: it's
  in your Windows hosts file. Now compare its IP with your Wi-Fi address from
  Day 12 (`192.168.1.23`). See Gotchas.

**Extend it:** add `Security.setProperty("networkaddress.cache.ttl", "0");` as
the first line of `main` and rerun. What happens to lookups 2 and 3? Then move
that line to *after* the loop and rerun. Why does it no longer have any effect?

<details>
<summary>Answer</summary>

With TTL 0 set first, every lookup goes back to Windows, so 2 and 3 cost
milliseconds again (often less than lookup 1, because now *Windows'* cache
answers). Moved after the first lookup, it has no effect: the JVM reads the
cache policy once, at its first lookup, and keeps it for the life of the process.
That's why it has to be set at startup.

</details>

### 9. Diagnose it (paper)

For each, name the most likely cause and the one command you'd run first.

1. After a deploy that moved `api.internal` to a new load balancer, 90% of
   traffic moved within a minute; one old service is still hitting the old LB an
   hour later. It's a Java service that hasn't restarted.
2. A pod can reach `orders` but `orders.default` fails, and an external API call
   from the same pod takes 200 ms longer than from your laptop.
3. Every service in a cluster starts failing at the same moment, including the
   monitoring dashboard's links.

<details>
<summary>Answers</summary>

1. Not the JVM's 30 s cache — an hour is too long. Most likely **long-lived
   pooled connections** that were opened to the old LB and never closed. Check
   its open connections (`ss -tn dst <old-lb-ip>`); fix with a connection
   `maxLifetime`. (A very old JVM with infinite DNS caching is the other suspect.)
2. `orders.default` should resolve via the `svc.cluster.local` search suffix,
   so first look at the pod's `cat /etc/resolv.conf` — a missing or edited search
   line explains it. The external slowness is **`ndots:5`**: the name is tried
   with every search suffix first. Fix with a trailing dot or a lower `ndots`.
3. Everything at once, including links → **DNS** (the cluster resolver,
   CoreDNS, or the upstream resolver). First command: `dig` a known name against
   the configured resolver, then against `8.8.8.8`.

</details>

---

## Gotchas

- **`dig` ≠ your application.** It skips `/etc/hosts`, `nsswitch.conf`, search
  domains (unless `+search`) and the JVM cache. Use `getent hosts` for the
  application's view on Linux.
- **TTLs aren't promises.** Your ISP's primary resolver held 60-second answers
  for ~30 minutes. Other resolvers do the same. Plan migrations with the old
  target kept alive.
- **Open connections never re-resolve.** DNS failover needs connection
  recycling (`maxLifetime`, idle timeouts) as well as a low TTL.
- **The JVM caches for 30 s regardless of TTL, and failures for 10 s.** Set
  `networkaddress.cache.ttl` before the first lookup if you need something else.
- **WSL regenerates `/etc/hosts` and `/etc/resolv.conf` on every start.** Edits
  inside WSL vanish. Change the Windows hosts file, or turn generation off in
  `/etc/wsl.conf`.
- **Stale hosts-file entries.** Your Windows hosts file maps
  `host.docker.internal` to **`192.168.1.16`**, but your Wi-Fi address is now
  `192.168.1.23`. Docker Desktop wrote those lines when your laptop had the old
  address and only refreshes them while it's running. Until then, anything using
  that name goes to whichever device has `.16` now. A hosts entry has no TTL — it
  stays wrong until something rewrites it.
- **WSL's DNS proxy reports TTL 0, has no `ra` flag, and doesn't answer DNS over
  TCP** (tested: `dig +tcp` to `172.24.160.1` timed out). Large answers that
  need TCP will fail through it. It also sometimes stuffs unrelated records
  into the answer section — a `www.github.com CNAME` query through it came back
  with A records for GitHub's name servers attached. Query `8.8.8.8` directly
  when you want clean, real answers.
- **Your WSL has no IPv6 route**, so `dig +trace` without `-4` prints "network
  unreachable" errors when it tries IPv6 name servers. Harmless noise; it falls
  back to IPv4.
- **`curl`'s "Could not resolve host" hides the real error.** NXDOMAIN, SERVFAIL
  and timeout all look the same there. `dig` tells them apart.
- **No CNAME at the zone apex.** Use Route 53 alias records.
- **Negative answers are cached too.** Creating a record right after something
  looked it up doesn't fix it immediately for that resolver — wait out the SOA's
  negative TTL.

---

## Recap

- **Chain:** app → stub resolver (`nsswitch`: `/etc/hosts` then DNS) →
  recursive resolver (`resolv.conf`) → root → TLD → **authoritative**. On your
  machine: WSL → Windows (`172.24.160.1`) → ISP (`103.57.86.10`) → the tree.
- **Delegation** via NS records; `dig +trace` shows each step. httpbin.org is
  hosted on Route 53, TTL 60.
- **Records:** A, AAAA, CNAME (no apex, no siblings), MX (priority), TXT
  (verification), NS (delegation), SRV (host + port), PTR (reverse), SOA.
- **TTL** controls caching at every layer — but resolvers can stretch it (yours
  turns 60 s into ~30 min), the JVM uses 30 s/10 s, and open connections never
  re-resolve.
- **`dig`:** `@server`, `+short`, `+trace`, `-x`, type argument; flags `aa`,
  `ra`, `rd`; status **NXDOMAIN / SERVFAIL / timeout** mean different things.
- **`/etc/hosts`** overrides DNS for apps, not for `dig`; WSL regenerates it.
- **Split horizon:** the answer depends on who asks. **Service discovery:**
  Docker's `127.0.0.11`, Kubernetes' CoreDNS with search domains and `ndots:5`.
- **DNS is a shared failure domain:** many unrelated failures at once → check DNS
  first.

---

## Say it out loud

1. Walk through what happens when your Spring app connects to
   `api.github.com` for the first time — every server involved, in order.
2. What's the difference between a recursive resolver and an authoritative
   server? Which one sets `aa`, and which `ra`?
3. You're moving a service to a new load balancer. How do you change DNS so the
   cut-over is quick, and why might some clients still hit the old one hours
   later?
4. Explain A, CNAME, MX, TXT and SRV records with one real use for each.
5. `dig` says a name resolves fine, but your Java app throws
   `UnknownHostException`. Give three possible reasons.
6. How does `http://orders:8080` work inside Kubernetes, and what's the `ndots`
   problem?

---

## Q&A

_Follow-up questions from reading this lesson go here._
