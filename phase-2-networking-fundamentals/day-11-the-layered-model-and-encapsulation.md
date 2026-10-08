# Day 11 — The Layered Model & Encapsulation

_Why this matters:_ Every "the service is down" message you will ever get is a
question about one specific layer, and the fastest engineers answer it by asking
"which layer is broken?" before touching anything. "Connection refused" and
"connection timed out" sound alike but point to completely different places.
Later phases lean on today's vocabulary constantly: an AWS **NLB** is a "layer 4"
load balancer and an **ALB** is "layer 7"; a Kubernetes Service works at L4 while
an Ingress works at L7; Istio's sidecar exists because it can see L7. If "L4" and
"L7" are just labels to you, those phases turn into memorising. Today makes them
mechanisms.

> **Phase 2 opener.** Days 11–20 cover how two programs on different machines
> actually find and talk to each other. Today is the map; every later day zooms
> into one region of it (IP on Day 12, TCP on Day 13, DNS on Day 14, HTTP on
> Day 16, TLS on Day 17).
>
> Three of the hands-on exercises need `sudo` (packet capture), so they run in
> your own **Ubuntu (WSL)** terminal, not the lab's Run button. They're marked.

---

## The one-paragraph version

Sending data across a network is split into jobs, and each job is done by a
separate **layer** that knows nothing about the others. Your program's job is
the *message* (an HTTP request). The layer below turns that into a reliable
stream between two programs on two machines (TCP, using **port** numbers). The
layer below that gets each chunk from one machine to another machine anywhere on
the internet (IP, using **IP addresses**). The bottom layer moves bits across
one physical link — one cable or one Wi-Fi hop — to the next box (Ethernet or
Wi-Fi, using **MAC addresses**). Going down, each layer wraps what it got from
above in its own **header**, like putting a letter in an envelope, then that
envelope in a bigger one — that's **encapsulation**. The key consequence:
**the IP addresses stay the same for the whole journey, but the MAC addresses
are rewritten at every router**, because MACs only mean anything on one local
link. To reach something on the local network your machine asks "who has this
IP?" (**ARP**); to reach anything else, it hands the packet to its **default
gateway** and lets routers take it from there. When something breaks, it breaks
at one layer — and the error message tells you which, if you know how to read it.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **protocol** | An agreed set of rules for how two sides talk | A specification of message formats and behaviour (HTTP, TCP, IP, Ethernet) |
| **layer** | One job in the chain of sending data, done by one protocol | A level in a protocol stack that uses the layer below and serves the layer above |
| **stack** | All the layers together, as implemented on a machine | Linux's network stack lives in the kernel from L2 to L4 |
| **OSI model** | A 7-layer textbook diagram of networking | ISO's reference model: Physical, Data Link, Network, Transport, Session, Presentation, Application |
| **TCP/IP model** | The 4-layer version that actually runs the internet | Link, Internet, Transport, Application |
| **L1, L2, L3, L4, L7** | Shorthand for "layer N" — OSI numbers, used everywhere | L1 physical, L2 link (MAC), L3 network (IP), L4 transport (TCP/UDP), L7 application (HTTP) |
| **header** | The label a layer sticks on the front of data | Fixed-format bytes carrying that layer's addressing and control info |
| **payload** | The data being carried inside a header | Everything after this layer's header — which is the layer above's header + data |
| **encapsulation** | Wrapping data in layer after layer of headers | Each layer prepends its header to the unit from the layer above |
| **frame / packet / segment** | The name of the "envelope" at L2 / L3 / L4 | PDU (protocol data unit) names: Ethernet frame, IP packet, TCP segment (UDP: datagram) |
| **NIC / interface** | Your network card, or a virtual one | `eth0`, `wlan0`, `lo` — a device the kernel can send frames through |
| **MAC address** | A hardware address for one network card, meaningful only on the local link | 48-bit L2 address, e.g. `00:15:5d:09:36:f4`; first 3 bytes identify the vendor |
| **IP address** | A machine's address on the internet (or a private network) | 32-bit IPv4 L3 address, e.g. `172.24.174.221` |
| **subnet** | The group of addresses that are "local" to you — reachable without a router | An IP prefix like `172.24.160.0/20` (Day 12 does the maths) |
| **switch** | A box that connects machines on one local network | L2 device: forwards frames by destination MAC |
| **router** | A box that connects different networks and forwards between them | L3 device: forwards packets by destination IP, using a routing table |
| **default gateway** | "The router I send everything to when I don't know better" | The next hop for the `default` route (`0.0.0.0/0`) |
| **hop** | One router-to-router step on the path | Each L3 device a packet passes through |
| **routing table** | The kernel's list of "to reach these addresses, send via there" | `ip route` — matched by longest prefix |
| **ARP** | "Who has IP X? Tell me your MAC" — shouted to the local network | Address Resolution Protocol: maps IPv4 → MAC on one link, via broadcast |
| **broadcast** | A message for everyone on the local network | Destination MAC `ff:ff:ff:ff:ff:ff` |
| **TTL** | A countdown on each packet so lost packets don't circle forever | IPv4 Time To Live: decremented by each router; dropped at 0 |
| **port** | A number that says which program on a machine the data is for | 16-bit L4 address (0–65535); `:80` HTTP, `:443` HTTPS, `:5432` Postgres |
| **socket** | Your program's handle on one network connection | The OS object your code reads/writes; identified by IP + port on each end |
| **loopback** | A fake network inside one machine | `lo`, `127.0.0.1` — never leaves the host |
| **MTU** | The biggest packet one link will carry | Maximum Transmission Unit: 1500 bytes of IP packet on standard Ethernet |
| **MSS** | The biggest chunk of *your data* that fits in one TCP segment | Maximum Segment Size = MTU − IP header − TCP header = 1460 on Ethernet |
| **NAT** | Rewriting addresses so many machines can share one address | Network Address Translation — preview today, full treatment Days 12 and 19 |
| **firewall** | A filter that allows or blocks traffic | Rules matching L3/L4 fields (IPs, ports, protocol), often tracking connections |
| **load balancer** | A box that spreads incoming traffic across several servers | L4 (forwards TCP connections) or L7 (reads HTTP and routes requests) |
| **proxy** | A middleman that receives your request and makes its own request onward | An L7 intermediary; "reverse proxy" sits in front of servers |
| **`tcpdump`** | A program that prints every packet going through an interface | Packet capture tool; needs root because it sees everyone's traffic |

---

## Before you read: three questions

1. A colleague says: "The payment service is broken. I can `ping` the box fine,
   but `curl` just hangs." Which layers has the ping proved are working, and
   which hasn't it tested at all?
2. Your laptop sends a packet to a server in AWS that is 15 routers away. Which
   address in that packet stays the same the entire way, and which one changes
   at every single router?
3. You run `ping 1.1.1.1`. Before sending anything to `1.1.1.1`, your machine
   broadcasts an ARP "who has …?" question. For *which* IP address?

Keep your answers. All three get answered below, with packets from your own
machine as evidence.

---

## Part 1 — Why layers exist at all

**In plain words:** imagine every program that wanted to send data had to know
whether the machine was on Wi-Fi or a cable, how to reach every other network on
earth, and what to do when a piece of data got lost. Nobody could write a web
server. Layers split the problem so each piece only solves one thing: your
Spring Boot app only thinks about HTTP; the kernel's TCP code only thinks about
reliable delivery; the IP code only thinks about "which way to the destination";
the network card only thinks about "get these bits to the next box". Any layer
can be swapped without the others noticing — which is why the same `curl`
works over Wi-Fi, Ethernet, 4G and a fibre backbone in one journey.

The precise rule behind this: **each layer talks only to the layer directly
above and below it on the same machine, and to the *same* layer on the other
machine.** Your HTTP code on your laptop "talks to" the HTTP code on the server;
your TCP talks to the server's TCP. Neither knows or cares what physical links
the bytes crossed in between.

Two properties fall out of that, and they explain most of the rest of today:

1. **A layer only reads its own header.** A switch reads MAC addresses and
   nothing else. A router reads the IP header. An L4 load balancer reads ports.
   An L7 proxy reads the HTTP request. What a device *can* do is decided by how
   deep it reads.
2. **A lower layer treats everything above it as opaque bytes.** IP doesn't know
   there's HTTP inside; to IP it's just payload. That's why you can put
   *anything* on top — HTTP, Postgres's wire protocol, Kafka's, gRPC.

> **Say this in an interview:** "Networking is layered so each protocol solves
> one problem and treats everything above it as opaque payload. Devices are
> classified by the deepest header they read — a switch reads L2, a router L3,
> an L4 load balancer reads ports, an L7 proxy reads the HTTP request."

---

## Part 2 — OSI's seven layers vs the four that actually run

**In plain words:** there are two diagrams of the same idea. The **OSI model**
has seven layers and is what everyone quotes. The **TCP/IP model** has four and
is what's actually built. People use the OSI *numbers* as vocabulary ("L4",
"L7") while running the TCP/IP *protocols*. You need to know both, and how they
line up.

| OSI # | OSI name | TCP/IP layer | Its one job | Address it uses | Unit | Examples |
|------:|----------|--------------|-------------|-----------------|------|----------|
| 7 | Application | **Application** | What the programs are saying | URLs, hostnames | message | HTTP, DNS, SMTP, Postgres protocol, gRPC |
| 6 | Presentation | ↑ (folded into app) | Encoding, encryption | — | — | TLS is often *called* L6 (see below) |
| 5 | Session | ↑ (folded into app) | Long-lived conversations | — | — | Mostly theory today |
| 4 | Transport | **Transport** | Program-to-program delivery on two machines | **port** | segment / datagram | TCP, UDP |
| 3 | Network | **Internet** | Machine-to-machine delivery across networks | **IP address** | packet | IPv4, IPv6, ICMP (`ping`) |
| 2 | Data Link | **Link** | Delivery to the next box on one local link | **MAC address** | frame | Ethernet, Wi-Fi (802.11), ARP |
| 1 | Physical | ↑ (part of Link) | Bits as voltages, light, radio | — | bits | Cables, fibre, radio |

What to take from the table:

- **You'll spend almost all your time on L2, L3, L4 and L7.** L5 and L6 exist in
  the OSI diagram but have no separate protocol in TCP/IP — your application does
  those jobs itself.
- **TLS doesn't fit cleanly.** It sits *on top of* TCP and *underneath* HTTP.
  People call it L6, or "between L4 and L7". Don't argue about it; know where it
  physically sits. (Day 17.)
- **Each layer has its own address** — MAC at L2, IP at L3, port at L4, hostname
  and URL path at L7. When someone asks "what does X route on?", the answer is
  one of these, and it tells you the layer.
- **Everything from L2 to L4 lives in the Linux kernel.** L7 lives in your
  process. The boundary between them is the **socket**: when your Java code calls
  `new Socket(host, 80)`, it's asking the kernel to handle L4 and below. Your
  code never builds a TCP header. You'll prove this in hands-on 7.

> **Say this in an interview:** "OSI is the seven-layer reference model; what
> runs is the four-layer TCP/IP stack — link, internet, transport, application.
> We still use OSI numbers as shorthand, so L3 is IP, L4 is TCP/UDP and ports,
> L7 is HTTP. On Linux, L2 through L4 are in the kernel and L7 is in the
> application; the socket API is the boundary."

---

## Part 3 — Encapsulation: what wraps what

**In plain words:** it's the envelope-inside-envelope idea, and it's literal —
the bytes on the wire really are nested headers. Your HTTP request is the
letter. TCP puts it in an envelope labelled with ports. IP puts that in a bigger
envelope labelled with IP addresses. Ethernet puts that in the outermost
envelope labelled with MAC addresses. At the other end, each layer opens its own
envelope and passes the contents up. Unlike real envelopes, each wrap adds
*bytes*, and there's a size limit on the outermost one — which matters.

```
Your code writes:                              GET /get HTTP/1.1\r\nHost: ...     (78 bytes)

L4 TCP adds a header:          [ TCP hdr 20–60 ][ GET /get HTTP/1.1 ...          ]  = segment
                                src port, dst port, seq/ack numbers, flags

L3 IP adds a header:  [ IP hdr 20 ][ TCP hdr   ][ GET /get HTTP/1.1 ...          ]  = packet
                       src IP, dst IP, TTL, protocol=6 (TCP)

L2 Ethernet wraps it:
[ Eth hdr 14 ][ IP hdr ][ TCP hdr ][ GET /get HTTP/1.1 ...          ][ FCS 4 ]      = frame
 dst MAC, src MAC, type=0x0800 (IPv4)                                  checksum
```

The one field in each header that says **what's inside the payload** is what
makes unwrapping possible:

| Header | "What's inside me" field | Values you'll see |
|--------|--------------------------|-------------------|
| Ethernet | **EtherType** | `0x0800` IPv4, `0x0806` ARP, `0x86dd` IPv6 |
| IPv4 | **Protocol** | `1` ICMP, `6` TCP, `17` UDP |
| TCP / UDP | **Destination port** (by convention) | `80` HTTP, `443` HTTPS, `53` DNS, `5432` Postgres |

The receiving kernel reads the EtherType to know the payload is IP, reads the IP
protocol field to know it's TCP, reads the destination port to know *which
socket* — which process — gets the bytes.

### The size limit: MTU and MSS

A standard Ethernet link carries at most **1500 bytes of IP packet** per frame.
That's the **MTU**. Subtract the headers and you get how much of *your* data
fits:

```
1500  MTU (max IP packet on Ethernet)
- 20  IPv4 header
- 20  TCP header (minimum)
----
1460  MSS — max bytes of your data per TCP segment
```

So a 1 MB JSON response doesn't travel as one thing. TCP chops it into ~720
segments of ≤1460 bytes, each wrapped in its own IP header and Ethernet frame,
and reassembles them at the other end. Your code never sees the chopping.

**This is not theory on your machine.** When I ran `tracepath` from your WSL
while writing this lesson, it reported:

```
 2:  192.168.1.1        3.903ms
 3:  192.168.1.1      574.994ms pmtu 1492
```

`pmtu 1492` means the path MTU drops from 1500 to **1492** at your home router.
The missing 8 bytes are a *PPPoE* header — the extra envelope many broadband
connections wrap around every packet between your router and your ISP. It's
encapsulation you can measure: one more layer of wrapping costs 8 bytes of
capacity in every packet. You'll see its effect on the MSS in hands-on 4.

**Why you care:** when some link on the path has a smaller MTU than the sender
assumes and the "too big" signal gets blocked, small requests work and large
ones silently hang. It looks exactly like a flaky app. VPNs, tunnels and
Kubernetes overlay networks (Phase 17) all add headers and all hit this.

> **Say this in an interview:** "Each layer prepends a header — Ethernet, then
> IP, then TCP, then the application payload — and each header has a field
> identifying the next one up: EtherType, IP protocol number, port. The link
> MTU caps the IP packet at 1500 bytes on Ethernet, so TCP's MSS is 1460. Any
> tunnel that adds headers shrinks it further, and MTU mismatches show up as
> large transfers hanging while small ones work."

---

## Part 4 — Local delivery vs global routing: MAC, ARP and IP

This is the most important part of the day. It answers questions 2 and 3.

**In plain words:** there are two kinds of address because there are two
different jobs. The **IP address** is like the address on a parcel — it's where
the parcel is finally going, and it doesn't change in transit. The **MAC
address** is like telling the courier "take this to the depot on the corner" —
it only means "the next box on this wire", so it's rewritten at every stop. Your
machine never needs the destination server's MAC address, and never learns it.
It only needs the MAC of the next box.

### The decision your kernel makes for every packet

For each outgoing packet, the kernel looks at the destination IP and asks one
question: **"Is this address on my local subnet?"**

- **Yes** — it's directly reachable. Find *that machine's* MAC (via ARP) and
  send the frame straight to it.
- **No** — it's somewhere else. Find the **default gateway's** MAC (via ARP) and
  send the frame to the gateway. The destination IP in the packet is still the
  far-away server.

Your kernel will tell you which way it would go. Run this in the lab:

```bash
# Where would a packet to each of these addresses be sent?
gw=$(ip route | awk '/^default/ {print $3; exit}')
for dst in 1.1.1.1 "$gw" 127.0.0.1; do
  echo "== $dst"
  ip route get "$dst" | head -1
done
```

On your machine this printed:

```
== 1.1.1.1
1.1.1.1 via 172.24.160.1 dev eth0 src 172.24.174.221 uid 1000
== 172.24.160.1
172.24.160.1 dev eth0 src 172.24.174.221 uid 1000
== 127.0.0.1
local 127.0.0.1 dev lo src 127.0.0.1 uid 1000
```

Read the difference: `1.1.1.1` has **`via 172.24.160.1`** — "not local, hand it
to the gateway". The gateway itself has no `via` — "local, deliver directly".
`127.0.0.1` goes to `lo` and never touches a network card.

### ARP: turning "next IP" into "next MAC"

Ethernet can only deliver to a MAC. So once the kernel knows the next hop's IP
(either the destination or the gateway), it needs that IP's MAC. If it's not
cached, it broadcasts:

```
00:15:5d:09:36:f4 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806): Request who-has 172.24.160.1 tell 172.24.174.221
00:15:5d:5d:25:5a > 00:15:5d:09:36:f4, ethertype ARP (0x0806): Reply 172.24.160.1 is-at 00:15:5d:5d:25:5a
```

That's a real capture from your WSL, taken after clearing the ARP cache and
running **`ping 1.1.1.1`**. Look at what it asked for: **not `1.1.1.1`**. It
asked for the gateway, `172.24.160.1` — because `1.1.1.1` isn't local, so the
only MAC it needs is the gateway's. That's question 3.

- The request goes to `ff:ff:ff:ff:ff:ff` — broadcast, every machine on the link
  hears it.
- Only the owner of `172.24.160.1` answers, directly (unicast) to the asker.
- The answer is cached in the **neighbour table** (`ip neigh`) for a while, so
  it doesn't happen per packet.
- ARP never crosses a router. It's L2 — it only exists on one link. (IPv6
  replaces ARP with *Neighbor Discovery*, which is why the modern command is
  `ip neigh`, not `arp`.)

### What happens at every router

When your packet reaches a router, the router:

1. **Strips the Ethernet frame** — the MACs have done their job.
2. Reads the **destination IP**, looks it up in its routing table, picks the
   next hop.
3. **Decrements the TTL** by 1 (drops the packet if it hits 0) and fixes the IP
   checksum.
4. **Builds a brand-new frame** with *its own* MAC as source and the next hop's
   MAC as destination — possibly a different L2 technology entirely (Wi-Fi in,
   PPPoE/fibre out).

So across the whole journey:

| Field | Changes at each hop? |
|-------|---------------------|
| Source / destination **MAC** | **Yes — every hop.** Only meaningful on one link |
| Source / destination **IP** | **No** — end to end (the exception is NAT, below) |
| **TTL** | Yes — down by 1 per router |
| Ports, TCP data, HTTP | No — routers don't read them (except NAT and middleboxes) |

That's question 2: the IPs survive the journey; the MACs are rewritten every
hop.

**TTL is how `tracepath` works.** It sends a packet with TTL=1; the first router
decrements it to 0, drops it, and sends back an error saying "expired" — which
reveals that router's address. Then TTL=2 reveals the second router, and so on.
The tool builds the path out of deliberate failures.

### NAT: the exception you're already living in

Your path has at least two machines that *do* rewrite IPs:

- **Windows** rewrites WSL's `172.24.x.x` source address to your Wi-Fi address
  (`192.168.1.23`) on the way out.
- **Your home router** rewrites `192.168.1.23` to the public address your ISP
  gave you.

That's why the server never sees `172.24.x.x`. Check what it does see:

```bash
echo "WSL's own address:     $(ip -4 -o addr show eth0 | awk '{print $4}')"
echo "What the server sees:  $(curl -s --max-time 10 https://httpbin.org/ip | tr -d ' \n{}"' | sed 's/origin://')"
```

Two different addresses. NAT is a Day 12 and Day 19 topic; for today, just know
that a NAT box is a router that also edits the IP (and port) fields.

> **Say this in an interview:** "MAC addresses are link-local — they get a frame
> to the next hop and are rewritten at every router. IP addresses are end to end
> and only change if there's NAT. For each packet the host checks whether the
> destination is on its subnet: if so it ARPs for the destination's MAC,
> otherwise it ARPs for the default gateway's MAC and lets the routers forward
> it by IP, decrementing TTL at each hop."

---

## Part 5 — Reading your own machine: `ip addr`, `ip route`, `ip neigh`

**In plain words:** three commands show you all three things from Part 4 on
your own machine — "what are my addresses", "where do I send things", and "whose
MACs have I learned". Here is every line, explained.

```bash
echo "===== ip addr";  ip addr
echo; echo "===== ip route"; ip route
echo; echo "===== ip neigh"; ip neigh
```

### `ip addr` — my interfaces and addresses

Your output (addresses will differ after a WSL restart — see Gotchas):

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:09:36:f4 brd ff:ff:ff:ff:ff:ff
    inet 172.24.174.221/20 brd 172.24.175.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::215:5dff:fe09:36f4/64 scope link
```

| Piece | Layer | Meaning |
|-------|-------|---------|
| `1: lo` | — | Interface index 1, the loopback. Traffic to `127.0.0.1` stays inside the machine |
| `mtu 65536` on `lo` | L2 | Loopback has no wire, so it allows huge packets |
| `2: eth0` | — | Your one real(ish) network interface. In WSL2 it's a virtual NIC on a Hyper-V virtual switch |
| `<BROADCAST,MULTICAST,UP,LOWER_UP>` | L1/L2 | Capabilities and state. **`UP`** = switched on by the admin. **`LOWER_UP`** = the "cable is plugged in" signal (carrier) — L1 is alive. `UP` without `LOWER_UP` means "enabled, but no link": check the cable/Wi-Fi |
| `mtu 1500` | L2 | Max IP packet size on this link (Part 3) |
| `qdisc mq … qlen 1000` | — | Outgoing queue settings. Ignore for now |
| `state UP` | L1/L2 | Operational state. `DOWN` here is the first thing to check when "nothing works" |
| `link/ether 00:15:5d:09:36:f4` | **L2** | **Your MAC address.** `00:15:5d` is Microsoft's vendor prefix — Hyper-V made it up |
| `brd ff:ff:ff:ff:ff:ff` | L2 | The broadcast MAC (what ARP requests go to) |
| `inet 172.24.174.221/20` | **L3** | **Your IPv4 address**, and `/20` says how big your local subnet is — which addresses count as "local" (Day 12) |
| `brd 172.24.175.255` | L3 | The subnet's broadcast IP |
| `scope global` | L3 | Usable for talking beyond this machine (vs `scope host` = only this machine) |
| `valid_lft forever` | L3 | How long the address lease lasts. `forever` because WSL assigns it statically |
| `inet6 fe80::…/64 scope link` | L3 | An automatic IPv6 *link-local* address — only valid on this one link. Notice it's built from your MAC (`215:5dff:fe09:36f4`) |

### `ip route` — where do packets go?

```
default via 172.24.160.1 dev eth0 proto kernel
172.24.160.0/20 dev eth0 proto kernel scope link src 172.24.174.221
```

| Line | Meaning |
|------|---------|
| `172.24.160.0/20 dev eth0 … scope link` | "Every address in my subnet is **directly reachable** out of `eth0` — no router needed (`scope link`)." When sending from here, use `src 172.24.174.221` |
| `default via 172.24.160.1 dev eth0` | "**Anything else**: send it to `172.24.160.1` out of `eth0`." That's your default gateway |
| `proto kernel` | Who created the route — here, set up automatically at boot (on a normal Ubuntu box you'd see `proto dhcp`) |

When several routes match, the **most specific** (longest prefix) wins — that's
why the `/20` route beats `default` (which is `0.0.0.0/0`, matching everything)
for local addresses. Day 12 explains the prefix lengths.

**Who is `172.24.160.1`?** It's Windows. On the Windows side, `ipconfig` shows an
adapter called `vEthernet (WSL (Hyper-V firewall))` with exactly that address.
WSL2 (in its default *NAT mode*) is a small virtual machine whose only router is
your Windows host.

```powershell
ipconfig | Select-String -Pattern "adapter|IPv4|Gateway"
```

### `ip neigh` — which MACs have I learned?

```
172.24.160.1 dev eth0 lladdr 00:15:5d:5d:25:5a REACHABLE
```

| Piece | Meaning |
|-------|---------|
| `172.24.160.1` | The neighbour's IP |
| `lladdr 00:15:5d:5d:25:5a` | Its **link-layer address** — MAC — learned via ARP |
| `REACHABLE` | Confirmed recently. Other states: **`STALE`** (cached, not recently confirmed — will be re-checked on next use, totally normal), **`INCOMPLETE`** (ARP asked, no answer *yet*), **`FAILED`** (ARP got no answer — nothing at that IP on this link) |

Notice there's usually **exactly one** entry: the gateway. You never talk
directly to any other machine on the WSL subnet, and everything else in the
world is reached through that one MAC.

> **Say this in an interview:** "`ip addr` shows interfaces with their L2 MAC and
> L3 addresses and link state; `ip route` shows the routing table — connected
> subnets plus a default route to the gateway, longest prefix wins; `ip neigh`
> is the ARP/neighbour cache mapping next-hop IPs to MACs. A `FAILED` neighbour
> entry means L2 can't find that host at all."

---

## Part 6 — L4 and L7, briefly: where your code actually lives

**In plain words:** IP gets bytes to the right *machine*. But a machine runs
dozens of programs — Postgres, your Spring Boot app, an SSH server. **Ports**
(L4) say which program. Then **L7** is the conversation itself — and it's
usually just text that your program writes. Everything below is done for you by
the kernel.

Today's version (TCP gets a full day on Day 13):

- A server **listens** on a port. A client connects to `IP:port`. The pair of
  `(IP, port)` on each end identifies one connection.
- **TCP** gives you a reliable, ordered byte stream — it resends lost segments
  and puts them back in order. **UDP** just sends individual datagrams, no
  guarantees. HTTP/1.1 and HTTP/2 run on TCP; DNS mostly uses UDP.
- **HTTP is plain text written into that stream.** Nothing magic. You can type
  it by hand:

```bash
# Talk HTTP by hand: nc opens a TCP connection (L4); we write the L7 text ourselves.
printf 'GET /get HTTP/1.1\r\nHost: httpbin.org\r\nConnection: close\r\n\r\n' \
  | timeout 10 nc httpbin.org 80 | head -8
```

`nc` (netcat) looked up `httpbin.org`'s IP, then asked the kernel for a TCP
connection to port 80. The kernel did all the L2–L4 work: handshake, IP
headers, Ethernet frames, ARP. The only "HTTP" involved is the four lines of
text you piped in. A server listening on a
port looks like this:

```bash
# Start a tiny web server on port 8000, see it LISTENING, talk to it, stop it.
python3 -m http.server 8000 --bind 127.0.0.1 >/dev/null 2>&1 &
pid=$!
sleep 1
ss -ltnp 'sport = :8000'          # -l listening, -t TCP, -n numeric, -p process
curl -s -o /dev/null -w 'server answered HTTP %{http_code}\n' http://127.0.0.1:8000/
kill "$pid"
```

`ss` shows `LISTEN … 127.0.0.1:8000 … users:(("python3",…))` — the kernel is
holding port 8000 open for that process. Day 13 explains every column.

> **Say this in an interview:** "IP delivers to a host; the port in the L4
> header delivers to a process. TCP gives a reliable ordered byte stream, UDP
> gives unreliable datagrams. HTTP/1.1 is just text written into a TCP stream —
> the application only ever sees bytes on a socket, and the kernel handles
> segmentation, IP and framing."

---

## Part 7 — Where each piece of infrastructure sits

**In plain words:** every box in a network diagram is defined by **how far into
the envelopes it looks**. A switch only reads the outer envelope (MAC). A router
opens one more (IP). A firewall might peek at the ports. An L7 load balancer
opens *everything*, reads your HTTP request, and makes decisions on the URL. The
deeper it reads, the smarter it can be — and the more work (and latency, and
things to misconfigure) it adds.

| Device | Layer | Reads | Decides / does | Rewrites | Where you'll meet it |
|--------|-------|-------|----------------|----------|----------------------|
| **Switch** | L2 | Dst MAC | Which physical port to send the frame out of | Nothing | Your home router's LAN ports; Hyper-V's virtual switch under WSL |
| **Router** | L3 | Dst IP | Next hop, from its routing table | MACs (new frame), TTL | Your home router; AWS VPC route tables (Phase 15) |
| **NAT gateway** | L3/L4 | IPs, ports | Share one public IP among many machines | **Src IP and port** | Windows under WSL, your home router, AWS NAT Gateway |
| **Firewall** | L3/L4 | IPs, protocol, ports (stateful ones track connections) | Allow or drop | Nothing | `iptables`/`ufw` (Day 19), **AWS Security Groups** (stateful) and NACLs (stateless) |
| **L4 load balancer** | L4 | IPs, ports | Picks a backend per **TCP connection**, forwards bytes without reading them | Dst IP (or nothing) | **AWS NLB**; a **Kubernetes Service** (kube-proxy iptables rules) |
| **L7 load balancer / reverse proxy** | L7 | The full HTTP request: host, path, headers | Routes per **request** (`/api/*` → service A), retries, TLS termination | **Everything** — it opens a *new* connection to the backend | **AWS ALB**, nginx, Envoy, Istio sidecar, Spring Cloud Gateway, Kubernetes Ingress |
| **Forward proxy** | L7 | HTTP request | Sits in front of *clients* going out (corporate proxy) | New outbound connection | `HTTPS_PROXY` in corporate networks |

The consequence that bites in real work: **an L7 proxy ends your TCP connection
and starts a new one.** So the backend sees the *proxy's* IP as the client, not
the user's. That's why `X-Forwarded-For` headers exist (Day 16), and why your
Spring app's "client IP" logging is wrong behind a load balancer unless you
configure it. Some L4 load balancers (an AWS NLB with instance targets, for
example) pass the client's IP through untouched — but no L4 balancer can route
by URL path, because it never reads the path.

> **Say this in an interview:** "A switch forwards on MAC, a router on IP; a
> firewall or security group filters on IPs and ports. An L4 load balancer like
> an NLB balances TCP connections without parsing them; an L7 load balancer like
> an ALB terminates the connection, parses HTTP and routes per request by host
> or path — so it can do path routing and TLS termination, but the backend sees
> the proxy's IP unless you read `X-Forwarded-For`."

---

## Part 8 — "It's a networking problem": finding the layer

**In plain words:** "the network is broken" is almost never true as stated.
Something specific at one layer is broken, and each layer fails with a different
*symptom*. The single most useful distinction: **was the error instant, or did
it hang?** An instant "refused" means a machine *answered* you — so everything up
to L3 worked. A hang means your packets vanished somewhere, and nobody answered
at all.

Watch four different failures, each at a different layer:

```bash
# Four failures, four layers. Watch the exit codes and how long each takes.
probe() {
  local label=$1; shift
  local start=$(date +%s%N)
  local err
  err=$(curl -sS -o /dev/null --connect-timeout 3 "$@" 2>&1)
  local rc=$?
  local ms=$(( ($(date +%s%N) - start) / 1000000 ))
  printf '%-34s exit=%-3s %5s ms  %s\n' "$label" "$rc" "$ms" "${err:-(no error)}"
}
probe "1. no such host (DNS)"            http://no-such-host.invalid/
probe "2. nothing listening (L4)"        http://127.0.0.1:9/
probe "3. packets dropped (L3/firewall)" http://10.255.255.1/
probe "4. server says 503 (L7)"          -f http://httpbin.org/status/503
```

| # | curl says | Exit | Time | What it proves | Layer |
|---|-----------|------|------|----------------|-------|
| 1 | `Could not resolve host` | 6 | fast | Never sent a single packet to the server — the *name* didn't turn into an IP | DNS (Day 14) |
| 2 | `Connection refused` / `Failed to connect` | 7 | **instant** | The host **is reachable** — its kernel answered with a TCP reset (RST) because no process listens on that port | L4 — service not running, wrong port, listening on the wrong interface |
| 3 | `Connection timed out` | 28 | **the full timeout** | Packets went out and *nothing ever came back*. Dropped by a firewall / security group, no route, or the host is down | L3 or a firewall |
| 4 | `The requested URL returned error: 503` | 22 | normal | **The network is completely fine.** DNS, IP, TCP all worked; the application answered with an error | L7 — an app problem |

Then the practical ladder, bottom-up — each step assumes the ones below it pass:

| Check | Command | If it fails, suspect |
|-------|---------|---------------------|
| Link up? | `ip link` → `state UP`, `LOWER_UP` | L1/L2: cable, Wi-Fi, VM adapter |
| Next hop known? | `ip neigh` → gateway not `FAILED` | L2: gateway not on this link |
| Route exists? | `ip route get <ip>` | L3: missing route → "Network is unreachable" |
| Name resolves? | `getent hosts <name>` | DNS |
| Port open? | `nc -zv -w 3 <host> <port>` | L4: refused = not listening; timeout = firewall |
| App healthy? | `curl -v http://<host>:<port>/health` | L7: status code and body |

**About `ping`** — question 1. A successful `ping` proves L1–L3 work to that
host (ICMP is an L3 protocol). It proves **nothing** about L4 or L7: the port can
be firewalled, the app can be dead. And a *failed* ping proves less than people
think, because many hosts and cloud security groups block ICMP. Your own WSL
gateway is one: `ping 172.24.160.1` loses 100% of packets (the Windows firewall
drops them) while every web request you make goes straight through it. So in the
colleague's case: ping proved the host is up and routable; the hang in `curl`
means TCP packets to that port are being silently dropped — a firewall/security
group, or something listening on the wrong port *behind* a firewall. Check with
`nc -zv -w 3 host port`.

> **Say this in an interview:** "I troubleshoot bottom-up and use the failure
> mode to pick the layer. Connection refused is fast and proves L3 reachability —
> the host's kernel sent a RST because nothing is listening. A timeout means
> packets are being silently dropped — usually a firewall or security group, or
> routing. A resolution error is DNS, and an HTTP 5xx means the network is fine
> and it's the application."

---

## Hands-on

Blocks tagged for the lab run with the Run button. The three **`sudo`**
exercises (3, 4, 5) must be run in your own **Ubuntu (WSL)** terminal — the lab
can't type your password. `tcpdump` is already installed.

### 1. Annotate your own machine (lab)

Run the Part 5 block. Then, *without looking back at Part 5*, write one line for
each of: your MAC, your IP, your subnet, your gateway's IP, your gateway's MAC,
your MTU. Check against the tables.

### 2. Ask the kernel for its routing decisions (lab)

Run the Part 4 `ip route get` block. Then add three destinations of your own to
the loop: `8.8.8.8`, an address inside your own `/20` that isn't yours (for
`172.24.174.221/20`, try `172.24.170.5`), and `192.168.1.1`. Predict for each
whether you'll see `via` before running it.

<details>
<summary>Answers</summary>

- `8.8.8.8` → `via 172.24.160.1` (not local).
- `172.24.170.5` → **no `via`**. It's inside `172.24.160.0/20`, so the kernel
  would ARP for it directly. Nothing lives there, so a real packet would leave an
  `INCOMPLETE`/`FAILED` entry in `ip neigh`.
- `192.168.1.1` → `via 172.24.160.1`. Your home router is on *Windows'* network,
  not WSL's. From inside WSL it's just another remote network behind the Windows
  gateway.

</details>

### 3. Watch ARP happen (Ubuntu terminal, `sudo`)

```
sudo -v                                   # type your password once, so it's cached
sudo ip neigh flush dev eth0              # forget every learned MAC
ip neigh                                  # empty
(sleep 1; ping -c 1 -W 1 1.1.1.1 >/dev/null) & sudo tcpdump -i eth0 -nn -e -c 2 arp
ip neigh                                  # the gateway is back
```

`-i eth0` which interface, `-nn` don't turn IPs/ports into names, `-e` print
the Ethernet (MAC) header, `-c 2` stop after two packets, `arp` only ARP frames.

Check: which IP did it ask for, and why is it not `1.1.1.1`? Which MAC did the
request go to, and which did the reply go to?

### 4. Capture one HTTP request, every layer (Ubuntu terminal, `sudo`)

Plain `http://`, not `https://`, so you can see the request text:

```
sudo -v
(sleep 1; curl -s -o /dev/null http://httpbin.org/get) & sudo tcpdump -i eth0 -nn -e -c 6 'tcp port 80'
```

Here's what that produced on your machine (trimmed of timestamps):

```
00:15:5d:09:36:f4 > 00:15:5d:5d:25:5a, ethertype IPv4 (0x0800), length 74: 172.24.174.221.60810 > 54.172.131.225.80: Flags [S], seq 3488537587, win 64240, options [mss 1460,sackOK,TS val 3011310533 ecr 0,nop,wscale 7], length 0
00:15:5d:5d:25:5a > 00:15:5d:09:36:f4, ethertype IPv4 (0x0800), length 74: 54.172.131.225.80 > 172.24.174.221.60810: Flags [S.], seq 120150364, ack 3488537588, win 26847, options [mss 1452,sackOK,TS val 721082701 ecr 3011310533,nop,wscale 8], length 0
00:15:5d:09:36:f4 > 00:15:5d:5d:25:5a, ethertype IPv4 (0x0800), length 66: 172.24.174.221.60810 > 54.172.131.225.80: Flags [.], ack 1, win 502, length 0
00:15:5d:09:36:f4 > 00:15:5d:5d:25:5a, ethertype IPv4 (0x0800), length 144: 172.24.174.221.60810 > 54.172.131.225.80: Flags [P.], seq 1:79, ack 1, win 502, length 78: HTTP: GET /get HTTP/1.1
```

Read one line left to right — it *is* the encapsulation diagram:

| Part of the line | Layer | Meaning |
|------------------|-------|---------|
| `00:15:5d:09:36:f4 > 00:15:5d:5d:25:5a` | L2 | From your MAC **to the gateway's MAC** — not the server's. The server is in AWS; its MAC is irrelevant |
| `ethertype IPv4 (0x0800)` | L2 | "My payload is an IPv4 packet" |
| `length 144` | L2 | Total frame bytes (FCS not shown) |
| `172.24.174.221 > 54.172.131.225` | L3 | Your IP to httpbin's IP — the end-to-end addresses |
| `.60810 > .80` | L4 | From port 60810 (a random *ephemeral* port your kernel picked) to port 80 |
| `Flags [S]`, `[S.]`, `[.]` | L4 | The TCP three-way handshake: SYN, SYN-ACK, ACK. Day 13 |
| `length 78: HTTP: GET /get HTTP/1.1` | L7 | Your 78 bytes of HTTP, finally |

Questions to answer from the output:

1. The HTTP packet is 144 bytes, of which 78 are HTTP. Where did the other 66
   go? (Hint: 14 + 20 + ?)
2. Your SYN says `mss 1460`. The server's SYN-ACK says `mss 1452`. Why 8 less?

<details>
<summary>Answers</summary>

1. 14 bytes Ethernet + 20 bytes IP + **32** bytes TCP (20-byte header + 12 bytes
   of *timestamp* option) = 66. Wrapping cost almost as much as the request.
2. 1492 − 40 = 1452. It's the PPPoE MTU from Part 3. Your side advertises 1460
   because *its* link (eth0) has MTU 1500 — it doesn't know about the smaller
   link further out. The 1452 was most likely written into the SYN-ACK by your
   home router ("MSS clamping" — routers doing PPPoE rewrite the MSS in passing
   handshakes so neither side sends packets too big for the PPPoE link). A router
   editing an L4 field is exactly the kind of "middlebox" exception Part 4
   mentioned.

</details>

### 5. Find the headers in raw bytes (Ubuntu terminal, `sudo`)

`-XX` prints the whole frame in hex. Capture just the first SYN:

```
sudo -v
(sleep 1; curl -s -o /dev/null http://httpbin.org/get) & sudo tcpdump -i eth0 -nn -e -XX -c 1 'tcp port 80 and tcp[tcpflags] & tcp-syn != 0'
```

Your machine produced these 74 bytes:

```
0x0000:  0015 5d5d 255a 0015 5d09 36f4 0800 4500
0x0010:  003c 50b6 4000 4006 d482 ac18 aedd 36ac
0x0020:  83e1 ed96 0050 b734 bf3a 0000 0000 a002
0x0030:  faf0 15b2 0000 0204 05b4 0402 080a b37c
0x0040:  fc3d 0000 0000 0103 0307
```

Each pair of hex digits is one byte; `0x0010` on the left means "this row starts
at byte 16". The map:

| Bytes (from 0) | Hex | Field | Value |
|---------------:|-----|-------|-------|
| 0–5 | `0015 5d5d 255a` | **Ethernet** dst MAC | `00:15:5d:5d:25:5a` — the gateway |
| 6–11 | `0015 5d09 36f4` | Ethernet src MAC | `00:15:5d:09:36:f4` — you |
| 12–13 | `0800` | EtherType | IPv4 |
| 14 | `45` | **IP** version / header length | version 4, header = 5 × 4 = 20 bytes |
| 16–17 | `003c` | IP total length | 60 = 20 IP + 40 TCP |
| 22 | `40` | **TTL** | 64 — Linux's default starting value |
| 23 | `06` | **Protocol** | 6 = TCP |
| 26–29 | `ac18 aedd` | Src IP | 172.24.174.221 |
| 30–33 | `36ac 83e1` | Dst IP | 54.172.131.225 |
| 34–35 | `ed96` | **TCP** src port | 60822 |
| 36–37 | `0050` | Dst port | 80 |
| 46 | `a0` | TCP header length | `a` = 10 × 4 = 40 bytes (20 + 20 of options) |
| 47 | `02` | Flags | SYN |
| 54–57 | `0204 05b4` | Option: MSS | `0x05b4` = 1460 |

Now do it on **your own** capture: find your TTL, protocol number, source IP and
both ports by counting bytes. Convert hex with `printf '%d\n' 0xed96` in the lab
if needed. This is the only time you'll decode by hand — Wireshark and `tcpdump
-v` do it for you — but having done it once, a header is never abstract again.

### 6. HTTP by hand (lab)

Run the `nc` block from Part 6. Then change `/get` to `/headers` and add a
header line of your own (`X-Learning: day11\r\n`) before the blank line. It
comes back in the response body — you just spoke raw L7.

### 7. Same thing from Java: your code only sees L7 (lab)

This is what every Java HTTP client does underneath. Note that this block runs on
**Windows**, not WSL — so watch which local address it reports.

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.nio.charset.StandardCharsets;

public class RawHttp {
    public static void main(String[] args) throws IOException {
        String host = "httpbin.org";

        try (Socket socket = new Socket()) {
            // One call = DNS lookup + TCP handshake. The kernel builds every
            // TCP/IP/Ethernet header; Java never sees them.
            socket.connect(new InetSocketAddress(host, 80), 5_000);
            socket.setSoTimeout(5_000);

            System.out.println("local  (me)  : " + socket.getLocalSocketAddress());
            System.out.println("remote (them): " + socket.getRemoteSocketAddress());
            System.out.println();

            // L7: the HTTP request is just text we write into the TCP stream.
            String request = "GET /get HTTP/1.1\r\n"
                    + "Host: " + host + "\r\n"
                    + "Connection: close\r\n"
                    + "\r\n";
            OutputStream out = socket.getOutputStream();
            out.write(request.getBytes(StandardCharsets.US_ASCII));
            out.flush();

            // Read the status line and headers, stop at the blank line before the body.
            BufferedReader in = new BufferedReader(
                    new InputStreamReader(socket.getInputStream(), StandardCharsets.US_ASCII));
            String line;
            while ((line = in.readLine()) != null && !line.isEmpty()) {
                System.out.println("< " + line);
            }
        }
    }
}
```

Look at the first two lines of output:

- **local** is `/192.168.1.23:<some port>` — your *Windows* Wi-Fi address, not
  WSL's `172.24.x.x`. Windows programs skip the WSL network entirely. The port is
  a random ephemeral port, like `60810` in the capture.
- **remote** is `httpbin.org/<ip>:80` — Java resolved the name (DNS) before
  connecting.

The layers in one sentence: *your* code wrote 4 lines of text; `connect()` and
`write()` asked the kernel to do L4 and below. `RestTemplate`, `WebClient` and
`java.net.http.HttpClient` are this, plus connection pooling, TLS and parsing.

### 8. The failure ladder (lab)

Run the Part 8 `probe` block. Then add a fifth probe: `http://httpbin.org:81/`
— a real host, but a port with nothing reachable. Predict: refused or timeout?

<details>
<summary>Answer</summary>

Timeout (exit 28, ~3 s). The host exists, but AWS's security group silently
**drops** traffic to port 81 instead of letting the server's kernel send a RST.
Same "nothing on that port" situation as probe 2 — but a firewall in the path
turns "refused" into "timeout". In AWS, a timeout to a port you *know* is
listening almost always means a security group rule.

</details>

### 9. Draw the path from your browser to AWS (paper or a text file)

This is the day's required practice. First collect the evidence:

```bash
# Your path to an AWS-hosted site, one router per line (no root needed).
timeout 15 tracepath -n -m 8 httpbin.org
```

`tracepath` (the no-root cousin of `traceroute`) uses the TTL trick from Part 4.
Lines of `no reply` are routers that don't send the "expired" message — normal.

Now draw the path a request takes from **your browser** (which runs on Windows)
to a server in AWS. For each box, write: its name, its address, the **highest
layer it reads**, and whether it rewrites anything. Mark where the MACs change,
where NAT happens, and where the PPPoE header gets added.

<details>
<summary>One correct drawing (yours will have different ISP hops)</summary>

```
 Browser (Windows process)                               L7: builds HTTP request
     │ socket → Windows kernel: TCP, IP (src 192.168.1.23), Wi-Fi frame
     │                                                   ── MACs: laptop → router
 Home router  192.168.1.1                                L3 (+L4 for NAT)
     │ NAT: src 192.168.1.23:port → <your public IP>:port
     │ strips Wi-Fi frame, wraps in PPPoE (+8 bytes, MTU 1492), clamps MSS
     │                                                   ── MACs rewritten
 ISP routers  103.57.86.x, 10.240.244.x, …               L3 only: TTL-1, new frame each hop
     │ internet backbone — 10 to 20 more hops
 AWS edge routers                                        L3
     │
 AWS load balancer (httpbin sits behind one)             L7 if ALB: ends your TCP connection,
     │                                                   reads HTTP, opens a new one to the server
 EC2 instance / container                                L7: the app finally reads "GET /get"
```

And from **WSL** (your `curl` runs) there's one extra box at the top: WSL
`172.24.174.221` → Windows `172.24.160.1` (NAT #1, rewrites to `192.168.1.23`) →
then the same path. That's the `172.24.160.1` then `192.168.1.1` you saw as the
first two tracepath hops.

The two things your drawing must show: **the destination IP never changes after
your router's NAT; the MAC addresses change at every arrow.**

</details>

---

## Gotchas

- **`ping` is not a health check.** It proves L3 only, and many hosts (your own
  WSL gateway, most AWS security groups by default) drop ICMP. "Ping fails" ≠
  "host down"; "ping works" ≠ "service up".
- **WSL's addresses can change every restart** — even the MAC. While this
  lesson was being tested, WSL went idle and restarted, and `eth0`'s MAC changed
  from `00:15:5d:09:36:f4` to `00:15:5d:09:3e:9e`. The IP (`172.24.174.221`) and
  gateway (`172.24.160.1`) can change too (`wsl --shutdown`, reboots). So your
  output won't match the lesson byte for byte. Never hard-code these in scripts;
  read them from `ip route` like the Part 4 block does.
- **WSL2 NAT mode is a separate network.** Other devices on your Wi-Fi can't
  reach a server in WSL directly; Windows forwards `localhost:PORT` into WSL for
  you, but LAN access needs extra setup (Day 19). Newer WSL has a *mirrored*
  networking mode that shares Windows' addresses — you're not using it, so
  lessons assume NAT mode.
- **Windows programs and WSL programs take different paths.** The Java block ran
  on Windows (`192.168.1.23`); `curl` in the Ubuntu tab ran from `172.24.x.x`.
  When debugging "works from Java, fails from curl", check which side you're on.
- **`tcpdump -i eth0` doesn't see loopback traffic.** A request to
  `127.0.0.1:8000` never touches `eth0`. Use `-i lo`, or `-i any`.
- **Capturing HTTPS shows you nothing readable.** Above L4 it's encrypted, which
  is why today used `http://`. You still see every L2–L4 header.
- **MTU problems look like app bugs.** Small requests fine, large uploads or
  responses hang — suspect MTU (VPNs, tunnels, PPPoE, Kubernetes overlays)
  before blaming the code.
- **"L4" and "L7" are OSI numbers, but the protocols are TCP/IP.** Nobody runs
  OSI's actual protocols. Use the numbers as vocabulary; don't go looking for
  the "session layer" in a packet capture.
- **`STALE` in `ip neigh` is fine.** It means "cached, will re-verify on next
  use", not "broken". `FAILED` is the one that means trouble.
- **`arp` and `ifconfig` are the old tools** (`net-tools`). Tutorials still use
  them; on modern Linux use `ip neigh`, `ip addr`, `ip route`.

---

## Recap

- Networking is split into layers; each reads only its own header and treats
  everything above as opaque payload. **L2 = MAC / link, L3 = IP / routing, L4 =
  ports / TCP & UDP, L7 = the application protocol (HTTP).**
- OSI (7 layers) is the vocabulary; TCP/IP (4 layers) is what runs. L2–L4 are
  in the kernel; L7 is in your process; the **socket** is the boundary.
- **Encapsulation:** HTTP → TCP segment → IP packet → Ethernet frame. Each
  header names what's inside the next (EtherType, IP protocol, port).
- **MTU** 1500 → **MSS** 1460. Every extra wrapper shrinks it — your PPPoE link
  makes it 1492 / 1452.
- **IPs are end to end; MACs change at every hop.** The kernel checks "is the
  destination on my subnet?" — if yes, ARP for it; if no, ARP for the **default
  gateway**. Routers rebuild the frame and decrement TTL.
- **NAT** is the exception that rewrites IPs — you're behind two of them.
- Infrastructure is defined by the deepest layer it reads: switch L2, router
  L3, firewall/SG L3–L4, NLB/k8s Service L4, ALB/nginx/Envoy/Ingress L7.
- Diagnose by symptom: **resolve error = DNS, instant refused = L4 (host
  reachable, nothing listening), hang = dropped at L3/firewall, HTTP 5xx = the
  network is fine.**

---

## Say it out loud

Explain each to a colleague in about 30 seconds, out loud, using the real
terms. If you stall, reread that part.

1. What's the difference between the OSI and TCP/IP models, and why do people
   still say "L4" and "L7"?
2. Walk through what happens, layer by layer, when your Spring Boot app calls
   `restTemplate.getForObject("http://orders:8080/orders/1", …)`, down to the
   frame leaving the network card.
3. Why does a packet's MAC address change at every router while its IP address
   doesn't? What is ARP for, and whose MAC does your laptop ARP for when it
   talks to a server in AWS?
4. What's the difference between an L4 and an L7 load balancer, and what can an
   ALB do that an NLB can't? What does it cost you?
5. `curl` gives "connection refused" on one server and hangs until timeout on
   another. What does each tell you, and where do you look next?
6. What are MTU and MSS, and why might large requests hang while small ones
   work?

---

## Q&A

_Follow-up questions from reading this lesson go here._
