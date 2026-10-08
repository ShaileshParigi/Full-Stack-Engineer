# Day 15 — Diagnostics Toolkit + Week 3 Review

_Why this matters:_ "The service is unreachable" is the most common message in
any on-call channel, and the most common mistake is to start guessing — restart
the pod, blame the network team, redeploy. Every one of Days 11–14 gave you a
layer and a way it fails. Today turns them into a **reflex**: a fixed order of
checks, one tool per layer, where each tool's output tells you whether to stop
or go down one more step. You'll build that order into a script, then break
your own machine one layer at a time and check the script names the right
culprit each time. It's also exactly what an interviewer means by "walk me
through how you'd debug a service that's unreachable".

> **Review day.** Parts 1–4 are the tools (some you've already met — they get a
> proper treatment here). Part 5 is the decision tree. The required practice is
> hands-on 6 and 7. The Week 3 recap is at the end; ask for the Week 3 quiz when
> you're ready.

---

## The one-paragraph version

Debugging connectivity is a ladder you climb from the bottom: **does the name
resolve → is there a route → does the port accept a connection → does TLS
succeed → what does the application answer?** Each rung has a tool
(`getent`/`dig`, `ip route get`, `nc -zv`, `openssl s_client`, `curl -v`) and
each failure has a signature you've already seen: "could not resolve", "no
route to host", **instant refusal vs slow timeout**, a certificate error, an
HTTP status. Stop at the first rung that fails — everything above it is
untested, not broken. Around that ladder sit the observation tools: `ping` and
`mtr` show whether packets reach a host and where they're lost (with big
caveats: many hosts ignore ping, and routers deprioritise replies), and
`ss`/`lsof` show what's listening on *your* machine and which process owns it.
Two traps run through all of it: a firewall can make a closed port look either
"refused" or "timed out", and DNS can be wrong rather than missing — which shows
up three rungs later as a timeout.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **ICMP** | The network's own messaging system: "unreachable", "too big", "expired", "are you there?" | Internet Control Message Protocol — L3, carried in IP, no ports |
| **echo request / reply** | "Are you there?" / "Yes" | ICMP types 8 and 0 — what `ping` sends and expects |
| **time exceeded** | "Your packet's TTL ran out at me" | ICMP type 11 — how traceroute discovers each hop (Day 11) |
| **destination unreachable** | "I can't deliver this" | ICMP type 3 — includes "port unreachable" (UDP, Day 13) and "fragmentation needed" (MTU, Day 11) |
| **rate limiting (ICMP)** | Routers answering only some pings so they don't get overloaded | Control-plane policing; produces "loss" at a hop that isn't real loss |
| **`mtr`** | `ping` and `traceroute` combined, repeated | Sends probes to every hop continuously and reports loss and latency per hop |
| **socket owner** | Which program has a port open | The process (PID) holding the socket's file descriptor |
| **PID** | A process's number | Process ID (Day 4) |
| **`lsof`** | "List open files" — including network sockets | Shows which process has which file or socket open |
| **`ss` / `netstat`** | List sockets and their states | `ss` (iproute2) is current; `netstat` (net-tools) is the old one |
| **`nc`** | A raw TCP/UDP client and server | netcat: connect, listen, or test a port (`-z`) |
| **`telnet`** | An old remote-login tool, now used to type a protocol by hand | An interactive TCP client |
| **`openssl s_client`** | A raw TLS client | Performs a TLS handshake and prints the certificate and result |
| **unreachable route** | A routing-table entry that says "you can't get there" | `ip route add unreachable …` — connect fails with "No route to host" |
| **DROP vs REJECT** | A firewall silently binning a packet vs answering "no" | DROP → the sender times out; REJECT → the sender gets an error (RST or ICMP) immediately |
| **decision tree** | A fixed order of yes/no checks | Here: DNS → route → TCP → TLS → HTTP |

---

## Before you read: three questions

1. `mtr` to a server shows 66% packet loss at hop 7, and 0% at every hop after
   it, including the destination. Is there packet loss on this path?
2. Spring Boot refuses to start: `Web server failed to start. Port 8080 was
   already in use.` Find the process holding it — in WSL, and on Windows.
3. You test a port and get `Connection refused`. A colleague says "so the
   service isn't running". What else could produce exactly that message?

---

## Part 1 — `ping` and ICMP

**In plain words:** `ping` asks a machine "are you there?" using the network's
own control messages (ICMP), not TCP or UDP, so no port is involved. A reply
proves that the machine is up and that packets can get there and back. No reply
proves very little — lots of machines and firewalls are told to ignore ping on
purpose.

```bash
# Three targets: one that answers, and two that are fine but ignore ping.
ping -c 3 -W 1 1.1.1.1      | tail -2
ping -c 2 -W 1 httpbin.org  | tail -2
gw=$(ip route | awk '/^default/ {print $3; exit}')
ping -c 2 -W 1 "$gw"        | tail -2
```

On your machine:

```
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 15.129/16.927/19.545/1.893 ms        ← 1.1.1.1: answers, ~17 ms away
2 packets transmitted, 0 received, 100% packet loss         ← httpbin.org: AWS blocks ICMP; HTTPS works fine
2 packets transmitted, 0 received, 100% packet loss         ← your own gateway: Windows firewall drops ping
```

Two of the three "fail", and both are perfectly healthy. Reading a successful
reply line:

```
64 bytes from 1.1.1.1: icmp_seq=1 ttl=56 time=15.2 ms
```

| Field | Meaning |
|-------|---------|
| `icmp_seq` | Sequence number. Gaps = lost packets; out-of-order = reordering |
| `ttl=56` | The TTL left on the **reply** when it arrived. Senders usually start at 64, 128 or 255, so 56 suggests it started at 64 and crossed 8 routers — matching the 8 hops before `1.1.1.1` in the `mtr` report below |
| `time` | Round-trip time. Compare with Day 13's `ss -ti` RTT to a US host (~245 ms) |

ICMP is more than ping. You've already relied on three other ICMP messages:
*time exceeded* (tracepath, Day 11), *fragmentation needed* (the PPPoE MTU
discovery, Day 11) and *port unreachable* (UDP to a closed port, Day 13).
**Blocking all ICMP breaks the MTU one** — and that's the "small requests work,
large ones hang" bug from Day 11. Firewalls should drop echo requests if they
like, but let "fragmentation needed" through.

> **Say this in an interview:** "Ping uses ICMP echo, so it tests L3
> reachability only — no ports, no application. A reply proves the host is up and
> routable; no reply proves little, because most cloud hosts and firewalls drop
> echo requests. And you shouldn't block all ICMP, because path MTU discovery
> depends on the 'fragmentation needed' message."

---

## Part 2 — `traceroute`, `tracepath` and `mtr`: reading hops

**In plain words:** these tools show the routers between you and a destination,
using the TTL trick from Day 11: send a packet that "dies" after 1 hop, then 2,
then 3, and each router that kills one reports back. `mtr` does this over and
over and keeps score, so you see loss and latency **per hop**. The skill is
reading it: a router being slow to *answer you* is not the same as a router
being slow to *forward your traffic*.

`traceroute` isn't installed in your WSL; `tracepath` and `mtr` are, and neither
needs root in the form used here.

```bash
# mtr in report mode: 3 probes to every hop, numeric addresses, wide output.
timeout 20 mtr -4 -n -r -c 3 -w 1.1.1.1
```

Your run:

```
HOST: Shailesh       Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 172.24.160.1    0.0%     3    0.6   0.6   0.6   0.6   0.0    ← Windows (WSL's gateway)
  2.|-- 192.168.1.1     0.0%     3    4.8   6.8   3.7  12.1   4.6    ← home router (Wi-Fi hop)
  3.|-- 103.57.86.10    0.0%     3    6.0   6.7   5.5   8.5   1.6    ← ISP — also its DNS resolver (Day 14)
  4.|-- 103.57.86.1     0.0%     3    7.5   7.2   6.2   7.9   0.9
  5.|-- 182.76.243.41   0.0%     3   12.2  11.9   6.6  16.9   5.2
  6.|-- 116.119.50.87   0.0%     3   22.2  22.4  19.6  25.4   2.9
  7.|-- 182.79.161.171 66.7%     3   18.4  18.4  18.4  18.4   0.0    ← "loss"
  8.|-- 162.158.52.39   0.0%     3   19.5  18.7  17.2  19.5   1.3    ← Cloudflare's network
  9.|-- 1.1.1.1         0.0%     3   15.5  15.5  15.1  16.0   0.5
```

| Column | Meaning |
|--------|---------|
| `Loss%` | Probes to **this hop** that got no reply |
| `Snt` | Probes sent |
| `Last / Avg / Best / Wrst` | Round-trip ms to this hop |
| `StDev` | Jitter — how much the RTT varies |

### How to read it — the rules that matter

1. **Loss that doesn't continue isn't loss.** Hop 7 shows 66.7%, but hops 8 and
   9 — which your packets reach *through* hop 7 — show 0%. So hop 7 forwards
   traffic fine; it just doesn't bother answering most probes *addressed to
   itself*. Routers handle "reply to a probe" on a slow path and rate-limit it.
   That's question 1: **no**, there's no loss on this path.
2. **Real loss starts at a hop and continues all the way to the destination.**
   If hop 6 showed 30% and every later hop also showed ~30%, the problem is at or
   just before hop 6.
3. **The same goes for latency.** A slow intermediate hop followed by fast hops
   is the router being slow to reply, not slow to forward. A jump that persists to
   the end is real — often geography (crossing an ocean adds ~100 ms).
4. **The last hop is the one that counts.** That's the path your traffic
   actually experiences.
5. **The return path may be different.** Each hop's reply comes back by its own
   route. `tracepath` printed `asymm` on Day 11 for exactly this reason.

**Why ICMP-based traces can lie about TCP:** a firewall may treat ICMP probes and
your real TCP traffic differently. `mtr -T -P 443 host` probes with TCP SYNs to
port 443 instead — closer to what your app does — but it needs `sudo`.

> **Say this in an interview:** "Traceroute and mtr use TTL expiry to reveal each
> hop. When reading mtr, loss or latency at an intermediate hop that doesn't
> persist to the destination is just that router rate-limiting replies to probes;
> real loss starts at a hop and carries through to the end. For TCP-specific
> problems I'd trace with TCP SYN probes to the actual port."

---

## Part 3 — `ss`, `netstat` and `lsof`: what's on *my* machine

**In plain words:** the first two parts look outward at the network. These tools
look at your own machine: which ports are open, what state each connection is
in, and — most usefully — **which program** owns a port. That's how you answer
"something is already using port 8080, but what?".

### `ss` — sockets (you've used it since Day 11)

| Flag | Meaning |
|------|---------|
| `-t` / `-u` | TCP / UDP |
| `-l` | Listening sockets only |
| `-a` | All (listening and connected) |
| `-n` | Numbers, don't resolve names/ports (faster, and no DNS — important when DNS is what's broken) |
| `-p` | Show the owning process (other users' processes need `sudo`) |
| `-i` | TCP internals: rtt, cwnd (Day 13) |
| `-s` | Summary counts |
| `state time-wait`, `'sport = :8080'`, `'dport = :443'` | Filters |

The daily one: **`ss -ltnp`** — "what is listening on TCP, numerically, and who
owns it".

### `netstat` — the old name for the same thing

You'll see it in older docs and on Windows. `netstat -tlnp` ≈ `ss -ltnp`. On
Linux it's installed in your WSL (`net-tools`) but deprecated; prefer `ss`.

### `lsof` — which process has this port

```bash
# Occupy port 8000, then find out who did it — two ways.
python3 -m http.server 8000 --bind 127.0.0.1 >/dev/null 2>&1 &
pid=$!
sleep 1
echo "== ss -ltnp";   ss -ltnp 'sport = :8000'
echo "== lsof";       lsof -nP -iTCP:8000 -sTCP:LISTEN
echo "== the started PID was $pid"
kill "$pid"
```

`lsof -nP -iTCP:8000 -sTCP:LISTEN`: `-n`/`-P` no name or port lookups, `-i` network
files, filtered to TCP port 8000 in state LISTEN. Its output gives the command,
PID and user directly. `fuser 8000/tcp` is a shorter way to get just the PID.

### Question 2: "Port 8080 was already in use"

The port can be held by a process **in WSL or on Windows** — and WSL forwards
`localhost` ports to Windows (Day 11), so they collide. Check both:

```
ss -ltnp 'sport = :8080'          # WSL side
lsof -nP -iTCP:8080 -sTCP:LISTEN
```

```powershell
# Windows side: who owns local port 8080?
Get-NetTCPConnection -LocalPort 8080 -State Listen |
  ForEach-Object { Get-Process -Id $_.OwningProcess } |
  Select-Object Id, ProcessName, Path
# The classic equivalent:  netstat -ano | findstr :8080   (last column = PID)
```

Usually the answer is a previous run of your own app (an IDE run configuration
that never stopped) or Docker publishing the same port.

> **Say this in an interview:** "On Linux, `ss -ltnp` lists listening TCP sockets
> with their owning processes, and `lsof -i :port` maps a port to a PID; on
> Windows it's `Get-NetTCPConnection` or `netstat -ano`. I use `-n` to avoid name
> lookups, especially when DNS might be the problem."

---

## Part 4 — `nc`, `telnet` and `openssl s_client`: testing one layer at a time

**In plain words:** `curl` tests everything at once, so when it fails you don't
know which part failed. These tools each test one layer: `nc` only tests "will
this port accept a TCP connection?", `openssl s_client` adds "does TLS work?",
and `telnet`/`nc` let you then type the application protocol by hand.

### `nc` — TCP and nothing else

| Command | Use |
|---------|-----|
| `nc -zv -w 3 host 5432` | Is the port open? `-z` connect and close, `-v` say what happened, `-w 3` give up after 3 s |
| `nc -l 9000` | Listen (Day 13) |
| `nc host 80` | Connect and type/pipe a protocol (Day 11) |
| `nc -u host 53` | UDP — remember "success" only means "sent" |

The three outcomes are the three signatures from Day 11, now from a tool that
doesn't involve HTTP at all:

```
Connection to 1.1.1.1 443 port [tcp/https] succeeded!       ← L1–L4 fine
nc: connect to 127.0.0.1 port 9 (tcp) failed: Connection refused     ← a RST came back
nc: connect to 10.255.255.1 port 80 (tcp) timed out: Operation now in progress   ← nothing came back
```

Every database and broker you'll run in Docker can be checked this way before
you blame your Spring config: `nc -zv localhost 5432` (Postgres), `6379`
(Redis), `9092` (Kafka).

### `telnet` — typing a protocol interactively

`telnet host port` opens a TCP connection and lets you type. It's useful for
line-based protocols (HTTP, SMTP, Redis) when `nc` isn't installed — common in
minimal containers and on Windows. **Don't pipe into it:** telnet does its own
newline translation, and when I piped an HTTP request into it, httpbin answered
`400 Bad Request`. Typed by hand, it works. For scripts, use `nc`. (Hands-on 5.)

### `openssl s_client` — TCP *plus* TLS

```bash
openssl s_client -connect httpbin.org:443 -servername httpbin.org -brief </dev/null 2>&1 | head -8
```

```
CONNECTION ESTABLISHED
Protocol version: TLSv1.2
Ciphersuite: ECDHE-RSA-AES128-GCM-SHA256
Peer certificate: CN = httpbin.org
Verification: OK
```

`-servername` sends the hostname in the handshake (SNI — Day 17 explains it), and
`</dev/null` makes it exit after the handshake instead of waiting for input.
Day 17 goes deep; today it's the TLS rung of the ladder. One trap, found while
building this lesson: **`s_client` doesn't check that the certificate matches
the hostname unless you add `-verify_hostname`.** Against
`wrong.host.badssl.com` — a certificate issued for a different name — it printed
`Verification: OK`. With `-verify_hostname wrong.host.badssl.com` it fails with
`hostname mismatch`. Code that "verifies the certificate but not the hostname"
is a real security bug class; Day 17 shows the Java version.

> **Say this in an interview:** "I isolate layers: `nc -zv` for whether the port
> accepts TCP, `openssl s_client` for the TLS handshake and certificate — with
> hostname verification switched on — and then `curl -v` for the HTTP exchange.
> Telnet or nc let me type line-based protocols by hand when nothing else is
> installed."

---

## Part 5 — The decision tree: DNS, route, port, TLS, or the app?

**In plain words:** check in the same order every time, from the bottom up, and
stop at the first thing that fails. Everything above a failure is *untested*,
not broken, so don't look at it yet. The order matters because each step needs
the one below it: no name, no address; no route, no packets; no TCP, no TLS; no
TLS, no HTTP.

```
"service unreachable"
        │
  1. NAME    getent hosts NAME            fails ──▶ DNS: dig NAME; dig @8.8.8.8 NAME; /etc/hosts?  (Day 14)
        │    answer looks wrong? ─────────────────▶ compare with dig @8.8.8.8 — /etc/hosts or split horizon
        ▼
  2. ROUTE   ip route get IP              fails ──▶ "No route to host": routing table / VPC route table (Day 12)
        ▼
  3. PORT    nc -zv -w 3 IP PORT
        │      refused  ─────────────────────────▶ host reachable; nothing listening, wrong port, bound to
        │                                          127.0.0.1 (Day 12) — or a firewall REJECT
        │      timeout  ─────────────────────────▶ dropped: firewall / security group, host down, or wrong IP
        ▼
  4. TLS     openssl s_client … -verify_hostname NAME
        │      fails    ─────────────────────────▶ expired / self-signed / wrong-name certificate (Day 17)
        ▼
  5. APP     curl -v URL
               4xx/5xx  ─────────────────────────▶ the network is fine — application, auth, upstream (Day 16)
```

The signatures, collected — what each layer's failure looks like from every tool
you'll use:

| Layer | `curl` | `nc` / shell | Java (`java.net.Socket`) | Java (`HttpClient`) |
|-------|--------|--------------|--------------------------|---------------------|
| DNS | `(6) Could not resolve host` | `getent` prints nothing | `UnknownHostException` | `ConnectException` ← `UnresolvedAddressException` |
| Route | `(7) Couldn't connect` | `No route to host` | `NoRouteToHostException` (Linux; not measured) | `ConnectException` (not measured) |
| Port, nothing listening | `(7) … Connection refused` (instant) | `Connection refused` | `ConnectException: Connection refused` | `ConnectException` ← `ClosedChannelException` |
| Port, dropped | `(28) Connection timed out` | `timed out` | `SocketTimeoutException: Connect timed out` | `HttpConnectTimeoutException` |
| TLS | `(60) SSL certificate problem…` | `s_client`: `verify error` | — | `SSLHandshakeException` ← `CertificateExpiredException` |
| App | status code (`-f` → exit 22) | — | — | `HttpResponse.statusCode()` |

The Java columns are measured on your JDK 21 (hands-on 8), except the route
row: the route break only exists inside WSL, and the lab's Java runs on
Windows. Notice how much the
modern `HttpClient` hides: a DNS failure and a refused connection are **both**
`ConnectException`, distinguishable only by the root cause at the bottom of the
stack trace. Always read to the last `Caused by:`.

### Three traps the tree has to handle

1. **A firewall REJECT looks like "nothing listening".** On your machine, an
   `iptables … -j REJECT --reject-with tcp-reset` rule made `nc` to `1.1.1.1:443`
   print `Connection refused` — and 1.1.1.1 definitely listens on 443. With
   `-j DROP` the same test **timed out**. So: timeout almost always means a
   firewall or wrong IP; refused *usually* means nothing listening, but not
   always. That's question 3. (AWS security groups only drop; they never
   reject.)
2. **A wrong DNS answer doesn't fail at DNS.** With `10.255.255.1 httpbin.org`
   in `/etc/hosts`, the name step "succeeds" and the failure appears at the port
   step as a timeout. That's why the script below compares the app's answer with
   public DNS.
3. **"Network is fine" needs proof.** `curl` reports HTTP code `000` when there
   was no HTTP response at all. My first version of the script treated `000` as an
   application error. Every tool only answers the exact question you ask it.

> **Say this in an interview:** "I go bottom-up and stop at the first failure:
> resolve the name the way the app does, check the route, test the port with
> `nc`, check TLS with `openssl s_client`, and only then look at the HTTP
> response. Refused versus timeout is the key split at the port step — refused
> means something answered, timeout means packets were dropped — with the caveat
> that a REJECT rule can fake a refusal, and a wrong DNS answer shows up later as a
> timeout."

---

## Hands-on

Blocks marked **lab** run with the Run button. Exercises 5 and 7 need your
**Ubuntu (WSL)** terminal; 4 is PowerShell.

### 1. Ping three ways (lab)

Run the Part 1 block. Then answer: which of the three hosts are actually up?
How would you prove the two silent ones are healthy without `ping`?

<details>
<summary>Answer</summary>

All three are up. Prove the silent ones with a TCP test against a port they
serve: `nc -zv -w 3 httpbin.org 443` (succeeds), and for your gateway, the fact
that every request you make leaves through it. "Does the thing I care about
work?" beats "does it answer ping?".

</details>

### 2. Read an `mtr` report (lab)

Run the Part 2 block. Your numbers will differ. For your run, answer: which hop
is the first one outside your home? Is any loss real? Where does the latency
jump, and does the jump persist to the end? Then run it against `httpbin.org`
with `-m 12` added (stop after 12 hops — otherwise mtr keeps probing silent AWS hops until the lab's timeout kills it): `timeout 20 mtr -4 -n -r -c 3 -w -m 12 httpbin.org`. What happens to the last few hops, and
why does that *not* mean httpbin is unreachable?

<details>
<summary>Answer for httpbin</summary>

The report ends at hop 7 with `???` and 100% loss, and httpbin never
appears: from there on nothing answers the probes (AWS's network doesn't, and
httpbin drops ICMP), so `mtr` stops listing silent hops. httpbin is fine — Part 1 already showed it ignores ping. An ICMP trace can
only tell you where ICMP stops being answered. The latency jump of ~200 ms at
the hop where traffic leaves India for the US *does* persist; that one is real
(it's the speed of light across an ocean). On 8 October it was hop 6:
`116.119.44.134  … Avg 219.9`, after five hops under 8 ms.

</details>

### 3. Who owns this port? (lab)

Run the Part 3 block. Confirm `ss -p`, `lsof` and the `$!` PID all agree. Then
change `--bind 127.0.0.1` to `--bind 0.0.0.0` and compare the Local Address
column. Which one could another machine connect to?

### 4. The Windows side of "port already in use" (PowerShell)

Start any server on Windows port 8080 — for example, run your Day 13 `PoolDemo`
with the port changed from `0` to `8080` and a `Thread.sleep(60_000)` before
`server.stop(0)` — then, while it runs, use the PowerShell commands from Part 3
to find its PID and process name. Which process name do you see for a Java
program? What would you kill?

### 5. Type HTTP by hand with telnet (Ubuntu terminal)

```
telnet httpbin.org 80
```

When it says `Connected`, type these lines exactly, then press Enter **twice**
(the blank line ends the request):

```
GET /get HTTP/1.1
Host: httpbin.org
Connection: close
```

You get a full response. Then try the same thing with `nc httpbin.org 80`. Then
try `telnet localhost 6379` (nothing there) and compare the error to `nc -zv
localhost 6379`.

### 6. Build the decision tree as a script (lab — required practice)

This is the Part 5 tree as a bash function, applied to five targets that each
fail at a different layer (and one healthy one). Read the function first — every
check is a tool from Parts 3–4.

```bash
# diagnose URL — walk the layers bottom-up and stop at the first one that fails.
diagnose() {
  local url=$1
  local scheme=${url%%://*} rest=${url#*://}
  local hostport=${rest%%/*}
  local host=${hostport%%:*} port
  if [[ $hostport == *:* ]]; then port=${hostport##*:}
  elif [[ $scheme == https ]]; then port=443
  else port=80; fi
  echo "== $url"

  # 1. Names: what the app would get (getent uses /etc/hosts, then DNS)
  local ip
  if [[ $host =~ ^[0-9.]+$ ]]; then ip=$host
  else ip=$(getent ahostsv4 "$host" | awk 'NR == 1 { print $1 }'); fi
  if [[ -z $ip ]]; then
    echo "  1 DNS    FAIL  '$host' does not resolve   -> next: dig $host ; dig @8.8.8.8 $host"; return 1
  fi
  echo "  1 DNS    ok    $host -> $ip"
  # A wrong answer is worse than no answer. Compare with what public DNS says.
  if [[ ! $host =~ ^[0-9.]+$ ]]; then
    local public
    public=$(dig +short +time=2 +tries=1 @8.8.8.8 "$host" A | grep -E '^[0-9.]+$')
    if [[ -n $public ]] && ! grep -qxF "$ip" <<<"$public"; then
      echo "           WARN  public DNS says $(head -1 <<<"$public"), not $ip -> check /etc/hosts (or split-horizon DNS)"
    fi
  fi

  # 2. Routing: does the kernel know a way there?
  local route
  if ! route=$(ip route get "$ip" 2>&1); then
    echo "  2 ROUTE  FAIL  $route   -> next: ip route"; return 1
  fi
  echo "  2 ROUTE  ok    $(head -1 <<<"$route" | sed 's/ uid .*//')"

  # 3. Transport: will anything accept a TCP connection on that port?
  local tcp
  tcp=$(nc -zv -w 3 "$ip" "$port" 2>&1)
  case $tcp in
    *succeeded*)    echo "  3 TCP    ok    port $port accepts connections" ;;
    *refused*)      echo "  3 TCP    FAIL  refused: host answered, nothing listening on $port (or a firewall REJECT)"; return 1 ;;
    *"timed out"*)  echo "  3 TCP    FAIL  timeout: packets silently dropped - firewall/security group, or host down"; return 1 ;;
    *)              echo "  3 TCP    FAIL  $tcp"; return 1 ;;
  esac

  # 4. TLS (https only): does the handshake succeed and the certificate verify?
  if [[ $scheme == https ]]; then
    local tls
    if ! tls=$(openssl s_client -connect "$ip:$port" -servername "$host" -verify_hostname "$host" -verify_return_error -brief </dev/null 2>&1); then
      echo "  4 TLS    FAIL  $(grep -m1 -iE 'verify error|alert|error' <<<"$tls")"; return 1
    fi
    echo "  4 TLS    ok    $(grep -m1 'Protocol version' <<<"$tls")"
  fi

  # 5. Application: what does the server say?
  local code
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 8 "$url")
  local rc=$?
  if [[ $code == 000 ]]; then
    echo "  5 HTTP   FAIL  no HTTP response at all (curl exit $rc) -> next: curl -v $url"; return 1
  elif [[ $code == 2* || $code == 3* ]]; then
    echo "  5 HTTP   ok    $code"
  else
    echo "  5 HTTP   FAIL  HTTP $code - the network is fine; look at the application"; return 1
  fi
}

for url in https://httpbin.org/get \
           https://no-such-host.invalid/ \
           http://127.0.0.1:9/ \
           http://10.255.255.1/ \
           https://self-signed.badssl.com/ \
           https://httpbin.org/status/503; do
  diagnose "$url" || true      # a failing target is the point here, not an error
done
```

Your run should show each target stopping at a different rung: healthy (all 5
ok), DNS, TCP refused, TCP timeout, TLS (`self-signed certificate`), HTTP 503.

Then:

1. Add `https://expired.badssl.com/` to the list. Which rung, and what's the
   message?
2. Why does step 1 use `getent` rather than `dig`? (Day 14.)
3. Why does step 3 use `nc` to the **IP** rather than the name?
4. Save the function (everything above the `for` loop) as `~/diagnose.sh` in
   your Ubuntu terminal. You'll `source` it in exercise 7.

<details>
<summary>Answers 2 and 3</summary>

2. `getent` resolves the way applications do — `/etc/hosts` first, then DNS —
   so it finds what `curl` or Java would actually connect to. `dig` would skip
   `/etc/hosts` and diagnose a different question.
3. So step 3 tests exactly the address step 1 found. If `nc` resolved the name
   again, it could get a different IP (8 A records for httpbin) and the steps
   would no longer be testing the same thing.

</details>

### 7. Break each layer on purpose (Ubuntu terminal, `sudo` — required practice)

Each block breaks exactly one layer, runs `diagnose`, and **puts it back**. Run
them one at a time and check, before reading the output, which rung you expect
to fail.

```
source ~/diagnose.sh
sudo -v

# A. DNS gives a WRONG answer (not a missing one)
sudo cp /etc/hosts /etc/hosts.bak
echo "10.255.255.1 httpbin.org" | sudo tee -a /etc/hosts
diagnose https://httpbin.org/get
sudo cp /etc/hosts.bak /etc/hosts

# B. No route to the destination
sudo ip route add unreachable 1.1.1.1/32
diagnose https://1.1.1.1/
sudo ip route del unreachable 1.1.1.1/32

# C. Firewall DROPs the port
sudo iptables -A OUTPUT -d 1.1.1.1 -p tcp --dport 443 -j DROP
diagnose https://1.1.1.1/
sudo iptables -D OUTPUT -d 1.1.1.1 -p tcp --dport 443 -j DROP

# D. Firewall REJECTs the port
sudo iptables -A OUTPUT -d 1.1.1.1 -p tcp --dport 443 -j REJECT --reject-with tcp-reset
diagnose https://1.1.1.1/
sudo iptables -D OUTPUT -d 1.1.1.1 -p tcp --dport 443 -j REJECT --reject-with tcp-reset

# E. The service stops listening
python3 -m http.server 8000 --bind 127.0.0.1 >/dev/null 2>&1 &
diagnose http://127.0.0.1:8000/
kill %1
diagnose http://127.0.0.1:8000/

# Prove everything is back to normal
sudo iptables -S OUTPUT                 # just: -P OUTPUT ACCEPT
ip route | grep unreachable             # nothing
diagnose https://1.1.1.1/
```

(The `iptables` rules affect **only** traffic to `1.1.1.1:443` and only until
you delete them or WSL restarts.)

What I got when testing these:

| Break | Rung that failed | Message |
|-------|------------------|---------|
| A — wrong `/etc/hosts` | **3 TCP** (timeout) — with a `WARN public DNS says 18.233.182.23, not 10.255.255.1` at step 1 | The real cause is DNS, but it fails later |
| B — unreachable route | 2 ROUTE | `RTNETLINK answers: No route to host` |
| C — DROP | 3 TCP | timeout (3 s) |
| D — REJECT | 3 TCP | **refused** — although 1.1.1.1 *is* listening |
| E — stopped listener | 5 HTTP ok, then 3 TCP refused | |

Write one sentence for each: what a colleague who only ran `curl` would have
concluded, and what the script told you instead.

### 8. The same layers, from Java (lab)

The exceptions your Spring service will throw for each layer, from both Java
APIs. Runs on Windows; takes about 12 seconds (two 3-second timeouts).

```java
import java.net.InetSocketAddress;
import java.net.Socket;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;

public class WhichLayer {

    record Target(String layer, String host, int port, String url) {}

    public static void main(String[] args) {
        Target[] targets = {
            new Target("DNS: no such name",      "no-such-host.invalid", 443, "https://no-such-host.invalid/"),
            new Target("L4: nothing listening",  "127.0.0.1",              9, "http://127.0.0.1:9/"),
            new Target("L3/firewall: dropped",   "10.255.255.1",          80, "http://10.255.255.1/"),
            new Target("TLS: expired cert",      "expired.badssl.com",   443, "https://expired.badssl.com/"),
            new Target("L7: app says 503",       "httpbin.org",          443, "https://httpbin.org/status/503"),
            new Target("healthy",                "httpbin.org",          443, "https://httpbin.org/status/200"),
        };

        System.out.println("== 1. Plain java.net.Socket: DNS + TCP connect only");
        for (Target t : targets) {
            long start = System.nanoTime();
            String result;
            try (Socket s = new Socket()) {
                s.connect(new InetSocketAddress(t.host(), t.port()), 3_000);   // resolves, then handshakes
                result = "connected (TCP is fine; TLS and HTTP not tested)";
            } catch (Exception e) {
                result = e.getClass().getName() + ": " + e.getMessage();
            }
            print(t.layer(), start, result);
        }

        System.out.println("\n== 2. java.net.http.HttpClient: the whole request");
        try (HttpClient client = HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(3)).build()) {
            for (Target t : targets) {
                long start = System.nanoTime();
                String result;
                try {
                    HttpResponse<Void> r = client.send(
                            HttpRequest.newBuilder(URI.create(t.url())).timeout(Duration.ofSeconds(8)).build(),
                            HttpResponse.BodyHandlers.discarding());
                    result = "HTTP " + r.statusCode();
                } catch (Exception e) {
                    Throwable root = e;
                    while (root.getCause() != null) root = root.getCause();   // the real reason is at the bottom
                    result = e.getClass().getSimpleName()
                            + (root == e ? ": " + e.getMessage() : "  <- root cause " + root.getClass().getName());
                }
                print(t.layer(), start, result);
            }
        }
    }

    static void print(String layer, long start, String result) {
        System.out.printf("  %-23s %5d ms  %s%n", layer, (System.nanoTime() - start) / 1_000_000, result);
    }
}
```

Your run:

```
== 1. Plain java.net.Socket: DNS + TCP connect only
  DNS: no such name         103 ms  java.net.UnknownHostException: no-such-host.invalid
  L4: nothing listening       2 ms  java.net.ConnectException: Connection refused: getsockopt
  L3/firewall: dropped     3014 ms  java.net.SocketTimeoutException: Connect timed out
  TLS: expired cert         492 ms  connected (TCP is fine; TLS and HTTP not tested)
  ...
== 2. java.net.http.HttpClient: the whole request
  DNS: no such name         112 ms  ConnectException  <- root cause java.nio.channels.UnresolvedAddressException
  L4: nothing listening      48 ms  ConnectException  <- root cause java.nio.channels.ClosedChannelException
  L3/firewall: dropped     3011 ms  HttpConnectTimeoutException  <- root cause java.net.ConnectException
  TLS: expired cert        2499 ms  SSLHandshakeException  <- root cause java.security.cert.CertificateExpiredException
  L7: app says 503         1664 ms  HTTP 503
```

Two things to take from it:

- The **timings** carry as much information as the exception names: 2 ms =
  refused, 3,000 ms = your connect timeout fired.
- With `HttpClient`, the top-level exception for DNS and for "refused" is the
  **same** class. In a log, scroll to the last `Caused by:`. If your service
  catches `ConnectException` and logs only `e.getMessage()`, you've thrown the
  layer away.

---

## Gotchas

- **No ping reply ≠ down.** Your gateway and httpbin both ignore ping and both
  work.
- **mtr loss at a middle hop that doesn't persist is not loss.** Read the last
  hop.
- **REJECT rules fake "connection refused".** And a wrong IP (from DNS or a
  config) looks like a firewall timeout.
- **`openssl s_client` doesn't check the hostname by default.** Add
  `-verify_hostname`. Its `Verification: OK` against a wrong-name certificate is
  the same bug as Java code that disables hostname verification.
- **Don't pipe into `telnet`** — use `nc` in scripts.
- **`ss -p` and `lsof` only show your own processes** without `sudo`. A port
  "in use" with no visible owner is probably root's (or another user's).
- **Use `-n` everywhere when DNS might be broken** — otherwise `ss`, `netstat`,
  `mtr` and `lsof` hang trying to resolve names for their output.
- **WSL and Windows share `localhost` ports.** "Already in use" may be the other
  side.
- **`wrong.host.badssl.com` was flaky from your network** while testing (its TCP
  connect timed out 2 times out of 3, on the same IP that worked for the other
  badssl hosts) — which is why the lab blocks use `self-signed` and `expired`
  instead.
- **The script's "WARN" can false-alarm** for names with geo-DNS or many
  rotating IPs (big CDNs return different sets to different resolvers) and for
  split-horizon names. It's a hint, not a verdict.

---

## Recap

- **Bottom-up, stop at the first failure:** name → route → port → TLS → app.
  Above a failure is untested, not broken.
- **`ping`/ICMP:** L3 only; silence proves little; don't block all ICMP (MTU).
- **`mtr`/`tracepath`:** per-hop loss and latency; only loss/latency that
  persists to the destination is real.
- **`ss -ltnp`, `lsof -i`, `Get-NetTCPConnection`:** what's listening and who
  owns it.
- **`nc -zv`** for TCP, **`openssl s_client -verify_hostname`** for TLS,
  **`curl -v`** for HTTP, **`telnet`** for typing protocols interactively.
- **Signatures:** resolve error → DNS; no route → routing; instant refused →
  nothing listening (or REJECT); timeout → dropped; cert error → TLS; status code
  → app.
- **Traps:** wrong DNS answers fail later; REJECT looks like refused; Java's
  `HttpClient` hides the layer in the root cause.

---

## Week 3 recap: addresses → transport → names

| Day | The idea | The command you'll reach for | The production failure it explains |
|-----|----------|------------------------------|------------------------------------|
| 11 — Layers | Each layer reads only its header; IPs end to end, MACs per hop | `ip addr`, `ip route`, `ip neigh`, `tcpdump -e` | "Is it the network?" → which layer; MTU black holes |
| 12 — Addressing | `/N` splits network and host; private ranges + NAT | `ip route get`, Python `ipaddress`, your `SubnetCalc` | VPC/EKS subnet exhaustion, overlapping CIDRs, public vs private subnets |
| 13 — Transport | Handshake, teardown, states, pooling, UDP | `ss -tan`, `ss -ti`, `ss -lnt` (Recv-Q) | TIME_WAIT port exhaustion, CLOSE_WAIT leaks, full accept queues, pool exhaustion |
| 14 — Names | Delegation, records, TTL and caches | `dig`, `dig +trace`, `getent hosts` | Stale DNS after a migration, JVM caching, ndots, DNS outages |
| 15 — Diagnostics | A fixed order of checks | `diagnose`, `mtr`, `nc -zv`, `openssl s_client` | "Service unreachable" — found in minutes, not hours |

The thread through all five: **a request is a name, turned into an address,
reached by a route, connected on a port, then spoken to in a protocol.** Every
outage you'll debug breaks exactly one of those, and now you have a tool for each.

**End of Week 3 — the quiz is available.** Ask for it when you're ready
("Phase 2 Week 3 quiz"); it'll be saved as `day-15-week-3-quiz.md`.

---

## Say it out loud

1. Walk me through how you'd debug "the payments service is unreachable" — in
   order, with the tool for each step.
2. `nc` says "connection refused". What does that prove, and what *doesn't* it
   prove?
3. How do you read an `mtr` report? When is loss at a hop real?
4. Port 8080 is "already in use". How do you find the culprit on Linux and on
   Windows?
5. Why can a bad `/etc/hosts` entry show up as a TCP timeout instead of a DNS
   error?
6. Your Java service logs `ConnectException`. How do you tell whether it was DNS
   or a refused connection?

---

## Q&A

_Follow-up questions from reading this lesson go here._
