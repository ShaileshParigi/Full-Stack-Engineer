# Phase 2 Quiz — Networking Fundamentals (Days 11–20)

**How to take it.** Give yourself 75–100 minutes. For every question, answer
**out loud or on paper first**, then open the answer. For the "predict"
questions, write your prediction down, *then* press Run — a prediction you make
after seeing the output doesn't count. Everything that runs is local: no internet
needed. The CIDR questions are meant to be done **without a calculator**.

**Scoring.** One point per question, 48 in total. Half a point if you had the
idea but not the words. **39 or more (80%)** — you're ready for Phase 3. Below
that, each answer tells you which day to reread; reread those parts, then retake
the questions you missed a day later.

This quiz covers Days 11–19 and the Day 20 checkpoint. Because it spans the whole
phase it also covers what a shorter Week 3 quiz (Days 11–15) would; ask if you'd
still like that one separately.

---

## Part A — Layers, addresses and NAT (Days 11–12)

**A1.** Your team is choosing between an AWS NLB and an ALB for a gRPC service
that needs mTLS end to end, and a separate public REST API that must route
`/orders/*` and `/users/*` to different services. Which load balancer goes where,
and which layer does each operate at?

<details>
<summary>Answer</summary>

An **NLB** is **L4**: it balances TCP connections by IP and port without parsing
them, so the TLS handshake (and client certificates) pass through untouched —
right for end-to-end mTLS. An **ALB** is **L7**: it terminates the TCP and TLS
connection, parses the HTTP request, and can route by host or **path** — right for
the REST API. The ALB can't do end-to-end mTLS because it ends the connection and
opens a new one to the backend; the NLB can't route by path because it never reads
the path. The ALB's backend also sees the ALB's IP unless it reads
`X-Forwarded-For`. → *Day 11, Day 16, Day 17*

</details>

**A2.** A packet travels from your laptop to a server in AWS through 15 routers.
Which addresses change at each hop, which don't, and whose MAC address does your
laptop ARP for?

<details>
<summary>Answer</summary>

The **source and destination MAC** addresses are rewritten at every hop — each
router strips the frame and builds a new one for the next link — and the **TTL**
drops by 1. The **IP addresses stay the same** end to end (unless a NAT device
rewrites them), and ports and payload are untouched. Your laptop ARPs only for its
**default gateway** — the destination isn't on its subnet, so the only MAC it
needs is the next hop's. Your own capture showed `who-has 172.24.160.1` (the
gateway) when you pinged `1.1.1.1`, not `1.1.1.1`. → *Day 11*

</details>

**A3.** Explain the difference between these two lines from `ip route get`, and
say what each tells the kernel to do:

```
1.1.1.1 via 172.24.160.1 dev eth0 src 172.24.174.221
172.24.170.5 dev eth0 src 172.24.174.221
```

<details>
<summary>Answer</summary>

The first has `via 172.24.160.1`: the destination is **not on the local subnet**,
so the kernel ARPs for the gateway's MAC and sends the frame there, with the
destination IP still `1.1.1.1`. The second has **no `via`**: `172.24.170.5` is
inside the connected subnet (`172.24.160.0/20`), so the kernel ARPs for *that
host's* MAC and delivers directly. Whether `via` appears is exactly the "is it on
my subnet?" decision. → *Day 11, Day 12*

</details>

**A4.** `tracepath` shows `pmtu 1492` at your home router, and a server's SYN-ACK
advertises `mss 1452`. Where do 1492 and 1452 come from? And what's the symptom if
a firewall on the path blocks all ICMP?

<details>
<summary>Answer</summary>

Standard Ethernet MTU is 1500; your ISP link uses **PPPoE**, whose extra header
costs **8 bytes**, so the path MTU is **1492**. MSS = MTU − 20 (IP) − 20 (TCP) =
**1452** (most likely the router rewriting the MSS in passing handshakes — "MSS
clamping"). If a
firewall drops all ICMP, the "fragmentation needed" messages that tell senders to
use smaller packets never arrive, so **small requests work but large responses or
uploads hang** — it looks like a flaky application. Never block all ICMP. → *Day
11, Day 15*

</details>

**A5.** *No calculator.* For each, give the network address, broadcast address,
usable range and number of usable hosts: **(a)** `172.20.37.200/21` and **(b)**
`10.4.130.77/26`.

<details>
<summary>Answer</summary>

**(a)** `/21` ends in the 3rd octet, with 3 host bits there → block size **8**.
37 rounds down to **32**. Network `172.20.32.0`, broadcast `172.20.39.255`, usable
`172.20.32.1`–`172.20.39.254`, **2^11 − 2 = 2,046** hosts.

**(b)** `/26` ends in the 4th octet, block size **64**. 77 rounds down to **64**.
Network `10.4.130.64`, broadcast `10.4.130.127`, usable `.65`–`.126`, **62** hosts.

→ *Day 12*

</details>

**A6.** *Predict, then Run.* Write down the six output lines before running this.

```bash
python3 - <<'EOF'
import ipaddress as ip

print(1, ip.ip_network("172.17.0.0/16").overlaps(ip.ip_network("172.16.0.0/12")))
print(2, ip.ip_network("10.0.0.0/16").overlaps(ip.ip_network("10.1.0.0/16")))
print(3, ip.ip_address("172.24.176.1") in ip.ip_network("172.24.160.0/20"))

thirds = list(ip.ip_network("10.0.0.0/16").subnets(new_prefix=19))
print(4, len(thirds), thirds[3])

net = ip.ip_network("172.20.32.0/21")
print(5, net.broadcast_address, len(list(net.hosts())))

try:
    ip.ip_network("10.0.8.0/20")
except ValueError:
    print(6, "ValueError")
EOF
```

<details>
<summary>Answer</summary>

```
1 True        Docker's 172.17.0.0/16 sits inside the private 172.16.0.0/12
2 False       adjacent /16s don't overlap — these two VPCs could peer
3 False       172.24.176.1 is just past 172.24.175.255, the next /20
4 8 10.0.96.0/19     a /16 holds 2^(19-16) = 8 /19s; they start at 0, 32, 64, 96...
5 172.20.39.255 2046  matches A5(a)
6 ValueError  10.0.8.0 isn't a multiple of 16 in the 3rd octet: misaligned
```

Line 4's fourth block starts at 96 because a `/19` is 32 wide in the third
octet. Line 6 is the alignment trap: a lenient tool would quietly turn it into
`10.0.0.0/20`. → *Day 12*

</details>

**A7.** You're given `10.0.0.0/16` for a VPC across 3 availability zones, with one
public and one private `/20` per AZ. Write the six CIDRs, say how many AWS-usable
addresses each has, and name one reason the private ones might be bigger for an EKS
cluster.

<details>
<summary>Answer</summary>

A `/16` splits into 16 `/20`s stepping 16 in the third octet. For example public
`10.0.0.0/20`, `10.0.16.0/20`, `10.0.32.0/20` and private `10.0.48.0/20`,
`10.0.64.0/20`, `10.0.80.0/20` — leaving ten spare `/20`s. Each has 4,096
addresses, **4,091 usable in AWS** (it reserves 5). On EKS with the VPC CNI **every
pod consumes a subnet IP**, so a busy node pool exhausts a small subnet and new
pods sit in `ContainerCreating`; AWS subnets **can't be resized**, so the private
ones are often `/19` or `/18`. → *Day 12 (Phase 13 and Phase 15 build this)*

</details>

**A8.** Your laptop is `192.168.1.23` and so is someone's laptop in another city.
Why does that not break the internet? And in AWS, what exactly makes a subnet
"public" or "private" — given that every subnet uses private addresses?

<details>
<summary>Answer</summary>

Private addresses (RFC 1918) are **never routed on the public internet** — a NAT
device rewrites the private source address (and, with PAT, the port) to its own
public one on the way out and un-rewrites replies using a translation table.
Unsolicited inbound traffic has no table entry and is dropped. In AWS,
**public/private is a property of the route table, not of the addresses**: a public
subnet sends `0.0.0.0/0` to an **Internet Gateway**; a private subnet sends it to a
**NAT Gateway**, so instances can start outbound connections but nothing on the
internet can start one inbound. → *Day 12*

</details>

---

## Part B — Transport and names (Days 13–14)

**B1.** "A server can only handle 65,535 connections, because there are only that
many ports." What's wrong with that, where *is* the real port limit, and what
sustained rate of **new** connections does it imply on a Linux client?

<details>
<summary>Answer</summary>

A connection is identified by the **4-tuple** (client IP, client port, server IP,
server port). A server listening on one port can hold millions of connections
because they differ in the client half. The real limit is on a **client talking to
a single destination**: it can only vary its own ephemeral port — on Linux
32768–60999, i.e. **28,232** ports. Each closed connection holds its port in
`TIME_WAIT` for **60 s**, so the sustained rate is about 28,232 / 60 ≈ **470 new
connections per second** to that destination; beyond it `connect()` fails with
"Cannot assign requested address". The fix is connection reuse (keep-alive,
pooling), not kernel tuning. → *Day 13*

</details>

**B2.** A client's `connect()` returns success, but the server application never
sees the connection and the client then hangs. On the server, `ss -ltn` shows the
listener with `Recv-Q 129  Send-Q 128`. What's happening? What do *other* clients
experience?

<details>
<summary>Answer</summary>

The **kernel completes the handshake** on the application's behalf and puts the
connection in the **accept queue** until the app calls `accept()`. On a `LISTEN`
line, `Recv-Q` is the number of connections *currently waiting* and `Send-Q` is
the queue's limit — so `Recv-Q ≥ Send-Q` means **the application has stopped
accepting** (every thread busy or deadlocked). The waiting clients' `connect()`
succeeded but nobody reads their request. Once the queue is full, the kernel drops
new SYNs, so *other* clients sit in `SYN-SENT` and see a **timeout** — which looks
exactly like a firewall. In Spring, this is Tomcat's `accept-count` filling after
the worker threads block on something slow. → *Day 13*

</details>

**B3.** Server A shows 3,000 sockets in `CLOSE-WAIT` and the number grows. Server
B shows 3,000 in `TIME-WAIT`. Which is a bug, whose is it, and what does the other
one mean?

<details>
<summary>Answer</summary>

`CLOSE-WAIT` is **always the local application's bug**: the peer closed, the
kernel acknowledged, and it's waiting for the program to call `close()` — a
leaked stream, an unclosed HTTP response body, a JDBC connection not returned. It
holds a file descriptor forever and ends in "Too many open files". `TIME-WAIT` is
**normal**: it's held for 60 s by the side that closed *first* (the active closer)
so stray old segments can't corrupt a new connection with the same 4-tuple. Lots of
them mean connections aren't being reused. → *Day 13*

</details>

**B4.** *Predict, then Run.* What will each line report — and which side holds the
`TIME-WAIT` sockets?

```bash
# 20 requests: a new connection each time, versus one connection reused.
tw() { ss -tan state time-wait 'dport = :8601' | tail -n +2 | wc -l; }

python3 - <<'EOF' &
import http.server as h
class Quiet(h.SimpleHTTPRequestHandler):
    protocol_version = "HTTP/1.1"
    def log_message(self, *args): pass
h.ThreadingHTTPServer(("127.0.0.1", 8601), Quiet).serve_forever()
EOF
server=$!
sleep 1

before=$(tw)
for i in $(seq 20); do curl -s -o /dev/null http://127.0.0.1:8601/; done
echo "20 separate curls:   TIME-WAIT +$(( $(tw) - before ))"

before=$(tw)
args=(); for i in $(seq 20); do args+=(-o /dev/null http://127.0.0.1:8601/); done
curl -s "${args[@]}"
echo "one curl, 20 URLs:   TIME-WAIT +$(( $(tw) - before ))"
kill "$server"
```

<details>
<summary>Answer</summary>

```
20 separate curls:   TIME-WAIT +20
one curl, 20 URLs:   TIME-WAIT +1
```

Each `curl` process closes its connection when it exits, so the **client** is the
active closer and holds one `TIME-WAIT` per connection (filtered on `dport` to
count the client side). One `curl` with 20 URLs reuses a single keep-alive
connection: one `TIME-WAIT` for the lot. That's the whole case for connection
pooling in one line. → *Day 13*

</details>

**B5.** Your Spring service gets `Connection reset by peer` on roughly one request
in a few hundred, mostly right after quiet periods, from a pooled HTTP client
behind an AWS ALB. Explain the mechanism and give two fixes.

<details>
<summary>Answer</summary>

A pooled connection sat **idle**. A middlebox on the path — the ALB (60 s idle
timeout by default), or a NAT gateway (350 s) — forgot the connection after its
own idle timeout. The pool still thinks it's open; the next request is sent on it,
and the far end answers with a **RST** ("reset by peer"). Fixes: set the pool's
**idle timeout / `maxLifetime` shorter than every middlebox's idle timeout**, so
the client retires connections before the network does; and/or enable connection
**validation** or TCP keep-alives (Linux's own keepalive starts after 7,200 s —
far too late to help by default). → *Day 13, Day 16*

</details>

**B6.** A client has no connect timeout set and the target host is silently
dropping packets. How long does the call hang on Linux, and why that number?

<details>
<summary>Answer</summary>

About **127 seconds**. With `tcp_syn_retries = 6` the kernel retries the SYN with
doubling waits — 1, 2, 4, 8, 16, 32 s — then waits 64 s more before giving up:
1+2+4+8+16+32+64 = 127. Every HTTP client and JDBC pool should set an explicit
connect timeout (and a read/total timeout). → *Day 13, Day 18*

</details>

**B7.** Describe who answers what when a WSL `curl` resolves `api.github.com` for
the first time on your machine: the chain of servers, which are recursive and
which authoritative, and which DNS flags tell you so in `dig` output.

<details>
<summary>Answer</summary>

`curl` calls the OS resolver, which checks `/etc/hosts` first (`hosts: files dns`),
then asks the `nameserver` in `resolv.conf` — **`172.24.160.1`, Windows' DNS proxy**.
Windows asks your ISP's **recursive resolver** (`103.57.86.10`), which, if not
cached, walks the tree: a **root** server → the **`.com`** registry servers (NS
delegation) → `github.com`'s **authoritative** servers (Route 53 / NS1), which
return the A record with its TTL. Recursive resolvers find and cache; authoritative
servers own the data. In `dig`: **`aa`** (authoritative answer) appears only from
the zone's own servers; **`ra`** (recursion available) from recursive resolvers —
Windows' proxy lacked it, hence "recursion requested but not available". → *Day 14*

</details>

**B8.** `app.example.com` has a 60-second TTL and its owner moves it to a new IP.
Name every place a stale answer can hang on, and roughly how long each can, on
your machine.

<details>
<summary>Answer</summary>

(1) The **recursive resolver** — normally 60 s, but your ISP's primary resolver
enforces a minimum of ~30 minutes (measured: 60 s records reported ~1,717 s).
(2) The **Windows DNS cache** — the TTL it was handed, so up to that long again.
(3) The **JVM** — caches successful lookups for **30 s regardless of TTL** (failures
for 10 s; set `networkaddress.cache.ttl` before the first lookup). (4) **Open
connections and pools** — they never re-resolve at all, until closed. Worst case
for you: about half an hour plus the JVM's 30 s, while a colleague on another ISP
switches in under a minute. → *Day 14*

</details>

**B9.** `dig` returns `NXDOMAIN`, `SERVFAIL`, "communications error … timed out"
and `NOERROR` with no answer lines, respectively. What does each mean, and what do
you check next?

<details>
<summary>Answer</summary>

**NXDOMAIN** — the authoritative server says the name doesn't exist (typo, wrong
environment, record deleted); cached, so ask the authoritative server directly with
`dig @<ns> name`. **SERVFAIL** — the resolver *couldn't get* a valid answer
(authoritative servers down or misconfigured, DNSSEC failure); `dig +trace` to see
which step breaks, and try another resolver. **Timeout** — no resolver answered at
all: is the `nameserver` reachable, is UDP 53 blocked; try `dig @8.8.8.8`.
**NOERROR with 0 answers** — the name exists but has **no record of that type**
(e.g. asked for AAAA on an IPv4-only name). `curl` reports all of these as "Could
not resolve host". → *Day 14*

</details>

**B10.** Two questions. **(a)** Inside a Kubernetes pod, `http://orders:8080` works
though no global DNS record named `orders` exists — how? **(b)** Why can
`api.stripe.com` be slow to resolve from that same pod?

<details>
<summary>Answer</summary>

**(a)** The pod's `resolv.conf` has **search domains** (`default.svc.cluster.local
svc.cluster.local cluster.local`). `orders` has no dots, so the resolver tries
`orders.default.svc.cluster.local` first, which CoreDNS answers. **(b)** `options
ndots:5` means any name with fewer than 5 dots is tried with every search suffix
**before** being tried as-is, so `api.stripe.com` (2 dots) causes several failed
NXDOMAIN lookups (for A and AAAA) first. Fix: write it fully qualified with a
trailing dot (`api.stripe.com.`) or lower `ndots` for that pod. → *Day 14 (Phase 15
covers Kubernetes DNS)*

</details>

---

## Part C — Diagnostics and HTTP (Days 15–16)

**C1.** You can't `ping` a server, but its web app works. Another host answers ping
but `curl` hangs. And `mtr` shows 66% loss at hop 7 and 0% at every hop after it.
For each of the three, what's your conclusion?

<details>
<summary>Answer</summary>

(1) **No conclusion that the host is down** — many hosts, cloud security groups and
even your own Windows gateway drop ICMP; the web app proves the host is up.
(2) Ping proves only **L3 reachability**; `curl` hanging means TCP/TLS/HTTP is
broken or **silently dropped** — likely a firewall/security group on the port (test
with `nc -zv -w 3 host port`). (3) **No real loss**: hop 7 rate-limits replies to
probes *addressed to itself* but forwards traffic fine — proven by 0% loss at every
later hop. Real loss starts at a hop and **persists to the destination**. → *Day 15*

</details>

**C2.** Name the layer for each: **(i)** `curl: (6) Could not resolve host`;
**(ii)** `curl: (7) Failed to connect … Connection refused` in 3 ms; **(iii)**
`curl: (28) Connection timed out` after exactly the connect timeout; **(iv)**
`curl: (60) SSL certificate problem`; **(v)** exit 0, but the page says "502".
What's the one-line command you'd run next for each?

<details>
<summary>Answer</summary>

(i) **DNS** — `dig name` vs `dig @8.8.8.8 name`. (ii) **L4, host reachable,
nothing listening** (the host's kernel sent a RST) — `ss -ltnp` on the server;
check port and bind address. (iii) **Packets dropped** — firewall/security group,
wrong IP, or host down — `nc -zv -w 3 host port`, `ip route get`. (iv) **TLS** —
`openssl s_client -connect host:443 -servername host -verify_hostname host`.
(v) **The network is fine; the application (or its upstream) returned an error**
— `curl -i` and the app/LB logs. Exit status 0 because without `-f` curl treats any
HTTP response as success. → *Day 15, Day 18*

</details>

**C3.** On your machine, adding an `iptables … -j REJECT --reject-with tcp-reset`
rule made `nc` to `1.1.1.1:443` print "Connection refused", though 1.1.1.1 does
listen there. And a bad `/etc/hosts` entry made an HTTPS call fail at the *TCP*
step, not DNS. Explain both and what they teach about reading the decision tree.

<details>
<summary>Answer</summary>

A **REJECT** rule sends a RST just as a closed port would, so "refused" *usually*
means nothing is listening but can also be a firewall that rejects (a **DROP** makes
the client time out instead; AWS security groups only drop). And a **wrong DNS
answer** — here a hosts-file entry pointing `httpbin.org` at an unroutable address
— passes the "does it resolve?" step perfectly, then fails later as a timeout.
Lesson: each rung only proves what it tests; compare what the app resolves
(`getent`) with what public DNS says (`dig @8.8.8.8`), and treat "refused" as strong
evidence, not proof. → *Day 15, Day 19*

</details>

**C4.** In Java's `HttpClient`, both a DNS failure and a refused connection arrive
as `ConnectException`. How do you tell them apart in a log, and what does the
*timing* of the failure add?

<details>
<summary>Answer</summary>

Read to the **last `Caused by:`** (root cause): DNS shows `UnresolvedAddressException`
(or `UnknownHostException` with plain `Socket`), a refusal shows
`ClosedChannelException` (or "Connection refused" with `Socket`). A connect timeout
is a different class (`HttpConnectTimeoutException`). The timing is a second clue:
refusal is near-instant (milliseconds), DNS failure takes about a resolver round
trip, and a dropped connection takes **exactly your connect timeout**. Logging only
`e.getMessage()` throws the layer away. → *Day 15*

</details>

**C5.** Classify each as safe / idempotent / neither, and say whether a client may
retry it automatically after a timeout: **(1)** `GET /orders/17` **(2)** `PUT
/orders/17/status {"status":"SHIPPED"}` **(3)** `POST /orders/17/refunds
{"amount":20}` **(4)** `DELETE /cart/items/3` **(5)** `GET /reports/daily?send=email`
(which emails the report). Then: how do you make (3) safe to retry?

<details>
<summary>Answer</summary>

(1) safe + idempotent — retry. (2) idempotent (sets state) — retry. (3) **neither**
— a retry could refund twice. (4) idempotent (the 2nd call may 404 but the *effect*
is the same) — retry. (5) a **broken contract**: it's a GET, so caches, crawlers and
retry logic assume it's safe and may repeat it, emailing every time — it should be a
`POST`. For (3): an **idempotency key** — the client generates a unique ID per
logical operation and reuses it on retries; the server stores key → result (stored
*before* the slow work, ideally via a unique constraint in the same transaction) and
returns the stored result for duplicates. → *Day 16*

</details>

**C6.** Pick the status code: **(a)** valid JSON, but the email fails validation;
**(b)** JSON with a missing closing brace; **(c)** a valid token, deleting someone
else's order; **(d)** an expired token; **(e)** creating a user whose email exists;
**(f)** limit exceeded (what header should accompany it?); **(g)** a request that
starts a 10-minute report job.

<details>
<summary>Answer</summary>

(a) **422** (or 400 if your API lumps validation in — be consistent). (b) **400**
— can't even parse it. (c) **403** — authenticated, not allowed (or 404 to avoid
revealing existence). (d) **401** with `WWW-Authenticate` — unauthenticated, so the
client should refresh its token. (e) **409 Conflict**. (f) **429** with
**`Retry-After`**. (g) **202 Accepted** with a `Location` for a status resource.
The key distinction: **401 = "who are you?", 403 = "I know who you are, and no"**.
→ *Day 16*

</details>

**C7.** *Predict, then Run.* A client POSTs a body through four different redirects.
For each, what method and body does the final request carry?

```bash
python3 - <<'EOF' &
import http.server as h

class Handler(h.BaseHTTPRequestHandler):
    def log_message(self, *args): pass

    def handle_any(self):
        length = int(self.headers.get("Content-Length", 0))
        body = self.rfile.read(length).decode()
        if self.path.startswith("/to-"):
            code = int(self.path[4:])
            self.send_response(code)
            self.send_header("Location", "/final")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return
        out = f"final saw: method={self.command} body={body!r}\n".encode()
        self.send_response(200)
        self.send_header("Content-Length", str(len(out)))
        self.end_headers()
        self.wfile.write(out)

    do_GET = do_POST = handle_any

h.HTTPServer(("127.0.0.1", 8602), Handler).serve_forever()
EOF
server=$!
sleep 1
for code in 301 303 307 308; do
  printf '%s -> ' "$code"
  curl -sL -d 'item=book' "http://127.0.0.1:8602/to-$code"
done
kill "$server"
```

Then: your team adds a `301` from `/v1/orders` to `/v2/orders`, and mobile apps
still `POST` to `/v1`. What happens to their orders, and what should the redirect be?

<details>
<summary>Answer</summary>

```
301 -> final saw: method=GET body=''
303 -> final saw: method=GET body=''
307 -> final saw: method=POST body='item=book'
308 -> final saw: method=POST body='item=book'
```

Clients follow `301`/`302`/`303` with a **GET and drop the body**. `307`/`308` keep
the method and body. So the mobile apps' orders are followed as body-less GETs to
`/v2/orders` — nothing is created, often with a `200` the app treats as success.
Use **`308 Permanent Redirect`**, or better, keep `/v1` serving the same handler;
not every client follows redirects at all, and many drop `Authorization` on a
cross-host redirect. → *Day 16*

</details>

**C8.** What does `Cache-Control: no-cache` mean — and how does it differ from
`no-store`? Then walk through a conditional `GET` with `ETag`: what's sent, what
comes back when the resource is unchanged, and what does that save?

<details>
<summary>Answer</summary>

**`no-cache` does not mean "don't cache"**: a copy may be stored but must be
**revalidated** with the server before each use. **`no-store`** means never store it
(use for sensitive data). Conditional `GET`: the first response carries
`ETag: "v1-abc"`. Next time the client sends `If-None-Match: "v1-abc"`; if
unchanged the server returns **`304 Not Modified` with no body** and the client
reuses its copy; if changed, `200` with the new body and tag. It saves the body's
bandwidth and serialisation work — the round trip still happens (add
`Cache-Control: max-age` to avoid that). `If-Match` on a `PUT` gives optimistic
concurrency (`412` if someone else changed it). → *Day 16*

</details>

**C9.** Your service is behind one AWS ALB and rate-limits by the **leftmost**
`X-Forwarded-For` value. How does an attacker get unlimited requests, and what
should you use instead?

<details>
<summary>Answer</summary>

The client can send its own `X-Forwarded-For: <random IP>` and the proxy **appends**
to it (your test: `"origin": "1.2.3.4, 103.214.63.244"`), so the leftmost entry is
whatever the attacker claimed — a fresh fake identity per request. Trust only the
entries added by proxies **you control**, counting from the **right**: with one
trusted proxy, the rightmost entry (what the ALB saw); with *n*, the *n*-th from the
right. In Spring, `server.forward-headers-strategy` makes `getRemoteAddr()` honour
the headers — only enable it when a proxy actually sets them. → *Day 16*

</details>

**C10.** What problem does HTTP/2 solve that HTTP/1.1 has, and what does it *not*
solve that HTTP/3 does? And: your SSE endpoint works locally but the user sees
nothing until the whole answer is ready, once it's deployed behind a proxy. Give
three possible causes.

<details>
<summary>Answer</summary>

HTTP/1.1 carries one request at a time per connection (HTTP-level head-of-line
blocking, worked around with many connections). **HTTP/2** multiplexes many streams
over **one TCP connection** (binary frames, header compression, negotiated by
ALPN) — but TCP delivers bytes strictly in order, so **one lost segment stalls
every stream**. **HTTP/3** runs on **QUIC over UDP**, where streams are
independently ordered. For streaming: (1) **proxy buffering** (nginx
`proxy_buffering on` by default; disable it or send `X-Accel-Buffering: no`);
(2) the server or a middleware **not flushing** per event (your JDK server held every
word until the end without `flush()`); (3) **compression** middleware that waits for
the body; (4) a client reading with `BodyHandlers.ofString()` instead of
`ofLines()`; and idle timeouts (ALB 60 s) cutting a slow stream. → *Day 16*

</details>

---

## Part D — TLS, curl, packets, firewalls and NAT (Days 17–19)

**D1.** Name the three guarantees TLS provides and the mechanism behind each. Then:
a colleague "fixes" a certificate error with `curl -k`. What exactly did they
switch off, and why is that worse than it sounds?

<details>
<summary>Answer</summary>

**Confidentiality** (symmetric encryption, AES), **integrity** (authenticated
encryption — a tampered byte fails the tag and kills the connection), and
**authentication** (a CA-signed certificate proving the server holds the private key
for that name). `-k` turns off **authentication**, not just a warning: the
connection is still encrypted, but to *whoever answered* — an attacker's server is
accepted as happily as the real one. Encryption without authentication protects you
from nobody. Same for a Java "trust-all" `TrustManager`. → *Day 17*

</details>

**D2.** Why does TLS use both asymmetric and symmetric cryptography? What is forward
secrecy, and how many round trips before data in TLS 1.2 versus 1.3?

<details>
<summary>Answer</summary>

Asymmetric (RSA/EC) solves **key agreement between strangers** over a watched
network but is ~1,000× too slow for bulk data; symmetric AES is fast but both sides
need the same secret. So the handshake uses public-key maths (an **ephemeral
ECDHE** exchange for the session key, plus a signature with the certificate's key to
prove identity), then AES-GCM for everything after. **Forward secrecy**: the session
key comes from throwaway key pairs that are deleted, so a recording of today's
traffic can't be decrypted even if the server's long-term key is stolen later.
**TLS 1.2 = 2 round trips** before data, **TLS 1.3 = 1** (and 1.3 encrypts the
certificate). → *Day 17*

</details>

**D3.** A certificate has `SAN: example.com, *.example.com`. Which of these does it
cover: `example.com`, `api.example.com`, `eu.api.example.com`, `10.0.1.5` (the
server's IP)? Is the `CN` consulted? What would you do to test against that server
by IP anyway?

<details>
<summary>Answer</summary>

Covered: `example.com` (listed explicitly) and `api.example.com` (a wildcard covers
**exactly one label**). Not covered: `eu.api.example.com` (two labels) and
`10.0.1.5` (an IP is only valid if the certificate has an IP SAN). **Modern clients
ignore the CN** — the SAN is what's checked. To test by IP while keeping validation
honest, connect with the name and pin the address: `curl --resolve
example.com:443:10.0.1.5 https://example.com/` — SNI, certificate check and `Host`
all still use the real name. → *Day 17, Day 18*

</details>

**D4.** Java throws `PKIX path building failed: unable to find valid certification
path` calling an internal API; `curl` to the same URL works from the same machine.
Explain, then give two correct fixes and one wrong one.

<details>
<summary>Answer</summary>

The chain doesn't reach a root in **the JVM's trust store**, but does reach one in
the **OS store** curl uses. You measured three separate stores: WSL `curl` 146 roots,
the **JDK's own `cacerts` 152**, and Windows' own. An internal CA installed by IT in
the OS store is absent from the JDK's. Also check the server isn't missing its
intermediate certificate (browsers fetch it; Java doesn't). Correct fixes: a Spring
Boot **SSL bundle** pointing at the CA's PEM, or a dedicated truststore via
`-Djavax.net.ssl.trustStore` (include public roots too if you call public APIs);
importing into `cacerts` with `keytool` works but is lost on JDK upgrades. **Wrong
fix:** a trust-all `TrustManager` or disabling hostname verification. → *Day 17*

</details>

**D5.** You build a private CA and issue a certificate for `localhost`. Running
`curl https://localhost:8443/` fails; `curl --cacert ca.crt …` works;
`curl --cacert ca.crt https://127.0.0.1:8443/` fails again. Explain all three. And:
a raw Java `SSLSocket` connected to a server that holds a *valid certificate for a
different name* completed the handshake. Why, and what one setting fixes it?

<details>
<summary>Answer</summary>

(1) Our CA isn't in curl's trust store, so the chain can't be verified. (2)
`--cacert` adds it as a trusted root — correct. (3) The certificate's SAN lists
`localhost` only, not `127.0.0.1` — a **hostname** mismatch even though the chain is
fine (add `IP:127.0.0.1` to the SAN to cover it). The Java case: a raw `SSLSocket`
verifies the **chain** but **not the hostname** unless you call
`SSLParameters.setEndpointIdentificationAlgorithm("HTTPS")`; `HttpClient`,
`HttpsURLConnection` and Spring's clients set it for you. (The same trap in
`openssl s_client`: add `-verify_hostname`.) → *Day 17*

</details>

**D6.** What is SNI, and why is it needed given that HTTP already has a `Host`
header? Then: an ALB terminates your TLS and forwards plain HTTP to the app. Who can
read the traffic between the ALB and the app, how does the app know the original
request was HTTPS, and when would you choose an NLB instead?

<details>
<summary>Answer</summary>

The server must pick a certificate **before** any HTTP is exchanged — the `Host`
header is inside the encryption that doesn't exist yet — so the client states the
hostname in plaintext in the ClientHello (**SNI**). It lets one IP serve many
HTTPS sites, and means the hostname is visible on the wire (without Encrypted
Client Hello). Behind a **terminating ALB**, the hop to the app is **plaintext**:
anyone inside the VPC path can read it, unless you re-encrypt. The app learns the
original scheme from **`X-Forwarded-Proto`**. Choose an **NLB (passthrough)** when
you need true end-to-end TLS or client-certificate mTLS to the app, accepting that
the load balancer can't route by path. → *Day 17, Day 11*

</details>

**D7.** Your mTLS server requires a client certificate signed by your CA. An
attacker presents a self-made certificate with exactly the same name,
`orders-service`. What happens, and what does that tell you about where identity
comes from? Where will you meet mTLS later?

<details>
<summary>Answer</summary>

It's rejected (`tlsv1 alert unknown ca`): the server verifies the client
certificate's **signature chain against the CA it trusts**, and a self-made
certificate doesn't chain to it. Identity comes from **the CA's signature plus the
client proving it holds the private key**, not from the name written on the
certificate. A request with no certificate at all fails earlier (`certificate
required`). You'll meet it as **service account / workload identity** with Spring
Security's X.509 support (Phase 8) and as the default inside **Istio**, where the
control plane issues short-lived certificates to every pod and enforces strict mTLS
(Phase 16). → *Day 17, Day 20*

</details>

**D8.** A health check prints these cumulative `curl -w` values. Compute the time
spent in each phase and say where the problem is.

```
time_namelookup:     0.020
time_connect:        0.300
time_appconnect:     1.200
time_pretransfer:    1.201
time_starttransfer:  3.401
time_total:          3.402
```

<details>
<summary>Answer</summary>

The values are **cumulative**, so subtract neighbours: DNS **20 ms**; TCP connect
0.300 − 0.020 = **280 ms** (≈ one round trip — a distant server); TLS 1.200 − 0.300
= **900 ms** (TLS 1.2's two round trips plus certificate work); server (first byte
minus request sent) 3.401 − 1.201 = **2,200 ms**; download 1 ms. The dominant
cost is the **server's think time, 2.2 s** — an application/database problem, not
DNS or network. Send the owner that breakdown rather than "your API is slow". The
TLS share is large too: a reused connection (pooling) would skip 1.2 s of it.
→ *Day 18, Day 13*

</details>

**D9.** Three curl traps. **(a)** `curl -d @orders.csv …` makes the server receive
one long line. **(b)** A health-check script keeps going after the endpoint
returns `500`. **(c)** You must test one specific backend of three behind
round-robin DNS without bypassing certificate validation. What flag or fix for each?

<details>
<summary>Answer</summary>

(a) `-d @file` **strips newlines** and sets a form `Content-Type`; use
**`--data-binary @orders.csv`** (and set `Content-Type` yourself), or `-F` for a
multipart upload. (b) curl **exits 0 on HTTP errors** by default; use **`-f` /
`--fail`** (or `--fail-with-body`) with `-sS`, and always set `--connect-timeout`
and `--max-time`. (c) **`--resolve host:443:<backend-ip>`** pins the address while
SNI, the certificate check and the `Host` header keep using the real name. → *Day 18*

</details>

**D10.** Read these three `tcpdump -nn` lines and say what each is. Then: what can
and can't a capture of an HTTPS request tell you?

```
10:11:07.412  172.24.174.221.49322 > 100.58.34.217.443: Flags [S],  seq 2021049354, win 64240, ... mss 1460 ...
10:11:07.830  100.58.34.217.443 > 172.24.174.221.49322: Flags [S.], seq 1918717871, ack 2021049355, ... mss 1452 ...
10:11:08.915  172.24.174.221.49322 > 100.58.34.217.443: Flags [F.], seq 934, ack 4876, ...
```

<details>
<summary>Answer</summary>

Line 1: the client's **SYN** from ephemeral port 49322 (offering MSS 1460). Line 2:
the server's **SYN-ACK** (`ack` = the client's ISN + 1; its MSS of **1452** shows the
PPPoE clamp). Line 3: the **client sends the first FIN** — the active closer, so
**the client side will hold `TIME_WAIT`**. The ~420 ms between lines 1 and 2 is one
RTT to the US. An **HTTPS capture shows**: IPs, ports, timing, sizes, the TCP
handshake and resets, the TLS version and cipher, the **SNI hostname**, and the
server certificate on TLS 1.2 (1.3 encrypts it). It **hides** the URL path, method,
headers (including `Authorization` and cookies), bodies and status codes — `GET
/get` appeared 0 times in your capture. → *Day 18*

</details>

**D11.** *Predict, then Run.* Fill in the 2×3 table of status codes before running.
Then answer: from **Windows**, which of the three servers can be reached through
`localhost`, and which through the WSL IP?

```bash
eth0_ip=$(ip -4 -o addr show eth0 | awk '{split($4, a, "/"); print a[1]}')
python3 -m http.server 8611 --bind 127.0.0.1  >/dev/null 2>&1 &
python3 -m http.server 8612 --bind 0.0.0.0    >/dev/null 2>&1 &
python3 -m http.server 8613 --bind "$eth0_ip" >/dev/null 2>&1 &
sleep 1
printf '%-16s %-8s %-8s %-8s\n' "reached via" "8611" "8612" "8613"
for target in 127.0.0.1 "$eth0_ip"; do
  row=""
  for port in 8611 8612 8613; do
    code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 2 "http://$target:$port/")
    row+=$(printf '%-9s' "$code")
  done
  [[ $target == 127.0.0.1 ]] && label="127.0.0.1" || label="eth0 address"
  printf '%-16s %s\n' "$label" "$row"
done
kill $(jobs -p)
```

<details>
<summary>Answer</summary>

```
reached via      8611     8612     8613
127.0.0.1        200      200      000
eth0 address     000      200      200
```

(`000` = no connection.) Each server answers only through the "doors" it stands
behind: loopback-bound (8611) only via loopback; `0.0.0.0` (8612) via both; the
`eth0` address (8613) only via that address. From **Windows** (measured):
`localhost` reaches 8611 and 8612 — **WSL relays Windows' `localhost` into WSL's
loopback** — but *not* 8613; the **WSL IP** reaches 8612 and 8613 but not 8611; and
**your phone** reaches none, because the relay listens only on Windows' loopback.
So "bound to 127.0.0.1" in WSL does **not** mean invisible to Windows. → *Day 19*

</details>

**D12.** You enable `ufw` in WSL with `deny 8102/tcp` and test from inside WSL with
`nc -zv 127.0.0.1 8102` — it succeeds. Explain, then say how you *should* test, and
what Windows `localhost:8102` returns and why.

<details>
<summary>Answer</summary>

Traffic from the machine to itself travels over the **loopback interface**, and
ufw's rules begin with `-A ufw-before-input -i lo -j ACCEPT` — **loopback is always
allowed**, so a local test proves nothing about the rule. Test from **another
host** — here Windows, with `Test-NetConnection <WSL IP> -Port 8102`, which
returns `False` (and `True` for an allowed port). Windows `localhost:8102` still
returns **200**, because WSL's relay connects from *inside* WSL via loopback, which
ufw also exempts — so **ufw can't block access that arrives through the localhost
relay**. → *Day 19*

</details>

**D13.** Three firewall questions. **(a)** What's the difference between a stateful
and a stateless filter, and what broke on your machine when you removed ufw's
`ESTABLISHED,RELATED` rule? **(b)** Security Group vs NACL: statefulness, rule types,
scope, evaluation. **(c)** How does a client experience `DROP` versus `REJECT`?

<details>
<summary>Answer</summary>

**(a)** A **stateful** filter tracks connections, so allowing the request allows the
reply; a **stateless** one judges each packet alone, so you must allow both
directions, for every protocol, including the client's ephemeral port range. Removing
the rule made outbound `curl` fail — the replies were dropped by default-deny — and
allowing TCP source 443 wasn't enough: the **DNS replies** (UDP source 53) were
dropped too until they got their own rule. **(b)** A **Security Group** is
**stateful, allow-only, attached to an instance**, rules evaluated together. A
**NACL** is **stateless, allow *and* deny, attached to a subnet**, evaluated in
**number order, first match wins** — so it also needs the outbound ephemeral-port
rule. **(c)** **DROP** → the client **times out** (looks like a dead host); **REJECT**
→ it fails immediately with "refused" (≈2 s from Windows, which retransmits the SYN
first). AWS security groups only drop. → *Day 19*

</details>

**D14.** How do Kubernetes Services and `docker run -p 8080:80` share a mechanism?
And what does `ssh -L 5433:db.internal:5432 ec2-user@bastion` do — including where
`db.internal` is resolved?

<details>
<summary>Answer</summary>

Both use **NAT in the kernel's netfilter**: a **DNAT** rule rewrites the destination
of packets arriving at a virtual/published address to the real target — `kube-proxy`
rewrites a Service's virtual IP to a pod IP; Docker rewrites host port 8080 to the
container's port 80. (Your `REDIRECT` experiment rewrote port 9999 to 8104 the same
way, and **MASQUERADE** is the outbound twin.) The SSH command opens **your
`localhost:5433`** and carries each connection through the SSH session to
**`db.internal:5432`, reached from the bastion** — so `db.internal` is resolved
**on the bastion**, not on your laptop (split-horizon DNS). It binds loopback by
default. (Not run on your machine — `sshd` isn't installed.) → *Day 19, Day 14*

</details>

---

## Part E — Put it together (Day 20 and the whole phase)

**E1.** "The orders API is unreachable." Here is what each command printed. Work
down the ladder and say where the problem is and what you'd do:

```
getent hosts orders.internal      → 10.0.2.15  orders.internal
dig +short @8.8.8.8 orders.internal   → (empty)
ip route get 10.0.2.15            → 10.0.2.15 via 172.24.160.1 dev eth0 src 172.24.174.221
nc -zv -w 3 10.0.2.15 8443        → nc: connect ... timed out
```

<details>
<summary>Answer</summary>

Rung 1 *looks* fine — the name resolves — but note **where** it resolved: `getent`
(which reads `/etc/hosts` first) found an answer that **public DNS doesn't have**.
That's either a stale/overriding `/etc/hosts` entry or a private (split-horizon)
name — check `grep orders /etc/hosts`. Rung 2 is fine (a route exists). Rung 3 is
where it fails: a **timeout**, meaning packets are silently **dropped**, not refused
— consistent with the address being wrong, unreachable from this network, or a
firewall/security group. So: confirm the intended IP for `orders.internal` from the
authoritative/internal DNS, then test that address from a host inside the right
network. Two takeaways: a wrong DNS answer shows up later as a timeout, and
`getent` vs `dig` disagreeing is itself a clue. → *Day 14, Day 15*

</details>

**E2.** Your `healthcheck.sh` exits `0` OK, `1` WARNING, `2` CRITICAL and `3` UNKNOWN.
Which code does each of these produce, and why is `3` a separate code rather than
folded into `2`? **(i)** `curl` gets "connection refused". **(ii)** The script is
run with `WARN_TOTAL_MS=fast`. **(iii)** HTTP `503` returned in 40 ms. **(iv)**
HTTP `200` with a 1.8 s server phase and a 1.5 s warning threshold on total.

<details>
<summary>Answer</summary>

(i) **2** — the check ran and the service is down, and the message names the layer
(`TCP:`). (ii) **3** — the *check* couldn't run (bad configuration). (iii) **2** —
a fast failure is still a failure; HTTP ≥ 400 is critical regardless of timing.
(iv) **1** — it works but is slow (total 1.8 s ≥ 1.5 s warn, assuming the critical
threshold isn't crossed). `3` is separate because **a broken monitor must never look
like a failing service** (or a healthy one): a typo in the threshold shouldn't page
someone at 3am for an outage that doesn't exist. This is the same contract Nagios
checks and Kubernetes `exec` probes use. → *Day 20*

</details>

**E3.** While writing the checkpoint's test harness, a test called "wrong hostname
(TLS)" passed — but the script had returned *connection refused*, not a TLS error.
Why did it pass, what was the real flaw, and how was it fixed? What's the general
lesson for tests?

<details>
<summary>Answer</summary>

The test only checked for **exit code 2**, and *every* failure — refused, DNS,
timeout, TLS — exits 2. The target `127.0.0.2:8443` wasn't listening at all, so the
script failed at the TCP rung for the wrong reason, and the test couldn't tell.
The fix was twofold: each test now **also asserts the layer text** (`TLS:`, `TCP:`,
`DNS:` …), and "wrong hostname" now reaches the real server under a name its
certificate doesn't cover — `RESOLVE=other.example:8443:127.0.0.1` — so TCP succeeds
and TLS genuinely refuses (`no alternative certificate subject name matches`).
Lesson: a test that can pass for the wrong reason isn't testing the thing; assert
*why* it failed, and prove the harness can fail by breaking the code on purpose.
→ *Day 20, Day 15*

</details>

**E4.** Describe the path from a Windows browser opening `https://localhost:8443/`
to a Python process in WSL: each hop, its layer, and where TCP and TLS each *end*.
What breaks if WSL restarts? Why does the browser warn about the certificate?

<details>
<summary>Answer</summary>

(1) The browser resolves `localhost` — Windows tries `::1` first (nothing listens;
measured warning), then `127.0.0.1`. (2) TCP connects to **Windows' `127.0.0.1:8443`**,
where a WSL-owned relay process listens. (3) The relay opens a **second** TCP
connection from inside WSL to `localhost:8443` over WSL's loopback. (4) `svc.py`
accepts it (`ss -ltnp`). **TCP ends twice** (browser↔relay, relay↔process) because
the relay is an L4 forwarder; **TLS ends once, at the process** — the relay copies
bytes without decrypting, which is why the browser sees the service's own
certificate. If WSL restarts, the service is gone and the relay's port closes (the
WSL IP also changes). The browser warns because the certificate was issued by a
**private CA that isn't in the Windows trust store** (`SEC_E_UNTRUSTED_ROOT`);
trusting the CA would silence it — and that CA key can then mint certificates
Windows accepts for any site, so only do that deliberately. → *Day 20, Day 19, Day 17*

</details>

**E5.** You're moving an API to a new load balancer at a new IP. Lay out the plan so
the cut-over is quick and reversible, and explain why some clients may still hit the
old one hours later.

<details>
<summary>Answer</summary>

(1) **Lower the TTL days ahead** (say 3600 → 60) and wait out the *old* TTL so every
cache has picked up the short one. (2) **Test the new target first** without DNS:
`curl --resolve api.example.com:443:<new-ip> https://api.example.com/health`
(certificate and SNI still validate). (3) **Change the record**, keeping the old
load balancer running. (4) **Watch traffic drain**; roll back by changing the record
back — fast because the TTL is low. (5) Raise the TTL again afterwards. Stragglers
persist because of resolvers that enforce **minimum TTLs** (your ISP's ~30 min), the
**JVM's 30 s cache**, and — longest — **pooled or long-lived connections that never
re-resolve** until closed (set `maxLifetime`, see B5). So the old LB must stay up
until its traffic is genuinely zero. → *Day 14, Day 13, Day 18*

</details>

**E6.** At 14:02 a dozen unrelated services start failing at once, and the monitoring
dashboard's own links stop loading. What do you suspect first and why? Give the two
commands you'd run, and name a real outage with this signature.

<details>
<summary>Answer</summary>

**DNS** — it's a shared failure domain: when names stop resolving, everything
looks down at once, and tools that depend on names (dashboards, health checks) fail
too, so you lose visibility exactly when you need it. Run `dig <known-name>` against
the **configured resolver** and again against **`@8.8.8.8`**: failure only on the
first points at the local/cluster resolver (CoreDNS, the VPC resolver); failure on
both points at the zone. Real examples: **Facebook, 4 October 2021** (routes to its
authoritative DNS servers withdrawn — names unresolvable worldwide for ~6 hours) and
**AWS `us-east-1`, 20 October 2025** (an empty DNS record for the DynamoDB endpoint
cascaded through dependent services). → *Day 14, Day 15*

</details>

---

## Results log

| Date | Score (/48) | Questions missed | Days to reread |
|---|---|---|---|
| | | | |

---

## Q&A

_Follow-up questions go here._
