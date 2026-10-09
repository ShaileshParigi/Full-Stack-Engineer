# Day 19 — Ports, Firewalls & NAT

_Why this matters:_ "Is it listening, and who is allowed to reach it?" is two
questions, and mixing them up causes the most common deployment failures there
are. A service bound to `127.0.0.1` inside a Docker container is unreachable from
anywhere — "connection refused", though the app is running. A database bound to
`0.0.0.0` on a machine with a public IP is open to the internet — and gets found
within hours. A security group rule that allows traffic in but forgets the way
back is a bug on one kind of firewall and not on another. Today you learn the two
layers of control — **where a program listens** and **what the network lets
through** — and then use them on your own machine, where one of them behaves
differently from what you'd expect.

> **Heads-up: the plan's practice exercise gives a surprising result.** The plan
> says "bind a service to `127.0.0.1` and confirm it's unreachable from Windows".
> On your machine it is **reachable** — WSL forwards Windows' `localhost` into
> WSL. Part 1 shows what actually happens, and why that matters.

> **Long day.** If it runs past two hours: Parts 1–4 and hands-on 1–5 today; Parts
> 5–7 and hands-on 6–9 next session. The required practice is hands-on 1, 4 and 5.

---

## The one-paragraph version

A program that wants network traffic **binds** to an address and a port. The
address decides *which doors it stands behind*: `127.0.0.1` is only the machine
itself, a specific interface address is only that network, and `0.0.0.0` means
every door it has — including any facing the internet. Binding is the first layer
of access control, and it's the cheapest. The second layer is a **firewall**: a
list of rules the kernel checks for every packet, top to bottom, first match wins,
with a **default policy** for anything unmatched. Linux's firewall is `iptables`
(Ubuntu's `ufw` is a friendlier front-end for it). A firewall is either
**stateful** — it remembers connections, so allowing a request automatically
allows its reply — or **stateless**, where you must write rules for both
directions. AWS Security Groups are stateful; NACLs are stateless. A rule can
**DROP** a packet silently (the sender times out) or **REJECT** it with an error
(the sender gets "refused"). The same kernel machinery can also **rewrite**
packets — NAT — which is how a port can be forwarded to another port or machine,
and how Kubernetes Services work. Under WSL2 there's a third piece: Windows
forwards its own `localhost` into WSL, which changes who can reach what.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **bind** | A program claiming an address and port to listen on | The `bind()` system call (Day 13) |
| **bind address** | Which network doors the program stands behind | `127.0.0.1` loopback, a specific interface IP, or `0.0.0.0` (all IPv4 interfaces) |
| **`0.0.0.0`** | As a bind address: "every interface" | The wildcard; `::` is the IPv6 equivalent |
| **exposure** | How far away someone can be and still reach the service | Determined by bind address **and** firewall rules **and** NAT |
| **`EADDRINUSE`** | The error when the port is already taken | "Address already in use" |
| **firewall** | Rules deciding which packets are allowed | In Linux, the kernel's netfilter, configured with `iptables` or `nft` |
| **`iptables`** | The classic tool for netfilter rules | Legacy name; on your Ubuntu it runs on the `nf_tables` backend |
| **`ufw`** | "Uncomplicated Firewall": simple commands that write iptables rules | Ubuntu's front-end; `ufw allow 8080/tcp` |
| **table** | A group of rules for one purpose | `filter` (allow/deny), `nat` (rewrite addresses), `mangle` (edit headers) |
| **chain** | An ordered list of rules at one point in a packet's journey | `INPUT` (to this machine), `OUTPUT` (from it), `FORWARD` (through it), `PREROUTING`, `POSTROUTING` |
| **rule** | One condition plus an action | "TCP, destination port 22 → ACCEPT" |
| **target** | What to do with a matching packet | `ACCEPT`, `DROP`, `REJECT`, or jump to another chain |
| **policy** | The default action when no rule matches | `iptables -P INPUT DROP`; ufw: "default deny incoming" |
| **DROP** | Discard silently | Sender sees nothing and times out |
| **REJECT** | Discard and tell the sender | Sends a TCP RST or ICMP unreachable; sender fails fast |
| **stateless filter** | Judges each packet on its own | Needs rules for requests **and** replies |
| **stateful filter** | Remembers connections | Connection tracking (`conntrack`): packets of an allowed connection are allowed automatically |
| **`ESTABLISHED,RELATED`** | "Part of a connection we already allowed" | The conntrack states that stateful rules match on |
| **Security Group** | AWS's per-instance firewall | Stateful, allow-rules only |
| **NACL** | AWS's per-subnet firewall | Stateless, ordered allow **and** deny rules |
| **NetworkPolicy** | Kubernetes' pod-level firewall | Selects pods and lists allowed traffic; needs a CNI plugin that enforces it |
| **NAT** | Rewriting source or destination addresses | SNAT/masquerade on the way out, DNAT/redirect on the way in (Day 12) |
| **port forwarding** | Passing traffic arriving at one port on to another port or machine | A DNAT rule, a proxy program, or an SSH tunnel |
| **SSH tunnel** | Carrying a TCP port through an SSH connection | `ssh -L local:remote-host:remote-port server` |
| **bastion / jump host** | A hardened machine you SSH to in order to reach private ones | Often the only thing with a public IP |
| **localhost forwarding** | WSL making Windows' `localhost:port` reach a WSL service | A relay running on Windows (on by default) |

---

## Before you read: three questions

1. You start a Spring Boot app in a Docker container. `curl localhost:8080`
   works *inside* the container, but the port you published (`-p 8080:8080`) gives
   "connection reset". What's the most likely cause?
2. A security group allows inbound TCP 443 to your load balancer. A colleague asks
   "do I need an outbound rule so the responses can get back?" What do you say?
   What about a NACL?
3. You run a server in WSL bound to `127.0.0.1`. Can a program on Windows reach it?
   Can your phone on the same Wi-Fi?

---

## Part 1 — Binding: `127.0.0.1` vs `0.0.0.0` vs one interface

**In plain words:** a machine usually has several "doors" — the loopback door that
only leads to itself, and one or more doors to real networks (your WSL has `lo`
and `eth0`; your Windows has Wi-Fi and the WSL adapter). When a program binds, it
decides which doors it stands behind. Stand behind the loopback door only, and
only programs on the same machine can reach you. Stand behind `0.0.0.0` and anyone
who can reach *any* of the machine's doors can knock.

| Bind to | Reachable from | Typical use |
|---------|----------------|-------------|
| `127.0.0.1` | Programs on **this machine** only | Dev servers, databases that only the local app uses, sidecars |
| A specific IP, e.g. `172.24.174.221` | Only via **that interface's** network | A service meant for one network (internal vs external) |
| `0.0.0.0` | Every interface the machine has | Servers meant for others; **containers** |

Spring Boot: `server.address=127.0.0.1` restricts it; the default is all
interfaces. PostgreSQL's `listen_addresses` and Redis's `bind` do the same job —
and "I bound the database to `0.0.0.0` to make it work" is how data ends up on
the internet.

### The measured matrix

I started three servers in your WSL — bound to `127.0.0.1`, `0.0.0.0` and your
`eth0` address — and tried to reach each from four places:

```
                     bound to       bound to       bound to
reached from         127.0.0.1      0.0.0.0        172.24.174.221 (eth0)
-----------------    -----------    -----------    ----------------------
WSL  → 127.0.0.1         200            200             FAIL
WSL  → 172.24.174.221    FAIL           200             200
Windows → localhost      200  (!)       200             FAIL
Windows → 172.24.174.221 FAIL           200             200
Windows Wi-Fi IP (192.168.1.23) — what other devices would use:  FAIL (tested on the 0.0.0.0 server)
```

Read it by columns:

- **`127.0.0.1`** — works from WSL's loopback, fails to WSL's `eth0` address (a
  different door), and **works from Windows' `localhost`** — the surprising one.
- **`0.0.0.0`** — every door works... except your phone's route in, the Windows
  Wi-Fi address (Part 7).
- **`eth0` address** — the mirror image of the first column: not via loopback,
  not via Windows' `localhost`.

**Why Windows reaches the loopback-bound server.** WSL runs a *relay* (this is inferred from the measurements below, not read from its source): when a
program in WSL starts listening, the relay opens the same port on Windows'
`127.0.0.1` and, for each connection, makes a new connection **from inside WSL to
WSL's `localhost`**. On Windows, `Get-NetTCPConnection` showed ports 8101 and
8102 listening on `127.0.0.1` owned by one process — the relay. The consequence
runs in both directions:

- Convenient: you can develop in WSL and open `http://localhost:8080` in a Windows
  browser.
- Not isolation: "bound to `127.0.0.1`" in WSL does **not** mean "invisible to
  Windows". It means "invisible to anything that isn't a program on this
  machine *or* Windows relaying for it".
- The relay binds Windows' **loopback** only, so other devices on your Wi-Fi can
  *not* use it. That's why the Wi-Fi-IP test failed.

### Containers: why `127.0.0.1` breaks them

Inside a container, `127.0.0.1` is the **container's own** loopback. A published
port (`-p 8080:8080`) forwards traffic arriving on the host to the container's
`eth0` address — where nothing is listening if the app bound to `127.0.0.1`. The
app must bind `0.0.0.0` inside the container (Spring does by default; question 1).
Control exposure at the *publish* instead: `-p 127.0.0.1:8080:8080` publishes on
the host's loopback only, while the default `-p 8080:8080` publishes on all host
interfaces.

> **Say this in an interview:** "Bind address is the first access control: loopback
> is host-local, a specific IP limits it to one network, and `0.0.0.0` listens
> everywhere. A containerised server must bind `0.0.0.0` inside, and you restrict
> exposure with the publish address or network policy. A database on `0.0.0.0` with
> a public IP and no firewall is a classic data leak."

---

## Part 2 — Port conflicts, and finding the offender

**In plain words:** two programs can't claim the same address and port. The second
one is refused with "Address already in use". The fix is finding who has it — Day
15's tools — and the trap is that the culprit may be on the other side of WSL.

```bash
# Occupy port 8300, try to take it again, then find the owner two ways.
python3 -m http.server 8300 --bind 127.0.0.1 >/dev/null 2>&1 &
holder=$!
sleep 1
echo "== second bind:"
python3 -m http.server 8300 --bind 127.0.0.1 2>&1 | tail -1
echo "== who has it:"
ss -ltnp 'sport = :8300' | tail -1
lsof -nP -iTCP:8300 -sTCP:LISTEN | tail -1
kill "$holder"
```

Same error from Java (on Windows, it came back as):

```
java.net.BindException: Address already in use: bind
```

| Where | Message |
|-------|---------|
| Python | `OSError: [Errno 98] Address already in use` |
| Java | `java.net.BindException: Address already in use` |
| Spring Boot | `Web server failed to start. Port 8080 was already in use.` |
| Node | `Error: listen EADDRINUSE: address already in use :::8080` |

Things that make it confusing:

- **The owner may be your own previous run** — an IDE session that didn't stop.
- **WSL and Windows share `localhost` ports** (Part 1): a Windows program on 8080
  blocks the relay for a WSL server on 8080, and vice versa.
- **`TIME_WAIT` can block a restart** (Day 13): a server restarted immediately may
  fail to rebind while old connections linger. That's why servers set
  `SO_REUSEADDR` (Java's `ServerSocket` does by default on Linux; Python's
  `http.server` sets it too).
- **Windows reserves port ranges.** On your machine, `netsh interface ipv4 show
  excludedportrange protocol=tcp` lists ranges like `49854–49953` and
  `50060–50159` (and `50000–50059`, marked `*` as administratively reserved).
  Binding inside one fails with a *permission* error that looks nothing like
  "in use" — Hyper-V and WSL reserve dynamic ranges, and they change between
  reboots. Pick ports below 49000, as Spring's 8080 and Postgres's 5432 are.

> **Say this in an interview:** "Address-in-use means another socket holds that
> address and port. I find the owner with `ss -ltnp` or `lsof -i :port` on Linux
> and `Get-NetTCPConnection` on Windows, remembering WSL shares localhost ports with
> Windows. On Windows, a permission error on a high port often means it falls in a
> reserved exclusion range."

---

## Part 3 — Firewalls: `iptables` and `ufw`, conceptually

**In plain words:** a firewall is a checklist the kernel runs for every packet. Go
down the list from the top; the first rule that matches decides — allow it, drop
it, or reject it. If nothing matches, the **default policy** decides. Everything
else is detail about *where* in the packet's journey the list is read.

### Where the checks happen

```
                      ┌───────────┐
 packet arrives ────▶ │PREROUTING │ (nat: rewrite destination — DNAT)
                      └─────┬─────┘
                  for me?   │      not for me (routing / forwarding)
                 ┌──────────┴──────────┐
            ┌────▼────┐           ┌────▼─────┐
            │  INPUT  │           │ FORWARD  │
            └────┬────┘           └────┬─────┘
          local process            ┌───▼───────┐
                 │                 │POSTROUTING│ (nat: rewrite source — SNAT/MASQUERADE)
            ┌────▼────┐            └───┬───────┘
            │ OUTPUT  │ ───────────────┘
            └─────────┘      (packets the machine itself sends)
```

| Chain | Sees | Example |
|-------|------|---------|
| `INPUT` | Packets **to** this machine | "Allow SSH from my subnet" |
| `OUTPUT` | Packets **from** this machine | "Don't let this box talk to the internet except on 443" |
| `FORWARD` | Packets passing **through** (routers, Docker hosts, WSL's Windows side) | Docker and Kubernetes manage rules here |
| `PREROUTING` / `POSTROUTING` | Before/after routing — where NAT happens | Port forwarding, masquerade |

A rule: `iptables -A INPUT -p tcp --dport 8104 -j ACCEPT` = "append to INPUT:
protocol TCP, destination port 8104, jump to ACCEPT". Targets are `ACCEPT`,
`DROP`, `REJECT`, or a jump to another chain.

On your WSL, with the firewall off, the state is:

```
-P INPUT ACCEPT       ← policy: accept everything not matched (and there are no rules)
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
```

**Order matters:** `-A` appends; `-I` inserts at the top. A broad `DROP` placed
above a specific `ACCEPT` makes the `ACCEPT` unreachable.

### `ufw`: the friendly front-end

```
sudo ufw default deny incoming       # the policy: nothing in unless allowed
sudo ufw default allow outgoing
sudo ufw allow 8104/tcp              # open one port
sudo ufw deny 8102/tcp               # explicitly drop one
sudo ufw reject 8102/tcp             # explicitly reject one
sudo ufw enable
sudo ufw status verbose
```

`ufw` is installed on your WSL (version 0.36.1) but **inactive**. Enabling it with
the rules above produced this (real output):

```
Status: active
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
8104/tcp                   ALLOW IN    Anywhere
8102/tcp                   DENY IN     Anywhere
```

### What I measured: three surprises

I tested from **Windows**, because from inside WSL the result is misleading:

1. **Inside WSL, `nc` to your own address ignores the rule.** With port 8102
   denied, `nc -zv 172.24.174.221 8102` *succeeded* — and so did `127.0.0.1`.
   Traffic to the machine's own IP travels over the loopback interface, and ufw's
   rules begin with `-A ufw-before-input -i lo -j ACCEPT`: **loopback is always
   allowed.** The plan's exercise ("block a port with ufw and verify with `nc`")
   only works if the `nc` comes from *another machine* — here, Windows.
2. **From Windows to the WSL IP, the rules worked** (PowerShell, `Test-NetConnection`):
   ```
   Windows -> WSL IP :8102 -> False     (denied)
   Windows -> WSL IP :8104 -> True      (allowed)
   Windows -> WSL IP :8105 -> False     (nothing listening, default deny)
   ```
3. **But Windows `localhost:8102` still returned 200 while "denied".** The relay
   from Part 1 connects *from inside WSL via loopback*, which ufw always allows. So
   **ufw cannot block access that arrives through the localhost relay.** To stop
   that, don't bind the server to loopback/`0.0.0.0` in the first place, or turn
   off localhost forwarding.

### DROP vs REJECT, timed from Windows

| Rule | What the sender experiences |
|------|------------------------------|
| `ALLOW` (port 8104) | Connected in ~1 ms |
| `DENY` = drop (and default deny, port 8105) | **Timeout** — still no answer at my 5,000 ms cap; a real client keeps waiting until its own connect timeout (Day 13) |
| `REJECT` (port 8102) | **Refused** after ~2,040 ms (two runs: 2,048 and 2,036). The cause is probably Windows retransmitting the SYN before accepting the reset — I didn't verify that — but the measurement is real: "fast failure" costs ~2 s from Windows. A Linux client fails instantly (Day 15's `nc` test) |

That's Day 11/15's diagnostic split from the firewall's side: **a firewall that
drops makes the failure look like a dead host; one that rejects makes it look like
a closed port.** AWS security groups only drop. If something that "should work"
times out, suspect a firewall; if it's refused, suspect nothing listening — or a
reject rule (Day 15).

> **Say this in an interview:** "Netfilter evaluates chains — INPUT, OUTPUT,
> FORWARD, plus PREROUTING and POSTROUTING for NAT — top to bottom, first match
> wins, with a default policy for the rest. `ufw` is a front-end that sets default
> deny incoming and adds allow rules. DROP makes the sender time out; REJECT makes
> it fail immediately. And loopback traffic is always accepted, so you must test
> firewall rules from another host."

---

## Part 4 — Stateful vs stateless filtering

**In plain words:** when you allow "incoming connections on port 443", you also
need the server's *replies* to get back out — and the *client's* later packets to
get in. A **stateless** firewall has no memory: every packet is judged alone, so
you must write a rule for each direction. A **stateful** firewall keeps a table of
connections it has seen start; once a connection is allowed, everything that
belongs to it is allowed automatically. Stateful is far easier to get right, and
it's what you want by default.

### The experiment: turn a stateful firewall stateless

ufw's `ufw-before-input` chain (real output from your WSL, trimmed) begins:

```
-A ufw-before-input -i lo -j ACCEPT
-A ufw-before-input -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT   ← the "stateful" rule
-A ufw-before-input -m conntrack --ctstate INVALID -j DROP
...
```

That second line is the whole of statefulness: "any packet belonging to a
connection we already track → accept". With ufw on and **default deny incoming**,
this line is what lets the replies to *your own* outgoing requests back in.

I deleted that one rule and ran an ordinary outbound request:

```
with the rule:       curl https://httpbin.org  →  200 in 1.1 s
rule removed:        curl https://httpbin.org  →  000 after 10 s (curl exit 28)
```

Nothing is blocking *outgoing* traffic; the firewall is silently dropping the
**replies**, because with default-deny-incoming, a packet coming in from
`httpbin.org:443` matches no rule. Then it got more interesting. I tried the
stateless fix — allow replies by source port:

```
iptables -I ufw-before-input -p tcp --sport 443 -j ACCEPT
→ curl still failed         (it couldn't even resolve the name)
iptables -I ufw-before-input -p udp --sport 53  -j ACCEPT
→ curl: 200 in 1.3 s
```

The first rule wasn't enough because the **DNS replies** (UDP, source port 53)
were being dropped too — `dig` timed out. A stateless firewall needs a reply rule
for **every protocol** you use: HTTPS replies, DNS replies, NTP, package
mirrors... and since a client's side of a connection uses an **ephemeral port**
(Day 13: 32768–60999 on Linux, 49152–65535 on Windows, 1024–65535 in the NACL
world), the standard stateless pattern is "allow inbound TCP to the *ephemeral
range*". (I restored the rule and checked `curl` worked before moving on.)

### Where this lands in AWS and Kubernetes

| | Security Group | NACL | Kubernetes NetworkPolicy |
|---|----------------|------|--------------------------|
| Attached to | Instance / network interface | **Subnet** | Pods (by label) |
| Stateful? | **Yes** | **No** | Yes (enforced by the CNI) |
| Rule types | **Allow only** | Allow **and** deny | Allow only |
| Evaluation | All rules together | Rules in **number order**; first match wins | Union of all policies selecting the pod |
| Default | Deny all inbound, allow all outbound | Default NACL allows all | **Allow all** until a policy selects the pod |
| Reference to other groups | Yes (`source = sg-xxxx`) | No — CIDRs only | Yes (pod/namespace selectors) |

Question 2: a Security Group is stateful, so **no outbound rule is needed** for
replies to allowed inbound requests (and the default allows all outbound anyway).
A **NACL is stateless**: you need an inbound rule for 443 *and* an outbound rule
for the ephemeral port range back to the client — forget it and you get exactly
the symptom above, everything "allowed" but nothing works. Phase 13 builds both.

Kubernetes' version: pods start with no isolation. The moment a NetworkPolicy
selects a pod, everything not explicitly allowed is denied for that pod — and it
only has an effect if your CNI plugin enforces policies (Phase 15).

> **Say this in an interview:** "A stateful firewall tracks connections, so
> allowing the request implicitly allows the reply; a stateless one judges each
> packet alone, so you must allow both directions — including the client's
> ephemeral port range. Security Groups are stateful, allow-only, instance-level;
> NACLs are stateless, ordered, allow-and-deny, subnet-level. A NetworkPolicy
> starts as allow-all and flips to default-deny for any pod it selects."

---

## Part 5 — NAT and port redirection

**In plain words:** the same kernel that decides whether to allow a packet can also
*rewrite* it. Rewrite the destination as it arrives ("anything for port 9999 should
really go to port 8104") and you've done port forwarding. Rewrite the source as it
leaves and you have the home-router NAT from Day 12. These happen in the `nat`
table, in `PREROUTING` and `POSTROUTING`.

A working miniature (as root; the lab can't run it, so use your Ubuntu terminal).
Port 9999 has nothing on it — then a redirect rule sends it to a server on 8104:

```
python3 -m http.server 8104 --bind 0.0.0.0 >/dev/null 2>&1 &

curl -s -o /dev/null -w 'before: %{http_code}\n' --max-time 2 http://127.0.0.1:9999/   # 000 — refused
sudo iptables -t nat -A OUTPUT -p tcp -d 127.0.0.1 --dport 9999 -j REDIRECT --to-ports 8104
curl -s -o /dev/null -w 'after:  %{http_code}\n' --max-time 2 http://127.0.0.1:9999/   # 200
sudo iptables -t nat -D OUTPUT -p tcp -d 127.0.0.1 --dport 9999 -j REDIRECT --to-ports 8104
kill %1
```

On your machine: `before: 000`, `after: 200`, then `000` again after deleting the
rule. The server on 8104 never changed; the kernel rewrote the destination port in
flight.

This is, in small, how **Kubernetes Services** work: a Service has a stable
virtual IP that no machine owns. `kube-proxy` writes `DNAT` rules so that packets to
that virtual IP:port are rewritten to one of the real pod IPs — chosen at
random per connection, using the same netfilter you just used (Phase 15). And
Docker's `-p 8080:80` is a DNAT rule from the host's port 8080 to the container's
port 80.

**MASQUERADE** is the outbound twin: `iptables -t nat -A POSTROUTING -s 172.17.0.0/16
-j MASQUERADE` rewrites the source address of container traffic to the host's, so
replies find their way back — the same trick as your Windows host NAT-ing WSL
(Day 12).

> **Say this in an interview:** "NAT rewrites addresses in the `nat` table: DNAT in
> PREROUTING rewrites destinations, which is how Docker port publishing and
> Kubernetes Services forward to backends; SNAT or masquerade in POSTROUTING
> rewrites sources so replies return. Connection tracking remembers the mapping so
> the reply is un-rewritten."

---

## Part 6 — Port forwarding and SSH tunnels

**In plain words:** sometimes the service you need is on a private network you
can't reach directly — a database in a private subnet, an admin UI on a server
with no public port. If you can SSH to *some* machine that *can* reach it, you can
ask SSH to carry a port: connections to a port on your laptop come out the other
end of the SSH connection and go to the private service. It's port forwarding done
inside an encrypted connection.

Two ways to build the same thing, in miniature:

### A program that forwards (lab-runnable)

A forwarder is nothing exotic — it accepts a connection, opens one to the real
target, and copies bytes both ways:

```bash
# Forward 127.0.0.1:8110 to a "private" web server on 127.0.0.1:8104. Like ssh -L, minus the encryption.
python3 -m http.server 8104 --bind 127.0.0.1 >/dev/null 2>&1 &
backend=$!

python3 - <<'EOF' &
import socket, threading

def pipe(src, dst):
    try:
        while data := src.recv(4096):
            dst.sendall(data)
    except OSError:
        pass
    finally:
        for s in (src, dst):
            try: s.shutdown(socket.SHUT_RDWR)
            except OSError: pass

listener = socket.socket()
listener.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
listener.bind(("127.0.0.1", 8110))
listener.listen()
listener.settimeout(4)                       # stop on its own after 4 s idle
try:
    while True:
        client, _ = listener.accept()
        upstream = socket.create_connection(("127.0.0.1", 8104))
        threading.Thread(target=pipe, args=(client, upstream), daemon=True).start()
        threading.Thread(target=pipe, args=(upstream, client), daemon=True).start()
except TimeoutError:
    pass
EOF
forwarder=$!
sleep 1

curl -s -o /dev/null -w 'direct   :8104 -> %{http_code}\n' http://127.0.0.1:8104/
curl -s -o /dev/null -w 'forwarded:8110 -> %{http_code}\n' http://127.0.0.1:8110/
wait "$forwarder"
kill "$backend"
```

Both return 200: the client talks to `:8110` and gets the answer from `:8104`
without knowing anything about it.

### SSH: the same thing through an encrypted hop

```
ssh -L 5433:db.internal:5432 ec2-user@bastion.example.com
```

> **Not run on your machine:** your WSL has the SSH *client* (OpenSSH 8.9) but no
> SSH *server*, and I had no reachable server to test against. The syntax below is
> standard OpenSSH; treat it as unverified here.

| Form | Meaning | Use |
|------|---------|-----|
| `-L 5433:db.internal:5432` | **Local** forward: connections to **your** `localhost:5433` go through the bastion to `db.internal:5432` | Reach a private database with a local client — `psql -h localhost -p 5433` |
| `-R 9000:localhost:3000` | **Remote** forward: the **server** listens on 9000 and sends it back to **your** `localhost:3000` | Expose a dev server through a reachable machine |
| `-D 1080` | **Dynamic** SOCKS proxy on your `localhost:1080` | Route a browser through the remote network |
| `-N` | Don't run a command, just hold the tunnel | Background tunnels |
| `-J bastion` | Jump host: SSH to the target through it | Reaching private hosts |

Properties worth knowing:

- `db.internal` in `-L` is resolved **on the bastion** (split-horizon DNS again —
  Day 14), not on your laptop.
- A forward binds to your **loopback** by default (Part 1: only you can use it). To
  let others use it you would bind `0.0.0.0` — and open that exposure deliberately.
- The first hop is encrypted; the last hop (bastion → database) is whatever the
  database protocol is.
- AWS's modern equivalent avoids opening SSH at all: **Session Manager port
  forwarding** (`aws ssm start-session --document-name AWS-StartPortForwardingSessionToRemoteHost`),
  with no inbound ports and IAM-controlled access (Phase 13).

To try SSH tunnels yourself later you'd need an SSH server — in your Ubuntu
terminal, `sudo apt install openssh-server` would add one. I haven't done that for
you.

> **Say this in an interview:** "An SSH local forward — `-L` — opens a port on my
> machine and carries connections through the SSH session to a host the bastion can
> reach, so I can use a private database from a local client; `-R` is the reverse and
> `-D` makes a SOCKS proxy. The target hostname is resolved on the far side. In AWS
> I'd prefer SSM Session Manager port forwarding so no inbound SSH is needed."

---

## Part 7 — WSL2 networking, and its limits

**In plain words:** WSL2 is a small virtual machine, and by default it sits behind
Windows the way a laptop sits behind a home router: it can reach out, outside
things can't reach in, and Windows does some friendly forwarding. You've now seen
the pieces separately; here is how they fit.

```
Phone on Wi-Fi (192.168.1.x)
      │  ✘ cannot reach WSL (the relay listens only on Windows' loopback)
      ▼
Windows 192.168.1.23 ──────────────────────────────────────────┐
  │ localhost:8102  ← relay (Windows loopback only)             │ NAT (Day 12)
  │      │ forwards into WSL's localhost                        ▼
  └─ vEthernet (WSL) 172.24.160.1 ──── Hyper-V virtual switch ── internet
                          │
                   WSL eth0 172.24.174.221   ← Windows CAN reach this directly
                   WSL lo   127.0.0.1
```

| From → To | Works? | Mechanism |
|-----------|--------|-----------|
| WSL → internet | ✔ | NAT through Windows |
| Windows → WSL service via `localhost:port` | ✔ | The relay (for ports bound to `127.0.0.1` or `0.0.0.0`, not `eth0`-only) |
| Windows → WSL via `172.24.174.221:port` | ✔ | Directly over the virtual network — **but the address changes** (Day 11) |
| WSL → Windows via the gateway `172.24.160.1` | ✔ | The WSL adapter on Windows |
| **Another device → WSL** | **✘** | Nothing listens on Windows' Wi-Fi address — measured: `192.168.1.23:8102` failed |

### Limits and how people work around them

- **LAN access (phone → WSL service).** Either forward a Windows port to WSL's
  address, or switch networking modes:
  ```powershell
  # Unverified here (needs an admin PowerShell and a Windows firewall rule). Forwards Windows :8080 to the WSL IP:
  netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=8080 connectaddress=172.24.174.221
  ```
  The WSL IP changes on every restart, so this goes stale — and `0.0.0.0` on
  Windows exposes it to every network the laptop joins (a café's Wi-Fi). The other
  route is **mirrored networking mode** (`networkingMode=mirrored` in
  `C:\Users\<you>\.wslconfig`), where WSL shares Windows' network and addresses;
  your WSL 2.0.14 is new enough to have it, but I haven't enabled it and it
  changes the earlier lessons' assumptions (Day 11).
- **Windows Firewall.** Windows has its own firewall in front of all of this,
  and the WSL adapter shows as `vEthernet (WSL (Hyper-V firewall))` — WSL has its
  own inbound firewall layer. On your machine neither blocked Windows → WSL traffic,
  but if "it works on `localhost` and not on the IP", look there.
- **`ufw` inside WSL does not see the relay's traffic** (Part 3, surprise 3).
- **No inbound from the outside world, ever**, in the default mode — which is a
  safe default for a dev machine and the reason you don't need `ufw` there.

> **Say this in an interview:** "WSL2 in NAT mode is a VM behind the Windows host:
> outbound goes through NAT, Windows' localhost is relayed into WSL, and Windows can
> also reach the WSL IP directly. Other devices can't reach it without a port proxy
> or mirrored mode. I treat 'bound to localhost' as 'visible to Windows' there."

---

## Hands-on

Blocks tagged **lab** run with the Run button; **Ubuntu** / **PowerShell** are
exercises for your own terminals.

### 1. The bind matrix, WSL side (lab — required practice)

```bash
# Three servers bound three ways; try to reach each via loopback and via eth0's address.
eth0_ip=$(ip -4 -o addr show eth0 | awk '{split($4, a, "/"); print a[1]}')
python3 -m http.server 8101 --bind 127.0.0.1 >/dev/null 2>&1 &
python3 -m http.server 8102 --bind 0.0.0.0   >/dev/null 2>&1 &
python3 -m http.server 8103 --bind "$eth0_ip" >/dev/null 2>&1 &
sleep 1
echo "listening:"; ss -ltn | grep -E ':810[123] '
echo
printf '%-26s %-10s %-10s %-10s\n' "reached from WSL via" "8101(lo)" "8102(any)" "8103(eth0)"
for target in 127.0.0.1 "$eth0_ip"; do
  row=""
  for port in 8101 8102 8103; do
    code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 2 "http://$target:$port/")
    row+=$(printf '%-11s' "$code")
  done
  printf '%-26s %s\n' "$target" "$row"
done
kill $(jobs -p)
```

`000` means no connection. Predict every cell before running (it's the left
two-thirds of the matrix in Part 1). Then, **with the servers still running**
(start them in your Ubuntu terminal instead if the lab stopped them), fill in the
Windows half in PowerShell:

```powershell
foreach ($p in 8101,8102,8103) {
  $r = try { (Invoke-WebRequest -UseBasicParsing -Uri "http://localhost:$p/" -TimeoutSec 4).StatusCode } catch { "FAIL" }
  "Windows localhost:$p -> $r"
}
foreach ($p in 8101,8102,8103) {
  $ok = Test-NetConnection -ComputerName <your-WSL-IP> -Port $p -WarningAction SilentlyContinue -InformationLevel Quiet
  "Windows -> WSL IP :$p -> $ok"
}
Get-NetTCPConnection -LocalPort 8101,8102,8103 -State Listen -ErrorAction SilentlyContinue |
  Select-Object LocalAddress, LocalPort, OwningProcess
```

Replace `<your-WSL-IP>` with the output of `ip -4 -o addr show eth0` in WSL (it
changes at every restart). Questions: which Windows `localhost` calls succeed? What
does `Get-NetTCPConnection` show listening — and on which Windows address? What
does that tell you about who can reach those ports from your phone?

### 2. Port conflict and finding the owner (lab)

Run the Part 2 block, then run this Java version on Windows:

```java
import java.net.BindException;
import java.net.InetAddress;
import java.net.ServerSocket;

public class Conflict {
    public static void main(String[] args) throws Exception {
        InetAddress loopback = InetAddress.getLoopbackAddress();
        try (ServerSocket first = new ServerSocket(8200, 50, loopback)) {
            System.out.println("first bind OK on " + first.getLocalSocketAddress());
            try (ServerSocket second = new ServerSocket(8200, 50, loopback)) {
                System.out.println("second bind OK?!");
            } catch (BindException e) {
                System.out.println(e.getClass().getName() + ": " + e.getMessage());
            }
        }
    }
}
```

Then, in PowerShell, run `netsh interface ipv4 show excludedportrange
protocol=tcp` and compare it with the port you'd choose for a new service. How many
ports does the biggest range cover?

### 3. Read your firewall (Ubuntu, `sudo`)

```
sudo iptables -S                 # all filter rules, in re-creatable form
sudo iptables -t nat -S          # the nat table
sudo ufw status verbose
```

On a fresh machine: all three policies `ACCEPT`, no rules, ufw `inactive`. Docker
Desktop and Kubernetes add their own chains here later — recognising the empty
baseline is the point.

### 4. Block a port with ufw — and test it from the right place (Ubuntu + PowerShell — required practice)

This **enables ufw** on your WSL. The last step turns it back off.

```
# Ubuntu terminal
python3 -m http.server 8102 --bind 0.0.0.0 >/dev/null 2>&1 &
python3 -m http.server 8104 --bind 0.0.0.0 >/dev/null 2>&1 &

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 8104/tcp
sudo ufw deny 8102/tcp
sudo ufw enable
sudo ufw status verbose

nc -zv -w 2 127.0.0.1 8102       # from inside WSL — what happens, and why?
```

```powershell
# PowerShell — replace the IP with your WSL address
Test-NetConnection 172.24.174.221 -Port 8102 -InformationLevel Quiet    # blocked
Test-NetConnection 172.24.174.221 -Port 8104 -InformationLevel Quiet    # allowed
Test-NetConnection 172.24.174.221 -Port 8105 -InformationLevel Quiet    # nothing there + default deny
(Invoke-WebRequest -UseBasicParsing http://localhost:8102/).StatusCode   # ??? — predict first
```

Predict the four PowerShell results and the `nc` result *before* running them, then
explain each. Then switch to a reject and time it:

```
sudo ufw --force delete deny 8102/tcp
sudo ufw reject 8102/tcp
```

```powershell
foreach ($p in 8102,8105) {
  $sw = [Diagnostics.Stopwatch]::StartNew()
  $c = New-Object Net.Sockets.TcpClient
  try { $t = $c.ConnectAsync('172.24.174.221', $p); if ($t.Wait(5000)) { $o = 'connected' } else { $o = 'TIMEOUT' } } catch { $o = 'REFUSED' }
  "port ${p}: $o after $($sw.ElapsedMilliseconds) ms"; $c.Dispose()
}
```

Clean up: `sudo ufw --force reset` (disables ufw and removes the rules), and
`kill %1 %2`.

<details>
<summary>Answers</summary>

`nc` to 127.0.0.1 **succeeds** (loopback is exempt). From Windows: 8102 `False`,
8104 `True`, 8105 `False`. `localhost:8102` returns **200** — it comes through the
relay from inside WSL. Reject: 8102 `REFUSED` after ~2 s; 8105 `TIMEOUT` after 5 s.

</details>

### 5. Make a firewall stateless (Ubuntu, `sudo` — required practice)

Do this **with ufw enabled and default deny incoming** (from exercise 4, before
the reset). If anything hangs, `sudo ufw --force reset` restores normality, and so
does `wsl --shutdown` from PowerShell (iptables rules live in memory).

```
curl -s -o /dev/null -w 'normal:    %{http_code}\n' --max-time 8 https://httpbin.org/get

sudo iptables -D ufw-before-input -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
curl -s -o /dev/null -w 'stateless: %{http_code}\n' --max-time 8 https://httpbin.org/get        # 000

sudo iptables -I ufw-before-input -p tcp --sport 443 -j ACCEPT
curl -s -o /dev/null -w 'tcp replies only: %{http_code}\n' --max-time 8 https://httpbin.org/get  # still fails — why?

sudo iptables -I ufw-before-input -p udp --sport 53 -j ACCEPT
curl -s -o /dev/null -w 'plus dns replies: %{http_code}\n' --max-time 8 https://httpbin.org/get  # works

# put it back
sudo iptables -D ufw-before-input -p tcp --sport 443 -j ACCEPT
sudo iptables -D ufw-before-input -p udp --sport 53 -j ACCEPT
sudo iptables -I ufw-before-input 2 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
curl -s -o /dev/null -w 'restored:  %{http_code}\n' --max-time 8 https://httpbin.org/get
```

Questions: (1) why did the third request still fail? (2) What would this firewall
need to allow *outgoing* SSH, NTP and package downloads statelessly? (3) Translate
what you just did into an AWS NACL: which direction and port range would the rule
you were missing have?

<details>
<summary>Answers</summary>

1. It couldn't resolve `httpbin.org`: the DNS replies (UDP, source port 53) were
   still being dropped.
2. A reply rule per protocol: TCP source 22, UDP source 123, TCP source 80/443 —
   or the whole ephemeral destination range, since the server's replies go *to*
   the client's ephemeral port.
3. An **inbound** rule allowing TCP on the ephemeral range (1024–65535) from
   0.0.0.0/0 — the replies to connections your instances initiate.

</details>

### 6. Redirect a port with NAT (Ubuntu, `sudo`)

Run the Part 5 block. Then list the rule with `sudo iptables -t nat -S OUTPUT`,
and delete it. Explain why the rule is in the `OUTPUT` chain of the `nat` table
rather than `PREROUTING` (hint: where does a packet from *this machine's own*
`curl` enter the stack?).

### 7. Build a port forwarder (lab)

Run the Part 6 block. Then change the backend to `8105` and the client request to
`/nonexistent` — what does the client see, and what does the *backend's* log say
the client's IP was? (A forwarder is an L4 proxy — Day 11: the backend sees the
forwarder as the client.)

### 8. Reachability from outside (PowerShell + your phone)

```powershell
Get-NetIPAddress -AddressFamily IPv4 -InterfaceAlias Wi-Fi | Select-Object IPAddress
```

With a server running in WSL on `0.0.0.0` (exercise 1), open
`http://<your Wi-Fi IP>:8102/` from your **phone** (same Wi-Fi). It won't load.
Which of Part 7's mechanisms would you add to make it load, and what are the
security implications of each?

### 9. Design questions (paper)

1. A web tier (public, port 443) talks to an app tier (port 8080) which talks to a
   database (port 5432). Write the Security Group rules (inbound only) for all
   three. Which uses a CIDR, and which should reference another security group?
2. Your NACL denies one abusive IP (`203.0.113.9`) and allows 443 from anywhere.
   In what order do you number the two rules, and why does it matter?
3. Your team enables a Kubernetes NetworkPolicy "allow ingress to the `orders` pods
   from the `gateway` pods". What happens to traffic from every other pod to
   `orders`? To traffic from `orders` to the internet?

<details>
<summary>Answers</summary>

1. Web SG: inbound 443 from `0.0.0.0/0`. App SG: inbound 8080 **from the web SG**.
   DB SG: inbound 5432 **from the app SG**. Only the first uses a CIDR; the rest
   reference groups, so they keep working as instances scale and IPs change.
2. The deny must have the **lower number** (e.g. 90 deny, 100 allow), because the
   NACL stops at the first match; reversed, the allow would match first and the deny
   would never apply.
3. All other pods' ingress to `orders` is now **denied** (a policy selecting a pod
   flips ingress to default-deny). Egress is unaffected — the policy only covers
   ingress — so `orders` can still reach the internet unless an egress policy also
   selects it.

</details>

---

## Gotchas

- **"Bound to localhost" in WSL ≠ invisible to Windows.** The relay forwards it.
  It *is* invisible to other devices on your network.
- **Local firewall tests are meaningless for loopback.** Test from another host
  (here: Windows).
- **ufw can't stop relayed traffic.** The relay originates inside WSL on loopback.
- **A silent drop looks like a dead host; a reject looks like a closed port.**
  Timeouts → suspect a firewall. Windows adds ~2 s even to refusals.
- **Stateless means every protocol, both ways.** Forgetting DNS replies is the
  classic miss; with NACLs, forgetting the ephemeral port range is.
- **Rule order matters.** First match wins; a broad deny above a specific allow
  makes the allow dead code.
- **Containers must bind `0.0.0.0`** inside; limit exposure with the publish
  address.
- **`0.0.0.0` is a promise to every network the machine joins.** Fine on a locked
  VPC subnet; risky on a laptop on public Wi-Fi.
- **WSL's IP changes** at every restart — never hard-code it in `netsh portproxy`
  rules or scripts.
- **Windows reserves port ranges** that can make an ordinary high port unusable
  with a confusing permission error.
- **`iptables -F` over SSH can lock you out** — flushing rules while the policy is
  DROP drops your own session. Change the policy first, or work from the console.
- **Rules you add with `iptables` live in memory** and vanish at reboot unless
  saved (ufw persists its own rules; raw iptables needs `iptables-persistent`).

---

## Recap

- **Bind address is access control:** `127.0.0.1` host-local, a specific IP one
  network, `0.0.0.0` everything. Containers must use `0.0.0.0`; restrict at publish.
- **On WSL, Windows' `localhost` is relayed** into WSL — loopback-bound servers are
  visible to Windows (not to the LAN); the WSL IP works from Windows but changes.
- **`EADDRINUSE`:** find the owner (`ss -ltnp`, `lsof`, `Get-NetTCPConnection`);
  Windows also reserves port ranges.
- **Netfilter:** tables → chains (INPUT, OUTPUT, FORWARD, PRE/POSTROUTING) → rules,
  first match wins, default policy decides the rest. `ufw` = default deny incoming
  + allow rules. **Loopback is always allowed.**
- **DROP** → timeout; **REJECT** → refused.
- **Stateful** firewalls remember connections (`ESTABLISHED,RELATED`); stateless
  ones need both directions, every protocol, and the ephemeral range. **SG:
  stateful, allow-only, instance. NACL: stateless, ordered, allow+deny, subnet.
  NetworkPolicy: allow-all until selected.**
- **NAT:** DNAT/REDIRECT for port forwarding (Docker `-p`, kube-proxy), SNAT/
  MASQUERADE for outbound.
- **SSH tunnels:** `-L` local, `-R` remote, `-D` SOCKS; target resolved on the far
  side. Prefer SSM in AWS.

---

## Say it out loud

1. What's the difference between binding to `127.0.0.1`, a specific IP, and
   `0.0.0.0`? Why does a containerised app have to bind `0.0.0.0`?
2. A port is "already in use". How do you find the process — on Linux, and on
   Windows when you're using WSL?
3. Explain how a Linux firewall evaluates a packet: tables, chains, rules,
   policy. What does DROP do compared with REJECT?
4. What's a stateful firewall, and why would a stateless one break your outbound
   HTTPS and DNS? Which AWS control is which?
5. How does Kubernetes make a Service's virtual IP reach a pod? Which netfilter
   mechanism is that?
6. How would you reach a private database from your laptop through a bastion?

---

## Q&A

_Follow-up questions from reading this lesson go here._
