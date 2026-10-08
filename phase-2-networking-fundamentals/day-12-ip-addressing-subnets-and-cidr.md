# Day 12 — IP Addressing, Subnets & CIDR

_Why this matters:_ Yesterday your kernel asked "is this destination on my
subnet?" for every packet. Today you learn to answer that question yourself, in
your head. It isn't trivia. In Phase 15 you'll carve an AWS VPC into subnets, and
a bad choice there can't be undone without rebuilding. In Phase 17 every
Kubernetes pod needs its own IP, and "pods stuck in `ContainerCreating`: failed
to assign an IP address" is a real 3am page caused by a subnet sized too small
months earlier. VPNs, VPC peering and Docker all break in confusing ways when
two networks use **overlapping** ranges. Every one of those problems is solved
with the arithmetic in this lesson, and interviewers expect you to do it out loud
without a calculator.

---

## The one-paragraph version

An IPv4 address is just a 32-bit number, written as four numbers from 0–255
separated by dots (`172.24.174.221`). Every address splits into two parts: the
**network part** (the first bits, shared by everyone on the same local network)
and the **host part** (the remaining bits, unique to each machine). The `/20` in
`172.24.174.221/20` says where the split is: the first 20 bits are the network,
the last 12 identify the host. That notation is **CIDR**. From the number after
the slash you can work out everything: how many addresses the subnet has
(2^(32 − prefix): a `/24` has 256, a `/20` has 4,096), the first address (the
**network address**, all host bits 0), the last (the **broadcast address**, all
host bits 1), and which addresses are usable for machines (everything in
between). Bigger number after the slash = smaller network. Three ranges —
`10.0.0.0/8`, `172.16.0.0/12` and `192.168.0.0/16` — are **private**: anyone can
use them, they're never routed on the public internet, and that's why your
laptop and millions of others can all be `192.168.1.something` at once. The
trick that lets private machines reach the internet anyway is **NAT**: a router
swaps the private source address for its own public one on the way out and
swaps it back on the way in.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **bit** | A single 0 or 1 | The smallest unit; an IPv4 address is 32 of them |
| **octet** | One of the four numbers in an IP address | 8 bits, value 0–255 |
| **dotted quad** | The `a.b.c.d` way of writing an IP | Four octets in decimal, separated by dots |
| **network part** | The part of the address shared by everyone on the same local network | The leading *prefix-length* bits |
| **host part** | The part that's different for each machine on that network | The remaining 32 − prefix bits |
| **prefix length** | The number after the slash | How many leading bits are the network part: `/20` = 20 bits |
| **CIDR** | The `address/number` notation, and the idea that the split can be anywhere | Classless Inter-Domain Routing (1993); replaced fixed "classes" |
| **subnet** | One network inside a bigger range, defined by a CIDR block | A contiguous, power-of-two-sized, aligned block of addresses |
| **subnet mask** | The older way of writing the prefix length | 32 bits: 1s for the network part, 0s for the host part — `/20` = `255.255.240.0` |
| **bitwise AND** | Keep a bit only where both inputs have a 1 | `ip & mask` = the network address. How every router "applies" a mask |
| **network address** | The first address in a subnet; names the subnet itself | Host bits all 0. Not assigned to a machine |
| **broadcast address** | The last address; "everyone on this subnet" | Host bits all 1. Not assigned to a machine |
| **usable hosts** | Addresses you can actually give to machines | Total − 2 (network + broadcast); AWS takes 5 |
| **block size** | How far apart subnets of a given size start | 256 − mask value in the "interesting" octet |
| **subnetting** | Cutting a big range into smaller pieces | Extending the prefix: each extra bit halves the size and doubles the count |
| **RFC 1918 / private range** | Addresses reserved for private networks | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — never routed on the internet |
| **public IP** | An address that's globally unique and reachable on the internet | Assigned via ISPs / cloud providers; your home has one |
| **NAT** | Rewriting addresses at a router so private machines can share a public IP | Network Address Translation, tracked in a translation table |
| **SNAT / DNAT** | Rewriting the *source* (going out) / the *destination* (coming in) | Source NAT and Destination NAT — both are `iptables` targets (Day 19) |
| **PAT / NAPT** | NAT that also rewrites ports, so thousands of connections share one IP | Port Address Translation — what every home router does |
| **port forwarding** | A fixed rule: "traffic arriving on my port X goes to that inside machine" | A static DNAT entry |
| **VPC** | Your own private network inside AWS | Virtual Private Cloud — one CIDR block (e.g. `10.0.0.0/16`) you split into subnets |
| **availability zone (AZ)** | One data centre (or group) inside an AWS region | Each AWS subnet lives in exactly one AZ |
| **longest prefix match** | When several routes match, use the most specific one | The routing rule: the matching route with the largest prefix length wins |
| **`/32`** | Exactly one address | A host route; "my IP only" in a security-group rule |
| **`0.0.0.0/0`** | Every IPv4 address | The default route; "anywhere" in a firewall rule |
| **loopback** | The machine talking to itself | `127.0.0.0/8` |
| **link-local** | Self-assigned addresses that only work on one link | `169.254.0.0/16` (IPv4), `fe80::/10` (IPv6) |
| **IPv6** | The 128-bit successor to IPv4 | Written in hex, e.g. `2001:db8::1`; standard subnet is `/64` |

---

## Before you read: three questions

1. Your WSL address is `172.24.174.221/20`. Is `172.24.170.5` on the same
   subnet? What about `172.24.176.1`?
2. A teammate says "let's give the new Kubernetes cluster a `/24` for pods".
   Roughly how many pods does that hold, and why might that be a problem in
   three months?
3. Your laptop is `192.168.1.23`. So is somebody's laptop in another city right
   now. Why doesn't that break the internet?

---

## Part 1 — An IP address is a 32-bit number

**In plain words:** `172.24.174.221` looks like four separate numbers, but
it's one number — the dots are only there to make it readable for humans. Each
of the four parts is 8 bits, which is why each one tops out at 255 (eight 1s in
binary = 255). Once you see the bits, everything else today is "draw a line
somewhere in these 32 bits".

Each octet is 8 bits, worth these place values:

```
bit value:   128  64  32  16   8   4   2   1
```

So `174` is `128 + 32 + 8 + 4 + 2` = `10101110`. Your whole WSL address in
binary:

```
172      .24       .174      .221
10101100 .00011000 .10101110 .11011101
```

The lab can print it for you. This block converts any dotted quad to binary,
using only bash arithmetic:

```bash
# Print each octet of an IPv4 address as 8 bits.
to_bin() {
  local n=$1 out="" i
  for (( i = 7; i >= 0; i-- )); do out+=$(( (n >> i) & 1 )); done
  printf '%s' "$out"
}
ip_bin() {
  local IFS=.          # split on dots...
  set -- $1            # ...deliberately unquoted, so the 4 octets become $1..$4
  printf '%-16s %s.%s.%s.%s\n' "$1.$2.$3.$4" "$(to_bin "$1")" "$(to_bin "$2")" "$(to_bin "$3")" "$(to_bin "$4")"
}
ip_bin "$(ip -4 -o addr show eth0 | awk '{split($4, a, "/"); print a[1]}')"
ip_bin 255.255.240.0
ip_bin 192.168.1.23
```

Two numbers worth memorising, because they come up constantly:

| Bits | Count | Where you meet it |
|-----:|------:|-------------------|
| 8 | 256 | One octet; a `/24` |
| 32 | 4,294,967,296 (~4.3 billion) | All of IPv4 — not enough for every device on earth, which is the root cause of NAT and private ranges |

**Java detail that will bite you:** Java's `int` is 32 bits and **signed**. Any
address whose first octet is 128 or more (`172.…`, `192.…`, `255.…`) has its top
bit set, so as a Java `int` it's **negative**. That's harmless for the bit
operations, but you must use `>>>` (unsigned shift) rather than `>>` when pulling
octets back out, and `Integer.toUnsignedLong` / `toUnsignedString` when printing
it as a number. You'll write exactly this in hands-on 4.

> **Say this in an interview:** "An IPv4 address is a 32-bit unsigned integer,
> written as four 8-bit octets in dotted-decimal. That gives about 4.3 billion
> addresses in total, which ran out years ago — that scarcity is why private
> ranges and NAT exist."

---

## Part 2 — The slash: network part and host part

**In plain words:** a postcode and a house number. Everyone on your street
shares the postcode; the house number tells them apart. In an IP address, the
first N bits are the "postcode" (the network) and the rest are the "house number"
(the host). The `/N` tells you where the postcode ends. Your kernel uses exactly
this: if a destination has the same network bits as you, it's on your street
(deliver directly, via ARP); if not, it goes to the gateway (Day 11).

Your WSL is `172.24.174.221/20`. Draw the line after bit 20:

```
                       network (20 bits)        | host (12 bits)
172.24.174.221    10101100.00011000.1010 | 1110.11011101
mask /20          11111111.11111111.1111 | 0000.00000000   = 255.255.240.0
------------------------------------------------------------- AND
network address   10101100.00011000.1010 | 0000.00000000   = 172.24.160.0
broadcast         10101100.00011000.1010 | 1111.11111111   = 172.24.175.255
```

- The **subnet mask** is the prefix written out as bits: twenty 1s, then twelve
  0s. `/20` and `255.255.240.0` are the same fact in two notations. Modern tools
  and AWS use the `/20` form; you'll still see masks in older configs and in
  Windows' `ipconfig` (which showed `Subnet Mask . . : 255.255.240.0` for the WSL
  adapter on Day 11).
- **`address AND mask` = network address.** Keep the network bits, zero the host
  bits. That one operation is how "is this on my subnet?" is answered: AND both
  addresses with the mask and compare.
- **Network address** (host bits all 0) = `172.24.160.0`. It's the subnet's
  *name*. Your routing table literally says `172.24.160.0/20 dev eth0`.
- **Broadcast** (host bits all 1) = `172.24.175.255`. Check Day 11's `ip addr`
  output: `brd 172.24.175.255`. The kernel did the same sum.
- **Usable** = everything in between: `172.24.160.1` to `172.24.175.254`. The
  gateway took the first one, `.160.1` — a very common convention.
- **Count** = 2^12 = 4,096 addresses, 4,094 usable.

Now question 1. Is `172.24.170.5` on your subnet? Its third octet is 170, and
your subnet covers third octets 160–175. **Yes.** `172.24.176.1`? 176 is past 175.
**No** — that's the next `/20`, and your kernel would send it to the gateway.

> **Say this in an interview:** "The prefix length says how many leading bits
> are the network portion. ANDing an address with the mask gives the network
> address; setting all host bits gives the broadcast. Two hosts are on the same
> subnet exactly when their addresses ANDed with the mask are equal — that's the
> check a host does before deciding to ARP directly or send to the gateway."

---

## Part 3 — Doing it in your head: the block-size method

**In plain words:** you don't actually convert to binary every time. Subnets
come in fixed sizes that line up neatly, like ruler markings. Find which of the
four numbers the line falls in, work out how wide one "step" is, and round down
to the nearest step. Three small steps, no binary.

### The method

1. **Find the interesting octet** — the one where the prefix ends.
   `/1`–`/8` → 1st octet, `/9`–`/16` → 2nd, `/17`–`/24` → 3rd, `/25`–`/32` → 4th.
   Octets before it are copied unchanged. Octets after it are 0 (network) or 255
   (broadcast).
2. **Block size** = 2^(how many host bits fall inside that octet). Equivalently,
   256 − the mask value in that octet.
3. **Network** = round the interesting octet *down* to a multiple of the block
   size. **Broadcast** = network + block − 1 in that octet, 255 in every octet
   after.

Worked: `172.24.174.221/20`.

1. 20 is in 17–24 → 3rd octet. `172.24` is copied.
2. 20 − 16 = 4 network bits in that octet, so 8 − 4 = 4 host bits → block = 2^4 =
   **16**. (Mask octet: 256 − 16 = 240 → `255.255.240.0`.)
3. 174 rounded down to a multiple of 16: 16 × 10 = 160. **Network
   `172.24.160.0`.** Broadcast: 160 + 16 − 1 = 175, then 255 → **`172.24.175.255`.**

### The table to memorise

The only mask values that exist in any octet are these eight. Learn this row and
the rest follows:

```
network bits in the octet:   1    2    3    4    5    6    7    8
mask value:                128  192  224  240  248  252  254  255
block size:                128   64   32   16    8    4    2    1
```

And the prefixes you'll actually use, with the counts that matter in AWS:

| Prefix | Mask | Addresses | Usable (normal) | Usable in AWS | Typical use |
|-------:|------|----------:|----------------:|--------------:|-------------|
| `/8` | 255.0.0.0 | 16,777,216 | 16,777,214 | — | The whole `10.x` private range |
| `/16` | 255.255.0.0 | 65,536 | 65,534 | — | **A VPC** (the largest AWS allows) |
| `/18` | 255.255.192.0 | 16,384 | 16,382 | 16,379 | Big EKS pod subnets |
| `/19` | 255.255.224.0 | 8,192 | 8,190 | 8,187 | EKS subnets |
| `/20` | 255.255.240.0 | 4,096 | 4,094 | 4,091 | AWS default-VPC subnets; your WSL |
| `/22` | 255.255.252.0 | 1,024 | 1,022 | 1,019 | |
| `/24` | 255.255.255.0 | 256 | 254 | 251 | A home network; a "small" subnet |
| `/26` | 255.255.255.192 | 64 | 62 | 59 | |
| `/28` | 255.255.255.240 | 16 | 14 | 11 | **Smallest AWS subnet** |
| `/30` | 255.255.255.252 | 4 | 2 | — | Old-style router-to-router links |
| `/31` | 255.255.255.254 | 2 | 2 | — | Point-to-point links (no network/broadcast needed — RFC 3021) |
| `/32` | 255.255.255.255 | 1 | 1 | — | One host: "my IP only" in a security group |
| `/0` | 0.0.0.0 | all ~4.3 billion | — | — | `0.0.0.0/0`: default route, "anywhere" |

Shortcuts that save you in an interview:

- **Every +1 on the prefix halves the size.** `/24` = 256, `/25` = 128, `/26` = 64.
- **Every −1 doubles it.** `/20` = 16 × `/24`s. `/16` = 256 × `/24`s.
- **Each 8 bits = ×256.** `/16` is 256 times a `/24`; `/8` is 256 times a `/16`.

### AWS takes five

In an AWS subnet, five addresses are reserved, not two. For `10.0.0.0/24`:
`.0` network, `.1` the VPC router, `.2` the Amazon DNS server, `.3` reserved for
future use, `.255` broadcast (AWS doesn't support broadcast, but reserves it
anyway). So a `/24` gives you **251**, and a `/28` only 11.

> **Say this in an interview:** "A `/24` is 256 addresses, 254 usable — 251 in
> AWS, which reserves the first four and the last. Each bit of prefix halves
> the block, so a `/20` is 4,096 and a `/16` is 65,536. To find a subnet
> boundary I take the block size in the interesting octet and round down to a
> multiple of it."

---

## Part 4 — Subnetting: carving a range into pieces

**In plain words:** you're given a big block and need to cut it into smaller,
equal blocks. Each extra bit you add to the prefix cuts every piece in half, so
adding *n* bits gives you 2^n pieces. The pieces sit side by side, and each one
starts at a multiple of its own size — no gaps, no overlaps.

**Splitting `10.0.0.0/16` into `/20`s:**

- `/16` → `/20` = 4 extra bits → **2^4 = 16 subnets**.
- Each `/20` is 4,096 addresses = a step of **16 in the third octet** (block size
  from Part 3).
- So they are `10.0.0.0/20`, `10.0.16.0/20`, `10.0.32.0/20`, … `10.0.240.0/20`.

**Alignment rule — the one beginners break:** a subnet must start on a multiple
of its own size. `10.0.8.0/20` is **not** a valid `/20` network — 8 isn't a
multiple of 16. Write it into AWS and it's rejected. Some tools silently "fix" it
to `10.0.0.0/20`, which is worse, because now it overlaps whatever you meant to
put there. An address with host bits set (`10.0.1.5/24`) describes a *host on* a
subnet, not the subnet itself.

**Mixed sizes are fine, if you allocate big ones first.** From one `/24` you
could cut a `/25` (128) plus two `/26`s (64 each): `.0/25`, `.128/26`, `.192/26`.
Putting the big block first keeps everything aligned.

### Why subnet sizing is a production decision, not a detail

On AWS EKS with the default networking plugin (the *VPC CNI*), **every pod gets
a real IP from the node's subnet**. A node running 30 pods uses ~30 subnet
addresses plus its own. A `/24` (251 usable) covers maybe 7–8 such nodes. When
it runs out, new pods sit in `ContainerCreating` with "failed to assign an IP
address to container", and you **cannot resize an AWS subnet** — you add new
subnets and migrate. That's question 2: a `/24` holds a couple of hundred pods,
which a growing cluster eats in weeks. EKS subnets are usually `/19` or `/18`.

Kubernetes clusters that *don't* give pods VPC IPs use a separate **pod CIDR**
(e.g. `10.244.0.0/16`), which is cut into one `/24` per node — so that cluster
tops out at 256 nodes. Same arithmetic, different consequence. Phase 17.

> **Say this in an interview:** "Subnetting extends the prefix: n extra bits
> gives 2^n subnets, each half the size per bit. Subnets must be aligned to their
> own size. I size subnets for growth because AWS subnets can't be resized — on
> EKS with the VPC CNI every pod consumes a subnet IP, so I'd use `/19`s rather
> than `/24`s."

---

## Part 5 — Private ranges, special ranges, and overlap

**In plain words:** some address ranges are reserved by agreement. The three
private ranges are like internal phone extensions: every company has an
extension 100, and that's fine, because you can't dial someone else's extension
from outside — calls go through the switchboard (NAT). The price is that two
networks using the *same* extensions can't be joined without confusion. That
"same extensions" problem is overlap, and it's the most common real-world
addressing bug.

### The ranges you'll meet

| Range | Size | What it is | Where you've seen it |
|-------|------|------------|----------------------|
| `10.0.0.0/8` | 16.7 M | Private (RFC 1918) | Most VPCs and corporate networks |
| `172.16.0.0/12` | 1 M | Private — **`172.16.0.0` to `172.31.255.255`** only | Your WSL (`172.24.160.0/20`); Docker's default bridge `172.17.0.0/16`; AWS default VPC `172.31.0.0/16` |
| `192.168.0.0/16` | 65,536 | Private | Your home Wi-Fi `192.168.1.0/24` |
| `127.0.0.0/8` | 16.7 M | Loopback — never leaves the machine | `localhost` |
| `169.254.0.0/16` | 65,536 | Link-local — self-assigned, one link only | **`169.254.169.254`** is the AWS instance metadata service, where EC2 apps fetch their IAM credentials |
| `100.64.0.0/10` | 4 M | Carrier-grade NAT (ISPs NAT-ing customers) | ISPs; also EKS "custom networking" for pods |
| `0.0.0.0` | — | As a *listen* address: "every interface on this machine" | `ss` showing `0.0.0.0:*`; `server.address` in Spring |

Note `172.16.0.0/12` is not "all of 172.x". `172.32.0.1` is a public address.
With a `/12` the interesting octet is the 2nd, block size 16: `172.16`–`172.31`.

### `127.0.0.1` vs `0.0.0.0` when a server *listens*

A server bound to `127.0.0.1:8080` accepts connections only from the same
machine. Bound to `0.0.0.0:8080`, it accepts them on every interface. The classic
bug: an app inside a Docker container listens on `127.0.0.1`, and nothing outside
the container can reach it — "connection refused", even though `curl localhost`
works *inside*. You'll hit it in Phase 4. Spring Boot listens on all interfaces
by default; Python's `http.server` from Day 11 was bound to `127.0.0.1` on
purpose.

### Overlap: the bug that shows up months later

Two CIDR blocks **overlap** if any address is in both. It's easy to check: two
blocks overlap exactly when the *larger* block contains the *smaller* block's
network address (aligned blocks either nest or are separate — they never
partially overlap).

Where it bites:

- **VPC peering / Transit Gateway** between two VPCs refuses overlapping CIDRs.
  If every team picked `10.0.0.0/16` because it was the example in the docs,
  none of them can ever be connected. Real companies keep a spreadsheet (or an
  IPAM tool) allocating non-overlapping blocks per team, per environment.
- **VPN from your laptop to a network that uses `192.168.1.0/24`** — the same
  range as your home Wi-Fi. Your routing table now has two routes for the same
  addresses, and half your traffic goes the wrong way.
- **Docker's `172.17.0.0/16`** overlapping a corporate or VPC range: containers
  can't reach those company hosts, because the kernel thinks they're on the local
  Docker bridge.

How the kernel resolves two matching routes is **longest prefix match**: the
most specific route wins. Given this table:

```
10.0.0.0/8      via vpn
10.0.5.0/24     dev eth0       (local)
0.0.0.0/0       via 172.24.160.1  (default)
```

`10.0.5.9` matches all three; `/24` is the longest → local. `10.1.2.3` matches
`/8` and `/0` → VPN. `8.8.8.8` matches only `/0` → default. That's also how your
`172.24.160.0/20` route beat `default` on Day 11.

> **Say this in an interview:** "RFC 1918 reserves `10/8`, `172.16/12` and
> `192.168/16` for private networks; they aren't routed on the internet, so
> they're reused everywhere behind NAT. The cost is overlap — peered VPCs and
> VPNs need non-overlapping CIDRs, so I'd plan address space centrally up front.
> When routes overlap, the kernel picks by longest prefix match."

---

## Part 6 — NAT: how private addresses reach the internet

**In plain words:** your home router is like a company receptionist with one
public phone number. When you call out, the receptionist notes "extension 23
called this number" and makes the call from the main line. When the answer comes
back, they check the note and put it through to extension 23. Nobody outside ever
sees your extension. And if a stranger calls the main line out of the blue,
there's no note to match, so the call goes nowhere — which is why you can't
easily run a server at home.

Question 3: `192.168.1.23` exists in millions of homes at once. That's fine
because **no packet with a private source address ever reaches the internet** —
each home's router rewrites it first.

### What a NAT router actually does (PAT, the common kind)

Your request from Day 11, leaving through the home router:

```
Inside (before NAT)                         Outside (after NAT)
src 192.168.1.23 : 51631  ─── router ──▶   src 205.254.166.2 : 40001     dst 54.172.131.225 : 80
                                  │
                  translation table entry:
                  192.168.1.23:51631  ⇄  205.254.166.2:40001  ⇄  54.172.131.225:80
```

- **Outbound:** rewrite the source IP (and usually the source port) to the
  router's public address — **SNAT**. Record the mapping.
- **Inbound reply:** look up the destination port `40001` in the table, rewrite
  the destination back to `192.168.1.23:51631`, forward it.
- **Unsolicited inbound:** no matching entry → dropped. To run a server behind
  NAT you add a fixed **port forwarding** rule (a static **DNAT**: "anything to my
  port 8080 goes to `192.168.1.23:8080`").
- **Ports are what let thousands share one IP.** Two laptops both using source
  port 51631 get different outside ports. That's PAT (Port Address Translation),
  and it's what "NAT" means in everyday use.

(`40001` is an illustration — the router picks the outside port, and nothing
tells you which. The rest of the numbers are your real ones.)

### Your actual NAT chain

You measured this on Day 11. A `curl` from WSL goes through **two** NATs:

```
WSL         172.24.174.221   ──NAT #1 (Windows)──▶   192.168.1.23    ──NAT #2 (home router)──▶   103.214.63.244   ──▶ internet
            private 172.16/12                        private 192.168/16                          public
```

On 27 September httpbin saw you as `205.254.166.2`; on 7 October, as
`103.214.63.244`. Your private addresses didn't move, but your **public** one
did: most home ISPs hand out a *dynamic* public IP that changes on reconnects or
lease renewals. That's one more reason home servers are awkward, and why an AWS
security-group rule like "allow SSH from my IP `/32`" quietly stops working one
morning. Hands-on 6 re-checks it.

### The AWS version — this is the public/private subnet definition

In Phase 15 you'll build exactly this, and interviewers ask it directly:

- A **public subnet** is one whose route table sends `0.0.0.0/0` to an
  **Internet Gateway**. Instances there with a public IP are reachable from the
  internet (subject to security groups).
- A **private subnet** sends `0.0.0.0/0` to a **NAT Gateway** that lives in a
  public subnet. Instances can *start* connections out (to pull Docker images,
  call APIs) but nothing on the internet can start a connection *in* — the same
  "no table entry, dropped" rule as your home router.

So "public" and "private" are not properties of the IP range — every subnet in a
VPC uses private RFC 1918 addresses. They're properties of the **route table**.

NAT also explains why Kubernetes Services work (DNAT from a virtual service IP to
a pod IP) and why a server behind NAT sees the NAT box's IP, not the client's.
Day 19 covers the `iptables` rules that implement it.

> **Say this in an interview:** "NAT rewrites the source address of outbound
> packets to a public IP and tracks the mapping, usually with port translation,
> so many hosts share one address; replies are matched against the table and
> unsolicited inbound traffic is dropped. In AWS, a subnet is public if its
> route table points `0.0.0.0/0` at an internet gateway, and private if it points
> at a NAT gateway — the CIDRs themselves are private either way."

---

## Part 7 — IPv6, at awareness level

**In plain words:** IPv6 is the long-term fix for running out of addresses. It
uses 128-bit addresses instead of 32, which is so many that every subnet gets more
addresses than the entire IPv4 internet, and NAT becomes unnecessary. You need to
recognise it and know the basics; you won't subnet it by hand.

- **Format:** eight groups of four hex digits, `2001:0db8:0000:0000:0000:0000:0000:0001`.
  Leading zeros in a group can be dropped and **one** run of all-zero groups can
  be written `::`, giving `2001:db8::1`.
- **Standard subnet is `/64`** — 2^64 addresses per subnet. You don't count
  hosts; you count subnets.
- **No broadcast.** IPv6 uses multicast; Neighbor Discovery replaces ARP (why
  it's `ip neigh`).
- **Link-local `fe80::/10`** addresses are automatic on every interface. You have
  one: `ip -6 addr` in WSL shows `fe80::…/64 scope link`, often built from the
  MAC. That's the only IPv6 you have — your WSL has no IPv6 route to the internet
  (checked while writing this), which is common.
- **Why it's suddenly relevant:** since February 2024 AWS charges for every
  public IPv4 address (about $3.60 a month each), so dual-stack and IPv6-only
  VPCs are becoming normal, and EKS supports IPv6 pod networking specifically
  to escape pod IP exhaustion.

> **Say this in an interview:** "IPv6 addresses are 128 bits in hex with `::`
> compression; LANs are `/64`, so address exhaustion and NAT go away. It uses
> multicast and Neighbor Discovery instead of broadcast and ARP. It matters in
> AWS now because public IPv4 costs money, and EKS can use IPv6 to avoid pod IP
> exhaustion."

---

## Hands-on

Do **1** with pen and paper *before* running anything — the tools in 2–4 give
the answers away.

### 1. The practice set (paper, no calculator)

For each, write: network address, broadcast address, usable range, usable host
count.

- a. `10.0.1.0/24`
- b. `172.31.0.0/20`
- c. `192.168.5.64/26`
- d. `192.168.5.100/27` *(the address isn't the network — find its subnet)*

Then plan a VPC: take `10.0.0.0/16` and give two **public** and two **private**
`/20` subnets, spread across two availability zones (`a` and `b`). For each,
write the CIDR, usable addresses in AWS, and what `0.0.0.0/0` routes to.

<details>
<summary>Answers</summary>

**a. `10.0.1.0/24`** — `/24` ends exactly at the end of the 3rd octet, so the
whole 4th octet is host bits. Network `10.0.1.0`, broadcast `10.0.1.255`,
usable `10.0.1.1`–`10.0.1.254`, **254** hosts.

**b. `172.31.0.0/20`** — 3rd octet, block 16. Network `172.31.0.0`, broadcast
`172.31.15.255`, usable `172.31.0.1`–`172.31.15.254`, **4,094** hosts. (This is
literally the first subnet of the AWS *default* VPC, `172.31.0.0/16`.)

**c. `192.168.5.64/26`** — 4th octet, 2 network bits → block 64. 64 is a
multiple of 64. Network `192.168.5.64`, broadcast `192.168.5.127`, usable
`.65`–`.126`, **62** hosts.

**d. `192.168.5.100/27`** — 4th octet, block 32. 100 rounded down to a multiple
of 32 = 96. Network `192.168.5.96`, broadcast `192.168.5.127`, usable
`.97`–`.126`, **30** hosts.

**VPC plan** — `/16` → `/20` is 4 extra bits = 16 possible subnets, stepping 16
in the 3rd octet:

| Subnet | CIDR | AZ | AWS usable | `0.0.0.0/0` → |
|--------|------|----|-----------:|---------------|
| public-a | `10.0.0.0/20` | a | 4,091 | Internet Gateway |
| public-b | `10.0.16.0/20` | b | 4,091 | Internet Gateway |
| private-a | `10.0.32.0/20` | a | 4,091 | NAT Gateway in public-a |
| private-b | `10.0.48.0/20` | b | 4,091 | NAT Gateway in public-b |

That uses 4 of 16 `/20`s; `10.0.64.0`–`10.0.255.255` is spare for databases,
more AZs, or EKS later. One NAT gateway per AZ means losing AZ `a` doesn't cut
off AZ `b`'s outbound traffic. A common variation in real designs: small public
subnets (only load balancers and NAT gateways live there) and big private ones
(where the apps and pods live).

</details>

### 2. Check your answers with Python's `ipaddress` (lab)

Python's standard library does CIDR maths correctly, which makes it a good
referee. Run it, compare with your paper answers, and look at the last two
lines carefully:

```bash
python3 - <<'EOF'
import ipaddress

for cidr in ["10.0.1.0/24", "172.31.0.0/20", "192.168.5.64/26", "192.168.5.100/27"]:
    net = ipaddress.ip_network(cidr, strict=False)   # strict=False: accept a host address
    hosts = list(net.hosts())
    print(f"{cidr:18} network {str(net.network_address):14} broadcast {str(net.broadcast_address):15}"
          f" usable {hosts[0]} - {hosts[-1]} ({len(hosts)})")

print()
vpc = ipaddress.ip_network("10.0.0.0/16")
subnets = list(vpc.subnets(new_prefix=20))
print(f"{vpc} splits into {len(subnets)} x /20:", ", ".join(str(s) for s in subnets[:4]), "...")

print()
try:
    ipaddress.ip_network("10.0.8.0/20")             # misaligned — strict by default
except ValueError as e:
    print("rejected:", e)
print("10.0.8.0/20 'fixed' to:", ipaddress.ip_network("10.0.8.0/20", strict=False))
EOF
```

The last line is the alignment trap from Part 4: asked for `10.0.8.0/20`, a
non-strict tool quietly gives you `10.0.0.0/20` — a different block that may
overlap one you already allocated.

### 3. Your own subnet, by hand then by machine (lab)

```bash
cidr=$(ip -4 -o addr show eth0 | awk '{print $4}')
echo "My address: $cidr"
ip -4 -o addr show eth0 | awk '{print "kernel says broadcast:", $6}'
ip route | grep -v '^default'
```

Before running: from Day 11 your address was `172.24.174.221/20`. If it has
changed since, work out the new network and broadcast with the block-size method
first, then run the block and check against the kernel's `brd` and the
`… dev eth0 scope link` route.

### 4. Build a subnet calculator in Java (lab)

The main exercise. Every operation from Parts 2–5 as bit operations on an
`int`. Read it before running: each method is one line of today's maths.

```java
import java.util.ArrayList;
import java.util.List;

public class SubnetCalc {

    /** A CIDR block, stored as its 32-bit network address plus prefix length. */
    record Cidr(int network, int prefix) {

        static Cidr parse(String text) {
            String[] parts = text.split("/");
            int prefix = Integer.parseInt(parts[1]);
            if (prefix < 0 || prefix > 32) {
                throw new IllegalArgumentException("prefix out of range: " + text);
            }
            int ip = toInt(parts[0]);
            int network = ip & maskOf(prefix);                 // AND with the mask = network address
            if (network != ip) {
                throw new IllegalArgumentException(text + " has host bits set; the network is "
                        + toDotted(network) + "/" + prefix);
            }
            return new Cidr(network, prefix);
        }

        int mask()        { return maskOf(prefix); }
        int broadcast()   { return network | ~mask(); }       // set every host bit to 1
        long size()       { return 1L << (32 - prefix); }      // long: a /0 is 2^32, too big for int

        long usable() {
            if (prefix == 32) return 1;
            if (prefix == 31) return 2;                        // point-to-point, RFC 3021
            return size() - 2;                                 // minus network and broadcast
        }

        long usableInAws() { return size() - 5; }

        boolean contains(int ip) { return (ip & mask()) == network; }

        /** Aligned blocks either nest or are disjoint, so compare under the shorter prefix. */
        boolean overlaps(Cidr other) {
            int m = maskOf(Math.min(prefix, other.prefix));
            return (network & m) == (other.network & m);
        }

        List<Cidr> split(int newPrefix) {
            if (newPrefix < prefix || newPrefix > 32) {
                throw new IllegalArgumentException("can't split /" + prefix + " into /" + newPrefix);
            }
            int count = 1 << (newPrefix - prefix);
            long step = 1L << (32 - newPrefix);
            List<Cidr> out = new ArrayList<>();
            for (int i = 0; i < count; i++) {
                out.add(new Cidr((int) (Integer.toUnsignedLong(network) + i * step), newPrefix));
            }
            return out;
        }

        @Override public String toString() { return toDotted(network) + "/" + prefix; }
    }

    static int maskOf(int prefix) {
        // NOT simply  -1 << (32 - prefix):  Java masks shift counts to 5 bits,
        // so  -1 << 32  is  -1 << 0  = all ones — a /0 would come out as a /32.
        return prefix == 0 ? 0 : -1 << (32 - prefix);
    }

    static int toInt(String dotted) {
        String[] o = dotted.split("\\.");
        if (o.length != 4) throw new IllegalArgumentException("not a dotted quad: " + dotted);
        int ip = 0;
        for (String part : o) {
            int v = Integer.parseInt(part);
            if (v < 0 || v > 255) throw new IllegalArgumentException("octet out of range: " + dotted);
            ip = (ip << 8) | v;
        }
        return ip;
    }

    static String toDotted(int ip) {
        // >>> (unsigned shift), because 172.x.x.x as an int is negative.
        return (ip >>> 24) + "." + ((ip >>> 16) & 0xFF) + "." + ((ip >>> 8) & 0xFF) + "." + (ip & 0xFF);
    }

    static void describe(String text) {
        Cidr c = Cidr.parse(text);
        System.out.printf("%-18s mask %-15s range %s - %s   %,d addresses, %,d usable, %,d in AWS%n",
                c, toDotted(c.mask()), toDotted(c.network()), toDotted(c.broadcast()),
                c.size(), c.usable(), Math.max(0, c.usableInAws()));
    }

    public static void main(String[] args) {
        System.out.println("== describe");
        for (String s : List.of("172.24.160.0/20", "10.0.1.0/24", "192.168.5.64/26",
                                "10.0.0.0/28", "10.0.0.0/31", "0.0.0.0/0")) {
            describe(s);
        }

        System.out.println("\n== Java's int is signed");
        int wsl = toInt("172.24.174.221");
        System.out.println("172.24.174.221 as int      = " + wsl);
        System.out.println("as unsigned                = " + Integer.toUnsignedLong(wsl));
        System.out.println("-1 << 32 (the shift trap)  = " + (-1 << 32) + "   (expected 0)");

        System.out.println("\n== same subnet as 172.24.160.0/20?");
        Cidr mine = Cidr.parse("172.24.160.0/20");
        for (String ip : List.of("172.24.174.221", "172.24.160.1", "172.24.170.5", "172.24.176.1", "192.168.1.23")) {
            System.out.printf("  %-15s %s%n", ip, mine.contains(toInt(ip)) ? "yes - deliver directly" : "no  - send to gateway");
        }

        System.out.println("\n== overlaps?");
        String[][] pairs = {
            {"172.17.0.0/16", "172.16.0.0/12"},     // Docker's bridge vs the whole private 172 range
            {"10.0.0.0/16",   "10.0.128.0/17"},     // a VPC and a block inside it
            {"10.0.0.0/16",   "10.1.0.0/16"},       // two VPCs that can peer
            {"192.168.1.0/24","192.168.0.0/24"},    // neighbours, not overlapping
        };
        for (String[] p : pairs) {
            System.out.printf("  %-16s vs %-16s %s%n", p[0], p[1],
                    Cidr.parse(p[0]).overlaps(Cidr.parse(p[1])) ? "OVERLAP" : "ok");
        }

        System.out.println("\n== split 10.0.0.0/16 into /20s (first 4 of " +
                Cidr.parse("10.0.0.0/16").split(20).size() + ")");
        Cidr.parse("10.0.0.0/16").split(20).stream().limit(4).forEach(s -> System.out.println("  " + s));

        System.out.println("\n== bad input");
        for (String bad : List.of("10.0.8.0/20", "10.0.1.5/24", "300.1.1.1/8")) {
            try {
                Cidr.parse(bad);
            } catch (IllegalArgumentException e) {
                System.out.println("  rejected: " + e.getMessage());
            }
        }
    }
}
```

Things to notice in the output:

- `172.24.174.221 as int` is **negative** (Part 1). The bit maths still works —
  only *printing* it needs `toUnsignedLong`.
- `-1 << 32` prints `-1`, not `0`. Without the special case in `maskOf`, a `/0`
  ("everywhere") would silently become a `/32` ("one address") — a firewall rule
  that was meant to allow everything would allow almost nothing, or vice versa.
  This is the "show the failure" of the day: a real class of bug in hand-written
  IP code.
- `overlaps` uses the fact from Part 5: compare both networks under the
  *shorter* prefix.

**Extend it (15 minutes):** add a method `Cidr containing(String ip, int
prefix)` that returns the subnet an address belongs to (practice-set question d),
and print `containing("192.168.5.100", 27)`.

### 5. Longest prefix match on your own routing table (lab)

```bash
# The route the kernel picks for each destination.
for dst in 172.24.170.5 172.24.176.1 8.8.8.8; do
  echo "== $dst  -> kernel picks: $(ip route get "$dst" | head -1)"
done
```

Explain each result using the two routes in your table (`172.24.160.0/20` and
`default` = `0.0.0.0/0`) and the longest-prefix rule. `172.24.176.1` is the
interesting one: it's only 1 past your broadcast address, but it goes `via` the
gateway.

### 6. See both NATs (lab)

```bash
echo "WSL (private, behind Windows NAT):  $(ip -4 -o addr show eth0 | awk '{print $4}')"
echo "What the internet sees (public):    $(curl -s --max-time 10 https://httpbin.org/ip | tr -d ' \n{}"' | sed 's/origin://')"
```

And on the Windows side, your Wi-Fi address — the middle of the chain:

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.PrefixOrigin -ne 'WellKnown' } | Select-Object InterfaceAlias, IPAddress, PrefixLength
```

Classify each address you see: private (which RFC 1918 range?) or public. Then
compare the public one with `103.214.63.244` (7 October) — has your ISP changed
it again?
Notice `PrefixLength` on Windows: `24` for Wi-Fi, `20` for the WSL adapter.

### 7. Docker's address range (when Docker Desktop is running)

Docker isn't integrated into your WSL distro, and Docker Desktop wasn't running
when this was written, so this one is for later (Phase 4 relies on it). From
PowerShell with Docker Desktop started:

```powershell
docker network inspect bridge --format '{{range .IPAM.Config}}{{.Subnet}} gw {{.Gateway}}{{end}}'
```

Expect `172.17.0.0/16 gw 172.17.0.1`. Which private range is that in? Does it
overlap your WSL subnet `172.24.160.0/20`? (Use the overlap rule — then confirm
with your Java `overlaps`.)

<details>
<summary>Answer</summary>

It's inside `172.16.0.0/12`. It does **not** overlap `172.24.160.0/20`: under
the shorter prefix `/16`, `172.17.0.0` stays `172.17.0.0` and `172.24.160.0`
becomes `172.24.0.0` — different. Docker and WSL each pick a range that avoids
the networks already on the machine — which is exactly why both chose something
in `172.16/12` that your home `192.168.1.0/24` doesn't touch.

</details>

---

## Gotchas

- **Bigger prefix = smaller network.** `/28` is tiny, `/16` is huge. Everyone
  flips this once under pressure.
- **`172.16.0.0/12` stops at `172.31.255.255`.** `172.32.x.x` and above are
  public.
- **AWS reserves 5 addresses per subnet**, not 2. A `/28` gives 11 usable, and
  `/28` is the smallest subnet AWS allows (`/16` the largest VPC).
- **AWS subnets can't be resized.** You can add secondary CIDR blocks to a VPC
  and create new subnets, but an existing subnet's range is fixed for life. Size
  for the cluster you'll have in two years.
- **Misaligned CIDRs.** `10.0.8.0/20` isn't a valid network. Strict tools reject
  it; lenient ones round it to `10.0.0.0/20` without telling you.
- **Default ranges collide.** `10.0.0.0/16` (every tutorial's VPC), `172.17.0.0/16`
  (Docker), `192.168.0.0/24` and `192.168.1.0/24` (home routers). Using them for
  networks that must later be connected is how overlap bugs are born.
- **Java's `int` is signed; shifts are mod 32.** Use `>>>` to extract octets, and
  special-case a prefix of 0 — `-1 << 32 == -1`.
- **Don't hand IP parsing to `InetAddress.getByName`.** It accepts old BSD
  shorthand forms: `getByName("10.1")` returns `10.0.0.1`, and
  `getByName("10.1.300")` returns **`10.1.1.44`** (300 spills into the last two
  octets) — both checked on your JDK 21. Given anything that isn't a literal IP
  it does a **DNS lookup**, which is slow and may succeed. And regexes miss
  range checks. Parse octets explicitly, or use a
  library (e.g. Apache Commons `SubnetUtils`, or Spring Security's
  `IpAddressMatcher`, which you'll meet in Phase 8).
- **WSL's subnet is chosen at boot.** It's been `172.24.160.0/20` for this whole
  phase so far, but WSL picks a range that doesn't collide with your other
  networks — join a VPN or a new Wi-Fi and it may move.
- **"Public subnet" isn't about the addresses.** In AWS every subnet uses private
  addresses; public/private is decided by the route table.

---

## Recap

- An IPv4 address is a **32-bit number**; the dots are for humans. Java holds it
  in a signed `int`.
- **`/N`** = the first N bits are the network part. `address AND mask` = network
  address; all host bits 1 = broadcast; in between are usable.
- Size = **2^(32 − N)**. `/24` = 256, `/20` = 4,096, `/16` = 65,536. Each +1
  halves. AWS takes 5 per subnet.
- **Block-size method:** find the interesting octet, block = 256 − mask value,
  round down to a multiple of the block.
- **Subnetting:** n extra prefix bits → 2^n subnets, aligned to their own size.
  Size AWS/EKS subnets for growth — they can't be resized, and on EKS every pod
  eats one.
- **Private ranges:** `10/8`, `172.16/12`, `192.168/16`. Plus `127/8` loopback,
  `169.254/16` link-local (AWS metadata), `100.64/10` CGNAT.
- **Overlap** breaks peering, VPNs and Docker; routes are chosen by **longest
  prefix match**.
- **NAT/PAT** rewrites source IP and port outbound and tracks it in a table;
  unsolicited inbound is dropped. You're behind two. In AWS: public subnet =
  `0.0.0.0/0 → Internet Gateway`, private = `0.0.0.0/0 → NAT Gateway`.
- **IPv6:** 128-bit hex, `/64` LANs, no NAT needed, increasingly relevant
  because AWS now charges for public IPv4.

---

## Say it out loud

1. What does `10.0.1.0/24` mean? Give the network, broadcast, usable range and
   count — and how many usable in AWS.
2. Walk through how you'd find the subnet that `172.24.174.221/20` belongs to,
   without binary.
3. What makes an AWS subnet public versus private? (Hint: it's not the CIDR.)
4. How does NAT let a whole office share one public IP, and why can't someone on
   the internet connect to a laptop in that office?
5. Two teams want to peer their VPCs and both used `10.0.0.0/16`. What happens,
   and how would you have prevented it?
6. Why would you choose `/19` subnets over `/24` subnets for an EKS cluster?

---

## Q&A

_Follow-up questions from reading this lesson go here._
