# Day 16 — HTTP Deep Dive

_Why this matters:_ HTTP is the protocol of almost everything you'll build:
every Spring `@RestController`, every call to another service, every load
balancer health check, and — in Phase 9 — every streamed token from an AI model.
Most HTTP bugs aren't syntax bugs; they're **meaning** bugs. A `POST` retried
after a timeout charges a customer twice. A `403` where a `401` belongs makes a
client give up instead of logging in again. A redirect silently turns a `POST`
into a `GET` and drops the body. A `502` gets blamed on the load balancer when
it's your app crashing. Behind a proxy, every request seems to come from the
same IP. Today is about what each part of an HTTP message *means*, so your APIs
behave correctly and you can read what other people's APIs are telling you.

> **Week 4 opener.** Week 3 got a byte stream from one program to another. This
> week is what travels over it: HTTP today, TLS tomorrow, then curl mastery,
> firewalls, and the Phase 2 checkpoint.
>
> **This is a long day.** If it runs past two hours, split it: Parts 1–4 and
> hands-on 1–6 today; Parts 5–8 and hands-on 7–11 next session. The required
> practice is hands-on 1, 3 and 5.

---

## The one-paragraph version

An HTTP exchange is a **request** (a method like `GET` or `POST`, a path, some
headers, maybe a body) and a **response** (a three-digit status code, headers,
maybe a body). It's text on HTTP/1.1 — you typed it by hand on Day 11 — and the
same meaning in binary frames on HTTP/2 and HTTP/3. The **method** is a promise
about what the request does: `GET` is *safe* (changes nothing), and `GET`, `PUT`
and `DELETE` are *idempotent* (doing them twice has the same effect as once), so
they can be retried after a timeout; `POST` is neither, which is why payment APIs
add an **idempotency key**. The **status code** is the server's verdict, and the
specific number matters: `401` means "who are you?", `403` means "I know who you
are, and no"; `502`, `503` and `504` say three different things about the server
behind a load balancer. **Headers** carry everything else: what format the body
is in, who you are, how long the response can be cached, what the response's
version tag (`ETag`) is so you can ask "has it changed?" and get a cheap `304`.
**Cookies** let a stateless protocol remember you. The connection underneath is
kept open and reused (Day 13), HTTP/2 lets many requests share it at once, and
**chunked transfer** lets a server start sending before it knows how long the
response will be — which is how AI chat endpoints stream tokens to you as
they're generated.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **request line** | The first line of a request | `METHOD path HTTP/version`, e.g. `GET /get HTTP/1.1` |
| **status line** | The first line of a response | `HTTP/version code reason`, e.g. `HTTP/1.1 200 OK` |
| **header** | A `Name: value` line of metadata | Field names are case-insensitive; HTTP/2 sends them lowercase |
| **body** | The content after the headers | Separated from the headers by one blank line (`\r\n\r\n`) |
| **method** | The verb: what the request wants to do | `GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS` |
| **safe** | Doesn't change anything on the server | `GET`, `HEAD`, `OPTIONS` — read-only by contract |
| **idempotent** | Doing it twice has the same effect as doing it once | `GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE` — not `POST`, not (generally) `PATCH` |
| **idempotency key** | A unique ID the client sends so the server can spot a retry | A header like `Idempotency-Key: …`; server stores key → result |
| **status code** | The server's three-digit verdict | 1xx info, 2xx success, 3xx redirect, 4xx client error, 5xx server error |
| **redirect** | "What you want is over there" | 3xx + a `Location` header |
| **content negotiation** | Client says what formats it accepts; server picks one | `Accept` request header → `Content-Type` response header |
| **media type** | The format label of a body | `application/json`, `text/html`, `text/event-stream` |
| **`Authorization`** | The request header carrying your credentials | `Basic …`, `Bearer <token>` (Phase 8) |
| **caching** | Keeping a copy of a response to reuse instead of asking again | Controlled by `Cache-Control`, validated with `ETag` / `Last-Modified` |
| **`ETag`** | A version tag for a response | An opaque identifier; changes when the content changes |
| **conditional request** | "Send it only if it changed" | `If-None-Match: <etag>` → `304 Not Modified` if unchanged |
| **`X-Forwarded-For`** | The chain of client IPs a proxy saw | Each proxy appends the address it received the request from |
| **cookie** | A small value the server asks the browser to send back on every request | Set with `Set-Cookie`, returned with `Cookie` |
| **session** | The server remembering who you are between requests | Usually: a cookie holds a session ID; the server stores the data |
| **keep-alive** | Reusing a connection for the next request (Day 13) | HTTP/1.1 persistent connections |
| **framing** | How the receiver knows where a message ends | `Content-Length`, or `Transfer-Encoding: chunked` |
| **chunked transfer** | Sending a body in pieces, each with its own size | `Transfer-Encoding: chunked`; ends with a zero-size chunk |
| **multiplexing** | Many requests interleaved on one connection at once | HTTP/2 streams |
| **ALPN** | Agreeing on HTTP/1.1 vs HTTP/2 during the TLS handshake | Application-Layer Protocol Negotiation |
| **QUIC** | A transport over UDP with built-in TLS and independent streams | What HTTP/3 runs on (Day 13) |
| **SSE** | A server pushing a stream of events over one long HTTP response | Server-Sent Events, `Content-Type: text/event-stream` |

---

## Before you read: three questions

1. A client sends `POST /payments`, gets a timeout, and retries. The customer is
   charged twice. Which property did the request lack, and what's the standard
   fix?
2. Your API returns `403` when a request has no token at all. A reviewer says it
   should be `401`. Who's right, and what actually changes for the client?
3. Behind an AWS load balancer, your service sometimes produces `502` and
   sometimes `504`. Your app never returns either. What is each one telling you
   about your app?

---

## Part 1 — The anatomy of an exchange

**In plain words:** an HTTP message is a letter with a strict format. First line:
what you want (or, in a reply, the verdict). Then a list of labelled details,
one per line. Then a blank line. Then, optionally, the contents. The blank line
matters — it's how the reader knows the details have ended.

`curl -v` shows you both sides. `>` lines are what curl sent, `<` lines what came
back, `*` lines are curl's own notes:

```bash
# A full exchange. -v shows the request (>) and response (<) headers; the body goes to /dev/null.
curl -sv -o /dev/null https://httpbin.org/get 2>&1 | grep -E '^[<>*]' | grep -vE '^\* +(CAfile|CApath|TLSv|old SSL|Server cert|subject|start date|expire date|issuer|SSL cert|Using Stream|Connection state|Copying)'
```

Your exchange (trimmed):

```
* Connected to httpbin.org (100.58.34.217) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* ALPN, server accepted to use h2                ← during TLS, both agreed on HTTP/2 (Part 7)
> GET /get HTTP/2                                ← request line: method, path, version
> Host: httpbin.org                              ← which site — one IP can host many (required)
> user-agent: curl/7.81.0
> accept: */*                                    ← "any format is fine"
>                                                ← blank line: end of headers, no body
< HTTP/2 200                                     ← status line
< date: Thu, 08 Oct 2026 03:34:44 GMT
< content-type: application/json                 ← what the body is
< content-length: 256                            ← how long the body is (framing — Part 6)
< server: gunicorn/19.9.0
```

The same message on HTTP/1.1, which you typed by hand on Day 11 and Day 15:

```
GET /get HTTP/1.1\r\n
Host: httpbin.org\r\n
Accept: */*\r\n
\r\n
```

HTTP/2 sends the same fields in binary frames and lowercases header names, but
the **meaning is identical** — methods, paths, headers, status codes. Everything
in Parts 2–5 applies to every version.

The `Host` header deserves a moment. `104.154.89.105` served `expired.badssl.com`,
`self-signed.badssl.com` and `wrong.host.badssl.com` on Day 15 — one IP, many
sites. The server picks which one from `Host` (and, for HTTPS, from the TLS SNI —
Day 17). That's *virtual hosting*, and it's why an L7 load balancer can route by
hostname (Day 11).

> **Say this in an interview:** "A request is a request line — method, target,
> version — then headers, a blank line, and an optional body; a response is a
> status line, headers, blank line, body. HTTP/2 and HTTP/3 change the framing to
> binary but keep the same semantics. `Host` is mandatory because one IP serves
> many sites."

---

## Part 2 — Methods, safety and idempotency

**In plain words:** the method is a promise about the request's effect. "Safe"
means it only reads. "Idempotent" means you can send it twice and the result is
the same as sending it once — like pressing a lift button twice. The reason this
matters: networks fail mid-request, and the client often can't tell whether the
server did the work before the connection died. If the request is idempotent, it
can simply be sent again. If not, a retry can do the work twice.

| Method | Purpose | Safe | Idempotent | Body | Typical success |
|--------|---------|:----:|:----------:|------|-----------------|
| `GET` | Read a resource | ✔ | ✔ | No | `200` |
| `HEAD` | `GET` but headers only | ✔ | ✔ | No | `200` |
| `OPTIONS` | "What can I do here?" (also CORS preflight) | ✔ | ✔ | No | `200`/`204` + `Allow` |
| `PUT` | **Replace** the resource at this URL with this body | ✘ | ✔ | Yes | `200`, `201`, `204` |
| `DELETE` | Remove it | ✘ | ✔ | Rarely | `204`, `200` |
| `POST` | Create something new / trigger a process | ✘ | **✘** | Yes | `201` + `Location`, `200`, `202` |
| `PATCH` | Partially modify | ✘ | Not by default | Yes | `200`, `204` |

Why each is idempotent or not:

- `PUT /users/42 {"name":"Ada"}` twice → user 42 is named Ada. Same end state.
- `DELETE /users/42` twice → user 42 is gone. (The second call may return `404`;
  idempotency is about the **effect on the server**, not an identical response.)
- `POST /orders {"item":"book"}` twice → **two orders**.
- `PATCH /account {"op":"increment","field":"balance","by":10}` twice → +20. A
  PATCH that sets fields (`{"name":"Ada"}`) happens to be idempotent; the method
  doesn't promise it.

**Idempotency is a contract you implement, not something HTTP enforces.** A `GET`
that sends an email, or a `PUT` that appends to a list, breaks the contract — and
then the proxies, clients and browsers that *rely* on it will hurt you.

### The retry problem, and the fix

What actually happens in question 1:

```
client ──POST /payments──▶ server       server charges the card...
client  (500 ms timeout)                ...and is still writing the response
client ──POST /payments──▶ server       (retry) server charges the card again
```

The client saw a timeout, not a failure. It can't know the first attempt
succeeded. The standard fix is an **idempotency key**: the client generates a
unique ID per *logical* operation (a UUID), sends it in a header, and reuses it on
every retry. The server stores `key → result`; if the key arrives again, it
returns the stored result instead of doing the work twice. Stripe's API works
exactly like this, and so should any API of yours that moves money or creates
things on retry-prone paths. Hands-on 10 runs both versions against a real (local)
server and counts the orders.

> **Say this in an interview:** "Safe methods don't change server state;
> idempotent methods have the same effect however many times they're applied —
> GET, HEAD, OPTIONS, PUT, DELETE. POST isn't, so a POST retried after a timeout
> can duplicate work. The fix is an idempotency key: the client sends a unique key
> per operation and reuses it on retries, and the server stores the key with the
> result and returns the stored result for duplicates."

---

## Part 3 — Status codes that carry real meaning

**In plain words:** the first digit is the category — 2 "it worked", 3 "look
elsewhere", 4 "*you* did something wrong", 5 "*I* (or something behind me) failed".
The other two digits are precise, and clients make real decisions on them:
whether to retry, whether to log in again, whether to show the user an error or
fix their input. Picking the right one is part of designing an API.

### Success and redirects

| Code | Meaning | Use it when |
|------|---------|-------------|
| `200 OK` | Success, here's a body | Default for successful `GET`/`PUT`/`PATCH` |
| `201 Created` | Something new exists; `Location` header points to it | Successful `POST` that created a resource |
| `202 Accepted` | Received, will be processed later | Async jobs: "your report is being generated" |
| `204 No Content` | Success, deliberately no body | `DELETE`, or a `PUT` with nothing to return |
| `301 Moved Permanently` | It lives at `Location` now, forever | Domain/URL changes; browsers and caches remember it |
| `302 Found` | It's at `Location` *for now* | Temporary redirects, e.g. after login |
| `304 Not Modified` | Your cached copy is still good | Response to a conditional request (Part 4) |
| `307` / `308` | Same as `302` / `301`, but **the method and body must be kept** | Redirecting a `POST` without breaking it |

**The redirect trap.** For historical reasons, clients follow a `301`, `302` or
`303` with a **`GET`** and drop the body. Verified on your machine — `curl -L`
POSTing `item=book` through each redirect type to httpbin:

```
301 -> GET form= {}
302 -> GET form= {}
303 -> GET form= {}
307 -> POST form= {'item': 'book'}
308 -> POST form= {'item': 'book'}
```

If you redirect API traffic (an old path, `http` → `https`), use `307`/`308`, or
every POST through it silently becomes an empty GET.

### Client errors — "you did something wrong"

| Code | Meaning | The distinction that matters |
|------|---------|------------------------------|
| `400 Bad Request` | I can't even parse this | Malformed JSON, wrong types, missing required header |
| `422 Unprocessable Content` | I parsed it fine, but it's not valid | Well-formed JSON that fails business validation (`"age": -3`). Spring's default for validation errors is `400`; many APIs use `422` to make the difference explicit |
| `401 Unauthorized` | **Who are you?** No valid credentials | Must include a `WWW-Authenticate` header saying how to authenticate. The client should log in / refresh its token and try again |
| `403 Forbidden` | **I know who you are, and no.** | Authenticated but not allowed. Retrying with the same identity won't help |
| `404 Not Found` | Nothing here | Also used deliberately instead of `403` to avoid revealing that a resource exists |
| `405 Method Not Allowed` | Resource exists, wrong method | Must include `Allow`. httpbin: `POST /get` → `405`, `allow: OPTIONS, HEAD, GET` |
| `406 Not Acceptable` | I can't produce any format you `Accept` | httpbin `/image` with `Accept: text/plain` → `406` |
| `409 Conflict` | Your request clashes with the current state | Duplicate username; editing a record someone else changed since you read it |
| `415 Unsupported Media Type` | I don't accept that `Content-Type` | Sending XML to a JSON-only endpoint |
| `429 Too Many Requests` | Slow down | Rate limiting; should include `Retry-After` (seconds). Phase 14 builds this with Redis |

Question 2: **the reviewer is right.** "No token" is an *authentication* failure
→ `401` with `WWW-Authenticate: Bearer`. A well-behaved client treats `401` as
"get a fresh token and retry" and `403` as "stop — you're not allowed". Return
`403` for a missing token and clients that would have refreshed their token give
up instead. (The name "Unauthorized" is a historical misnomer — it really means
*unauthenticated*. Phase 8 is entirely about this distinction.)

```bash
# 401 comes with instructions; supplying credentials fixes it.
curl -si https://httpbin.org/basic-auth/user/pass | grep -iE '^(HTTP|www-authenticate)'
curl -s  -u user:pass https://httpbin.org/basic-auth/user/pass
```

### Server errors — and what a load balancer is telling you

| Code | Generated by | Means | Look at |
|------|--------------|-------|---------|
| `500 Internal Server Error` | **Your app** | Unhandled exception | Your app's logs and stack trace |
| `502 Bad Gateway` | **The proxy / LB** | It reached your app, but got an invalid response or the connection was **reset/closed** mid-request | App crashes, OOM kills, the keep-alive mismatch from Day 13 (your app closing an idle connection the LB was about to reuse) |
| `503 Service Unavailable` | Proxy/LB **or** your app | Nobody available to serve this: no healthy targets, app overloaded or in maintenance | Health checks failing, all instances down, deploy in progress; may carry `Retry-After` |
| `504 Gateway Timeout` | **The proxy / LB** | Your app accepted the request but didn't answer in time | Slow queries, thread pool or DB pool exhaustion (Day 13), a downstream call with no timeout |

Question 3: your app never *returns* `502` or `504` — the load balancer generates
them **about** your app. `502` = "I talked to your app and what came back was
broken or cut off" (a crash, or a connection reset). `504` = "I talked to your app
and it went silent past my timeout" (it's alive but stuck). Different causes,
different fixes. For AWS ALB, the 504 timeout is the LB's idle timeout (60 s
default).

```bash
# Every status code is just a number the server chose to send. One curl, one reused connection.
args=()
for code in 201 204 301 400 401 403 404 409 422 429 500 502 503 504; do
  args+=(-o /dev/null "https://httpbin.org/status/$code")
done
curl -s -w '%{url_effective} -> %{http_code}
' "${args[@]}" | sed 's|https://httpbin.org/status/|asked for |' | paste - -
```

> **Say this in an interview:** "401 means unauthenticated — no or invalid
> credentials, with a `WWW-Authenticate` challenge — and the client should
> re-authenticate; 403 means authenticated but not permitted. 400 is a malformed
> request, 422 a well-formed one that fails validation, 409 a conflict with
> current state, 429 rate limiting with `Retry-After`. Behind a load balancer,
> 502 means the upstream returned an invalid response or reset the connection,
> 503 means no healthy upstream, and 504 means the upstream didn't respond in
> time."

---

## Part 4 — Headers you'll live with

**In plain words:** headers are the labels on the envelope. Some describe the
contents ("this is JSON"), some say what you'd like back ("I prefer JSON"), some
prove who you are, some say how long you can keep a copy, and some are added by
the machines the message passed through on the way.

### `Content-Type` and `Accept` — content negotiation

- **`Content-Type`** describes *the body this message carries*. On a request:
  "I'm sending you JSON" (`application/json`). On a response: "this is JSON".
  Wrong or missing on a request → `415`, or your framework fails to parse the body
  (a Spring `@RequestBody` with `Content-Type: text/plain` → `415`).
- **`Accept`** is a request header: "formats I can handle, in order of
  preference". The server picks one and labels the response with `Content-Type`.
  If it can't satisfy any → `406`.

```bash
curl -s -o /dev/null -w 'Accept: image/webp  -> %{http_code} %{content_type}\n' -H 'Accept: image/webp'  https://httpbin.org/image
curl -s -o /dev/null -w 'Accept: text/plain  -> %{http_code} %{content_type}\n' -H 'Accept: text/plain'  https://httpbin.org/image
```

### `Authorization`

`Authorization: <scheme> <credentials>`. The two you'll meet:

- `Basic dXNlcjpwYXNz` — that's `user:pass` in **Base64, which is encoding, not
  encryption**: anyone who sees it can decode it (`echo dXNlcjpwYXNz | base64
  -d`). Only acceptable over HTTPS.
- `Bearer eyJhbGciOi…` — a token (often a JWT). Whoever *bears* it is treated as
  the owner, so it's as sensitive as a password. Phase 8 is all about these.

Never log the `Authorization` header. Spring and most HTTP client loggers mask
it — check yours does.

### `Cache-Control` — who may keep a copy, and for how long

| Directive | Means |
|-----------|-------|
| `max-age=60` | Fresh for 60 seconds — reuse without asking |
| `public` / `private` | May shared caches (CDNs, proxies) store it, or only the user's own browser? |
| `no-cache` | **May** store it, but must check with the server (revalidate) before each use |
| `no-store` | Must not store it at all — use for anything sensitive |

`no-cache` vs `no-store` is the classic confusion: `no-cache` does *not* mean
"don't cache". httpbin's `/cache/60` answers with `cache-control: public,
max-age=60`. An API returning account data should send `Cache-Control: no-store`
(or at least `private`).

### `ETag` and conditional requests — the cheap "has it changed?"

The server labels a response with a version tag: `ETag: "v1-abc"`. Next time,
the client asks *"only send it if it's no longer `v1-abc`"*:
`If-None-Match: "v1-abc"`. Unchanged → `304 Not Modified` with **no body**: the
client reuses its copy. Changed → `200` with the new body and a new `ETag`.

```bash
# 1) get the resource and its ETag  2) ask again with that ETag  3) ask with a stale one
curl -si https://httpbin.org/etag/v1-abc | grep -iE '^(HTTP|etag)'
curl -si -H 'If-None-Match: "v1-abc"' https://httpbin.org/etag/v1-abc | grep -iE '^(HTTP|etag|content-length)'
curl -si -H 'If-None-Match: "v0-old"' https://httpbin.org/etag/v1-abc | grep -iE '^HTTP'
```

`ETag`s also prevent lost updates: `PUT` with `If-Match: "v1-abc"` means "only
apply this if nobody changed it since I read it" — the server answers
`412 Precondition Failed` if they did. That's optimistic locking over HTTP; you'll
see the database version of it with JPA's `@Version` in Phase 7.

`Last-Modified` / `If-Modified-Since` does the same with a timestamp instead of a
tag (httpbin `/cache` returns `304` for any `If-Modified-Since`).

### `X-Forwarded-For` — the client's IP, as told by the proxies

On Day 11 you saw that an L7 proxy ends your TCP connection and opens a new one.
So the app behind it sees the *proxy's* IP as the client. Proxies therefore append
the address they received the request from to `X-Forwarded-For`. httpbin (behind
an AWS load balancer) reports it as `origin`:

```bash
curl -s https://httpbin.org/get | grep origin
curl -s -H 'X-Forwarded-For: 1.2.3.4' https://httpbin.org/get | grep origin
```

```
  "origin": "103.214.63.244",
  "origin": "1.2.3.4, 103.214.63.244",
```

The second line is the security lesson: **the client can write anything into
`X-Forwarded-For`**, and the proxy just appends to it. The leftmost entry is
whatever the client claimed. Only entries added by proxies **you** control can be
trusted — count from the right, one per trusted proxy. An app that takes the
leftmost value for rate limiting or IP allow-lists can be bypassed with one
header. In Spring Boot, `server.forward-headers-strategy=native` (or `framework`)
makes `request.getRemoteAddr()` honour these headers — only enable it when the app
is actually behind a proxy that sets them. Related headers: `X-Forwarded-Proto`
(was the original request `https`?), `X-Forwarded-Host`, and the standard
`Forwarded` header that combines them.

> **Say this in an interview:** "`Content-Type` labels the body; `Accept` drives
> content negotiation. `Authorization` carries Basic or Bearer credentials —
> Basic is only Base64. `Cache-Control` sets freshness and who may store it —
> `no-cache` means revalidate, `no-store` means never store. `ETag` with
> `If-None-Match` gives cheap `304` revalidation, and with `If-Match` gives
> optimistic concurrency. `X-Forwarded-For` carries the client IP through proxies,
> but only the entries appended by trusted proxies can be believed."

---

## Part 5 — Cookies and sessions

**In plain words:** HTTP has no memory: each request stands alone, so the server
can't tell that this request comes from the same person as the last one. Cookies
fix that. The server says "here's a ticket, show it every time" (`Set-Cookie`),
and the browser sends it back on every later request to that site (`Cookie`).
Usually the ticket is just a random ID, and the server keeps the real data — who
you are, what's in your cart — in a **session** stored on its side.

```bash
# The server sets a cookie with a redirect; a cookie jar stores it and sends it back.
curl -si 'https://httpbin.org/cookies/set?session=abc123' | grep -iE '^(HTTP|set-cookie|location)'
jar=$(mktemp)
curl -s -c "$jar" -L 'https://httpbin.org/cookies/set?session=abc123' >/dev/null   # -c: save cookies
curl -s -b "$jar" https://httpbin.org/cookies                                       # -b: send them
echo "--- the jar file:"; grep -v '^#' "$jar" | sed '/^$/d'
rm -f "$jar"
```

### Cookie attributes that matter for security

`Set-Cookie: SESSION=8f2c…; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=1800`

| Attribute | Effect |
|-----------|--------|
| `HttpOnly` | JavaScript can't read it — stops an injected script stealing the session |
| `Secure` | Only sent over HTTPS |
| `SameSite=Lax` / `Strict` / `None` | Whether it's sent on requests started by *other* sites — the main defence against CSRF |
| `Max-Age` / `Expires` | Lifetime; without either it's a *session cookie* that dies with the browser |
| `Domain` / `Path` | Which hosts and paths receive it |

### Sessions: where the state lives

- **Server-side session** (Spring's default, `JSESSIONID`): cookie holds a random
  ID; the server keeps the data. Easy to revoke (delete the session). But with
  three app instances behind a load balancer, instance B doesn't have the session
  instance A created. Two fixes: **sticky sessions** (the LB pins a user to one
  instance — breaks when that instance dies) or a **shared session store**
  (Spring Session + Redis — Phase 14).
- **Token-based** (JWT in an `Authorization: Bearer` header or a cookie): the
  token itself carries the identity, signed, so any instance can check it
  without shared storage. Harder to revoke before it expires. Phase 8 compares
  the two properly — "sessions vs tokens" is a guaranteed interview question.

> **Say this in an interview:** "HTTP is stateless; cookies add state by having
> the client echo back a value the server set. Session cookies should be
> `HttpOnly`, `Secure` and have a `SameSite` policy. Server-side sessions are
> easy to revoke but need sticky routing or a shared store like Redis when you
> scale out; tokens like JWTs are stateless but harder to revoke."

---

## Part 6 — Keep-alive and connection reuse

**In plain words:** you saw on Day 13 that opening a connection costs a round
trip or more, so HTTP/1.1 keeps the connection open after a response for the next
request. For that to work, the client has to know exactly where one response ends
and the next begins on the same stream of bytes — so every response must declare
its own length, or be sent in self-describing chunks.

```bash
# Two requests in one curl: watch it reuse the connection.
curl -sv -o /dev/null -o /dev/null --http1.1 http://httpbin.org/get http://httpbin.org/get 2>&1 \
  | grep -iE 'Connected to|Re-using|left intact'
```

You'll see `Re-using existing connection! (#0)` for the second request.

The rules:

- **HTTP/1.1 connections are persistent by default.** Either side sends
  `Connection: close` to end it after this response (Day 13 hands-on 9: one header
  turned 1 connection into 10).
- **Framing makes reuse possible.** A response body ends either after
  `Content-Length` bytes, or at the final zero-size chunk of
  `Transfer-Encoding: chunked` (Part 8). With neither, the only way to mark the
  end is to close the connection — HTTP/1.0's way, and why it couldn't reuse.
- **On HTTP/1.1, one connection carries one request at a time.** The next
  request waits until the previous response finishes — HTTP-level head-of-line
  blocking (Day 13). Clients work around it by opening several connections per
  host (browsers: 6; your pool: whatever you configure).
- **Idle connections get closed by someone.** The pool, the server, or a load
  balancer in between. When those timeouts disagree, you get the `Connection
  reset` / `502` problems from Day 13 and Part 3.

> **Say this in an interview:** "HTTP/1.1 keeps connections open by default and
> reuses them for sequential requests, which needs explicit framing —
> `Content-Length` or chunked encoding — so the client knows where each response
> ends. `Connection: close` ends it. One request at a time per connection is
> HTTP/1.1's head-of-line blocking, which is why clients pool several
> connections."

---

## Part 7 — HTTP/1.1 vs HTTP/2 vs HTTP/3

**In plain words:** same meaning, three different ways of putting it on the wire.
HTTP/1.1 is text, one request at a time per connection. HTTP/2 is binary and lets
many requests share one connection at the same time, each tagged with a stream
number, so nothing waits in line. HTTP/3 does the same over QUIC, on UDP, so that
a lost packet only delays the one stream it belonged to instead of all of them.

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---------|--------|--------|
| Transport | TCP | TCP | **QUIC over UDP** |
| Format | Text | Binary frames | Binary frames |
| Concurrent requests per connection | 1 | **Many (multiplexed streams)** | Many |
| Header compression | None | HPACK | QPACK |
| Head-of-line blocking | HTTP-level and TCP-level | TCP-level only — one lost packet stalls every stream | **None between streams** |
| Handshakes before first request | TCP + TLS | TCP + TLS | One combined QUIC+TLS handshake (can be 0-RTT on reconnect) |
| How it's chosen | Default | **ALPN** during TLS (`ALPN, server accepted to use h2`) | Server advertises `Alt-Svc: h3=":443"`; client tries QUIC next time |

See multiplexing for yourself — five parallel requests, counting new connections:

```bash
# 5 parallel requests: HTTP/1.1 needs a connection each; HTTP/2 shares one.
u=https://httpbin.org/get
for v in --http1.1 --http2; do
  printf '%-9s ' "$v"
  curl -sZ $v -o /dev/null -o /dev/null -o /dev/null -o /dev/null -o /dev/null \
       -w '%{http_version}:%{num_connects} ' $u $u $u $u $u
  echo
done
echo "Google advertises HTTP/3: $(curl -sI https://www.google.com/ | grep -i '^alt-svc' | cut -c1-40)"
```

On your machine: `1.1:1 1.1:1 1.1:1 1.1:1 1.1:1` (five connections, one each) vs
`2:0 2:0 2:0 2:0 2:1` (one connection carrying all five — which transfer gets
credited with opening it depends on which finishes first). Your `curl` 7.81 can't
speak HTTP/3 itself, but you can see Google advertising it.

Where you'll meet each: HTTP/1.1 between many internal services and in most load
balancer → app hops; **HTTP/2 for gRPC** (service-to-service RPC) and between
browsers and edges; HTTP/3 at CDNs and big public sites. Java's `HttpClient`
prefers HTTP/2 over HTTPS by default and falls back to 1.1.

> **Say this in an interview:** "HTTP/1.1 is text with one in-flight request per
> connection. HTTP/2 uses binary framing and multiplexes many streams over one TCP
> connection with header compression, negotiated via ALPN — but a lost TCP segment
> still stalls every stream. HTTP/3 runs over QUIC on UDP, so streams are
> independent, and the transport and TLS handshakes are combined."

---

## Part 8 — Chunked transfer and streaming (the mechanism behind SSE)

**In plain words:** normally a server builds the whole response, measures it, and
sends it with a `Content-Length`. But sometimes it doesn't know the length yet —
because it's still producing the content. An AI model generates a reply a word at
a time over 10 seconds; making the user stare at a spinner for 10 seconds and
then dumping the whole answer is a bad experience. **Chunked transfer** lets the
server send the body in pieces as they become ready, each piece prefixed with its
own size, with a zero-size piece at the end. **SSE** is a simple text format on
top of that for "a stream of events".

### Chunked on the wire

```bash
# httpbin streams 3 JSON lines with Transfer-Encoding: chunked (forced to HTTP/1.1 to show the header).
curl -sN --http1.1 -D - https://httpbin.org/stream/3 | grep -iE '^(HTTP|transfer-encoding|content-length)|"id"' | cut -c1-80
echo
# drip: 5 bytes spread over 2 seconds. First byte arrives early; total takes the full time.
curl -sN --http1.1 -o /dev/null -w 'first byte after %{time_starttransfer}s, done after %{time_total}s\n' \
  'https://httpbin.org/drip?numbytes=5&duration=2&delay=0'
```

There's `Transfer-Encoding: chunked` and **no** `Content-Length`. On the raw wire,
each chunk is `<size in hex>\r\n<data>\r\n`, ending with `0\r\n\r\n` — curl
removes that framing for you. (HTTP/2 doesn't use chunked encoding; it already
sends bodies as a sequence of DATA frames, which stream the same way.)

### Server-Sent Events

SSE is a text format inside a streaming response:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Transfer-Encoding: chunked

data: Streaming

data: lets

event: done
data: [DONE]

```

- Each event is one or more `field: value` lines, ended by a **blank line**.
- `data:` is the payload; `event:` names the event type; `id:` lets a client
  resume after a disconnect (`Last-Event-ID` header).
- One direction only: server → client, over one ordinary HTTP response. (For
  both directions you'd use WebSockets.)

**This is how AI APIs stream.** OpenAI's and Anthropic's APIs, with `stream:
true`, return `text/event-stream` and send each generated piece as a `data:`
event; Spring AI's streaming calls in Phase 9 consume exactly this, and in Spring
you'd produce it with `SseEmitter` or a `Flux<ServerSentEvent<…>>`. Hands-on 11
builds both ends in plain Java so you can see the tokens arrive.

### What breaks streaming in production

- **Buffering proxies.** nginx buffers upstream responses by default
  (`proxy_buffering on`), so the user gets everything at the end anyway. Streaming
  endpoints need buffering off (nginx honours an `X-Accel-Buffering: no` response
  header).
- **Compression** middleware that waits for the whole body.
- **Idle timeouts.** A load balancer closes a connection that sends nothing for 60
  seconds (ALB default) — a slow model that thinks for a minute gets cut off.
  Servers send periodic comment lines (`: keep-alive`) to prevent it.
- **Reading the whole body in your client.** `BodyHandlers.ofString()` waits for
  the end; streaming needs `ofLines()` or `ofInputStream()`.

> **Say this in an interview:** "Chunked transfer encoding lets a server send a
> body of unknown length as size-prefixed chunks terminated by a zero-length
> chunk, so it can start responding before it's done. Server-Sent Events is a
> `text/event-stream` format on top of that — `data:` lines separated by blank
> lines, one direction, server to client — and it's how LLM APIs stream tokens.
> In production you have to disable proxy buffering and handle idle timeouts."

---

## Hands-on

All blocks are **lab** blocks unless marked.

### 1. Read a full exchange (lab — required practice)

Run the Part 1 block. Then answer from your output: which HTTP version was used,
and which line proves it was negotiated rather than assumed? Which request header
is mandatory? Which response header lets the connection be reused?

Then rerun it with `--http1.1` added after `-sv`. What changes in the request
line, and in the case of the header names?

### 2. Methods and their contracts (lab)

```bash
# PUT and DELETE work on their endpoints; POST to a GET-only endpoint gets 405 + Allow.
curl -s -X PUT -H 'Content-Type: application/json' -d '{"name":"Ada"}' https://httpbin.org/put | grep -A3 '"json"'
printf 'DELETE -> %s\n' "$(curl -s -o /dev/null -w '%{http_code}' -X DELETE https://httpbin.org/delete)"
curl -si -X POST https://httpbin.org/get | grep -iE '^(HTTP|allow)'
curl -sI https://httpbin.org/get | head -1                 # HEAD: headers only
```

Then, on paper: classify each as safe / idempotent / neither, and say whether a
client may automatically retry it after a timeout:

1. `GET /orders/17`
2. `PUT /orders/17/status {"status":"SHIPPED"}`
3. `POST /orders/17/refunds {"amount":20}`
4. `DELETE /cart/items/3`
5. `PATCH /accounts/9 {"balanceDelta": -50}`
6. `GET /reports/daily?send=email` (which emails the report)

<details>
<summary>Answers</summary>

1. Safe and idempotent — retry freely.
2. Idempotent (sets a state; twice = same state) — retry.
3. Neither — a retry could refund twice. Needs an idempotency key.
4. Idempotent — second call may 404, but the effect is the same. Retry.
5. Not idempotent — a *delta* applied twice moves the balance twice. Make it
   `{"balance": 450}` with `If-Match`, or add an idempotency key.
6. **Broken contract.** It's a `GET`, so caches, prefetchers, crawlers and retry
   logic will assume it's safe and may send it repeatedly — emailing the report
   each time. It should be a `POST`.

</details>

### 3. Redirects (lab — required practice)

```bash
# A real 301: http -> https on github.com. Without -L you see the redirect; with -L curl follows it.
curl -sI http://github.com | grep -iE '^(HTTP|location)'
curl -sIL -o /dev/null -w 'after -L: %{http_code} at %{url_effective}, %{num_redirects} redirect(s)\n' http://github.com

# What each redirect type does to a POST and its body
for c in 301 302 303 307 308; do
  printf '%s -> ' "$c"
  curl -sL -d 'item=book' "https://httpbin.org/redirect-to?url=/anything&status_code=$c" \
    | python3 -c 'import json, sys; d = json.load(sys.stdin); print(d["method"], "body:", d["form"])'
done
```

Question: your team moves an API from `/v1/orders` to `/v2/orders` and adds a
`301` from the old path. Mobile apps still `POST` to `/v1/orders`. What happens to
their orders? What should the redirect have been — or should it be a redirect at
all?

<details>
<summary>Answer</summary>

Their POSTs are followed as **GETs without a body** — so they hit `GET
/v2/orders` (a list, or a 405) and no order is created, often with a 200
response the app treats as success. Use `308 Permanent Redirect`, which preserves
the method and body. Better still for APIs: keep `/v1` working (route it to the
same handler, or translate it) — not every HTTP client follows redirects at all,
and following a redirect to another host would drop the `Authorization` header in
many clients.

</details>

### 4. Status code tour (lab)

Run the status-code loop from Part 3, and the `401` block. (The loop passes all
14 URLs to **one** `curl` so they share one connection — about 5 s. A separate
`curl` per code took 18.5 s, too close to the lab's 20-second limit: Day 13's
connection-reuse lesson, applied.) Then decide the right
status for each:

1. JSON body is `{"email": "not-an-email"}` — valid JSON, invalid email.
2. Body is `{"email": "a@b.com"` — missing closing brace.
3. Valid token, but the user tries to delete someone else's order.
4. Expired token.
5. Creating a user whose email already exists.
6. A client sent 300 requests in a minute; the limit is 100.
7. `POST /reports` starts a 10-minute report job.

<details>
<summary>Answers</summary>

1. `422` (or `400` if your API convention lumps validation in — be consistent).
2. `400` — can't even parse it.
3. `403` — authenticated, not allowed. (`404` if you don't want to reveal the
   order exists.)
4. `401` with `WWW-Authenticate: Bearer error="invalid_token"` — client should
   refresh its token.
5. `409 Conflict`.
6. `429` with `Retry-After`.
7. `202 Accepted` with a `Location` pointing at a status resource
   (`/reports/jobs/123`).

</details>

### 5. Conditional requests and caching (lab — required practice)

Run the `ETag` block from Part 4. You should see `200` + `etag`, then **`304`**
for the matching tag, then `200` for the stale tag. Then:

```bash
curl -sI https://httpbin.org/cache/60 | grep -iE '^(HTTP|cache-control)'
curl -si -H 'If-Modified-Since: Thu, 01 Jan 2026 00:00:00 GMT' https://httpbin.org/cache | grep -iE '^HTTP'
```

Questions: what does the `304` response *not* contain, and why is that the
point? If a mobile app polls `GET /orders/17` every 5 seconds and the order
changes twice a day, what does adding `ETag` support save?

<details>
<summary>Answer</summary>

A `304` has no body — the client reuses the copy it already has. For the polling
app, nearly every poll becomes a tiny `304` instead of a full JSON body: less
bandwidth, less serialisation work, and the server can often answer from a cheap
version check without loading the whole order. The round trip still happens; to
avoid that too you'd add `Cache-Control: max-age`.

</details>

### 6. Spoofing `X-Forwarded-For` (lab)

Run the `X-Forwarded-For` block from Part 4. Then answer: your service is behind
exactly one AWS load balancer and rate-limits by client IP using the **leftmost**
`X-Forwarded-For` value. How does an attacker get unlimited requests? Which value
should you use instead?

<details>
<summary>Answer</summary>

Send a different fake `X-Forwarded-For: <random IP>` on every request — each looks
like a new client. With one trusted proxy, use the **rightmost** entry (the one
the load balancer appended — `103.214.63.244` in your test). With *n* trusted
proxies, take the *n*-th from the right.

</details>

### 7. Cookies (lab)

Run the Part 5 cookie block. In the jar file, find the cookie's name, value and
domain. Then: which three attributes would you require on a production session
cookie, and what attack does each stop?

### 8. Connection reuse and multiplexing (lab)

Run the Part 6 block, then the Part 7 block. Explain the `num_connects` numbers
in one sentence each. Then: on HTTP/1.1, how would a client send five requests at
the same time *without* five connections? (Trick question — why?)

<details>
<summary>Answer to the trick question</summary>

It can't, usefully. HTTP/1.1 *pipelining* (sending several requests before the
responses arrive) exists in the spec, but responses must come back in order, so
one slow response still blocks the rest, and proxies handled it so badly that
browsers disabled it. Real HTTP/1.1 concurrency means more connections; real
multiplexing needs HTTP/2.

</details>

### 9. Watch a body stream (lab)

Run the Part 8 chunked block. Confirm the response has `Transfer-Encoding:
chunked` and no `Content-Length`. In the `drip` line, why is "first byte" so much
earlier than "done"? What would a client using `BodyHandlers.ofString()` show the
user during that gap?

### 10. Idempotency keys vs a naive retry (lab)

A local server where the first `POST /orders` is slower than the client's
timeout. The client retries once, as many clients do. Count the orders.

```java
import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpServer;
import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.http.HttpTimeoutException;
import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class RetryDemo {

    // The "database": every order the server has created.
    static final List<String> orders = new CopyOnWriteArrayList<>();
    // Idempotency-Key -> the order id created for it.
    static final Map<String, String> seenKeys = new ConcurrentHashMap<>();

    public static void main(String[] args) throws Exception {
        ExecutorService workers = Executors.newFixedThreadPool(4);
        HttpServer server = HttpServer.create(new InetSocketAddress("127.0.0.1", 0), 0);
        server.setExecutor(workers);                      // so a slow request doesn't block the retry
        server.createContext("/orders", RetryDemo::createOrder);
        server.start();
        URI url = URI.create("http://127.0.0.1:" + server.getAddress().getPort() + "/orders");

        try (HttpClient client = HttpClient.newHttpClient()) {
            System.out.println("== 1. POST with a naive retry (no idempotency key)");
            postWithRetry(client, url, null);
            Thread.sleep(1_000);                          // let the slow server finish its work
            System.out.println("   orders in the database: " + orders);

            orders.clear();
            System.out.println("\n== 2. POST with an Idempotency-Key, same retry");
            postWithRetry(client, url, "key-7f3a");
            Thread.sleep(1_000);
            System.out.println("   orders in the database: " + orders);
        }
        server.stop(0);
        workers.shutdown();
    }

    /** Try once with a short timeout; on timeout, retry once — what many clients do by default. */
    static void postWithRetry(HttpClient client, URI url, String idempotencyKey) throws Exception {
        for (int attempt = 1; attempt <= 2; attempt++) {
            HttpRequest.Builder b = HttpRequest.newBuilder(url)
                    .timeout(Duration.ofMillis(500))
                    .POST(HttpRequest.BodyPublishers.ofString("{\"item\":\"book\",\"qty\":1}"));
            if (idempotencyKey != null) b.header("Idempotency-Key", idempotencyKey);
            try {
                HttpResponse<String> r = client.send(b.build(), HttpResponse.BodyHandlers.ofString());
                System.out.println("   attempt " + attempt + ": HTTP " + r.statusCode() + " " + r.body());
                return;
            } catch (HttpTimeoutException e) {
                System.out.println("   attempt " + attempt + ": timed out after 500 ms -> retrying");
            }
        }
    }

    static void createOrder(HttpExchange ex) throws IOException {
        ex.getRequestBody().readAllBytes();
        String key = ex.getRequestHeaders().getFirst("Idempotency-Key");
        String id;
        int status;
        if (key != null && seenKeys.containsKey(key)) {
            id = seenKeys.get(key);                       // seen this key: return the original result
            status = 200;
        } else {
            id = "order-" + (orders.size() + 1);
            if (key != null) seenKeys.put(key, id);       // remember the key BEFORE the slow part
            orders.add(id);
            status = 201;
            sleep(orders.size() == 1 ? 800 : 0);          // the first request is slower than the client's timeout
        }
        byte[] body = ("{\"id\":\"" + id + "\"}").getBytes(StandardCharsets.UTF_8);
        ex.getResponseHeaders().set("Content-Type", "application/json");
        ex.sendResponseHeaders(status, body.length);
        try (var out = ex.getResponseBody()) { out.write(body); }
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

Your run:

```
== 1. POST with a naive retry (no idempotency key)
   attempt 1: timed out after 500 ms -> retrying
   attempt 2: HTTP 201 {"id":"order-2"}
   orders in the database: [order-1, order-2]

== 2. POST with an Idempotency-Key, same retry
   attempt 1: timed out after 500 ms -> retrying
   attempt 2: HTTP 200 {"id":"order-1"}
   orders in the database: [order-1]
```

In case 1 the client *thinks* it created `order-2`, and there's an `order-1`
nobody knows about — the double charge from question 1. Then:

1. Why does the server store the key **before** the slow part, not after it
   finishes? What would happen with the order swapped?
2. Case 2 returns `200` on the retry, not `201`. Is that right? What do real APIs
   (Stripe) do?
3. `seenKeys` lives in one JVM's memory. What breaks with three instances behind
   a load balancer, and where would the keys live instead?

<details>
<summary>Answers</summary>

1. If the key were stored only after the work finished, the retry would arrive
   while the first attempt was still running, find no key, and create a second
   order — the same bug. Real implementations go further: they record the key as
   "in progress" atomically (e.g. a unique constraint in the database) and make a
   concurrent duplicate wait or get `409`.
2. Either is defensible; the key point is "same result as the first time".
   Stripe replays the **original** response — same status, same body — so the
   client can't tell it was a retry. Returning the original `201` is the more
   faithful choice.
3. A retry can land on a different instance that has never seen the key. The keys
   must live in shared storage — the database (a unique constraint on the key, in
   the same transaction as the order) or Redis with an expiry.

</details>

### 11. Stream tokens like an AI endpoint (lab)

Both ends of SSE in plain Java: a server that sends one word every 250 ms as a
`data:` event over a chunked response, and a client that prints each event the
moment it arrives.

```java
import com.sun.net.httpserver.HttpServer;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.util.stream.Stream;

public class StreamDemo {
    public static void main(String[] args) throws Exception {
        HttpServer server = HttpServer.create(new InetSocketAddress("127.0.0.1", 0), 0);
        server.createContext("/chat", ex -> {
            ex.getResponseHeaders().set("Content-Type", "text/event-stream");
            ex.getResponseHeaders().set("Cache-Control", "no-cache");
            ex.sendResponseHeaders(200, 0);                       // 0 = length unknown -> chunked
            try (OutputStream out = ex.getResponseBody()) {
                for (String token : "Streaming lets the user read while the model is still writing .".split(" ")) {
                    out.write(("data: " + token + "\n\n").getBytes(StandardCharsets.UTF_8));
                    out.flush();                                  // push this chunk onto the wire now
                    Thread.sleep(250);                            // pretend the model is thinking
                }
                out.write("event: done\ndata: [DONE]\n\n".getBytes(StandardCharsets.UTF_8));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        server.start();
        URI url = URI.create("http://127.0.0.1:" + server.getAddress().getPort() + "/chat");

        long start = System.nanoTime();
        try (HttpClient client = HttpClient.newHttpClient()) {
            HttpResponse<Stream<String>> resp = client.send(
                    HttpRequest.newBuilder(url).header("Accept", "text/event-stream").build(),
                    HttpResponse.BodyHandlers.ofLines());           // lines as they arrive, not the whole body
            System.out.printf("%5d ms  headers arrived: HTTP %d, Transfer-Encoding=%s%n",
                    ms(start), resp.statusCode(), resp.headers().firstValue("transfer-encoding").orElse("-"));
            resp.body()
                .filter(line -> line.startsWith("data: "))
                .forEach(line -> System.out.printf("%5d ms  %s%n", ms(start), line.substring(6)));
        }
        server.stop(0);
    }

    static long ms(long start) { return (System.nanoTime() - start) / 1_000_000; }
}
```

Your run:

```
  510 ms  headers arrived: HTTP 200, Transfer-Encoding=chunked
  521 ms  Streaming
  741 ms  lets
  994 ms  the
 1259 ms  user
 ...
 3312 ms  .
 3565 ms  [DONE]
```

Each word is printed ~250 ms after the previous one — the client is reading the
response while the server is still writing it. Then break it, two ways:

1. Change `BodyHandlers.ofLines()` to `BodyHandlers.ofString()` (and print the
   whole string at the end). When does anything appear now?
2. Put `ofLines()` back and comment out `out.flush()`. Does it still stream?
   Rerun and look at the timestamps.

<details>
<summary>What you should see</summary>

1. Nothing appears until ~3.5 s, then everything at once — `ofString()` waits for
   the end of the body. The server streamed; the **client** threw that away. This
   is the most common reason "streaming doesn't work" in a client.
2. It stops streaming. On your JDK the headers still arrived at ~500 ms, but
   **every word arrived together at ~3,550 ms** — the server buffered the small
   writes until the response ended. The client was doing everything right; the
   *server* held the data back. Whether a write reaches the wire immediately
   depends on buffering at every hop — which is why streaming frameworks flush per
   event, and why proxy buffering (Part 8) has to be switched off.

</details>

---

## Gotchas

- **301/302/303 turn a POST into a GET and drop the body.** Use 307/308 for
  anything that isn't a GET.
- **`401` is "who are you?", `403` is "no".** Getting it backwards stops clients
  from refreshing tokens.
- **Your app doesn't send 502/504 — the load balancer does, about your app.** 502
  = it got a broken or reset response; 504 = it got nothing in time.
- **Idempotency is a promise you keep in code.** A `GET` with side effects will be
  replayed by caches, crawlers and retry logic.
- **`no-cache` ≠ "don't cache".** It means "revalidate first". `no-store` means
  don't store.
- **`X-Forwarded-For` is client-controlled on the left.** Trust only what your own
  proxies appended.
- **Basic auth is Base64, not encryption.** And never log `Authorization`.
- **Header names are case-insensitive** — `Content-Type` and `content-type` are
  the same header. HTTP/2 always lowercases them; don't string-compare them
  case-sensitively.
- **Streaming dies in buffers:** proxy buffering, compression, or a client reading
  with `ofString()`. And idle timeouts cut long silent streams.
- **httpbin is a test service.** It returned an `ETag` without the quotes the
  standard requires (`etag: v1-abc` rather than `"v1-abc"`); real servers quote
  them. Don't copy a test server's quirks into your API.

---

## Recap

- **Message:** request line / status line, headers, blank line, body. Same
  meaning on HTTP/1.1, /2, /3.
- **Methods:** safe (GET, HEAD, OPTIONS), idempotent (those + PUT, DELETE), neither
  (POST, usually PATCH). Retries are only safe for idempotent requests — otherwise
  use an **idempotency key**.
- **Status codes:** 201 + `Location`, 202, 204; 301/302 vs **307/308** (keep the
  method); 400 vs 422; **401 vs 403**; 404, 405 + `Allow`, 406, 409, 415, **429 +
  `Retry-After`**; 500 (your app) vs **502/503/504** (the proxy, about your app).
- **Headers:** `Content-Type`/`Accept` (negotiation, 415/406), `Authorization`
  (Basic is Base64; Bearer tokens), `Cache-Control` (`no-cache` vs `no-store`),
  **`ETag` + `If-None-Match` → 304**, `If-Match` → 412, `X-Forwarded-For` (trust
  from the right).
- **Cookies** carry state; `HttpOnly`, `Secure`, `SameSite`. Server sessions need
  sticky routing or a shared store; tokens don't.
- **Keep-alive** needs framing (`Content-Length` or chunked); HTTP/1.1 = one
  request per connection at a time.
- **HTTP/2** multiplexes over one TCP connection (ALPN); **HTTP/3** over QUIC/UDP
  removes cross-stream head-of-line blocking.
- **Chunked + SSE** = start responding before you're done — how AI endpoints
  stream; buffering and `ofString()` silently break it.

---

## Say it out loud

1. What's the difference between safe and idempotent? Why does it matter when a
   request times out? How do idempotency keys work?
2. 401 vs 403, and 400 vs 422 — when do you use each?
3. A load balancer returns 502, 503 and 504 at different times. What does each
   tell you, and where do you look first?
4. Walk me through a conditional `GET` with `ETag`. What does the `304` save?
5. How does HTTP/2 improve on HTTP/1.1, and what problem does HTTP/3 solve that
   HTTP/2 can't?
6. How does an LLM API stream tokens over HTTP, and what can stop the stream
   reaching the user?

---

## Q&A

_Follow-up questions from reading this lesson go here._
