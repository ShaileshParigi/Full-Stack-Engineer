# Day 13 — TCP, UDP & Connection State

_Why this matters:_ Almost every production networking incident you'll be paged
for is a TCP incident wearing a costume. "HikariPool-1 - Connection is not
available, request timed out after 30000ms." A load test that dies with "Cannot
assign requested address". A service whose `ss` output shows thousands of
sockets in `CLOSE_WAIT`. "Connection reset by peer" every few minutes on a
service behind a load balancer. Calls that hang for exactly two minutes. Each of
these is one specific TCP mechanism — the handshake, teardown, `TIME_WAIT`, the
accept queue, connection reuse — and once you can see those mechanisms in `ss`
and `tcpdump`, these stop being mysteries. It's also the heart of Phase 5:
connection pools and thread pools are the same idea, and they fail together.

> **This day is long.** If it runs past two hours, split it: Parts 1–5 and
> hands-on 1–5 today; Parts 6–9 and hands-on 6–10 next session. Hands-on 1, 6
> and 9 are the required practice.

---

## The one-paragraph version

IP (Day 11) only promises "best effort" — packets can be lost, duplicated, or
arrive out of order. **TCP** builds a reliable, ordered stream of bytes on top of
that: it numbers every byte, the receiver acknowledges what it got, and anything
unacknowledged is resent. Before any data flows, the two sides do a **three-way
handshake** (SYN, SYN-ACK, ACK) to agree on starting numbers — which costs one
full round trip before your first byte, and TLS adds more on top. Closing is its
own exchange (FIN in each direction), and the side that closes first then sits
in **`TIME_WAIT`** for 60 seconds holding that port. Every connection is
identified by four things: client IP, client port, server IP, server port.
Opening a new connection per request is therefore slow (handshakes), wasteful
(TIME_WAIT sockets pile up, ports run out) and expensive for the server — which
is why every serious client keeps finished connections open and reuses them:
**connection pooling**. HikariCP does it for your database, your HTTP client does
it for HTTP. **UDP** is the opposite trade: no handshake, no ordering, no
retries — just send a packet — and it's used where speed matters more than
completeness, or where the application wants to handle reliability itself (DNS,
video calls, HTTP/3).

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **port** | A number naming one program's mailbox on a machine | 16-bit L4 field, 0–65535 |
| **well-known port** | Ports reserved for standard services | 0–1023 (22 SSH, 53 DNS, 80 HTTP, 443 HTTPS); binding one needs root on Linux |
| **ephemeral port** | A temporary port the OS picks for the client side of a connection | Linux default range 32768–60999 (`ip_local_port_range`) |
| **socket** | Your program's handle on a network endpoint | A kernel object; the file descriptor your code reads and writes |
| **4-tuple** | The four numbers that identify one TCP connection | (src IP, src port, dst IP, dst port) |
| **listen** | "I'm open for connections on this port" | `bind()` + `listen()`: the kernel accepts handshakes on the app's behalf |
| **accept** | The app picking up the next finished connection | `accept()` takes a completed connection from the accept queue |
| **backlog / accept queue** | Connections that finished the handshake but the app hasn't picked up yet | Bounded by `listen(backlog)` and `net.core.somaxconn` |
| **segment** | One TCP packet | TCP header + up to MSS bytes of data |
| **sequence number** | The position of a byte in the stream | 32-bit counter; starts at a random *ISN* (initial sequence number) |
| **ACK** | "I've received everything up to byte N" | Acknowledgement number = next byte expected; also a header flag |
| **flags** | One-bit markers on a segment saying what it is | `SYN` start, `ACK` acknowledgement, `FIN` finished sending, `RST` abort, `PSH` push |
| **handshake** | The three-message setup before data flows | SYN → SYN-ACK → ACK |
| **RTT** | How long a message takes there and back | Round-trip time; ~245 ms from your WSL to httpbin in the US |
| **retransmission** | Resending data the other side didn't acknowledge | Triggered by a timeout (RTO) or duplicate ACKs |
| **flow control** | "Don't send faster than I can read" — the *receiver* protects itself | The receive window (`win` in tcpdump) |
| **congestion control** | "Don't send faster than the network can carry" — the *sender* protects the network | The congestion window (`cwnd`); algorithms like CUBIC |
| **slow start** | Starting carefully and speeding up while nothing is lost | `cwnd` begins at 10 segments and roughly doubles each RTT |
| **teardown** | Closing a connection politely | FIN from each side, each ACKed |
| **RST / reset** | Slamming the connection shut | A segment with the RST flag; "connection refused" / "reset by peer" |
| **`TIME_WAIT`** | A 60-second "cooling off" state after closing | Held by the side that closed first; protects against stray old packets |
| **`CLOSE_WAIT`** | "The other side hung up, and my app hasn't closed its end" | Kernel waiting for the application to call `close()` |
| **half-open** | A handshake that started but never finished | A socket in `SYN-RECV` |
| **SYN flood** | An attack that fills the server's half-open table with fake SYNs | Defeated by SYN cookies |
| **head-of-line blocking** | One stuck item holding up everything queued behind it | In TCP: one lost segment delays all later bytes |
| **UDP** | Send a packet, hope it arrives | User Datagram Protocol: connectionless, unordered, unreliable |
| **datagram** | One self-contained UDP message | Delivered whole or not at all |
| **QUIC** | A newer protocol giving TCP-like reliability over UDP | The transport under HTTP/3 |
| **keep-alive (HTTP)** | Leaving the connection open after a response for the next request | HTTP/1.1 default persistent connections |
| **connection pool** | A set of open connections kept ready for reuse | HikariCP for JDBC; the pool inside every HTTP client |

---

## Before you read: three questions

1. Your load-test machine fires requests at a service, each on a new
   connection. After about a minute, the load tester starts failing with
   "Cannot assign requested address". The service is healthy. What ran out, and
   on which machine?
2. `ss` on one of your servers shows 3,000 sockets in `CLOSE-WAIT`, and the
   number grows every hour. Whose bug is it: the client's, the network's, or
   your server application's?
3. A client's `connect()` call returns successfully. The server application
   never sees the connection. How is that possible?

---

## Part 1 — Ports, sockets, and what "listening" means

**In plain words:** a machine has one IP address but runs many programs, so
each program that wants network traffic claims a numbered mailbox — a port. A
server claims a fixed, well-known number so clients know where to find it (443
for HTTPS). A client doesn't care which number it gets, so the OS hands it a
random temporary one for the length of the conversation. And a single
conversation is identified by *both* ends: my address and mailbox, your address
and mailbox. That's why one server port can hold a million conversations at once.

### Port ranges

| Range | Name | Who uses it |
|-------|------|-------------|
| 0–1023 | **well-known** | Standard services. On Linux, binding one needs root (or the `CAP_NET_BIND_SERVICE` capability) — which is why your Spring Boot app uses 8080, not 80 |
| 1024–49151 | **registered** | Common app ports by convention: 5432 Postgres, 6379 Redis, 9092 Kafka, 8080 |
| **32768–60999** | **ephemeral (Linux)** | What your kernel assigns to outgoing connections. Your WSL: `cat /proc/sys/net/ipv4/ip_local_port_range` → `32768 60999` |
| 49152–65535 | ephemeral (Windows, IANA) | Same idea on Windows — you saw Java on Windows use 51631 on Day 11 |

### The 4-tuple

A TCP connection is identified by **(client IP, client port, server IP, server
port)**. Change any one and it's a different connection. Consequences:

- A server on `:443` can hold millions of connections — they all share the
  server half of the tuple and differ in the client half. "A server can only have
  65,535 connections" is a classic interview *wrong* answer.
- A **client** talking to **one** destination (`10.0.1.5:5432`) varies only its
  own port. On Linux that's ~28,000 ports (32768–60999). That limit *is* real,
  and it's question 1 — Part 6.

### What `listen` actually does

A server makes three system calls: `bind(port)` claims the port, `listen()`
tells the kernel "accept connections here", and then the app calls `accept()` in
a loop to pick up each new connection.

The important part: **the kernel does the handshake, not your app.** Completed
connections wait in an **accept queue** until the app calls `accept()`. If the
app is slow (all threads busy), they wait there. If that queue is full, the
kernel silently drops new SYNs, and the clients just... wait.

That's question 3. Here's it happening — a server that calls `listen(2)` and
then never calls `accept()`:

```bash
# A server that listens with backlog 2 but never accepts. Watch what clients see.
python3 - <<'EOF'
import socket, subprocess
srv = socket.socket()
srv.bind(("127.0.0.1", 9200))
srv.listen(2)                       # accept queue: 2 (Linux allows backlog + 1)
clients = []
for i in range(5):
    c = socket.socket()
    c.settimeout(1.5)
    try:
        c.connect(("127.0.0.1", 9200))
        print(f"client {i}: connect() returned OK")
    except Exception as e:
        print(f"client {i}: {type(e).__name__}: {e}")
    clients.append(c)
print(subprocess.run(["ss", "-tan", "sport = :9200 or dport = :9200"],
                     capture_output=True, text=True).stdout)
EOF
```

On your machine:

```
client 0: connect() returned OK
client 1: connect() returned OK
client 2: connect() returned OK
client 3: TimeoutError: timed out
client 4: TimeoutError: timed out
State    Recv-Q Send-Q Local Address:Port  Peer Address:Port
LISTEN   3      2          127.0.0.1:9200       0.0.0.0:*
ESTAB    0      0          127.0.0.1:35272    127.0.0.1:9200
...
SYN-SENT 0      1          127.0.0.1:35292    127.0.0.1:9200
SYN-SENT 0      1          127.0.0.1:35304    127.0.0.1:9200
```

- Three clients "connected" — `connect()` returned success — and the server
  **application never saw any of them**. They're `ESTAB` in the kernel, sitting
  in the accept queue.
- On a **`LISTEN`** line, `ss` repurposes the columns: **`Recv-Q` = connections
  waiting in the accept queue now (3)**, **`Send-Q` = the queue's limit (2)**.
  `Recv-Q` at or above `Send-Q` on a listener means "this app has stopped
  accepting". Remember that — it's a 3am diagnostic.
- Clients 3 and 4 are stuck in **`SYN-SENT`**: their SYNs were dropped, so from
  the client's side it looks exactly like a firewall — a **timeout**, not a
  refusal.

**In Spring Boot** this is Tomcat: `server.tomcat.threads.max` (200) worker
threads, `server.tomcat.max-connections` (8192) connections it will hold, then
`server.tomcat.accept-count` (100) more in the kernel's accept queue. When all
of that fills — usually because every thread is blocked on a slow database —
new clients time out while the process looks "up". Phase 5 and 6 come back to
this.

> **Say this in an interview:** "A TCP connection is identified by the 4-tuple,
> so a server port can hold far more than 65K connections; the real port limit is
> on the client side, to a single destination. When a server listens, the kernel
> completes handshakes into an accept queue and the app drains it with
> `accept()`. If the app stops accepting, the queue fills, new SYNs are dropped
> and clients see timeouts — `ss -lnt` shows it as `Recv-Q` reaching `Send-Q` on
> the listener."

---

## Part 2 — What TCP guarantees, and how

**In plain words:** IP is like posting loose pages: some get lost, some arrive
twice, some arrive in the wrong order. TCP numbers every page, the receiver keeps
sending back "I've got everything up to page 40", and the sender reposts anything
not acknowledged in time. On top of that, two speed limits: the receiver says
"I can only hold this much right now" (so it isn't overwhelmed), and the sender
starts slowly and speeds up until it notices pages getting lost (so it doesn't
overwhelm the network in between). Your Java code sees none of this — it just
reads a stream of bytes in the right order.

| Guarantee | Mechanism | Where you see it |
|-----------|-----------|------------------|
| **Ordering** | Every byte has a **sequence number**; the receiver reassembles by number | `seq 1:7` in tcpdump — bytes 1 through 6 |
| **Reliability** | Receiver **ACKs** the next byte it expects; unacknowledged data is **retransmitted** after a timeout (RTO) | `ack 7`; `rto:630` in `ss -ti` |
| **No duplicates** | Receiver discards bytes it already has, by sequence number | — |
| **Flow control** | Receiver advertises a **window**: how many more bytes it can buffer | `win 64240` in tcpdump |
| **Congestion control** | Sender keeps a **congestion window** (`cwnd`): data in flight before it must wait for ACKs. Grows while things go well (**slow start**), shrinks sharply on loss | `cubic … cwnd:10` in `ss -ti` |

### Seeing them on a real connection

`ss -ti` prints TCP's internal state for live connections. This was a real
HTTPS request from your WSL to httpbin, captured mid-flight (trimmed):

```
ESTAB 0 0  172.24.174.221:45174  54.159.186.149:443
   cubic wscale:8,7 rto:630 rtt:244.615/52.477 mss:1440 cwnd:10
   bytes_sent:905 bytes_received:4421 ... minrtt:243.183
```

| Field | Meaning |
|-------|---------|
| `cubic` | The congestion-control algorithm (Linux default) |
| `rtt:244.615/52.477` | Smoothed round-trip time 244.6 ms, variation 52.5 ms. India ↔ US east coast |
| `rto:630` | Retransmission timeout: if a segment isn't ACKed in 630 ms, resend it. Derived from the RTT |
| `mss:1440` | Max data per segment. Day 11's 1452 (PPPoE) minus 12 bytes of timestamp option |
| `cwnd:10` | Congestion window: 10 segments (~14 KB) may be in flight unacknowledged |

### Why that matters for performance: bandwidth isn't the only limit

With `cwnd:10` and a 245 ms RTT, a brand-new connection can send at most ~14 KB
per round trip — about **57 KB/s** at first, however fast your line is. Slow
start then roughly doubles `cwnd` each RTT. So a **new** connection is slow twice
over: the handshake costs a round trip before any data, and then it starts with a
small window. A **reused** connection has already grown its window. That's a
second reason (after the handshake) why pooling matters on high-latency links.

> **Say this in an interview:** "TCP gives an ordered, reliable byte stream over
> unreliable IP: bytes carry sequence numbers, the receiver sends cumulative
> ACKs, and unacknowledged data is retransmitted after an RTO derived from the
> measured RTT. Flow control is the receiver's advertised window; congestion
> control is the sender's congestion window, which starts small in slow start and
> backs off on loss. That's why new connections are slow on high-latency links."

---

## Part 3 — The three-way handshake

**In plain words:** before talking, both sides need to agree on where their
page numbering starts, and prove they can hear each other. Client: "I'd like to
talk; my numbering starts at X" (**SYN**). Server: "OK, I heard X; mine starts at
Y" (**SYN-ACK**). Client: "I heard Y" (**ACK**). Three messages, one full round
trip, before a single byte of your request moves.

Captured on your WSL loopback (`sudo tcpdump -i lo -nn 'tcp port 9000'`) while
`nc` sent the 6 bytes `hello\n`:

```
127.0.0.1.57992 > 127.0.0.1.9000: Flags [S],  seq 1700866182, win 65495, options [mss 65495,...]
127.0.0.1.9000 > 127.0.0.1.57992: Flags [S.], seq 191828866, ack 1700866183, ...
127.0.0.1.57992 > 127.0.0.1.9000: Flags [.],  ack 1, ...
127.0.0.1.57992 > 127.0.0.1.9000: Flags [P.], seq 1:7, ack 1, ..., length 6
127.0.0.1.9000 > 127.0.0.1.57992: Flags [.],  ack 7, ...
```

| # | Flags | Meaning |
|--:|-------|---------|
| 1 | `[S]` SYN | Client: "start; my sequence numbers begin at 1700866182." The client socket is now **`SYN-SENT`** |
| 2 | `[S.]` SYN + ACK | Server: "mine begin at 191828866, and `ack 1700866183` — I expect your byte after the SYN." Server's half-made connection is **`SYN-RECV`** |
| 3 | `[.]` ACK | Client: "got yours." Both sides are **`ESTABLISHED`** |
| 4 | `[P.]` data | `seq 1:7` = bytes 1–6, the 6 bytes of `hello\n`. tcpdump switches to *relative* numbers after the handshake. `P` = push: deliver to the app now |
| 5 | `[.]` ACK | Server: `ack 7` — "I have everything up to byte 6, send byte 7 next" |

Notice the SYN itself "uses up" one sequence number (ack = ISN **+ 1**). The
starting numbers are random, which stops an attacker from guessing them and
injecting fake segments into your connection.

### What it costs

One round trip before data can flow. To httpbin that's ~245 ms. HTTPS then needs
a TLS handshake on top (Day 17). `curl -w` shows the bill for a real request from
your WSL:

```
connect 0.313144s  tls 0.949092s  total 1.213440s   new-connections 1    ← first request
connect 0.000033s  tls 0.000033s  total 0.261326s   new-connections 0    ← second request, reused
```

**Almost a second of setup** for a request whose actual work took a quarter of a
second. The second request reused the connection and skipped all of it. Hold
that thought for Part 9.

### When nobody answers the SYN: the two-minute hang

If a SYN gets no reply at all (host down, packet dropped by a firewall), Linux
retries with doubling waits: 1, 2, 4, 8, 16, 32 seconds — `tcp_syn_retries` is 6
on your WSL — then waits 64 s more before giving up. **About 127 seconds.** Any
Java client without an explicit connect timeout inherits that. Every HTTP client
and JDBC pool you configure should set one (`connectTimeout`, Hikari's
`connectionTimeout`).

### SYN floods

**In plain words:** the server has to remember every half-finished handshake
(it sent SYN-ACK, waiting for the ACK). An attacker sends huge numbers of SYNs
from fake addresses and never finishes any of them, so the server's table of
half-finished handshakes fills, and real clients can't get in.

The precise version: half-open connections sit in the **SYN queue** in state
`SYN-RECV` (your WSL allows `tcp_max_syn_backlog` = 256). The defence is **SYN
cookies** (`tcp_syncookies` = 1 on your machine, the Linux default): when the
queue is full, the server stops storing state and instead encodes it into the
sequence number of its SYN-ACK. A real client's ACK echoes that number back, so
the server can rebuild the connection from it; fake SYNs cost it nothing. In AWS,
load balancers and AWS Shield absorb SYN floods before they reach your
instances.

> **Say this in an interview:** "The three-way handshake is SYN, SYN-ACK, ACK —
> both sides exchange random initial sequence numbers, and it costs one RTT
> before any data, plus TLS on top for HTTPS. A SYN flood fills the server's
> queue of half-open connections; SYN cookies defend against it by encoding the
> connection state in the SYN-ACK's sequence number instead of storing it.
> Unanswered SYNs retry with exponential backoff for about two minutes on Linux,
> which is why clients need explicit connect timeouts."

---

## Part 4 — Closing: FIN, RST, and the difference

**In plain words:** there are two ways to end a phone call. The polite way:
"I've finished talking" — "OK, me too" — both hang up; each side says it's done
separately, because one side might still have things to say. That's **FIN**.
The rude way: slam the phone down. That's **RST**, and it means something went
wrong.

### Polite close: FIN

TCP is two independent one-way streams, so each side closes its own direction.
Continuing the capture above:

```
127.0.0.1.57992 > 127.0.0.1.9000: Flags [F.], seq 7, ack 1        ← client: "I'm done sending"
127.0.0.1.9000 > 127.0.0.1.57992: Flags [F.], seq 1, ack 8        ← server: "got it — and I'm done too"
127.0.0.1.57992 > 127.0.0.1.9000: Flags [.],  ack 2               ← client: "got yours"
```

The textbook teardown is four segments (FIN, ACK, FIN, ACK). Here it's three,
because the server had nothing left to send and **combined** its ACK with its own
FIN. Both are normal. Like SYN, a FIN uses one sequence number (`ack 8` after
byte 7).

The side that sends the first FIN is the **active closer**. Remember that phrase
for Part 6.

### Abortive close: RST

A RST means "this connection does not exist / is being killed now". No
acknowledgement, no waiting. You meet it as:

| What you see | What happened |
|--------------|---------------|
| **`Connection refused`** | Your SYN reached a host with nothing listening on that port; its kernel answered `[R.]`. Captured on your machine: `127.0.0.1.9001 > 127.0.0.1.40624: Flags [R.], seq 0, ack 3028047615, win 0` — instant, one packet |
| **`Connection reset by peer`** | The other side killed an *established* connection: the process crashed, it closed with unread data, or a **middlebox** (load balancer, NAT, firewall) had forgotten the connection and rejected your next segment |
| `Broken pipe` | You wrote to a connection the other side had already reset |

The "forgot the connection" case is the one that pages you. Part 9 covers it.

> **Say this in an interview:** "FIN is a graceful half-close — each direction
> is closed independently and acknowledged, so teardown is up to four segments.
> RST aborts immediately: it's what a closed port answers to a SYN, and what you
> get as 'connection reset by peer' when the peer or a middlebox drops state for
> a live connection."

---

## Part 5 — Connection states you'll see in `ss`

**In plain words:** each socket is always in one named state, and `ss -tan`
prints it. Most of the time you only care about four: `LISTEN` (a server
waiting), `ESTAB` (talking), `TIME-WAIT` (recently closed, cooling off) and
`CLOSE-WAIT` (the other side hung up and *we* haven't). The rest are short-lived
steps you'll only catch if something's stuck.

```
              client                                  server
                                                      LISTEN
   SYN-SENT   ──── SYN ──────────────────────────▶   SYN-RECV
   ESTAB      ◀─── SYN-ACK ───────────────────────
              ──── ACK ──────────────────────────▶   ESTAB
                         ... data both ways ...
   FIN-WAIT-1 ──── FIN ──────────────────────────▶   CLOSE-WAIT   ← waits for the APP to close()
   FIN-WAIT-2 ◀─── ACK ───────────────────────────
                                                      (app calls close())
              ◀─── FIN ───────────────────────────   LAST-ACK
   TIME-WAIT  ──── ACK ──────────────────────────▶   CLOSED
   (60 s)
   CLOSED
```

(Here the client closes first; either side can.)

| State (`ss` name) | Side | Meaning | Normal? |
|-------------------|------|---------|---------|
| `LISTEN` | server | Waiting for connections | Yes |
| `SYN-SENT` | client | Sent SYN, waiting for a reply | Briefly. Many stuck = target not answering (firewall, full accept queue) |
| `SYN-RECV` | server | Got SYN, sent SYN-ACK, waiting for the ACK | Briefly. Thousands = SYN flood |
| `ESTAB` | both | Connected | Yes |
| `FIN-WAIT-1` | closer | Sent FIN, waiting for its ACK | Briefly |
| `FIN-WAIT-2` | closer | Our FIN is ACKed; waiting for the other side's FIN | Briefly. Stuck = the other app hasn't closed |
| **`CLOSE-WAIT`** | receiver of FIN | Peer closed; **the kernel is waiting for our application to call `close()`** | Briefly. **Piling up = a bug in this app** |
| `LAST-ACK` | receiver of FIN | We've sent our FIN, waiting for the final ACK | Briefly |
| **`TIME-WAIT`** | active closer | Closed; holding the 4-tuple for 60 s | Yes — but many of them mean no connection reuse |
| `CLOSING` | both | Both sent FIN at the same moment | Rare |

### `CLOSE-WAIT` is always your application's fault

That's question 2. The kernel handled the peer's FIN fine; it's waiting for
the program to close its socket. If the program never does — an `InputStream`
not closed, an HTTP response body not consumed or closed, a JDBC `Connection`
leaked out of a pool — the socket sits in `CLOSE-WAIT` **forever**, holding a
file descriptor. Eventually the process hits its file-descriptor limit and
starts failing with "Too many open files".

Watch it happen: a server that sends `bye` and closes, and a client that reads it
and never closes:

```bash
# Server closes first; client "forgets" to close. Look at both states.
python3 - <<'EOF' &
import socket, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(("127.0.0.1", 9100)); s.listen()
conn, _ = s.accept()
conn.sendall(b"bye\n")
conn.close()                      # server: polite close (sends FIN)
time.sleep(3)
EOF
sleep 0.5
python3 - <<'EOF' &
import socket, time
c = socket.create_connection(("127.0.0.1", 9100))
print("client read:", c.recv(100))
time.sleep(2)                     # the bug: no c.close()
EOF
sleep 1
ss -tan 'sport = :9100 or dport = :9100'
wait
```

```
State      Recv-Q Send-Q Local Address:Port  Peer Address:Port
LISTEN     0      128        127.0.0.1:9100       0.0.0.0:*
CLOSE-WAIT 0      0          127.0.0.1:60464    127.0.0.1:9100     ← the leaky client
FIN-WAIT-2 0      0          127.0.0.1:9100     127.0.0.1:60464    ← server, waiting for a FIN that never comes
```

The fix in Java is always the same: `try`-with-resources on anything that wraps
a socket — `Socket`, `InputStream`, `Connection`, `ResultSet`, HTTP response
objects.

> **Say this in an interview:** "`TIME_WAIT` is on the side that closed first
> and is normal; lots of it means connections aren't being reused. `CLOSE_WAIT`
> means the peer closed and our application never called `close()` — it's always
> a resource leak in the local app, like an unclosed stream or response body,
> and it ends in file-descriptor exhaustion."

---

## Part 6 — `TIME_WAIT`, and why it appears in load tests

**In plain words:** after a connection closes, the side that hung up first keeps
the line "reserved" for a minute before reusing the same number. It's a safety
measure — a delayed packet from the old call arriving on a new call with the same
numbers would corrupt it. Normally harmless. But a client that opens and closes
thousands of connections per second to the *same* server reserves thousands of
lines per second, and there are only about 28,000 lines to reserve.

### Why it exists

1. **Stray old segments.** A segment from the old connection, delayed in the
   network, could arrive after a *new* connection with the identical 4-tuple
   starts, and be mistaken for new data. Waiting lets old segments expire.
2. **A lost final ACK.** If the last ACK is lost, the other side resends its FIN;
   the closer must still be around to ACK it again rather than answer with a RST.

On Linux, `TIME_WAIT` lasts **60 seconds**, hard-coded in the kernel.
(`tcp_fin_timeout` — 60 on your machine — is a different timer, for
`FIN-WAIT-2`, despite what many blog posts claim.)

### Who gets it

**The active closer** — whichever side sent the first FIN. You can prove this on
your machine. 100 HTTP requests to a local server, three ways:

```
100 separate curls, HTTP/1.1 server:  client-side TIME-WAIT +100   server-side +0
1 curl, 100 URLs (connection reused): client-side TIME-WAIT +1     server-side +0
100 separate curls, HTTP/1.0 server:  client-side TIME-WAIT +2     server-side +99
```

- Each separate `curl` process closes its connection when it exits → client
  closes first → TIME-WAIT on the **client**.
- One `curl` with 100 URLs reuses one connection (HTTP keep-alive) → **one**
  TIME-WAIT in total.
- Python's `http.server` speaks HTTP/1.0 and closes after every response → the
  **server** is the active closer and collects the TIME-WAITs.

### Question 1: ephemeral port exhaustion

A load generator opening a new connection per request to `10.0.1.5:8080`:

- Usable client ports: 60999 − 32768 + 1 = **28,232**.
- Each closed connection holds its port in TIME-WAIT for **60 s**.
- So the sustained limit is about 28,232 / 60 ≈ **470 new connections per
  second** to that one destination. Push past it and `connect()` fails with
  **`EADDRNOTAVAIL` — "Cannot assign requested address"**: the client ran out of
  ports. The service is fine; the load tester is the bottleneck.

The same thing happens in production when a service makes a new connection per
outbound call to one downstream — or behind a NAT gateway, which has its own
port limit per destination.

**The fix is connection reuse** (HTTP keep-alive and pooling, Part 9) — not
kernel tuning. You'll see advice to set `tcp_tw_recycle`; it broke clients behind
NAT and was **removed from Linux in 4.12**. `tcp_tw_reuse` (yours is `2` =
loopback only) is safer, but it treats the symptom.

> **Say this in an interview:** "`TIME_WAIT` lasts 60 seconds on Linux on
> whichever side closed first; it stops delayed segments from an old connection
> polluting a new one with the same 4-tuple. In a load test without keep-alive,
> the client accumulates them until it runs out of ephemeral ports — about
> 28,000 per destination, so roughly 470 new connections a second — and
> `connect()` fails with 'cannot assign requested address'. The fix is
> connection reuse, not kernel tuning."

---

## Part 7 — Head-of-line blocking

**In plain words:** a supermarket queue with one till. If the person at the
front has a problem with their card, everyone behind waits — even though their
shopping is fine. TCP is a single queue of bytes delivered strictly in order, so
when one segment is lost, every byte behind it is held back until that one is
resent, even if the later bytes have already arrived.

It happens at two levels:

| Level | What blocks | Who fixed it |
|-------|-------------|--------------|
| **HTTP/1.1** | One connection carries one request at a time; a slow response blocks the requests queued behind it. Browsers open ~6 connections per host to work around it | **HTTP/2** multiplexes many requests as independent streams over **one** connection |
| **TCP** | One lost segment stalls *all* HTTP/2 streams on that connection, because TCP must deliver bytes in order — it doesn't know they belong to different streams | **HTTP/3** runs on **QUIC** over UDP: each stream is ordered independently, so a loss only stalls its own stream |

So HTTP/2 actually suffers *more* from packet loss than several HTTP/1.1
connections would: one loss freezes everything. That's the main reason HTTP/3
exists. Day 16 compares the HTTP versions properly.

> **Say this in an interview:** "Head-of-line blocking is when one stalled item
> blocks everything behind it. HTTP/2 fixed it at the HTTP layer with
> multiplexed streams, but TCP still delivers bytes strictly in order, so one lost
> segment stalls every stream on the connection. HTTP/3 moves to QUIC over UDP,
> where streams are independently ordered."

---

## Part 8 — UDP: no handshake, no promises

**In plain words:** UDP is a postcard. You write the address and drop it in the
box. No "hello" first, no confirmation it arrived, no guarantee postcards arrive
in the order you sent them. That sounds strictly worse — until you need speed,
or you'd rather lose a little data than wait for it.

| | TCP | UDP |
|---|-----|-----|
| Setup | 3-way handshake (1 RTT) | None — first packet carries data |
| Delivery | Reliable, retransmitted | Best effort — lost is lost |
| Order | Guaranteed | Not guaranteed |
| Unit | Byte stream (no message boundaries) | Datagrams (each delivered whole or not at all) |
| Header | 20–60 bytes | 8 bytes |
| Connection state | Yes — all of Part 5 | None |
| Flow / congestion control | Built in | Application's job |

Watch the "no promises" part. TCP to a closed port is refused instantly; UDP to
a closed port "succeeds":

```bash
# TCP to a closed port: refused. UDP to a closed port: silence, exit status 0.
nc -zv -w 2 127.0.0.1 9001;            echo "TCP exit status: $?"
echo hello | nc -u -w 1 127.0.0.1 9002; echo "UDP exit status: $?"
```

UDP sent the datagram and considers its job done. (The receiving kernel actually
replies with an ICMP "port unreachable", but the sender only notices that in
special cases. You can't rely on it.)

**Who wants that:**

- **DNS** — a query and its answer each fit in one packet. A handshake would
  double the cost. If the answer doesn't come, the resolver just asks again
  (Day 14). DNS falls back to TCP for large answers.
- **Video calls, voice, game state** — a frame that arrives late is useless.
  Better to skip it than freeze everything waiting for it (that's head-of-line
  blocking again).
- **QUIC / HTTP/3** — builds its own reliability, encryption and streams on top
  of UDP, so it can avoid TCP's head-of-line blocking and do the handshake and
  TLS together.
- **Metrics** (StatsD), **VPN tunnels** (WireGuard), **DHCP**, **NTP**.

> **Say this in an interview:** "UDP is connectionless: no handshake, no
> retransmission, no ordering, just datagrams with an 8-byte header. You choose
> it when latency matters more than completeness, like voice or video, when a
> request fits in one packet, like DNS, or when you want to build your own
> transport on top, like QUIC for HTTP/3."

---

## Part 9 — Connection pooling: why HikariCP and your HTTP client both have one

**In plain words:** opening a connection is like setting up a phone call through
a switchboard — dialling, waiting, introductions, checking ID — before you can say
anything. If you hang up after every sentence and redial for the next one, you
spend most of your time dialling. A pool keeps a few lines open and hands one to
whoever needs to talk next.

### What a new connection really costs

| Step | HTTPS to an API | JDBC to Postgres |
|------|-----------------|------------------|
| TCP handshake | 1 RTT | 1 RTT |
| TLS handshake | 1–2 RTT (Day 17) | 1–2 RTT if SSL is on |
| Authentication | — | Several round trips (SCRAM password exchange) |
| Server-side work | Allocate connection state | **Postgres forks a new backend process** per connection (megabytes of memory) |
| Slow start | Small `cwnd` again | Small `cwnd` again |
| On close | A TIME-WAIT socket | A TIME-WAIT socket |

Your `curl` above: 0.95 s of setup vs 0.26 s for the actual request. For a
database a few milliseconds away it's smaller per call but multiplied by every
query.

### What happens without pooling

- **Latency:** every call pays the setup cost.
- **Port exhaustion:** TIME-WAIT pile-up (Part 6).
- **Server overload:** Postgres's default `max_connections` is **100**. Twenty
  app instances each opening connections freely can hit it, and then *everyone*
  gets "sorry, too many clients already".

### What a pool does

Keeps N connections open; a request **borrows** one, uses it, **returns** it
(that's what `close()` on a pooled `Connection` actually does — returns it, not
closes it). Your Java demo in hands-on 9 shows it from the server's side:

```
A) new HttpClient per request
  10 requests arrived from client ports [56717, 56718, ..., 56726]
  -> 10 TCP connection(s)

B) one shared HttpClient
  10 requests arrived from client ports [56727, 56727, 56727, ..., 56727]
  -> 1 TCP connection(s)
```

Same port = same 4-tuple = same TCP connection, reused ten times.

### The pools you'll configure

| Pool | Defaults worth knowing | Failure message |
|------|------------------------|-----------------|
| **HikariCP** (Spring Boot's JDBC pool) | `maximumPoolSize` 10, `connectionTimeout` 30 s (wait to *borrow*), `idleTimeout` 10 min, `maxLifetime` 30 min | `HikariPool-1 - Connection is not available, request timed out after 30000ms` |
| **JDK `HttpClient`** | Pools automatically, per client instance — so create **one** and share it | — |
| **Apache HttpClient 5** (common under `RestTemplate`) | **5 connections per route** (host), 25 total | Requests queue waiting for a connection; latency climbs under load |
| **Reactor Netty** (under `WebClient`) | Its own pool, per `HttpClient` instance | `PoolAcquirePendingLimitException` / acquire timeouts |

### The ways pools fail

1. **Pool exhaustion.** All connections borrowed, and requests wait. The Hikari
   message above does **not** mean the database is down. It means 10
   connections were all busy for 30 s — slow queries, a transaction held open
   during an HTTP call, or a leak (a connection never returned). Raising the pool
   size usually just moves the queue into the database. Hikari's own guidance
   starts from about `(CPU cores × 2) + disks`, which is smaller than people
   expect.
2. **A pool per request.** `new HttpClient()` / `new RestTemplate()` inside a
   method, so every call gets an empty pool — hands-on 9's case A. In Spring,
   make it a bean.
3. **Stale connections and middlebox idle timeouts.** A pooled connection sits
   idle. Meanwhile a load balancer or NAT gateway on the path drops it from its
   tables after its own idle timeout (AWS ALB: 60 s default; AWS NAT Gateway:
   350 s). The pool still thinks it's open. The next request on it gets
   **`Connection reset by peer`**. The fix: set the pool's idle timeout / max
   lifetime **shorter** than every middlebox's idle timeout, and/or enable
   validation or keep-alives. (Linux's own TCP keepalive starts after 7,200 s —
   `tcp_keepalive_time` on your machine — far too late to help by default.)
4. **Leaks.** Borrowed and never returned — the same missing `close()` that
   produces `CLOSE-WAIT`. Hikari's `leakDetectionThreshold` logs where a
   connection was borrowed if it's held too long.

> **Say this in an interview:** "A new connection costs a TCP handshake, often a
> TLS handshake and authentication, server-side resources — a whole process in
> Postgres — and a TIME_WAIT socket afterwards. Pools amortise that by reusing
> connections. The classic failures are pool exhaustion from slow or leaked
> borrows, creating a client per request so the pool is never reused, and stale
> pooled connections being reset because a load balancer's idle timeout is
> shorter than the pool's."

---

## Hands-on

Blocks marked **lab** run with the Run button. **Ubuntu terminal** exercises need
several terminals or `sudo`.

### 1. Watch a connection live, by hand (Ubuntu terminal — required practice)

Open **three** Ubuntu (WSL) tabs.

```
# Tab 1 — the server
nc -l 9000

# Tab 3 — the observer. Run this before, during, and after each step:
watch -n 0.5 "ss -tan '( sport = :9000 or dport = :9000 )'"

# Tab 2 — the client
nc 127.0.0.1 9000
```

1. With only tab 1 running, tab 3 shows one `LISTEN`.
2. Connect from tab 2: two `ESTAB` lines appear — one per side. Find the client's
   ephemeral port. Is it in 32768–60999?
3. Type a line in tab 2 and press Enter. It appears in tab 1. Type back from tab
   1.
4. Press **Ctrl+C in tab 2** (the client closes first). What state does the
   client-side socket enter, and for how long does it stay? (Count.)
5. Repeat, but this time press Ctrl+C in **tab 1** first. Which side gets the
   `TIME-WAIT` now?

Then the scripted version, for the record:

```bash
# Listener, client, close — and the state at each step.
nc -l 9000 >/dev/null &
listener=$!
sleep 0.5
echo "--- listening";          ss -tan 'sport = :9000 or dport = :9000'
(sleep 2) | nc -N 127.0.0.1 9000 &
client=$!
sleep 0.5
echo "--- connected";          ss -tan 'sport = :9000 or dport = :9000'
wait "$client"
sleep 0.3
echo "--- client closed first"; ss -tan 'sport = :9000 or dport = :9000'
kill "$listener" 2>/dev/null || true   # it may already have exited
```

(`nc -N` closes the connection when its input ends — here, after 2 seconds.)

### 2. Capture the handshake and teardown (Ubuntu terminal, `sudo`)

```
sudo -v
sudo tcpdump -i lo -nn -c 10 'tcp port 9000 or tcp port 9001' &
sleep 1
nc -l 9000 >/dev/null &
sleep 0.3
printf 'hello\n' | nc -N 127.0.0.1 9000
nc -z 127.0.0.1 9001
```

`-i lo` because this traffic never leaves the machine (Day 11 gotcha). Match
your output to Parts 3 and 4: three handshake segments, one data segment and its
ACK, the FIN exchange, and finally a SYN to 9001 answered by `[R.]`. Then
answer:

1. Why is the data segment `seq 1:7` for `hello\n`?
2. After the client's FIN at `seq 7`, why does the server ACK `8`?
3. Is the teardown three segments or four in your capture? Why can it be either?

### 3. The accept queue (lab)

Run the Part 1 `listen(2)` block. Then change `srv.listen(2)` to `srv.listen(0)`
and predict how many clients get in. Then, in the printed `ss` output, point at
the line that would tell you, on a production box, that the app has stopped
accepting.

<details>
<summary>Answer</summary>

With `listen(0)`, Linux still allows one connection (backlog + 1), so client 0
gets in and 1–4 time out. The tell-tale line is the `LISTEN` row: `Recv-Q`
(currently queued) ≥ `Send-Q` (the limit). On a real server, run
`ss -lnt 'sport = :8080'` and watch that `Recv-Q` column.

</details>

### 4. A `CLOSE-WAIT` leak (lab)

Run the Part 5 block. Then fix the bug — add `c.close()` before `time.sleep(2)`
in the client — and run it again. What happens to both sockets?

<details>
<summary>Answer</summary>

Client closes → its FIN reaches the server, which was in `FIN-WAIT-2` waiting
for exactly that. The server side goes to `TIME-WAIT` (it was the active closer),
and the client's socket disappears (`LAST-ACK` → `CLOSED`, too quick to catch).
No `CLOSE-WAIT` remains.

</details>

### 5. TCP refused vs UDP silence (lab)

Run the Part 8 block. Then explain, in two sentences, why a health check that
"sends a UDP packet and checks the exit status" is worthless.

### 6. TIME_WAIT pile-up vs keep-alive (lab — required practice)

```bash
# 50 requests: new connection each time vs one reused connection.
tw() { ss -tan state time-wait "$@" | tail -n +2 | wc -l; }

# A local HTTP/1.1 server (keeps connections open between requests).
python3 -c "
import http.server as h
h.SimpleHTTPRequestHandler.protocol_version = 'HTTP/1.1'
h.ThreadingHTTPServer(('127.0.0.1', 8001), h.SimpleHTTPRequestHandler).serve_forever()" >/dev/null 2>&1 &
server=$!
sleep 1

c0=$(tw 'dport = :8001'); s0=$(tw 'sport = :8001')
for i in $(seq 50); do curl -s -o /dev/null http://127.0.0.1:8001/; done
echo "50 curls (new connection each):  client TIME-WAIT +$(( $(tw 'dport = :8001') - c0 )), server +$(( $(tw 'sport = :8001') - s0 ))"

c0=$(tw 'dport = :8001'); s0=$(tw 'sport = :8001')
args=(); for i in $(seq 50); do args+=(-o /dev/null http://127.0.0.1:8001/); done
curl -s "${args[@]}"
echo "1 curl, 50 URLs (reused):        client TIME-WAIT +$(( $(tw 'dport = :8001') - c0 )), server +$(( $(tw 'sport = :8001') - s0 ))"

kill "$server"
echo; echo "TIME-WAIT sockets on this machine right now: $(tw)"
```

(It runs 50 requests rather than the 100 in Part 6 so it stays inside the lab's 20-second limit — each `curl` process takes ~0.15 s to start in WSL.)

Run it twice in a row, within 60 seconds. Does the final total roughly double?
Wait over a minute and run just `ss -tan state time-wait | wc -l`. Where did they
go?

Then the arithmetic, on paper: your load tester opens 1,000 new connections per
second to one service, each closed by the client. How many seconds until it runs
out of ephemeral ports? What two changes would fix it?

<details>
<summary>Answer</summary>

Net consumption ≈ 1,000/s while each port is held 60 s, against 28,232 ports →
exhaustion after roughly **28 seconds**. Fixes: **reuse connections** (keep-alive
/ a shared pooled client in the load tester — the real fix), or spread load across
more destination IPs/ports or source IPs (each new 4-tuple combination gets its
own 28K). Widening `ip_local_port_range` or enabling `tcp_tw_reuse` helps at the
margin but treats the symptom.

</details>

### 7. TCP's internals on a live connection (lab)

```bash
# Start a slow HTTPS request in the background and inspect it mid-flight.
curl -s -o /dev/null https://httpbin.org/delay/3 &
sleep 1.5
ss -tin 'dport = :443' | head -4
wait
```

Find `rtt`, `rto`, `mss` and `cwnd` in your output. Compare `rtt` with the
`connect` time in exercise 8. They should be close — why?

### 8. What reuse saves (lab)

```bash
# Two requests in one curl: the second reuses the first's connection.
curl -s -o /dev/null -o /dev/null \
  -w 'connect %{time_connect}s  tls %{time_appconnect}s  total %{time_total}s  new-connections %{num_connects}\n' \
  https://httpbin.org/get https://httpbin.org/get
```

`time_connect` is when the TCP handshake finished; `time_appconnect` when TLS
finished. The second line's zeros and `new-connections 0` are the pool at work.
(The network to the US is noisy — run it a few times; the pattern holds even
when the numbers jump around.)

<details>
<summary>Why connect ≈ RTT (exercise 7)</summary>

`curl` considers the connection made when it has received the SYN-ACK — one
round trip after sending the SYN. So the TCP connect time *is* one RTT, plus a
little local overhead.

</details>

### 9. Pooling in Java, seen from the server (lab — required practice)

The server records which client port each request came from. Same port = same
TCP connection.

```java
import com.sun.net.httpserver.HttpServer;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Set;

public class PoolDemo {

    // Every request the server sees, recorded as the client's port number.
    static final List<Integer> clientPorts = new ArrayList<>();

    public static void main(String[] args) throws Exception {
        // A tiny HTTP/1.1 server on 127.0.0.1, port 0 = "kernel, pick any free port".
        HttpServer server = HttpServer.create(new InetSocketAddress("127.0.0.1", 0), 50);
        server.createContext("/", exchange -> {
            synchronized (clientPorts) {
                clientPorts.add(exchange.getRemoteAddress().getPort());
            }
            byte[] body = "ok".getBytes(StandardCharsets.UTF_8);
            exchange.sendResponseHeaders(200, body.length);   // known length -> connection can be reused
            try (OutputStream out = exchange.getResponseBody()) {
                out.write(body);
            }
        });
        server.start();
        URI url = URI.create("http://127.0.0.1:" + server.getAddress().getPort() + "/");
        System.out.println("server listening on " + server.getAddress());

        // A) The anti-pattern: a new client per request. Each client has its own,
        //    empty pool, so every request pays for a fresh TCP handshake.
        for (int i = 0; i < 10; i++) {
            try (HttpClient fresh = HttpClient.newHttpClient()) {
                fresh.send(HttpRequest.newBuilder(url).build(), HttpResponse.BodyHandlers.discarding());
            }
        }
        report("A) new HttpClient per request");

        // B) One shared client (what a Spring bean gives you). Its pool keeps the
        //    connection open after each response and hands it to the next request.
        try (HttpClient shared = HttpClient.newHttpClient()) {
            for (int i = 0; i < 10; i++) {
                shared.send(HttpRequest.newBuilder(url).build(), HttpResponse.BodyHandlers.discarding());
            }
        }
        report("B) one shared HttpClient");

        server.stop(0);
    }

    static void report(String label) {
        synchronized (clientPorts) {
            Set<Integer> distinct = new LinkedHashSet<>(clientPorts);
            System.out.printf("%n%s%n  10 requests arrived from client ports %s%n  -> %d TCP connection(s)%n",
                    label, clientPorts, distinct.size());
            clientPorts.clear();
        }
    }
}
```

Read the output, then answer:

1. In case A, the ports increase by one each time. Where do those numbers come
   from? (This runs on Windows — check them against the Windows ephemeral range
   in Part 1.)
2. Which side closed the connections in case A, and so which side is holding the
   `TIME-WAIT` sockets now?
3. **Break it:** in the handler, just before `sendResponseHeaders`, add
   `exchange.getResponseHeaders().set("Connection", "close");` and rerun. How
   many connections does case B use now, and why — even though the client is
   shared?
4. Remove that line again. Now change `sendResponseHeaders(200, body.length)` to
   `sendResponseHeaders(200, 0)` (0 means "length unknown — send it in
   chunks"). Predict first: does case B still reuse one connection? What does
   the client need to know about a response before it can send the next request
   on the same connection?

<details>
<summary>Answers</summary>

1. The Windows kernel picks the client's ephemeral port for each new connection,
   sequentially from its range (49152–65535). One new port = one new connection.
2. Each `HttpClient` is closed by `try`-with-resources, so the **client** closed
   first → the TIME-WAITs are on the client side (Windows), ten of them, for case
   A. Case B left at most one.
3. **10 connections** in case B too (verified: ten different ports). The server
   announces it will close after this response and then does, so the shared
   client's pool has nothing to reuse. A pool can only reuse what the *other
   side* keeps open — and the server is now the active closer, so the TIME-WAITs
   move to the server. Same effect as the HTTP/1.0 Python server in Part 6.
4. **Still 1 connection** (verified). With chunked encoding the body is sent in
   pieces, and a final empty chunk marks the end. What the client needs is to
   know **where this response ends**, so the next bytes on the connection can be
   read as the next response. `Content-Length` and chunked framing both give it
   that. When neither is available, the only way to mark the end is to close the
   connection — which is how HTTP/1.0 worked, and why it couldn't reuse
   connections. Day 16 covers this framing.

</details>

### 10. Inventory your machine (lab)

```bash
echo "Ephemeral port range:        $(cat /proc/sys/net/ipv4/ip_local_port_range)"
echo "SYN retries (connect hang):  $(cat /proc/sys/net/ipv4/tcp_syn_retries)"
echo "SYN cookies:                 $(cat /proc/sys/net/ipv4/tcp_syncookies)"
echo "Max accept backlog:          $(cat /proc/sys/net/core/somaxconn)"
echo "TCP keepalive starts after:  $(cat /proc/sys/net/ipv4/tcp_keepalive_time) s"
echo
echo "Sockets by state right now:"
ss -tan | awk 'NR > 1 { print $1 }' | sort | uniq -c | sort -rn
```

For each setting, say which Part it belongs to and what would go wrong if it
were much smaller.

---

## Gotchas

- **`connect()` succeeding doesn't mean the app accepted you.** The kernel
  completes handshakes into the accept queue on its own (Part 1).
- **A full accept queue looks like a firewall.** Dropped SYNs → client timeouts,
  not refusals. Check `Recv-Q` vs `Send-Q` on the `LISTEN` line before blaming
  the network.
- **`CLOSE-WAIT` is never the network's fault.** It's a missing `close()` in the
  process that owns the socket.
- **`TIME-WAIT` is on whoever closed first** — which may be your *server* if it
  closes connections (HTTP/1.0, `Connection: close`, or a server-side idle
  timeout).
- **`tcp_fin_timeout` does not control `TIME_WAIT`.** It's fixed at 60 s on Linux.
  And `tcp_tw_recycle` no longer exists — ignore any guide that sets it.
- **No connect timeout = up to ~127 s hang** on an unreachable host
  (`tcp_syn_retries` = 6). Set connect *and* read timeouts on every client.
- **One pool per request is no pool.** `new HttpClient()`, `new RestTemplate()`
  or `WebClient.create()` inside a method throws the pool away each call.
- **Pool idle timeout must be shorter than every middlebox's** (ALB 60 s, NAT
  Gateway 350 s), or you get intermittent "connection reset by peer".
- **UDP "success" means "sent".** Nothing more.
- **`ss` column meanings change on `LISTEN` lines** — `Recv-Q`/`Send-Q` are the
  accept queue's current length and limit, not byte counts.
- **WSL vs Windows.** Lab `bash` blocks run in WSL (Linux ranges, `ss`); `java`
  blocks run on Windows (`netstat -ano` there, Windows ephemeral range
  49152–65535). Don't look for the Java demo's sockets with `ss` in WSL.

---

## Recap

- A connection = **4-tuple**. Servers can hold millions; a client has ~28K ports
  **per destination**.
- **`listen`**: the kernel handshakes into an **accept queue**; the app
  `accept()`s. Full queue → dropped SYNs → client **timeouts**.
- **TCP** = sequence numbers + ACKs + retransmission (reliable, ordered) +
  receive window (**flow control**) + `cwnd` (**congestion control**, slow start).
- **Handshake** SYN / SYN-ACK / ACK = 1 RTT before data; TLS adds more. Unanswered
  SYNs retry for ~127 s. **SYN floods** fill the half-open queue; **SYN cookies**
  defend.
- **FIN** = graceful, per direction. **RST** = abort: "refused", "reset by peer".
- **States:** `LISTEN`, `ESTAB`, `TIME-WAIT` (closer, 60 s, normal), `CLOSE-WAIT`
  (**your app's leak**).
- **TIME_WAIT + no reuse** → ephemeral port exhaustion → "Cannot assign
  requested address" at ~470 new connections/s per destination.
- **Head-of-line blocking:** HTTP/2 fixed it for HTTP, TCP still has it; QUIC/HTTP/3
  removes it.
- **UDP:** no handshake, no order, no retries; DNS, media, QUIC.
- **Pooling** amortises handshakes, TLS, auth and server resources. Failures:
  exhaustion, pool-per-request, stale connections vs middlebox idle timeouts,
  leaks.

---

## Say it out loud

1. Walk through the three-way handshake and the four-segment teardown, naming
   each state each side passes through.
2. A server shows thousands of `TIME_WAIT` sockets; another shows thousands of
   `CLOSE_WAIT`. What does each mean, and which one is a bug?
3. Your load test fails with "Cannot assign requested address". Explain the
   cause with numbers, and the fix.
4. Why do HikariCP and every HTTP client keep a connection pool? What breaks if
   you create a new client per request?
5. What's head-of-line blocking, and how do HTTP/2 and HTTP/3 each deal with it?
6. When would you choose UDP over TCP? Give three real examples.

---

## Q&A

_Follow-up questions from reading this lesson go here._
