# Day 18 — curl Mastery & Packet Inspection

_Why this matters:_ When an API misbehaves, the fastest route to the truth is
to take your application out of the picture and talk to the endpoint directly.
`curl` is how: it's on every Linux box, every container image worth debugging,
every CI runner, and it can reproduce any request your Java code makes — with
exactly the headers, body, auth and target IP you choose. Its `-w` option
answers the question that comes up in every latency incident, **"which part is
slow — DNS, the connection, TLS, or the server?"**, in one line. And when even
`curl` isn't enough — when you need to know what *actually* crossed the wire —
`tcpdump` shows you the packets. You've used both tools for a week; today you
learn them properly.

> **Two halves.** Parts 1–6 are `curl` (all lab-runnable). Parts 7–8 are
> `tcpdump`, which needs `sudo`, so those exercises run in your Ubuntu terminal.
> The required practice is hands-on 5 and 9.

---

## The one-paragraph version

`curl` builds an HTTP request from flags — method (`-X`), headers (`-H`), body
(`-d`, `--data-binary`, `-F`), credentials (`-u`, an `Authorization` header) —
sends it, and shows you as much of the exchange as you ask for (`-i` headers,
`-v` everything). You can steer **where** it connects regardless of DNS
(`--resolve`), **how long** it waits (`--connect-timeout`, `--max-time`), and save
the whole recipe in a config file to replay it (`-K`). Most usefully, `-w` prints
timestamps for each phase of the request — name lookup, TCP connect, TLS done,
first byte back, finished — **each measured from the start**, so the differences
between them tell you exactly where the time went. `tcpdump` sits underneath all
that: it captures the packets on an interface, filtered by host, port or protocol,
and prints them or saves them to a file you can read later. On plain HTTP and DNS
it shows you everything. On HTTPS it shows you who talked to whom, when, how much,
and — via SNI — which site, but not a single byte of the request itself.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **`-w` / write-out** | A template curl fills in with facts about the request after it finishes | `%{variable}` placeholders: timings, status, sizes, IPs |
| **format file** | The `-w` template saved in a file | `-w @timing.txt` |
| **time_namelookup** | When DNS finished | Seconds from the start |
| **time_connect** | When the TCP handshake finished | Seconds from the start (includes DNS) |
| **time_appconnect** | When the TLS handshake finished | Seconds from the start; 0 for plain HTTP |
| **time_starttransfer (TTFB)** | When the first byte of the response arrived | "Time to first byte" — includes the server's processing |
| **time_total** | When the last byte arrived | |
| **config file** | A file of curl options, one per line | Read with `-K file`; `~/.curlrc` is read automatically |
| **multipart form** | The body format for file uploads | `multipart/form-data`, sent with `-F` |
| **URL encoding** | Escaping special characters in URLs and forms | `space` → `%20`; `--data-urlencode` |
| **packet capture** | A recording of the packets on an interface | What `tcpdump` (and Wireshark) produce |
| **pcap** | The standard file format for a capture | `tcpdump -w file.pcap`, read with `-r` |
| **BPF filter** | The small expression saying which packets to capture | Berkeley Packet Filter syntax: `tcp port 443 and host httpbin.org` |
| **snap length** | How many bytes of each packet to keep | `-s 0` = whole packet |
| **Wireshark** | A graphical packet analyser | Opens pcap files; decodes every protocol |

---

## Before you read: three questions

1. A call to a partner API takes 2.4 seconds. Without touching any code, how do
   you find out whether the time goes on DNS, the connection, TLS, or the
   partner's server?
2. You upload a CSV with `curl -d @orders.csv …`. The server receives one long
   line. What happened?
3. The security team asks: "If someone captures our HTTPS traffic, what can they
   still learn?" What do you tell them?

---

## Part 1 — Seeing the exchange: output flags

**In plain words:** by default `curl` prints only the response body. Everything
else — status, headers, what it sent, what went wrong — is hidden unless you ask.
The output flags decide how much you see and where it goes.

| Flag | Shows / does | Use it when |
|------|--------------|-------------|
| (none) | Body only | Piping JSON into `jq` |
| `-i` | Response headers + body | Checking status and headers quickly |
| `-I` | Headers only (sends a **`HEAD`** request) | Is it up? What would it send? (Some APIs don't support HEAD — use `-i -o /dev/null` or `-D -` instead) |
| `-v` | Everything: connection, TLS, request (`>`), response (`<`) | Debugging — Day 16 and 17 |
| `-s` | Silent: no progress bar or errors | Scripts |
| `-S` | With `-s`: show errors anyway | Always pair as `-sS` in scripts |
| `-o FILE` | Write the body to a file (`-o /dev/null` to discard) | Downloads; timing-only runs |
| `-O` | Write the body to a file named like the URL's last part | Downloading `…/robots.txt` → `robots.txt` |
| `-D -` | Dump response headers to stdout (works with any method) | Headers *and* body, separately |
| `-L` | Follow redirects | Day 16 |
| `-f` / `--fail` | Exit with code 22 on HTTP ≥ 400 (and print no body) | Scripts and health checks |
| `--fail-with-body` | Like `-f`, but still prints the error body | When the error JSON matters |

The script default to memorise: **`curl -sS -f`** — quiet, errors visible, non-zero
exit status on HTTP errors. Without `-f`, `curl` exits **0** on a `500`, and your
script carries on happily (the Day 9 lesson: fail loudly).

```bash
# Exit status: without -f a 500 is "success"; with -f it isn't.
curl -s -o /dev/null https://httpbin.org/status/500;    echo "without -f: exit $?"
curl -sS -f -o /dev/null https://httpbin.org/status/500; echo "with -f:    exit $?"
cd "$(mktemp -d)" && curl -sS -O https://httpbin.org/robots.txt && ls -l robots.txt
```

> **Say this in an interview:** "In scripts I use `curl -sS -f`, so errors are
> printed and HTTP 4xx/5xx produce a non-zero exit status — by default curl exits
> zero on a 500. `-i` shows response headers, `-v` shows the whole exchange
> including TLS, and `-o /dev/null` with `-w` gives me just the metrics."

---

## Part 2 — Sending things: method, headers, body

**In plain words:** to reproduce what your Java code sends, you control three
things: the method, the headers, and the body. The trap is that some body flags
quietly change your data or add headers you didn't ask for.

| Flag | What it does | Watch out for |
|------|--------------|---------------|
| `-X PUT` | Sets the method | Unneeded with `-d` (implies POST) or `-I` (HEAD) |
| `-H 'Name: value'` | Adds/replaces a header | `-H 'Accept:'` (empty) *removes* one |
| `-d 'a=1'` / `-d @file` | Body; **sets `Content-Type: application/x-www-form-urlencoded`**; implies POST | **`-d @file` strips newlines and carriage returns** |
| `--data-binary @file` | Body exactly as the file is, byte for byte | Set `Content-Type` yourself |
| `--data-urlencode 'q=a b&c'` | URL-encodes the value | For form fields with special characters |
| `-G` | Puts `-d` data in the URL query string instead (GET) | `curl -G -d q=java https://…/search` |
| `-F 'file=@report.pdf'` | `multipart/form-data` upload | What HTML file-upload forms send |
| `--json '{…}'` | Body + JSON headers in one flag | **Not in your curl** — added in 7.82; you have 7.81. Use `-H 'Content-Type: application/json' -d '…'` |

The newline trap, on your machine:

```bash
cd "$(mktemp -d)"
printf 'line one\nline two\n' > body.txt
echo "== -d @file (newlines stripped, sent as a form):"
curl -s -d @body.txt https://httpbin.org/post | grep -A1 '"form"'
echo "== --data-binary @file (byte for byte):"
curl -s --data-binary @body.txt -H 'Content-Type: text/plain' https://httpbin.org/post | grep '"data"'
```

```
== -d @file (newlines stripped, sent as a form):
  "form": {
    "line oneline two": ""
== --data-binary @file (byte for byte):
  "data": "line one\nline two\n",
```

That's question 2: `-d @file` joined the lines **and** labelled the body as a
form. For files, always `--data-binary` (or `-F` for uploads).

The JSON pattern you'll use daily (until you have `--json`):

```bash
curl -sS -f -X POST https://httpbin.org/post \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  --data-binary '{"item":"book","qty":2}' | grep -A3 '"json"'
```

> **Say this in an interview:** "`-d` sends a form-encoded body and strips
> newlines from files; `--data-binary` sends bytes exactly; `-F` does multipart
> uploads. For JSON I set `Content-Type` explicitly and use `--data-binary`."

---

## Part 3 — Authentication

**In plain words:** most APIs want proof of who you are in every request — a
username and password, or a token. `curl` can send both, but the way you pass the
secret matters, because command lines are not private.

```bash
# Basic auth: curl builds the header for you.
curl -s -u user:pass https://httpbin.org/basic-auth/user/pass
# Bearer token: you build the header. Take the token from a variable, never type it inline.
TOKEN=my-test-token
curl -s -H "Authorization: Bearer $TOKEN" https://httpbin.org/bearer
```

| Method | Flag | What's sent |
|--------|------|-------------|
| Basic | `-u user:pass` | `Authorization: Basic dXNlcjpwYXNz` (Base64 — Day 16) |
| Basic, prompt for password | `-u user` | curl asks; nothing in history |
| Bearer token | `-H "Authorization: Bearer $TOKEN"` | the token |
| Header from a file | `-H @headers.txt` | each line of the file as a header |

Where secrets leak:

- **Shell history.** `curl -H 'Authorization: Bearer eyJ…'` is now in
  `~/.bash_history`. Use a variable set from a file or a prompt
  (`read -rs TOKEN`), or `-H @file`.
- **The process list.** While curl runs, its full command line is visible to every
  user on the machine via `ps aux`. On shared machines and CI runners, prefer
  `-H @file` or a config file (Part 6).
- **`-v` output.** It prints your `Authorization` header. Don't paste `-v` output
  into tickets or chat without redacting it.

> **Say this in an interview:** "Basic auth with `-u`, bearer tokens with an
> `Authorization` header. I keep tokens out of the command line — from an
> environment variable, a header file, or a config file — because command lines
> end up in shell history and are visible in `ps`."

---

## Part 4 — Where it connects, and how long it waits

**In plain words:** sometimes you need to send a request to one specific server —
the new one you're about to switch DNS to, or the one backend out of three that's
misbehaving — while still using the real hostname, so TLS and `Host` work.
`--resolve` overrides DNS for a single command. And by default curl will wait a
very long time; in scripts you always set limits.

| Flag | Meaning |
|------|---------|
| `--resolve httpbin.org:443:3.90.119.127` | For this command, `httpbin.org` on port 443 **is** `3.90.119.127`. SNI, the certificate check and `Host` all still use `httpbin.org` |
| `--connect-to httpbin.org:443:other.host:8443` | Connect to a different host/port entirely, while still requesting `httpbin.org` |
| `--connect-timeout 3` | Give up if the TCP (+TLS) connection takes more than 3 s |
| `--max-time 10` | Give up if the **whole** request takes more than 10 s |
| `--retry 3` | Retry *transient* failures — timeouts and HTTP 408, 429, 500, 502, 503, 504 — with backoff |
| `-4` / `-6` | Force IPv4 / IPv6 |

```bash
# Pin httpbin.org to one of its 8 IPs (Day 14) — the certificate still validates.
curl -s -o /dev/null -w 'connected to %{remote_ip}, HTTP %{http_code}\n' \
  --resolve httpbin.org:443:3.90.119.127 https://httpbin.org/get
# Compare: an IP in the URL fails the certificate's name check (Day 17).
curl -sS -o /dev/null https://3.90.119.127/get 2>&1 | head -1
```

This is the right way to test a server before a DNS cut-over (Day 14), to hit one
backend behind round-robin DNS, or to reproduce Day 17's question 2 without the
certificate error.

Why both timeouts: `--connect-timeout` catches the Day 13 two-minute SYN hang;
`--max-time` catches a server that accepts the connection and then never answers
(Day 16's 504 situation). A health check needs both.

> **Say this in an interview:** "`--resolve` pins a hostname to an IP for one
> request while keeping SNI, certificate validation and the Host header correct —
> ideal for testing a specific backend or a migration target. I always set
> `--connect-timeout` and `--max-time` in scripts, and `--retry` only retries
> transient failures."

---

## Part 5 — `-w`: which part is slow?

**In plain words:** a request happens in stages — look up the name, connect, set up
TLS, wait for the server to think, download the answer. `-w` tells you the moment
each stage *finished*, all measured from the same starting gun. Subtract
neighbouring numbers and you get how long each stage took. Whichever stage is big
is your problem.

Save this as a format file (hands-on 5 does it for you):

```
     dns lookup:  %{time_namelookup}s\n
    tcp connect:  %{time_connect}s\n
  tls handshake:  %{time_appconnect}s\n
   request sent:  %{time_pretransfer}s\n
     first byte:  %{time_starttransfer}s\n
          total:  %{time_total}s\n
   http %{http_code}, %{size_download} bytes from %{remote_ip}:%{remote_port}\n
```

Your local server vs httpbin, measured while writing this lesson:

```
                 local (127.0.0.1:8000)    remote (httpbin.org, HTTPS)
dns lookup:      0.000032s                 0.018153s
tcp connect:     0.000300s                 0.271121s
tls handshake:   0.000000s  (plain HTTP)   1.151447s
request sent:    0.000356s                 1.151603s
first byte:      0.002550s                 1.581023s
total:           0.002617s                 1.581148s
```

**The numbers are cumulative** — each is "seconds since the start", not "seconds
this phase took". So read the *gaps*:

| Phase | Calculation | httpbin | What a big number means |
|-------|-------------|--------:|--------------------------|
| DNS | `namelookup` | 0.018 | Slow resolver, or cache misses (Day 14) |
| TCP connect | `connect − namelookup` | 0.253 | **≈ 1 RTT** (Day 13). Much more = far away, or a lost SYN retransmitted (+1 s) |
| TLS | `appconnect − connect` | 0.880 | TLS 1.2's 2 RTTs + server crypto (Day 17). Huge = slow server, or no session reuse |
| Server | `starttransfer − pretransfer` | 0.429 | **TTFB: the server's think time** + 1 RTT. Your app, its database, its downstreams |
| Download | `total − starttransfer` | 0.000 | Big response or slow bandwidth |

Same tool on `/delay/2` (the server deliberately waits 2 s) shows the server gap
jumping by ~2 s while the connection phases stay the same — the signature of "it's
the backend, not the network".

That answers question 1: run the request once with the format file. If `connect`
is large, it's the network path; if `appconnect − connect` is large, it's TLS; if
`starttransfer − pretransfer` is large, it's their server; and you can send that
breakdown to the partner instead of "your API is slow".

Remember that the network to the US is noisy from your connection — run any
comparison several times and look at the pattern, not one number.

> **Say this in an interview:** "`curl -w` with a format file gives cumulative
> timestamps: namelookup, connect, appconnect, pretransfer, starttransfer and
> total. Connect minus namelookup is roughly one RTT; appconnect minus connect is
> the TLS handshake; starttransfer minus pretransfer is time to first byte — the
> server's processing. That tells you immediately whether latency is network, TLS
> or backend."

---

## Part 6 — Reusable requests: config files

**In plain words:** a long curl command with six headers and a body is hard to
retype and easy to get subtly wrong. Save it in a file once, and replay it with
one flag — and the secrets in it stay out of your shell history and the process
list.

A config file has one option per line, long names without the `--`:

```
# req.curl — a saved request
url = "https://httpbin.org/bearer"
header = "Authorization: Bearer my-test-token"
silent
write-out = "%{http_code}\n"
output = "/dev/null"
```

Run it with `curl -K req.curl`. Anything on the command line is added, so
`curl -K req.curl -v` replays it verbosely.

- **`~/.curlrc` is read by every `curl` you run** — handy for defaults like
  `--connect-timeout`, dangerous if it contains something you forget about
  (`insecure`!). `curl -q` (as the first argument) ignores it.
- For a family of requests against one API, a small bash function works too:

  ```
  api() { curl -sS -f -H "Authorization: Bearer $TOKEN" -H 'Accept: application/json' "https://api.example.com$1" "${@:2}"; }
  api /orders/17
  api /orders -X POST --data-binary @order.json -H 'Content-Type: application/json'
  ```

> **Say this in an interview:** "I keep repeatable requests in curl config files
> run with `-K`, which also keeps tokens out of history and `ps`. I avoid putting
> risky options in `~/.curlrc` because every curl invocation picks it up."

---

## Part 7 — `tcpdump`: capturing, filtering, reading

**In plain words:** `curl` tells you what *it* thinks happened. `tcpdump` records
what *actually* went over the network card — every packet, with timestamps — so
you can see things no application will tell you: retransmissions, resets, which
DNS server really answered, whether a packet left the machine at all. The skill is
filtering: a busy machine sends thousands of packets a second, and you want the
dozen that matter.

You've been using it since Day 11. The anatomy of a command:

```
sudo tcpdump -i eth0 -nn -c 20 -w out.pcap  'tcp port 443 and host httpbin.org'
             └ where  └ no   └ stop └ save   └ the filter (BPF) — always quote it
                        names  after   to file
```

### Options

| Option | Meaning |
|--------|---------|
| `-i eth0` / `-i lo` / `-i any` | Interface. Loopback traffic is only on `lo` (Day 11) |
| `-nn` | Don't resolve names or port numbers — faster, and no extra DNS traffic in your capture |
| `-c N` | Stop after N packets |
| `-e` | Show MAC addresses (Day 11) |
| `-v` / `-vv` | More decoding (IP TTL, DNS details) |
| `-A` / `-X` | Print payload as ASCII / hex+ASCII |
| `-s 0` | Capture whole packets (the default on modern tcpdump, but explicit is safer) |
| `-w file.pcap` | Write raw packets to a file instead of printing |
| `-r file.pcap` | Read a saved file — **no `sudo` needed** |

### Filters (BPF)

| Filter | Captures |
|--------|----------|
| `host 8.8.8.8` | To or from that IP |
| `src host X` / `dst host X` | One direction |
| `net 172.24.160.0/20` | A whole subnet (Day 12) |
| `port 53` / `tcp port 443` / `udp port 53` | By port and protocol |
| `portrange 8000-8100` | A range |
| `icmp` | Pings, unreachables, time-exceeded |
| `and` / `or` / `not`, parentheses | Combine — quote the whole filter |
| `'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'` | Only connection starts, ends and resets — a compact view of connection activity |

### Reading a DNS capture

Hands-on 8 captures DNS while you run `dig`. On your machine:

```
172.24.174.221.49267 > 172.24.160.1.53: 19889+ [1au] A? github.com. (51)
172.24.160.1.53 > 172.24.174.221.49267: 19889-$ 1/0/0 A 20.207.73.82 (54)
172.24.174.221.57475 > 8.8.8.8.53: 54623+ [1au] MX? example.org. (52)
8.8.8.8.53 > 172.24.174.221.57475: 54623$ 1/0/1 MX . 0 (55)
```

| Piece | Meaning |
|-------|---------|
| `.49267 > …53` | Your random source port to DNS port 53, over **UDP** (Day 13) |
| `19889` | Query ID — the reply carries the same ID; that's how they're matched |
| `+` | Recursion desired (`rd` in `dig`, Day 14) |
| `[1au]` | One "additional" record in the query — the EDNS option `dig` adds |
| `A?` / `MX?` | The question type |
| `-` (in a reply) | Recursion **not** available — the WSL proxy, exactly as `dig` warned on Day 14 |
| `$` | Authenticated-data flag (`ad` in `dig`) |
| `1/0/0` | Answer / authority / additional record counts |
| `(51)` | Size in bytes |

Compare the two replies: the Windows proxy's has the `-`, Google's doesn't. Day 14's
"recursion requested but not available" warning, visible on the wire.

### Wireshark

For anything bigger than a few dozen packets, save with `-w` and open the file in
**Wireshark** on Windows (free; open `\\wsl$\Ubuntu\home\<you>\capture.pcap` from
Windows Explorer). It decodes every field of every protocol and can follow a TCP
stream. Same packets, much easier to read.

> **Say this in an interview:** "I capture with `tcpdump -nn -i <iface>` and a
> tight BPF filter — host, port, protocol — usually writing to a pcap with `-w` to
> analyse in Wireshark. `-nn` avoids name lookups polluting the capture, and the
> filter `tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0` gives a quick view of
> connection setup, teardown and resets."

---

## Part 8 — What a capture can (and can't) show you on HTTPS

**In plain words:** TLS encrypts the *contents* of the conversation, not the fact
that it happened. A capture of HTTPS is like watching sealed envelopes go through a
sorting office: you can't read the letters, but you can see the addresses, the
times, how many envelopes, how thick each one is — and, with HTTPS, the name of the
site is written on the outside of the first envelope.

I captured one `curl https://httpbin.org/get` to a pcap file and searched it:

```
$ tcpdump -nn -A -r https.pcap | grep -c 'GET /get'
0                                      ← the request: invisible
$ tcpdump -nn -A -r https.pcap | grep -o 'httpbin\.org'
httpbin.org                            ← in the ClientHello (SNI)
httpbin.org                            ← in the certificate (TLS 1.2 sends it unencrypted)
httpbin.org
```

The same request over plain `http://` shows everything: `GET /get HTTP/1.1`,
`Host: httpbin.org`, every header, the body.

| Visible in an HTTPS capture | Hidden |
|-----------------------------|--------|
| Source and destination IPs and ports | The URL path and query string |
| Timing of every packet; the TCP handshake, retransmissions, resets | Every request and response header, including `Authorization` and cookies |
| How many bytes each way — often enough to guess what was fetched | Request and response bodies |
| **The hostname, via SNI** (plaintext in the ClientHello) | The HTTP method and status code |
| The TLS version and cipher suite chosen | |
| The server's certificate — on **TLS 1.2** (1.3 encrypts it) | |
| DNS queries for the name — unless DNS is encrypted too | |

That's question 3. And the things that *are* visible are exactly what you need for
most network debugging: did the SYN get an answer, did TLS complete, who sent the
RST, where did the delay happen.

### When you need to see inside your own HTTPS

- Debug at a point where it's already plaintext: the app's own logs, or behind the
  TLS-terminating load balancer (Day 17, Part 8).
- **`SSLKEYLOGFILE`:** set this environment variable and `curl` (and browsers)
  write the session keys to that file; Wireshark can then decrypt a capture of
  those sessions. Only works for programs that support it — the JDK doesn't
  natively — and the key file is as sensitive as the traffic.

> **Say this in an interview:** "A capture of HTTPS shows IPs, ports, timing,
> sizes, the TLS handshake, the SNI hostname and, with TLS 1.2, the certificate —
> but not the method, URL, headers or bodies. That's still enough to debug most
> connection problems. To inspect payloads you either capture where TLS is already
> terminated or use a key log with tools that support `SSLKEYLOGFILE`."

---

## Hands-on

Blocks marked **lab** run with the Run button; **Ubuntu terminal** ones need `sudo`.

### 1. Output flags drill (lab)

Run the Part 1 block. Then, using only curl flags (no `grep`), produce:

1. Only the status code of `https://httpbin.org/status/418`.
2. Only the response headers of `https://httpbin.org/get`, without sending a HEAD
   request.
3. The body of `https://httpbin.org/status/404`'s error, *and* a non-zero exit
   status.

<details>
<summary>Answers</summary>

1. `curl -s -o /dev/null -w '%{http_code}\n' https://httpbin.org/status/418`
2. `curl -s -D - -o /dev/null https://httpbin.org/get`
3. `curl -sS --fail-with-body https://httpbin.org/status/404; echo "exit $?"` —
   httpbin's 404 here has an empty body, but the exit status is 22.

</details>

### 2. Bodies that survive the trip (lab)

Run the Part 2 newline block and the JSON block. Then send the same two-line file
with `-F 'file=@body.txt'` to `https://httpbin.org/post` and find where it appears
in httpbin's reply (`"files"`). Which `Content-Type` did `-F` set? (Add `-v` and
look for it.)

### 3. Tokens without leaking them (lab)

Run the Part 3 block. Then put the header in a file and send it from there:

```bash
cd "$(mktemp -d)"
printf 'Authorization: Bearer from-a-file\n' > auth.header
chmod 600 auth.header
curl -s -H @auth.header https://httpbin.org/bearer
```

Why `chmod 600` — and why does that help less on `/mnt/d` than in `~` (Day 9)?

### 4. Pin a request to one backend (lab)

Run the Part 4 block. Then:

1. Get httpbin's current IPs with `dig +short @8.8.8.8 httpbin.org`, and use
   `--resolve` to send one request to each of three of them, printing
   `%{remote_ip}` and `%{http_code}`.
2. Think this one through without running it: what would
   `curl --resolve httpbin.org:443:127.0.0.1 https://httpbin.org/` fail with if
   nothing listens on 443 locally? And if Day 17's `localhost` HTTPS server were
   listening there?

<details>
<summary>Answer to 2</summary>

With nothing on 443: `Connection refused` (exit 7) — instantly, at the TCP rung
(Day 15). With Day 17's server there: the TCP connect succeeds, but TLS fails
twice over — its certificate isn't from a trusted CA, and it's for `localhost`, not
`httpbin.org`. `--resolve` changes *where* you connect, never *who you expect to
find there*.

</details>

### 5. The timing breakdown: local vs remote (lab — required practice)

```bash
cd "$(mktemp -d)"
# The format file: each value is seconds since the START of the request.
cat > timing.txt <<'EOF'
     dns lookup:  %{time_namelookup}s\n
    tcp connect:  %{time_connect}s\n
  tls handshake:  %{time_appconnect}s\n
   request sent:  %{time_pretransfer}s\n
     first byte:  %{time_starttransfer}s\n
          total:  %{time_total}s\n
   http %{http_code}, %{size_download} bytes from %{remote_ip}:%{remote_port}\n
EOF

python3 -m http.server 8000 --bind 127.0.0.1 >/dev/null 2>&1 &
server=$!
sleep 1
echo "== local:  http://127.0.0.1:8000/";      curl -s -o /dev/null -w @timing.txt http://127.0.0.1:8000/
echo "== remote: https://httpbin.org/get";     curl -s -o /dev/null -w @timing.txt https://httpbin.org/get
echo "== remote, slow server: /delay/2";      curl -s -o /dev/null -w @timing.txt https://httpbin.org/delay/2
kill "$server"
```

For each of the three, compute the five gaps from the Part 5 table (DNS, TCP,
TLS, server, download). Then answer:

1. Why is the local `tls handshake` exactly `0`?
2. In the `/delay/2` run, which gap grew by ~2 s? What would you tell the API's
   owner?
3. The remote `tcp connect` gap should be close to the RTT from Day 13's
   `ss -ti` (~245 ms). Is it? If a run shows ~1.25 s instead, what happened?
4. Save `timing.txt` into `~` permanently and add `alias curlt='curl -s -o
   /dev/null -w @$HOME/timing.txt'` to your `~/.bashrc`. You'll use it for the
   rest of the course.

<details>
<summary>Answers</summary>

1. Plain `http://` — there is no TLS, so curl reports 0.
2. The server gap (`first byte − request sent`). Tell them: "DNS, TCP and TLS are
   normal; your server takes ~2 s to produce the first byte" — with the numbers.
3. A connect gap of about RTT + 1 s means the first SYN was lost and retransmitted
   after the initial 1-second retransmission timeout (Day 13). One occurrence is
   noise; a pattern is packet loss on the path.

</details>

### 6. The same breakdown from Java (lab)

`curl -w`'s phases measured by hand in Java: DNS (`InetAddress`), TCP (`Socket`),
TLS (`startHandshake`), first byte, rest. Takes ~5–8 seconds.

```java
import java.io.InputStream;
import java.io.OutputStream;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.nio.charset.StandardCharsets;
import javax.net.ssl.SSLParameters;
import javax.net.ssl.SSLSocket;
import javax.net.ssl.SSLSocketFactory;

/** curl -w, by hand: time each phase of one HTTPS request separately. */
public class PhaseTimer {

    public static void main(String[] args) throws Exception {
        time("httpbin.org", "/get");
        time("httpbin.org", "/delay/1");          // the server deliberately waits 1 s
    }

    static void time(String host, String path) throws Exception {
        long t0 = System.nanoTime();

        InetAddress ip = InetAddress.getByName(host);                     // DNS
        long dns = System.nanoTime();

        Socket tcp = new Socket();
        tcp.connect(new InetSocketAddress(ip, 443), 5_000);              // TCP handshake: 1 RTT
        long connect = System.nanoTime();

        SSLSocket tls = (SSLSocket) ((SSLSocketFactory) SSLSocketFactory.getDefault())
                .createSocket(tcp, host, 443, true);
        SSLParameters p = tls.getSSLParameters();
        p.setEndpointIdentificationAlgorithm("HTTPS");                   // check the hostname (Day 17)
        tls.setSSLParameters(p);
        tls.setSoTimeout(10_000);
        tls.startHandshake();                                            // TLS handshake
        long handshake = System.nanoTime();

        OutputStream out = tls.getOutputStream();
        out.write(("GET " + path + " HTTP/1.1\r\nHost: " + host + "\r\nConnection: close\r\n\r\n")
                .getBytes(StandardCharsets.US_ASCII));
        out.flush();
        InputStream in = tls.getInputStream();
        int first = in.read();                                           // wait for the first byte of the response
        long firstByte = System.nanoTime();
        byte[] rest = in.readAllBytes();                                 // the rest of the response
        long total = System.nanoTime();
        tls.close();

        String status = ((char) first + new String(rest, 0, Math.min(rest.length, 40), StandardCharsets.US_ASCII))
                .lines().findFirst().orElse("?");
        System.out.printf("== https://%s%s  (%s)%n", host, path, status);
        row("dns lookup", t0, dns);
        row("tcp connect", dns, connect);
        row("tls handshake", connect, handshake);
        row("server time (TTFB)", handshake, firstByte);
        row("download", firstByte, total);
        row("TOTAL", t0, total);
        System.out.println();
    }

    static void row(String phase, long from, long to) {
        System.out.printf("  %-20s %6d ms%n", phase, (to - from) / 1_000_000);
    }
}
```

One run on your machine:

```
== https://httpbin.org/get  (HTTP/1.1 200 OK)
  dns lookup               21 ms
  tcp connect             295 ms
  tls handshake          1841 ms
  server time (TTFB)      476 ms
  download                 44 ms
  TOTAL                  2679 ms

== https://httpbin.org/delay/1  (HTTP/1.1 200 OK)
  dns lookup                0 ms
  tcp connect            1052 ms
  tls handshake          1221 ms
  server time (TTFB)     1917 ms
  download                  0 ms
  TOTAL                  4192 ms
```

Explain, using this week's material, each of: the second DNS lookup taking 0 ms;
the first TLS handshake taking much longer than curl's; the second TCP connect
taking 1,052 ms; the second TTFB being ~1.4 s more than the first.

<details>
<summary>Answers</summary>

- **DNS 0 ms:** the JVM's own 30-second DNS cache (Day 14).
- **First TLS handshake ~1.8 s:** the first TLS connection in a JVM also pays
  one-time costs — loading and initialising the crypto classes and the trust store
  — on top of the network round trips. Later handshakes in the same JVM are cheaper.
  (Connection pooling avoids the handshake entirely — Day 13.)
- **TCP 1,052 ms:** ~1 s more than the RTT — the first SYN was lost and retransmitted
  after the initial retransmission timeout (Windows, like Linux, starts at ~1 s).
- **TTFB +1.4 s:** `/delay/1` waits 1 s on purpose; the rest is noise on a busy
  path. Run it three times before believing any single number.

</details>

### 7. A saved request (lab)

```bash
cd "$(mktemp -d)"
cat > req.curl <<'EOF'
# A saved request: one option per line, long names without the dashes
url = "https://httpbin.org/bearer"
header = "Authorization: Bearer my-test-token"
silent
write-out = "HTTP %{http_code}\n"
output = "/dev/null"
EOF
curl -K req.curl
curl -K req.curl -w 'HTTP %{http_code} in %{time_total}s\n'    # a later -w replaces the file's
curl -K req.curl -v 2>&1 | grep -iE '^> (GET|authorization)'   # extra flags add to it
```

(Not every option combines the way you'd guess: adding `-o -` here does **not**
print the body — the file's `output` setting still wins. When in doubt, check
with `-v`.)

Change the token line to an empty `header = "Authorization:"` and rerun. What
status do you get, and which Day 16 status-code rule does it demonstrate?

### 8. Capture DNS while running `dig` (Ubuntu terminal, `sudo` — required practice)

```
sudo -v
sudo tcpdump -i eth0 -nn -c 4 'udp port 53' &
sleep 1
dig +short github.com
dig +short @8.8.8.8 example.org MX
wait
```

Match your four lines to the Part 7 table. Then:

1. Which reply came from the WSL proxy and which from Google — and which flag
   character tells you, without looking at the IPs?
2. Run `sudo tcpdump -i eth0 -nn -v -c 2 'udp port 53'` with
   `dig +short @8.8.8.8 httpbin.org`. How many A records does the reply carry, and
   what's the IP TTL on Google's reply packet (not the DNS TTL)?
3. What would you add to the filter to capture only the *queries*, not the replies?

<details>
<summary>Answers</summary>

1. The reply with `-` after the ID came from the WSL proxy (recursion not
   available); Google's has no `-`.
2. Eight A records (Day 14). The IP TTL was 121 on your test — Google's
   servers start at 128 and the reply crossed 7 routers.
3. `'udp dst port 53'`.

</details>

### 9. What HTTPS hides — and doesn't (Ubuntu terminal, `sudo` — required practice)

Capture one HTTPS request and one HTTP request to files, then analyse the files
**without** `sudo`:

```
cd ~
sudo -v
# timeout stops each capture after 6 s. (Not "sudo kill %1": %1 is a shell job number,
# which only the shell's own kill understands — sudo would pass the literal "%1".)
sudo timeout 6 tcpdump -i eth0 -nn -s 0 -w https.pcap 'tcp port 443 and host httpbin.org' &
sleep 1; curl -s -o /dev/null https://httpbin.org/get; wait

sudo timeout 6 tcpdump -i eth0 -nn -s 0 -w http.pcap 'tcp port 80 and host httpbin.org' &
sleep 1; curl -s -o /dev/null http://httpbin.org/get; wait

ls -l https.pcap http.pcap                   # owned by user "tcpdump" — it drops root after starting
sudo chown "$USER" https.pcap http.pcap      # so you can delete them later

# From here on, no sudo: reading a capture needs no privileges
tcpdump -nn -r https.pcap | head -12
tcpdump -nn -A -r https.pcap | grep -c 'GET /get'        # 0
tcpdump -nn -A -r https.pcap | grep -o 'httpbin\.org'   # SNI + certificate
tcpdump -nn -A -r http.pcap  | grep -E 'GET /get|Host:|User-Agent'
tcpdump -nn -r https.pcap 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'
```

Then answer:

1. In the HTTPS capture, find the packet that carries the ClientHello (hint: the
   first packet *from* you with data after the handshake — `seq 1:518` on my run).
   About how big was it, and why is SNI inside it readable?
2. Which side sent the first `FIN`? Which side will hold `TIME_WAIT` (Day 13)?
3. Open `https.pcap` in Wireshark on Windows (`\\wsl$\Ubuntu\home\<you>\https.pcap`
   in Explorer, if you have Wireshark). Find the *Server Name Indication* field in
   the Client Hello.

### 10. The exam question (paper)

A user reports: "Checkout is slow, sometimes 6 seconds." You have `curl`, a
format file, and `tcpdump`. Write the three commands you'd run first, and what
result from each would point you at (a) DNS, (b) packet loss, (c) the backend.

<details>
<summary>One good answer</summary>

1. `curlt https://checkout.example.com/health` several times (the Part 5 format
   file). Large `dns lookup` → **DNS**. Large and variable `tcp connect` (RTT + 1 s,
   + 3 s…) → **packet loss** (SYN retransmissions). Large `first byte − request
   sent` → **backend**.
2. `dig checkout.example.com` against the configured resolver and `@8.8.8.8`, with
   `+stats` for query time — confirms or rules out DNS, and shows TTLs (Day 14).
3. `sudo tcpdump -nn -i eth0 'host <checkout-ip> and tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'`
   while reproducing — repeated SYNs for the same connection = **loss**; quick
   SYN/SYN-ACK and then a long silence before the response = **backend**.

</details>

---

## Gotchas

- **curl exits 0 on HTTP errors** unless you pass `-f` / `--fail-with-body`.
- **`-d @file` strips newlines and sets a form `Content-Type`.** Use
  `--data-binary` for files.
- **`--json` doesn't exist in your curl 7.81** (it arrived in 7.82). Set the
  headers yourself.
- **`-I` sends a HEAD request**, which some APIs answer differently from GET (or
  reject with 405). Use `-D - -o /dev/null` to see a GET's headers.
- **`-w` timings are cumulative.** Subtract neighbours to get phases.
- **`-w` applies to the whole command** — with several URLs in one curl, use
  `%{url_effective}` in the format to tell the lines apart (Day 16's status loop).
- **Tokens on the command line leak** into history and `ps`. Use `-H @file`, `-K`,
  or a variable read from a file.
- **`--resolve` changes the IP, not the identity.** The certificate must still
  match the hostname.
- **`~/.curlrc` affects every curl**, including other people's scripts you run.
- **tcpdump: always `-nn`, always quote the filter**, and pick the right
  interface (`lo` for localhost).
- **Captures are sensitive.** A plain-HTTP or DNS capture can contain tokens,
  cookies and personal data. Don't share pcaps casually; delete them when done.

---

## Recap

- **Output:** `-i`, `-I` (HEAD), `-v`, `-sS`, `-o`/`-O`, `-D -`, `-L`, and **`-f`**
  so HTTP errors fail.
- **Sending:** `-X`, `-H`, `-d` (form, strips newlines), **`--data-binary`** (exact),
  `--data-urlencode`, `-F` (multipart), `-G`.
- **Auth:** `-u` (Basic), `Authorization: Bearer` header — kept out of the command
  line with `-H @file` or `-K`.
- **Steering:** `--resolve` (pin an IP, keep the name), `--connect-to`,
  `--connect-timeout` + `--max-time`, `--retry` (transient only).
- **`-w` format file:** namelookup → connect → appconnect → pretransfer →
  starttransfer → total; the **gaps** are DNS, TCP (≈ RTT), TLS, server (TTFB),
  download.
- **Config files** (`-K`) make requests repeatable and keep secrets out of
  history.
- **tcpdump:** `-i`, `-nn`, `-c`, `-v`, `-A`, `-w`/`-r`, BPF filters (`host`,
  `port`, `net`, `and/or/not`, tcp flags). Reading a pcap needs no root.
- **HTTPS captures** show IPs, ports, timing, sizes, handshake, SNI (and the cert on
  TLS 1.2) — not method, URL, headers or bodies.

---

## Say it out loud

1. A request is slow. Show me how you'd use `curl` to find out which phase is slow,
   and how you read the numbers.
2. What's the difference between `-d` and `--data-binary`? When does it matter?
3. How would you send one request to a specific backend server while keeping TLS
   validation working?
4. What would you put in a `tcpdump` command to see only connection setups and
   resets to one host?
5. What can someone learn from a packet capture of your HTTPS traffic?
6. Why should a health-check script use `-f`, `--connect-timeout` and
   `--max-time`?

---

## Q&A

_Follow-up questions from reading this lesson go here._
