# Day 20 — Checkpoint: `netcheck`, a Connectivity Diagnostic Report

_Why this matters:_ This is what Days 11–19 were for. Somebody tells you "the
service is down", and you have an hour. This checkpoint makes you build the kit
you'd reach for: a map of exactly how a request gets from a browser to your
process, proof you can read a TLS handshake, a health check that tells you *which
layer* failed and exits with a code a machine can act on, a packet capture of the
connection being made, and a runbook that turns symptoms into next steps. The
health-check script is also something you'll genuinely reuse: the same
exit-code contract is what Kubernetes probes, cron alerts and monitoring systems
expect.

> **Capstone note — this is a deliberate side-quest.** The capstone codebase
> starts with Spring Boot in Phase 6. But four ideas here come back for real: the
> **exit-code contract** (0 / 1 / 2 / 3) is how Kubernetes `exec` probes and every
> Nagios-style monitor work (Phase 15, 17); the **per-phase timing breakdown** is
> what Prometheus' blackbox exporter and every APM report (Phase 17); the **private
> CA and TLS inspection** is the groundwork for Istio's mTLS (Phase 16); and the
> **runbook** is the first draft of the operations docs your capstone will need.

> **How to use this file.** Run Step 0 to get the service and the test harness.
> Then build the five deliverables yourself, milestone by milestone, and run the
> harness after each. Fill in the **Build log** as you go. The reference
> implementation is at the bottom — open it only when you've finished, or after 30
> minutes stuck on one milestone. Plan on **two sessions**.

---

## The one-paragraph version

You run a small HTTPS service in WSL (given to you, with its own private
certificate authority). Then you answer five questions about it, with evidence from
real commands: **How does a request get to it?** (a written path from a Windows
browser to the process, every hop named). **What happens when TLS connects?** (an
`openssl s_client` handshake, explained). **Is it healthy, and if not, where is it
broken?** (a `curl`-based script that prints a timing breakdown and exits `0` OK,
`1` warning, `2` critical or `3` unknown — and names the failing layer when `curl`
itself fails). **What does the connection look like on the wire?** (a `tcpdump`
capture of the TCP and TLS handshake, annotated). **What do I do when it breaks?**
(a runbook: symptom → tool → likely cause). A provided test harness breaks the
service in fourteen ways and checks your script reports each correctly.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **health check** | A cheap request that answers "is this thing working?" | An endpoint (`/health`) plus a client that judges the answer |
| **probe** | A health check run on a schedule by a platform | Kubernetes liveness, readiness and startup probes |
| **exit-code contract** | Agreed meanings for a program's exit numbers | Here: 0 OK, 1 WARNING, 2 CRITICAL, 3 UNKNOWN — from the Nagios plugin convention |
| **UNKNOWN vs CRITICAL** | "I couldn't run the test" vs "I ran it and it failed" | A broken monitor must never look like a healthy (or a failing) service |
| **threshold** | The value at which a measurement becomes a warning or an alarm | Here, milliseconds per phase |
| **phase timing** | How long each stage of a request took | `curl -w` cumulative timestamps, subtracted (Day 18) |
| **runbook** | Written steps for diagnosing and fixing a known failure | Symptom → check → cause → fix |
| **path document** | A hop-by-hop description of how traffic reaches a process | Names, addresses, layers and what each hop rewrites |
| **private CA** | A certificate authority you run yourself | Signs certificates only you (or your systems) trust (Day 17) |
| **handshake capture** | A recording of the packets that set up a connection | `tcpdump -w`, read back with `-r` (Day 18) |
| **assertion on the layer** | Checking *why* something failed, not just that it did | A test that also requires the right error text (see the harness) |

---

## What you're building

| # | Deliverable | File | Days it draws on |
|---|-------------|------|------------------|
| D1 | Request path: Windows browser → process | `PATH.md` | 11, 12, 14, 19 |
| D2 | TLS handshake, captured and explained | `TLS.md` | 17 |
| D3 | Health-check script with timing and exit codes | `healthcheck.sh` | 9, 15, 16, 18 |
| D4 | TCP/TLS handshake packet capture, annotated | `handshake.pcap` + `CAPTURE.md` | 13, 17, 18 |
| D5 | Troubleshooting runbook | `RUNBOOK.md` | 15 (all of them) |

All of it lives in `~/netcheck` — in the Linux filesystem, not `/mnt/d` (Day 1,
Day 9: fast, and `chmod` works).

---

## Step 0 — Get the service and the harness

This creates `~/netcheck` with: `svc.py` (the HTTPS service), `make-certs.sh`
(a private CA and a `localhost` certificate — Day 17's recipe), `run-tests.sh`
(the harness), and a **stub** `healthcheck.sh` that just says it isn't written yet.
(It exits `99`, a code your script must never use.) It won't overwrite a `healthcheck.sh` you've already started. Run it once:

```bash
mkdir -p ~/netcheck && cd ~/netcheck

cat > svc.py <<'SVC_EOF'
#!/usr/bin/env python3
"""A tiny HTTPS service to diagnose.

  python3 svc.py [--bind 127.0.0.1] [--port 8443]

  GET /health        200 {"status":"ok"}
  GET /slow?ms=1500  sleeps, then 200 -- a stand-in for a slow database
  GET /fail          503 -- a stand-in for a broken dependency
  anything else      404
"""
import argparse
import http.server
import json
import ssl
import sys
import time
import urllib.parse


class Handler(http.server.BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"          # keep-alive, like a real service (Day 13, Day 16)

    def do_GET(self):
        url = urllib.parse.urlparse(self.path)
        query = urllib.parse.parse_qs(url.query)
        if url.path == "/fail":
            return self.reply(503, {"status": "unavailable"})
        if url.path == "/slow":
            time.sleep(int(query.get("ms", ["1000"])[0]) / 1000)
        if url.path in ("/", "/health", "/slow"):
            return self.reply(200, {"status": "ok", "path": url.path})
        return self.reply(404, {"status": "not found"})

    def reply(self, code, payload):
        body = (json.dumps(payload) + "\n").encode()
        self.send_response(code)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, fmt, *args):
        print(f"svc: {self.client_address[0]}:{self.client_address[1]} {fmt % args}", file=sys.stderr)


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--bind", default="127.0.0.1")
    parser.add_argument("--port", type=int, default=8443)
    parser.add_argument("--cert", default="server.crt")
    parser.add_argument("--key", default="server.key")
    args = parser.parse_args()

    server = http.server.ThreadingHTTPServer((args.bind, args.port), Handler)
    context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    context.load_cert_chain(args.cert, args.key)
    server.socket = context.wrap_socket(server.socket, server_side=True)
    print(f"svc: listening on https://{args.bind}:{args.port}", file=sys.stderr)
    server.serve_forever()


if __name__ == "__main__":
    main()
SVC_EOF

cat > make-certs.sh <<'CERTS_EOF'
#!/usr/bin/env bash
# make-certs.sh -- a private CA, and a server certificate for localhost signed by it (Day 17).
set -euo pipefail

openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout ca.key -out ca.crt -days 30 -subj "/CN=Learn Daily Dev CA" 2>/dev/null

openssl req -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout server.key -out server.csr -subj "/CN=localhost" 2>/dev/null

printf 'subjectAltName=DNS:localhost,IP:127.0.0.1\nextendedKeyUsage=serverAuth\nbasicConstraints=critical,CA:FALSE\n' > server.ext

openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 30 -extfile server.ext 2>/dev/null

openssl verify -CAfile ca.crt server.crt
CERTS_EOF

cat > run-tests.sh <<'TESTS_EOF'
#!/usr/bin/env bash
# run-tests.sh -- start the service, break it on purpose, and check healthcheck.sh names each failure.
# Every test checks the exit status AND that the message names the right layer, so a test
# can't pass for the wrong reason.
set -uo pipefail
cd "$(dirname "$0")"

pids=()
cleanup() { for p in "${pids[@]}"; do kill "$p" 2>/dev/null || true; done; }
trap cleanup EXIT

[[ -f server.crt ]] || ./make-certs.sh >/dev/null
python3 svc.py --port 8443 2>svc.log &
pids+=($!)
sleep 1

pass=0 fail=0
# expect "name" EXPECTED_EXIT "text the output must contain"  command...
expect() {
  local name=$1 want=$2 text=$3; shift 3
  local out got
  out=$("$@" 2>&1); got=$?
  if [[ $got -eq $want && $out == *"$text"* ]]; then
    printf 'PASS  %-34s exit %s   %s\n' "$name" "$got" "$out"; pass=$((pass + 1))
  else
    printf 'FAIL  %-34s wanted exit %s containing "%s"\n      got exit %s: %s\n' "$name" "$want" "$text" "$got" "$out"
    fail=$((fail + 1))
  fi
}

export CACERT="$PWD/ca.crt"
base=https://localhost:8443

expect "healthy"                      0 "OK "        ./healthcheck.sh "$base/health"
expect "slow server -> WARNING"       1 "WARNING"    env WARN_SERVER_MS=500  CRIT_SERVER_MS=2000 ./healthcheck.sh "$base/slow?ms=800"
expect "very slow server -> CRITICAL" 2 "server 1"   env WARN_SERVER_MS=500  CRIT_SERVER_MS=1000 ./healthcheck.sh "$base/slow?ms=1500"
expect "app returns 503"              2 "HTTP 503"   ./healthcheck.sh "$base/fail"
expect "404"                          2 "HTTP 404"   ./healthcheck.sh "$base/nope"
expect "nothing listening (refused)"  2 "TCP:"       ./healthcheck.sh https://localhost:8444/health
expect "untrusted CA"                 2 "TLS:"       env CACERT= ./healthcheck.sh "$base/health"
# Reach the real server under a name its certificate doesn't cover: TCP works, TLS must refuse.
expect "wrong hostname"               2 "TLS:"       env RESOLVE=other.example:8443:127.0.0.1 ./healthcheck.sh https://other.example:8443/health
expect "DNS failure"                  2 "DNS:"       ./healthcheck.sh https://no-such-host.invalid/
expect "total time limit"             2 "TIMEOUT:"   env MAX_TIME_S=1 ./healthcheck.sh "$base/slow?ms=2500"
expect "bad URL"                      3 "UNKNOWN"    ./healthcheck.sh not-a-url
expect "no arguments"                 3 "usage"      ./healthcheck.sh
expect "bad threshold"                3 "whole number" env WARN_TOTAL_MS=fast ./healthcheck.sh "$base/health"
expect "missing CA file"              3 "not readable" env CACERT=/no/such/ca.crt ./healthcheck.sh "$base/health"

echo
echo "$pass passed, $fail failed"
exit $(( fail > 0 ))
TESTS_EOF

if [[ ! -f healthcheck.sh ]]; then
  cat > healthcheck.sh <<'STUB_EOF'
#!/usr/bin/env bash
echo "healthcheck.sh is not written yet" >&2
exit 99
STUB_EOF
fi

chmod +x svc.py make-certs.sh run-tests.sh healthcheck.sh
./make-certs.sh
echo
echo "Step 0 done. Files in ~/netcheck:"
ls -1
```

You should see `server.crt: OK`. Now start the service **in its own Ubuntu tab**
and leave it running:

```
cd ~/netcheck && python3 svc.py
```

(It logs one line per request to that tab. `Ctrl+C` stops it. The harness starts
its own copy on the same port, so stop yours before running `run-tests.sh`.)

Quick look, from a second tab:

```
cd ~/netcheck
curl -sS --cacert ca.crt https://localhost:8443/health
curl -sS https://localhost:8443/health          # fails - Day 17: our CA isn't in the trust store
```

---

## Milestone 1 — The request path (D1: `PATH.md`)

**Goal:** a document a colleague could use to find which hop is at fault. For the
request *Windows browser → `https://localhost:8443/health`*, name every hop from
the browser to the Python process: what name or address is used at each step, which
layer it works at (Day 11), what it rewrites or forwards, and **how you proved it**
with a command.

Gather evidence with this (read-only; run it in WSL, with the service running):

```bash
cd ~/netcheck
echo "== 1. the service: what is listening, and who owns it (Day 15)"
ss -ltnp 'sport = :8443'

echo; echo "== 2. WSL's addresses and its route out (Day 11, 12)"
ip -4 -br addr show eth0
ip route | grep default

echo; echo "== 3. how names resolve in WSL (Day 14)"
grep -E '^hosts' /etc/nsswitch.conf
getent hosts localhost

echo; echo "== 4. what the service presents (Day 17)"
openssl s_client -connect localhost:8443 -servername localhost -CAfile ca.crt -brief </dev/null 2>&1 | head -6
```

And from the Windows side, in PowerShell:

```powershell
# Which Windows address does 'localhost' mean, and what is listening on 8443 on Windows? (Day 19)
Resolve-DnsName localhost -Type A_AAAA | Select-Object Name, Type, IPAddress
Get-NetTCPConnection -LocalPort 8443 -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess
Test-NetConnection localhost -Port 8443 -InformationLevel Quiet     # note the IPv6 warning, if any
curl.exe -sS --cacert "$HOME\ca.crt" --ssl-no-revoke https://localhost:8443/health   # after copying ca.crt there
```

Copy `ca.crt` to Windows first: `cp ~/netcheck/ca.crt /mnt/c/Users/<you>/`. Two
Windows-specific things to expect: `curl.exe` uses Windows' own TLS (schannel), so
without the CA you get `SEC_E_UNTRUSTED_ROOT`; and even *with* `--cacert` it also
checks certificate **revocation**, which a private CA can't answer — hence
`--ssl-no-revoke` (measured: without it, `the revocation status is unknown`). Then
answer in `PATH.md`:

1. List the hops in order. For each: layer, address in use, evidence command.
2. Where does the TCP connection from the browser actually **end**? Where does TLS
   end? (They're different places.)
3. Which hop would break if you restarted WSL, and why? (Day 11, Day 19.)
4. What happens, hop by hop, if the service is rebound to `0.0.0.0`? To your WSL
   `eth0` address only? (Day 19 matrix.)
5. What does a Windows browser say about the certificate, and why — and what would
   you have to do to make it go away? What's the risk?

---

## Milestone 2 — The TLS handshake (D2: `TLS.md`)

**Goal:** capture the handshake with `openssl s_client` and explain it.

```bash
cd ~/netcheck
# 1. A normal, verifying connection
openssl s_client -connect localhost:8443 -servername localhost -CAfile ca.crt -verify_hostname localhost -brief </dev/null

# 2. Every handshake message, in order (>>> client sent, <<< server sent)
openssl s_client -connect localhost:8443 -servername localhost -CAfile ca.crt -msg </dev/null 2>&1 | grep -E '^(>>>|<<<).*(Handshake|ChangeCipher)'

# 3. Force TLS 1.2 and compare
openssl s_client -connect localhost:8443 -servername localhost -CAfile ca.crt -tls1_2 -brief </dev/null

# 4. Without our CA: the failure you saw on Day 17
openssl s_client -connect localhost:8443 -servername localhost -verify_return_error -brief </dev/null 2>&1 | head -3

# 5. The certificate itself
openssl s_client -connect localhost:8443 -servername localhost </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

In `TLS.md` explain, quoting your own output:

1. Which TLS version and cipher suite were negotiated? Decode the cipher suite name
   piece by piece (Day 17).
2. Which messages did each side send, in order, for TLS 1.3? Which of them are
   *encrypted* in 1.3 but were plaintext in 1.2?
3. How many certificates did the server send? Why isn't the root among them?
   What does the client use to decide to trust the server's certificate?
4. What does `Verified peername` prove, and what did you pass to make it work?
5. Why does connection 4 fail, and what exactly was missing?
6. Why does `-servername localhost` matter here, when there's only one site?

---

## Milestone 3 — The health check (D3: `healthcheck.sh`)

**Goal:** `./healthcheck.sh URL` probes one endpoint with `curl`, prints **one line**
with a per-phase timing breakdown, and exits with a code that means something.

**Requirements:**

- `set -euo pipefail` at the top, a `trap` that removes its temp file (Day 9).
- **Exit codes:** `0` OK, `1` WARNING, `2` CRITICAL, `3` UNKNOWN. UNKNOWN means *the
  check couldn't be run* — bad URL, no arguments, a non-numeric threshold, an
  unreadable CA file. It must never be confused with a healthy or failing service.
- **Settings from environment variables** (Day 10's pattern), all optional, with
  defaults: `WARN_CONNECT_MS`/`CRIT_CONNECT_MS`, `WARN_SERVER_MS`/`CRIT_SERVER_MS`,
  `WARN_TOTAL_MS`/`CRIT_TOTAL_MS`, `CONNECT_TIMEOUT_S`, `MAX_TIME_S`, `CACERT`
  (passed as `--cacert`), `RESOLVE` (passed as `--resolve`). Validate that numbers
  are numbers.
- **Always set both timeouts** on `curl` (Day 18): `--connect-timeout` and
  `--max-time`.
- Use **`curl -w`** to get the timestamps (Day 18) and subtract them into phases:
  DNS, TCP connect, TLS, server (first byte minus request sent), and total, in
  milliseconds.
- **When `curl` itself fails**, map its exit code to the layer that broke (Day 15):
  `6` DNS, `7` TCP refused/unreachable, `28` timeout, `35/51/58/60/77/83` TLS — and
  exit `2` with a message naming the layer.
- When `curl` succeeds: HTTP `>= 400` is CRITICAL regardless of timing; each phase
  is compared with its thresholds; the overall severity is the worst one; the line
  says which threshold was crossed.
- Output example:
  ```
  OK https://localhost:8443/health  http=200 total=48ms (dns 0 | tcp 0 | tls 4 | server 44) ip=127.0.0.1
  ```

**The test harness** (`./run-tests.sh`, ~5 seconds) starts the service and runs
fourteen scenarios — healthy, slow, very slow, 503, 404, refused, untrusted CA,
**wrong hostname**, DNS failure, a total-time timeout, and four ways of misusing the
script. It checks the exit code **and** that the message names the right layer. Run
it first, with the stub, to see all fourteen fail; then make them pass one at a
time.

Hints, if stuck for more than ten minutes:

- `curl -w` still prints its template when the request fails; the numbers are just
  zero.
- `curl ... || rc=$?` stops `set -e` from killing the script on a curl failure so
  you can inspect `rc`.
- A function ending in `(( x > y )) && something` returns non-zero when the
  comparison is false, which trips `set -e` (Day 9). End such functions with
  `return 0`.
- `-w` timings are cumulative: connect time is `connect - namelookup`.
- To test "wrong hostname" you need to reach the real server under a name the
  certificate doesn't cover: that's what `RESOLVE` is for (Day 18).

---

## Milestone 4 — Capture the handshake (D4: `handshake.pcap` + `CAPTURE.md`)

**Goal:** a packet capture of one health check's connection, with each packet
accounted for.

In an Ubuntu terminal, with the service running (`sudo` for the capture only —
reading the file back needs none, Day 18):

```
cd ~/netcheck
sudo -v
sudo timeout 6 tcpdump -i lo -nn -s 0 -w handshake.pcap 'tcp port 8443' &
sleep 1
CACERT=$PWD/ca.crt ./healthcheck.sh https://localhost:8443/health
wait
sudo chown "$USER" handshake.pcap

tcpdump -nn -r handshake.pcap                                                    # everything
tcpdump -nn -r handshake.pcap 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'  # connection setup and teardown only
tcpdump -nn -A -r handshake.pcap | grep -ac localhost                            # how many times is the name visible?
```

`-i lo` because the service is on loopback (Day 11). `wait` instead of `kill`:
`tcpdump` finishes cleanly on its own after the 6-second timeout. (An earlier
version of this exercise killed it after one second and lost the closing
packets.)

In `CAPTURE.md`, annotate:

1. The three-way handshake: which packets, which flags, what each side's starting
   sequence number is (Day 13).
2. The first data packet from the client. How big is it (`seq 1:518` → 517 bytes)?
   What is it? Why can you find the word `localhost` in it?
3. The server's reply. In TLS 1.3 the whole server flight — ServerHello, certificate,
   signature, Finished — fits in one segment. How big? Why can't you read the
   certificate in the capture?
4. The application data: the request and the response. What sizes? Why can't you
   read `GET /health`?
5. The teardown: who sends the first `FIN`? Which side will hold `TIME_WAIT` (Day 13)?
6. Compare with Day 18's capture of `httpbin.org`: what's different when the
   connection never leaves the machine (TTL, MSS, timing)?

---

## Milestone 5 — The runbook (D5: `RUNBOOK.md`)

**Goal:** a table you could hand to a teammate at 3am. Write it **last**, after you
have broken the service and watched the tools. Each row is a *symptom a user would
report* → *the one command to run* → *what its output means* → *the likely cause*
→ *the fix*.

Required rows (at least these ten — add your own):

1. "I can't reach it at all" (connection refused)
2. "It hangs and then times out"
3. "It says the site can't be found" (DNS)
4. "Certificate error" — three variants: untrusted CA, wrong name, expired
5. "It works for me but not for them" (resolver, TTL, bind address, firewall)
6. "It's slow" — and how to tell DNS / network / TLS / server apart from a timing line
7. "It returns an error" (4xx vs 5xx vs 502/503/504 behind a load balancer)
8. "It works on `localhost` but not by IP / from another machine" (bind address)
9. "Port already in use"
10. "Connection reset by peer" / intermittent failures after idle (Day 13, Day 16)

For each: what you'd run in this project (`./healthcheck.sh`, `ss`, `dig`, `nc`,
`openssl s_client`, `tcpdump`) and a one-line way to confirm the cause.

---

## Build log

Fill this in as you go. Be honest about what broke — that's the useful part.

| Milestone | Time spent | What I got wrong first | What I learned |
|-----------|-----------:|------------------------|----------------|
| 0 Setup | | | |
| 1 Path (`PATH.md`) | | | |
| 2 TLS (`TLS.md`) | | | |
| 3 `healthcheck.sh` | | | |
| 4 Capture (`CAPTURE.md`) | | | |
| 5 `RUNBOOK.md` | | | |

**Harness results over time** (`./run-tests.sh` → `N passed, M failed`):

| Attempt | Passed / 14 | What I fixed next |
|--------:|:-----------:|-------------------|
| 1 (stub) | 0 | |
| | | |
| | | |

---

## Stretch goals (optional)

1. **Run it on a schedule:** a cron entry (Day 5) running `healthcheck.sh` every
   minute and appending its one-line output to `~/netcheck/health.log`. Stop the
   service for three minutes, then start it. Read the log.
2. **Check the real internet:** `./healthcheck.sh https://httpbin.org/get` with
   default thresholds, then with `WARN_CONNECT_MS=100`. Which phases dominate, and
   why does the default `CRIT_TOTAL_MS` need to be higher for remote services?
3. **Make the service LAN-visible** with `--bind 0.0.0.0` and confirm from
   Windows by WSL IP — then confirm your phone still can't reach it (Day 19).
4. **Add certificate expiry** to `healthcheck.sh`: warn if the certificate expires
   within 14 days (`openssl s_client … | openssl x509 -noout -checkend`), in a
   separate check so a TLS problem is distinguished from a slow response.
5. **JSON output:** a `--json` flag that prints the same breakdown for a log
   pipeline (Day 8's `jq`).

---

## Reference implementation

**Open this only when you've finished or you've been stuck for 30 minutes.**

### `healthcheck.sh`

This is the file `run-tests.sh` is validated against (14 passed, 0 failed).

```
#!/usr/bin/env bash
# healthcheck.sh -- probe one URL, print a timing breakdown, exit by severity.
#
#   healthcheck.sh URL
#
# Exit codes (the same contract Nagios checks, Kubernetes exec probes and cron alerts use):
#   0 OK        1 WARNING        2 CRITICAL        3 UNKNOWN (bad usage or configuration)
#
# Settings (environment variables, all optional):
#   WARN_CONNECT_MS=300  CRIT_CONNECT_MS=1000   TCP connect time        (about 1 round trip)
#   WARN_SERVER_MS=800   CRIT_SERVER_MS=2000    first byte after request sent (server think time)
#   WARN_TOTAL_MS=1500   CRIT_TOTAL_MS=3000     whole request
#   CONNECT_TIMEOUT_S=3  MAX_TIME_S=10          hard limits handed to curl
#   CACERT=/path/ca.crt                         trust this CA (private CAs, Day 17)
#   RESOLVE=host:port:ip                        pin the target IP (Day 18)
set -euo pipefail

readonly OK=0 WARN=1 CRIT=2 UNKNOWN=3

: "${WARN_CONNECT_MS:=300}"  "${CRIT_CONNECT_MS:=1000}"
: "${WARN_SERVER_MS:=800}"   "${CRIT_SERVER_MS:=2000}"
: "${WARN_TOTAL_MS:=1500}"   "${CRIT_TOTAL_MS:=3000}"
: "${CONNECT_TIMEOUT_S:=3}"  "${MAX_TIME_S:=10}"
: "${CACERT:=}"              "${RESOLVE:=}"

die_unknown() { echo "UNKNOWN $*" >&2; exit "$UNKNOWN"; }

# --- 1. validate -------------------------------------------------------------
[[ $# -eq 1 ]] || die_unknown "usage: healthcheck.sh URL"
url=$1
[[ $url == http://* || $url == https://* ]] || die_unknown "not an http(s) URL: $url"
for var in WARN_CONNECT_MS CRIT_CONNECT_MS WARN_SERVER_MS CRIT_SERVER_MS WARN_TOTAL_MS CRIT_TOTAL_MS \
           CONNECT_TIMEOUT_S MAX_TIME_S; do
  [[ ${!var} =~ ^[0-9]+$ ]] || die_unknown "$var must be a whole number, got '${!var}'"
done
[[ -z $CACERT || -r $CACERT ]] || die_unknown "CACERT not readable: $CACERT"

# --- 2. run curl; the -w template becomes key=value lines --------------------
tmp=$(mktemp)
trap 'rm -f "$tmp"' EXIT

curl_args=(-sS -o /dev/null --connect-timeout "$CONNECT_TIMEOUT_S" --max-time "$MAX_TIME_S"
  -w 'dns=%{time_namelookup}\nconnect=%{time_connect}\ntls=%{time_appconnect}\npretransfer=%{time_pretransfer}\nstart=%{time_starttransfer}\ntotal=%{time_total}\ncode=%{http_code}\nip=%{remote_ip}\n')
[[ -n $CACERT ]]  && curl_args+=(--cacert "$CACERT")
[[ -n $RESOLVE ]] && curl_args+=(--resolve "$RESOLVE")

rc=0
curl "${curl_args[@]}" "$url" >"$tmp" 2>"$tmp.err" || rc=$?
err=$(head -n1 "$tmp.err" 2>/dev/null || true)
rm -f "$tmp.err"

# --- 3. a curl failure tells us WHICH layer broke (Day 15) -------------------
if (( rc != 0 )); then
  case $rc in
    6)           layer="DNS: cannot resolve the hostname" ;;
    7)           layer="TCP: connection refused or unreachable (nothing listening / no route)" ;;
    28)          layer="TIMEOUT: no answer within ${CONNECT_TIMEOUT_S}s connect / ${MAX_TIME_S}s total (dropped, or server stuck)" ;;
    35|51|58|60|77|83) layer="TLS: handshake or certificate failure" ;;
    *)           layer="curl error $rc" ;;
  esac
  echo "CRITICAL $url  $layer  [${err}]"
  exit "$CRIT"
fi

# --- 4. turn the cumulative timestamps into per-phase milliseconds -----------
declare -A t
while IFS='=' read -r key value; do t[$key]=$value; done <"$tmp"

ms() { awk -v s="$1" 'BEGIN { printf "%d", s * 1000 + 0.5 }'; }
dns=$(ms "${t[dns]}")
connect=$(( $(ms "${t[connect]}") - dns ))
tls=0
if [[ ${t[tls]} != 0.000000 ]]; then tls=$(( $(ms "${t[tls]}") - $(ms "${t[connect]}") )); fi
server=$(( $(ms "${t[start]}") - $(ms "${t[pretransfer]}") ))
total=$(ms "${t[total]}")
code=${t[code]}

# --- 5. judge -----------------------------------------------------------------
severity=$OK
reasons=()
raise() { (( $1 > severity )) && severity=$1; reasons+=("$2"); return 0; }   # explicit "return 0": a false (( )) must not trip set -e

if (( code >= 400 )); then raise "$CRIT" "HTTP $code"; fi
if   (( connect >= CRIT_CONNECT_MS )); then raise "$CRIT" "connect ${connect}ms >= ${CRIT_CONNECT_MS}"
elif (( connect >= WARN_CONNECT_MS )); then raise "$WARN" "connect ${connect}ms >= ${WARN_CONNECT_MS}"; fi
if   (( server  >= CRIT_SERVER_MS  )); then raise "$CRIT" "server ${server}ms >= ${CRIT_SERVER_MS}"
elif (( server  >= WARN_SERVER_MS  )); then raise "$WARN" "server ${server}ms >= ${WARN_SERVER_MS}"; fi
if   (( total   >= CRIT_TOTAL_MS   )); then raise "$CRIT" "total ${total}ms >= ${CRIT_TOTAL_MS}"
elif (( total   >= WARN_TOTAL_MS   )); then raise "$WARN" "total ${total}ms >= ${WARN_TOTAL_MS}"; fi

case $severity in 0) label=OK ;; 1) label=WARNING ;; *) label=CRITICAL ;; esac
detail="http=$code total=${total}ms (dns ${dns} | tcp ${connect} | tls ${tls} | server ${server}) ip=${t[ip]}"
if (( ${#reasons[@]} )); then
  echo "$label $url  $detail  <- ${reasons[*]}"
else
  echo "$label $url  $detail"
fi
exit "$severity"
```

Design notes:

- **Why exit `3` for bad input and `2` for curl failures.** A script that can't run
  its test must not report "service down" — you'd page someone because of a typo in
  the monitor.
- **Why subtract the cumulative `-w` values.** `time_connect` includes DNS;
  `time_appconnect` includes connect. The gaps are the phases.
- **Why the layer text is asserted in the harness.** My first harness passed
  "wrong hostname" because the target address wasn't listening at all — the script
  returned `2` for the wrong reason. Checking only the exit code can't tell the
  layers apart; that is the entire skill this phase teaches.
- **Why `HTTP >= 400` is CRITICAL even when fast.** A health endpoint that answers
  `503` quickly is still down. Fast failure is not health.
- **Why both timeouts.** `--connect-timeout` bounds the Day 13 SYN hang;
  `--max-time` bounds a server that accepts and then never answers.

### A sample `PATH.md` skeleton (your values will differ)

```
Request: Windows browser -> https://localhost:8443/health

1. Browser resolves "localhost"           L7/DNS   built into Windows (the hosts-file lines are commented out) -> ::1 and 127.0.0.1;
                                                   tries ::1 first (nothing listens there), then falls back to 127.0.0.1
2. TCP to Windows 127.0.0.1:8443          L4       a Windows-side relay owns this port (Get-NetTCPConnection -> one PID)
3. Relay opens a connection inside WSL    L4       to localhost:8443, over WSL's loopback (lo); ufw never sees it
4. svc.py accepts it                      L7       ss -ltnp -> LISTEN 127.0.0.1:8443 users:(("python3",...))
5. TLS runs BROWSER <-> svc.py            L6/L7    the relay copies bytes; it never decrypts -> the browser sees svc's own
                                                   certificate, issued by "Learn Daily Dev CA"
End of TCP: two connections (browser-relay, relay-svc.py).  End of TLS: svc.py, end to end.
Breaks if: WSL restarts (service gone, relay port closes); the service is rebound to eth0 only (relay can't reach it);
           port 8443 is taken on Windows.
```

### A sample `RUNBOOK.md` (abridged)

| Symptom | Run | Reading it | Likely cause | Fix / confirm |
|---------|-----|-----------|--------------|---------------|
| Connection refused | `./healthcheck.sh URL` → `TCP:` | Host answered with a RST | Nothing listening, wrong port, bound to the wrong address | `ss -ltnp 'sport = :PORT'`; check bind address |
| Hangs, then times out | `./healthcheck.sh URL` → `TIMEOUT:`; `nc -zv -w 3 HOST PORT` | Packets dropped, no reply | Firewall / security group, wrong IP, host down | `ip route get`; check SG/ufw; try another network |
| "Could not resolve host" | `./healthcheck.sh URL` → `DNS:`; `dig NAME` vs `dig @8.8.8.8 NAME` | NXDOMAIN, SERVFAIL or timeout? | Typo, record missing, resolver down, stale cache | `dig +trace`; compare resolvers; check `/etc/hosts` |
| Cert: untrusted CA | `TLS:` + `unable to get local issuer certificate` | Chain doesn't reach a trusted root | Private CA not in the trust store (Java: its own `cacerts`) | Give the client the CA (`--cacert`, SSL bundle) |
| Cert: wrong name | `TLS:` + `no alternative certificate subject name matches` | Hostname not in the SAN | Connected by IP / a different name | Use a name in `openssl x509 -ext subjectAltName` |
| Cert: expired | `TLS:` + `certificate has expired` | `notAfter` is in the past | Renewal missed | `openssl s_client … \| openssl x509 -noout -enddate`; renew; add monitoring |
| Slow | `./healthcheck.sh URL`, read the phases | big `dns` → resolver; big `tcp` → distance / lost SYN (+1 s); big `tls` → handshake / server; big `server` → application | see phase | DNS: `dig`; network: `mtr`; server: app logs |
| 5xx | `curl -i URL` | 500 app; 502 bad/reset response; 503 no healthy backend; 504 backend too slow | app crash, pool exhaustion, deploy | App logs; LB target health; check timeouts |
| Works on localhost, not by IP | `ss -ltn` | `127.0.0.1:PORT` | Bound to loopback | Rebind to `0.0.0.0` (mind exposure) |
| Port in use | `ss -ltnp` / `lsof -i :PORT` | Names the owner | Previous run still alive, or Windows owns the port (WSL shares localhost) | Stop it; check `Get-NetTCPConnection` |
| Reset after idle | `tcpdump` for `RST`; compare idle timeouts | A middlebox forgot the connection | LB / NAT idle timeout < pool idle timeout | Shorten pool `maxLifetime` / enable keep-alives |

---

## When you're done

- Run `./run-tests.sh` one last time: **14 passed, 0 failed**.
- Re-read your `RUNBOOK.md`. Does every row name a command *and* what its output
  means?
- Stop the service tabs. Optionally remove the test files:
  `rm -rf ~/netcheck` (or keep it — Phase 12 and 17 will reuse the health check).
- **Phase 2 quiz.** It's ready: [`day-20-week-4-quiz.md`](./day-20-week-4-quiz.md)
  — 48 questions covering Days 11–20, 75–100 minutes. Take it before starting
  Phase 3. (Quizzes aren't a separate day.)

---

## Say it out loud

Final check — explain these to a colleague, using the real terms.

1. Walk through exactly how a request gets from a Windows browser to a process
   in WSL, and where TCP and TLS each end.
2. What do `0`, `1`, `2` and `3` mean in your health check, and why is "UNKNOWN"
   its own code?
3. Your health check reports a 4.1-second total, mostly in `tcp`. What does that
   tell you, and what would you do next?
4. What does a packet capture of an HTTPS request show, and what's invisible?
5. You get "certificate verify failed" from a Java service but `curl` works. Walk
   through your debugging.
6. Pick any row of your runbook and explain how the command distinguishes that
   cause from its neighbours.

---

## Q&A

_Follow-up questions go here._
