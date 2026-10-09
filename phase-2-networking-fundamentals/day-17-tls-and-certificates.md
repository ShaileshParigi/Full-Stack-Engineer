# Day 17 — TLS & Certificates

_Why this matters:_ Almost every connection your services make is TLS — to
databases, to other services, to Stripe, to an LLM API — and TLS fails in ways
that look nothing like normal bugs. The most common Java error you'll ever
Google is `PKIX path building failed: unable to find valid certification path
to requested target`, and it nearly always means one specific thing (a trust
store problem), not "the network is broken". Certificates **expire on a date
someone forgot**, and expired certificates have taken down Microsoft Teams, a
national mobile network, and thousands of systems at once when a root
certificate retired. And in Phase 16 you'll run a service mesh where **every**
service-to-service call is mutual TLS — which is today's material with
automation on top. This is the foundation, so don't skim it.

> **This is a long day.** If it runs past two hours, split it: Parts 1–5 and
> hands-on 1–4 today; Parts 6–10 and hands-on 5–8 next session. The required
> practice is hands-on 1, 2 and 4.

---

## The one-paragraph version

TLS wraps a TCP connection so that nobody in between can **read** it
(confidentiality), **change** it unnoticed (integrity), or **pretend to be the
server** (authentication). The first two are done with fast *symmetric*
encryption using a key both sides share; the trick is agreeing on that key over
a network anyone can watch, which needs slower *asymmetric* (public-key)
cryptography — so TLS uses both: public-key maths in the **handshake** to agree
on a key, then symmetric encryption for the data. Authentication is the
**certificate**: a document that says "this public key belongs to `httpbin.org`",
digitally signed by a **certificate authority (CA)**. Your client checks that
signature, then the signature on the CA's certificate, up a **chain** until it
reaches a **root** CA it already trusts — one listed in its **trust store**. It
also checks the name you connected to is in the certificate, and that today's
date is inside its validity period. Fail any check and the connection is refused
— which is what self-signed certificates, expired certificates and wrong
hostnames all trigger. **SNI** tells the server which site's certificate to use
when one IP hosts many. **mTLS** is the same thing in both directions: the client
proves its identity with a certificate too.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **TLS** | The "S" in HTTPS: an encrypted, authenticated wrapper around a connection | Transport Layer Security; current versions 1.2 and 1.3 |
| **SSL** | The old name for TLS | SSL 3.0 and older are broken; people still say "SSL" meaning TLS |
| **confidentiality** | Nobody else can read it | Encryption |
| **integrity** | Nobody can change it without being caught | Message authentication (AEAD ciphers like AES-GCM) |
| **authentication** | You're talking to who you think you are | Proven with a certificate and a signature |
| **symmetric encryption** | One shared key locks and unlocks | AES — fast, but both sides need the same secret |
| **asymmetric / public-key** | A key *pair*: what one key locks, only the other unlocks | RSA, elliptic curves (EC) — slow, but the public half can be shared openly |
| **public key / private key** | The half you hand out / the half you never reveal | |
| **key exchange** | Two strangers agreeing on a shared secret while everyone listens | (EC)DHE — Diffie-Hellman with throwaway keys |
| **digital signature** | Proof that the holder of a private key approved this exact data | Hash the data, sign the hash with the private key; anyone verifies with the public key |
| **hash** | A short fingerprint of data; any change gives a different fingerprint | SHA-256 |
| **forward secrecy** | Stealing the server's key later doesn't unlock recordings of old traffic | Session keys come from ephemeral (EC)DHE keys that are thrown away |
| **certificate** | An ID card: "this public key belongs to this name", signed by an issuer | X.509 certificate |
| **subject / issuer** | Who the certificate is for / who signed it | Distinguished names like `CN=httpbin.org` |
| **SAN** | The list of names the certificate is valid for | Subject Alternative Name extension — the field clients actually check |
| **validity** | The dates the certificate is good between | `notBefore` / `notAfter` |
| **CA** | An organisation trusted to sign certificates | Certificate Authority |
| **root / intermediate / leaf** | The top CA / a CA the root vouches for / the server's own certificate | A chain: leaf ← signed by intermediate ← signed by root |
| **trust store** | The list of root CAs a client trusts | `/etc/ssl/certs` on Ubuntu; `cacerts` in the JDK; the Windows certificate store |
| **self-signed** | A certificate that vouches for itself | Issuer = subject, signed with its own key — no CA behind it |
| **CSR** | A request to a CA: "please sign this public key for this name" | Certificate Signing Request |
| **PKIX** | The standard rules for checking a certificate chain | What Java's error message names |
| **SNI** | The client saying which site it wants, at the start of the handshake | Server Name Indication — sent in plaintext |
| **cipher suite** | The set of algorithms a connection uses | e.g. `ECDHE-RSA-AES128-GCM-SHA256` |
| **TLS termination** | The point where TLS is decrypted | Often a load balancer, not your app |
| **mTLS** | Both sides prove who they are with certificates | Mutual TLS — the client sends a certificate too |

---

## Before you read: three questions

1. Your Java service calls an internal API and fails with `PKIX path building
   failed`. From the same machine, `curl` to the same URL works perfectly. How?
2. A server's certificate is valid for `api.example.com`. You run
   `curl https://10.0.1.5/` (its IP) and get a certificate error. The certificate
   isn't expired or self-signed. What's wrong?
3. An AWS load balancer serves your API over HTTPS. Is the traffic between the
   load balancer and your Spring app encrypted?

---

## Part 1 — What TLS provides

**In plain words:** without TLS, everything you send crosses many machines you
don't control — your Wi-Fi router, Windows, your ISP, a dozen routers (Day 11) —
and any of them could read it, change it, or answer instead of the real server.
TLS gives you three guarantees against those three attacks.

| Guarantee | Stops | How |
|-----------|-------|-----|
| **Confidentiality** | Someone on the path reading passwords, tokens, data | Symmetric encryption (AES) with a key only the two ends know |
| **Integrity** | Someone changing data in transit ("pay £10" → "pay £1000") | Every record carries an authentication tag; a changed byte fails the check and the connection is killed |
| **Authentication** | Someone pretending to be the server (a hijacked DNS answer, a rogue Wi-Fi hotspot) | The server proves it holds the private key for a certificate a trusted CA issued for that name |

The third is the one people underestimate. Encryption alone is useless if you're
encrypting to the attacker — you'd have a perfectly private conversation with the
wrong person. That's why certificate checks matter, and why `curl -k` and Java
"trust all certificates" code are so dangerous: they keep the encryption and
throw away the authentication.

What TLS does **not** hide: the IP addresses and ports (Day 11's headers are
outside it), the amount and timing of traffic, and — usually — the hostname you
asked for (SNI, Part 7). Day 18 shows exactly what a packet capture can still see.

> **Say this in an interview:** "TLS provides confidentiality through symmetric
> encryption, integrity through authenticated encryption so tampering is detected,
> and server authentication through a CA-signed certificate. Without
> authentication, encryption is worthless — you could be encrypting to a
> man-in-the-middle — which is why disabling certificate validation is a real
> vulnerability."

---

## Part 2 — Symmetric vs asymmetric: why TLS uses both

**In plain words:** imagine a lockbox. **Symmetric** is a normal lock: one key
locks and unlocks, so both people need a copy — but how do you get the key to the
other person without someone copying it on the way? **Asymmetric** is a special
padlock with two keys: anyone can *close* it with the public key you hand out, but
only your private key opens it. That solves the "how do we share a key" problem,
but it's about a thousand times slower. So TLS uses the special padlock just once,
at the start, to agree on a normal key, and then uses the fast normal key for
everything else.

| | Symmetric (AES) | Asymmetric (RSA, EC) |
|---|-----------------|----------------------|
| Keys | One shared secret | A public/private pair |
| Speed | Very fast — hardware accelerated, gigabytes per second | Slow — used for small things only |
| Problem it solves | Encrypting lots of data | Agreeing on a secret, and signatures |
| Problem it has | Both sides must already share the key | Too slow for bulk data |
| In TLS | Encrypts every byte after the handshake | Key exchange + proving identity in the handshake |

### The two jobs public-key crypto does in a handshake

1. **Key exchange (ECDHE).** Both sides generate a throwaway key pair, swap the
   public halves, and each combines its own private half with the other's public
   half. The maths gives both the **same** secret, while an eavesdropper who saw
   both public halves can't compute it. That secret becomes the AES key. Because
   the throwaway keys are deleted afterwards, a recording of today's traffic can't
   be decrypted even if the server's long-term key is stolen next year —
   **forward secrecy**. (The `ECDHE` at the start of `ECDHE-RSA-AES128-GCM-SHA256`
   is this.)
2. **Signatures (RSA or ECDSA).** The server signs part of the handshake with the
   private key matching its certificate. Only the real owner of that certificate
   could produce that signature. (The `RSA` in the suite name.) Signing is also
   how a CA "vouches" for a certificate (Part 5).

Read the whole cipher suite your machine negotiated with httpbin:

| `ECDHE` | `RSA` | `AES128-GCM` | `SHA256` |
|---------|-------|--------------|----------|
| key exchange | server's signature algorithm (its cert has an RSA key) | symmetric cipher for the data, with integrity built in | hash used in the handshake's key derivation |

> **Say this in an interview:** "Asymmetric crypto solves key distribution but is
> slow; symmetric crypto is fast but needs a shared key. TLS uses ephemeral
> elliptic-curve Diffie-Hellman to agree on a session key, the server's
> certificate key to sign the handshake, and then AES-GCM for the data. Ephemeral
> key exchange gives forward secrecy."

---

## Part 3 — The handshake

**In plain words:** before any HTTP is sent, client and server have a short
conversation. Client: "Hi, here's what I support, and I want `httpbin.org`." Server:
"Let's use these settings; here's my ID card, and here's my half of the key
exchange, signed." Client checks the ID card, sends its half, and both work out the
same secret key. Each confirms with a "finished" message encrypted under the new
key. Then the real request flows, encrypted.

`curl -v` narrates it. Your machine against httpbin (TLS 1.2), trimmed:

```
* TLSv1.3 (OUT), TLS handshake, Client hello (1):         ← versions, cipher suites, SNI "httpbin.org", ALPN h2/http1.1
* TLSv1.3 (IN),  TLS handshake, Server hello (2):         ← server picks TLS 1.2 + one suite
* TLSv1.2 (IN),  TLS handshake, Certificate (11):         ← the certificate chain (Part 5)
* TLSv1.2 (IN),  TLS handshake, Server key exchange (12): ← server's ECDHE public half, SIGNED with its cert's key
* TLSv1.2 (IN),  TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16): ← client's ECDHE public half
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):  ← "from now on, encrypted"
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN),  TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN, server accepted to use h2
*  subject: CN=httpbin.org
*  subjectAltName: host "httpbin.org" matched cert's "httpbin.org"
*  issuer: C=US; O=Amazon; CN=Amazon RSA 2048 M01
*  SSL certificate verify ok.
```

(The first two lines say `TLSv1.3` because curl offered 1.3; the server chose
1.2. The `TLS header, Certificate Status` lines in your raw output are curl
labelling record headers — ignore them.)

```
 client                                          server
   │── ClientHello: versions, suites, SNI, ALPN ──▶│
   │◀─ ServerHello, Certificate, KeyExchange(sig) ─│   1 round trip
   │   [client verifies chain, name, dates, sig]   │
   │── KeyExchange, ChangeCipherSpec, Finished ───▶│
   │◀──────────── ChangeCipherSpec, Finished ──────│   2nd round trip
   │══════════ encrypted HTTP request ═══════════▶│
```

### TLS 1.2 vs 1.3

| | TLS 1.2 | TLS 1.3 |
|---|---------|---------|
| Round trips before data | **2** | **1** (0 on resumption, with replay caveats) |
| Certificate sent | In plaintext | **Encrypted** |
| Algorithms | Many, some weak (RSA key exchange, CBC) | Only modern ones; forward secrecy mandatory |
| Your examples | httpbin.org (`TLSv1.2`) | 1.1.1.1 (`TLSv1.3`, `TLS_AES_256_GCM_SHA384`) |

That extra round trip is why httpbin's `curl -w` from Day 13 showed ~0.95 s to
finish TLS on a ~0.25 s RTT link: TCP's 1 RTT + TLS 1.2's 2 RTTs + the server's
work.

**Where verification happens:** after the `Certificate` message the client checks
(1) the chain up to a trusted root, (2) the name, (3) the dates, and then (4)
the signature on the key exchange proves the server actually holds the private
key. A failure at any point aborts the handshake — which is why TLS errors in
Java appear as `SSLHandshakeException`.

> **Say this in an interview:** "The client sends ClientHello with supported
> versions, cipher suites, SNI and ALPN; the server answers with ServerHello, its
> certificate chain and a signed ephemeral key share; the client validates the
> chain, hostname and dates, completes the key exchange, and both send Finished
> messages under the new keys. TLS 1.2 takes two round trips, TLS 1.3 one, and
> 1.3 encrypts the certificate."

---

## Part 4 — Certificates: what's inside one

**In plain words:** a certificate is an ID card for a server. It names the server,
says who issued the card, when it's valid, and contains the server's public key —
and the whole card is signed by the issuer so it can't be forged or edited.

Decode httpbin's (hands-on 2 is this command):

```
subject=CN = httpbin.org
issuer=C = US, O = Amazon, CN = Amazon RSA 2048 M01
notBefore=Jun 19 00:00:00 2026 GMT
notAfter=Jan  2 23:59:59 2027 GMT
X509v3 Subject Alternative Name:
    DNS:httpbin.org, DNS:*.httpbin.org
X509v3 Key Usage: critical
    Digital Signature, Key Encipherment
X509v3 Extended Key Usage:
    TLS Web Server Authentication
X509v3 Basic Constraints: critical
    CA:FALSE
Public-Key: (2048 bit)          Signature Algorithm: sha256WithRSAEncryption
sha256 Fingerprint=73:8E:8A:F7:38:B6:19:CA:...
```

| Field | Meaning | What breaks if it's wrong |
|-------|---------|---------------------------|
| **Subject** `CN=httpbin.org` | Who it's for. The `CN` (Common Name) is legacy — **modern clients ignore it for hostname checks** | Nothing on its own |
| **SAN** `httpbin.org, *.httpbin.org` | **The names it's valid for.** This is what's checked | Connecting with any other name → hostname mismatch |
| `*.httpbin.org` | A wildcard covers **exactly one** label: `api.httpbin.org` ✔, `a.b.httpbin.org` ✘, `httpbin.org` ✘ (hence both entries) | |
| **Issuer** | Who signed it — the next link up the chain | Untrusted issuer → `PKIX` / `unable to get local issuer certificate` |
| **Validity** | 19 June 2026 → 2 January 2027: **197 days** | Outside it → `certificate has expired` / `not yet valid` (check clocks!) |
| **Key Usage / EKU** | What the key may be used for: "TLS Web Server Authentication" | A client certificate used as a server certificate, or vice versa, is rejected |
| **Basic Constraints** `CA:FALSE` | This certificate may **not** sign other certificates | Without it, the owner of any leaf could mint certificates for any site |
| **Public key** | The key the server proves it holds | — |
| **Signature** | The issuer's signature over all of the above | Any edit invalidates it |
| **Fingerprint** | SHA-256 hash of the whole certificate — a unique ID | Used for *pinning* (fragile; avoid unless you must) |

Question 2's answer is in that table: `curl https://10.0.1.5/` checks whether
`10.0.1.5` is in the SAN list. It isn't — the certificate says `api.example.com`.
The fix is to connect by name (or, for testing, `curl --resolve
api.example.com:443:10.0.1.5 https://api.example.com/` — Day 18). Certificates
*can* contain IP SANs (1.1.1.1's does), but most don't.

> **Say this in an interview:** "An X.509 certificate binds a public key to names
> in the Subject Alternative Name extension, with an issuer, validity period, key
> usage constraints and a CA flag, all signed by the issuer. Clients check the
> hostname against the SANs — the CN is legacy — and a wildcard matches exactly
> one label."

---

## Part 5 — The chain of trust, and trust stores

**In plain words:** your computer doesn't know httpbin. But it was shipped with a
short list of organisations it trusts (about 150 root CAs). httpbin's certificate
is signed by an Amazon CA; that CA's certificate is signed by Amazon's root; and
Amazon's root is on your list. Each signature vouches for the next link down, so
trust flows from the root you already had to the server you've never met.

The chain httpbin sends (hands-on 1):

```
 0 s:CN = httpbin.org                               ← leaf: the server's certificate
   i:C = US, O = Amazon, CN = Amazon RSA 2048 M01
   v:NotBefore: Jun 19 00:00:00 2026 GMT; NotAfter: Jan  2 23:59:59 2027 GMT
 1 s:C = US, O = Amazon, CN = Amazon RSA 2048 M01   ← intermediate
   i:C = US, O = Amazon, CN = Amazon Root CA 1
   v:NotBefore: Aug 23 2022; NotAfter: Aug 23 2030
 2 s:C = US, O = Amazon, CN = Amazon Root CA 1      ← root (a cross-signed copy — see below)
   i:C = US, ... CN = Starfield Services Root Certificate Authority - G2
   v:NotBefore: May 25 2015; NotAfter: Dec 31 2037
```

`s:` is subject, `i:` issuer. Each certificate's issuer is the next one's
subject. Verifying:

1. Check the leaf's signature using the intermediate's public key.
2. Check the intermediate's signature using `Amazon Root CA 1`'s public key.
3. Is `Amazon Root CA 1` in my trust store? **Yes** → trusted. Stop.

Certificate 2 is interesting: it's Amazon's root, signed by an *older* root
(Starfield). That's a **cross-signature** — for very old clients that have
Starfield in their trust store but not Amazon's root. Modern clients stop at step
3 and ignore it.

**Why intermediates exist:** the root's private key is kept offline in a vault
and used rarely. Day-to-day signing happens with intermediates, which can be
revoked and replaced if compromised without touching every trust store on earth.

**A classic misconfiguration — the missing intermediate.** If a server sends only
its leaf, clients can't build the chain. Browsers often fetch the missing piece
themselves (so "it works in Chrome"), but Java and `curl` don't — they fail with
`unable to get local issuer certificate` / `PKIX path building failed`. Always
serve the full chain (leaf + intermediates; the root is optional).

### There are three trust stores on your machine

| Who | Trust store | Count on your machine |
|-----|-------------|----------------------:|
| `curl`, `openssl`, Python in WSL | `/etc/ssl/certs/ca-certificates.crt` (Ubuntu's `ca-certificates` package) | 146 |
| **Java** (any JVM) | **The JDK's own `lib/security/cacerts`** | 152 (your Temurin 21) |
| Browsers, PowerShell, .NET on Windows | Windows certificate store | (its own list) |

That's question 1. A company's internal CA gets installed into the **operating
system** store by IT, so `curl` trusts it — but the JVM doesn't read the OS
store. It uses its own `cacerts`, which doesn't have it. Result: `PKIX path
building failed` from Java, success from `curl`. Fixes (most to least
recommended for a Spring service):

- **Spring Boot SSL bundles** — point the client at the CA explicitly:
  ```yaml
  spring:
    ssl:
      bundle:
        pem:
          internal-ca:
            truststore:
              certificate: classpath:certs/internal-ca.pem
  ```
- A dedicated trust store for the JVM: `-Djavax.net.ssl.trustStore=…` (replaces
  the default list entirely — include the public roots if you also call public
  APIs).
- Import into the JDK's `cacerts` with `keytool -importcert` — works, but is lost
  whenever the JDK is upgraded or the container image changes.
- **Never** a "trust all certificates" `TrustManager` or disabled hostname
  verification. That turns off authentication (Part 1) for every connection.

> **Say this in an interview:** "The server sends its leaf and intermediate
> certificates; the client verifies each signature up the chain until it reaches
> a root in its trust store. Java uses its own `cacerts` rather than the OS store,
> which is why an internal CA can work for curl and fail in Java with 'PKIX path
> building failed'. The fix is to add the CA to the application's trust
> configuration, not to disable validation."

---

## Part 6 — Self-signed certificates: why they fail, and when that's fine

**In plain words:** a self-signed certificate is an ID card you printed yourself.
It may be perfectly accurate, but nobody else has a reason to believe it — no
trusted authority vouched for it. So clients refuse it, unless you've told them in
advance "trust this one". That's fine when *you* control both ends (development,
internal systems with your own private CA), and wrong when strangers need to trust
you (anything public — use a real CA; Let's Encrypt is free).

Two kinds you'll make:

| | Self-signed leaf | Your own private CA |
|---|------------------|---------------------|
| What | One certificate, issuer = subject | A CA certificate (`CA:TRUE`) that signs your server certificates |
| Client must trust | That exact certificate | The CA, once — then every certificate it signs |
| Good for | Quick local test | Dev environments, internal services, **mTLS** (Part 9) |
| Rotating a server cert | Every client must be updated | Clients don't change |

Hands-on 4 builds a private CA, signs a `localhost` certificate with it, serves
HTTPS with it, and shows you every outcome:

```
--- 1. default trust store           curl: (60) SSL certificate problem: unable to get local issuer certificate
--- 2. -k (skip ALL verification)    200
--- 3. --cacert ca.crt (trust our CA)  200
--- 4. right CA, wrong name           curl: (60) SSL: no alternative certificate subject name matches target host name '127.0.0.1'
```

The right answer is 3. Option 2, `-k` (`--insecure`), is fine for a one-off look
at a dev server and **never** belongs in a script, a health check, or code — it
accepts *any* certificate, including an attacker's.

> **Say this in an interview:** "A self-signed certificate fails validation
> because no trusted CA vouches for it. It's appropriate where you control every
> client — dev, or internal systems using a private CA that you distribute to
> clients' trust stores. The fix is always to trust the specific CA, never to
> disable verification."

---

## Part 7 — SNI: one IP, many HTTPS sites

**In plain words:** on Day 16, the `Host` header told a server which site you
wanted. But with HTTPS, the server has to pick a certificate *before* it can read
any HTTP — the `Host` header is inside the encryption it hasn't set up yet. So the
client says the hostname in the very first handshake message, in the clear. That's
SNI.

On Day 15 you noticed `expired.badssl.com`, `self-signed.badssl.com` and
`badssl.com` all live on `104.154.89.105`. Same IP, same port — different
certificates, chosen by SNI:

```
expired.badssl.com       subject=OU = Domain Control Validated, OU = PositiveSSL Wildcard, CN = *.badssl.com
self-signed.badssl.com   subject=C = US, ST = California, L = San Francisco, O = BadSSL, CN = *.badssl.com
badssl.com               subject=CN = *.badssl.com
(no SNI)                 subject=... O = BadSSL Fallback. Unknown subdomain or no SNI., CN = badssl-fallback-unknown-subdomain-or-no-sni
```

Without SNI, the server has to guess and sends a default — here a deliberately
named fallback. Consequences:

- **Every load balancer, CDN and Kubernetes ingress hosting many sites depends on
  SNI.**
- **SNI is plaintext.** Anyone on the path can see which site you're visiting even
  though the content is encrypted (Day 18 finds it in a capture). *Encrypted
  Client Hello* (ECH) fixes this, but it's only starting to be deployed.
- In code, connecting by IP sends no hostname SNI. Use the name and let DNS (or
  `--resolve`) supply the IP.

> **Say this in an interview:** "SNI is a plaintext field in the ClientHello that
> names the requested host, so a server with one IP can choose the right
> certificate before any HTTP is exchanged. It's how load balancers and CDNs host
> many TLS sites, and it means the hostname is visible on the wire unless
> Encrypted Client Hello is used."

---

## Part 8 — TLS termination vs end-to-end encryption

**In plain words:** TLS protects one hop: from the client to wherever the TLS
ends. In most real systems it ends at a load balancer, which decrypts the request
(so it can read the HTTP and route it — Day 11's L7) and then sends it on to your
app, either in plain HTTP or in a *new* TLS connection. Whether that second hop is
encrypted is a decision someone made — or forgot to make.

```
A) Terminate at the LB (common default)
   client ══TLS══▶ ALB ──plain HTTP──▶ app:8080          inside the VPC, unencrypted

B) Terminate and re-encrypt
   client ══TLS══▶ ALB ══new TLS══▶ app:8443             two separate TLS sessions; LB sees plaintext

C) Passthrough (end to end)
   client ══════════════TLS══════════════▶ app:8443      NLB forwards TCP; only the app can decrypt

D) Service mesh (Phase 16)
   app ─plain─▶ sidecar ══mTLS══▶ sidecar ─plain─▶ app  every hop between pods encrypted and authenticated
```

| | Terminate | Re-encrypt | Passthrough |
|---|-----------|------------|-------------|
| LB can route by path/host, add headers | ✔ | ✔ | ✘ (sees only bytes; can still use SNI) |
| Certificates to manage | One, on the LB (AWS ACM renews it for you) | LB + every app | Every app |
| Traffic inside the network | **Plaintext** | Encrypted | Encrypted |
| App knows the original request was HTTPS | Only via `X-Forwarded-Proto` | Same | Yes |

Question 3: **by default, no.** An ALB with an HTTP target group sends plaintext
to your app. Whether that's acceptable depends on your threat model and compliance
rules (many — PCI DSS, HIPAA environments — require encryption in transit inside
the network too). Two practical consequences for a Spring app behind option A:
it must trust `X-Forwarded-Proto` (`server.forward-headers-strategy`) or it will
build `http://` redirect URLs and think requests are insecure; and anything that
must be end-to-end (mTLS with client certificates) can't terminate at an L7 LB.

> **Say this in an interview:** "TLS is hop-by-hop: a load balancer that
> terminates TLS sees plaintext and forwards either plaintext or a re-encrypted
> connection, while passthrough keeps it end to end but loses L7 routing. In AWS,
> an ALB terminates; an NLB can pass through. Behind a terminating proxy the app
> relies on `X-Forwarded-Proto`, and a mesh like Istio adds mTLS on every internal
> hop."

---

## Part 9 — mTLS preview: both sides show a certificate

**In plain words:** normal TLS proves the *server's* identity; the client stays
anonymous at the TLS level and proves who it is later, with a password or token
inside the HTTP. **Mutual TLS** adds one step: the server also asks for the
client's certificate and checks it against a CA it trusts. Now both sides know who
they're talking to before a single byte of HTTP. For one service talking to
another — no human, no password to type — that's an excellent identity.

The handshake gains two messages: the server sends a **CertificateRequest**
("show me yours, signed by one of these CAs"), and the client replies with its
**Certificate** plus a signature proving it holds the private key.

Hands-on 5 runs a server that requires client certificates and greets the caller
by the name in its certificate:

```
--- no client certificate          curl: (56) ... tlsv13 alert certificate required
--- certificate from our CA        hello, orders-service
--- self-made certificate          curl: (56) ... tlsv1 alert unknown ca
```

The third line is the point: a certificate with the right *name* but not signed by
the CA the server trusts is rejected. Identity comes from the signature, not the
name on the card.

Where this goes:

- **Phase 8 (Spring Security):** a client certificate is one way for a *service
  account* — a machine identity — to authenticate. Spring Security's
  `x509()` support maps the certificate's subject to a principal.
- **Phase 16 (Istio):** the mesh gives every pod a certificate automatically
  (SPIFFE identities), rotates them every few hours, and enforces mTLS between all
  services — "strict mTLS". It's exactly hands-on 5, run by a control plane.

> **Say this in an interview:** "In mutual TLS the server sends a
> CertificateRequest and the client presents a certificate chained to a CA the
> server trusts, proving possession of the private key. It authenticates both
> sides at the transport layer, which makes it a strong workload identity for
> service-to-service traffic — it's what service meshes enforce."

---

## Part 10 — Certificate expiry: a leading cause of outages

**In plain words:** every certificate has an end date. When it passes, every
client refuses the connection — instantly, completely, at a moment nobody chose.
Nothing about the code or the servers changed; a date just arrived. Because
renewal happens rarely, the knowledge of how to do it leaves with whoever did it
last.

Real ones:

- **December 2018 — O2 (UK) and SoftBank (Japan):** an expired certificate in
  Ericsson software took down mobile data for tens of millions of customers for
  most of a day.
- **February 2020 — Microsoft Teams:** an authentication certificate expired; Teams
  was down for hours.
- **30 September 2021 — Let's Encrypt's old root (DST Root CA X3) expired.** Sites
  were fine, but older clients that didn't have the newer root in their trust
  store suddenly couldn't connect to millions of sites — including old JVMs and
  embedded devices.

And it's getting more frequent by design: the CA/Browser Forum has voted to
shrink maximum public certificate lifetimes — **200 days** for certificates issued
from March 2026 (httpbin's is 197), 100 days from 2027, **47 days from 2029**.
Manual renewal is no longer viable. The answer is automation:

| Where | Automation |
|-------|------------|
| AWS load balancers | **ACM** (AWS Certificate Manager) issues and renews automatically |
| Kubernetes | **cert-manager** renews certificates via ACME (Let's Encrypt) |
| Anything else | An ACME client (`certbot`), plus **monitoring of expiry dates** |

And monitor anyway — automation fails silently too. Hands-on 7 builds the check.

> **Say this in an interview:** "Certificate expiry is a top outage cause because
> renewal is rare and manual. Public certificate lifetimes are dropping towards 47
> days, so issuance must be automated — ACM, cert-manager, ACME — and expiry must
> be monitored independently, with alerts weeks ahead."

---

## Hands-on

All blocks are **lab** blocks.

### 1. Inspect a real chain (lab — required practice)

```bash
# The chain httpbin.org sends: s = subject, i = issuer, v = validity.
openssl s_client -connect httpbin.org:443 -servername httpbin.org -showcerts </dev/null 2>/dev/null \
  | grep -E '^ *[0-9] s:|^ +i:|^ +v:'
echo
echo "Roots in WSL's trust store: $(grep -c 'BEGIN CERTIFICATE' /etc/ssl/certs/ca-certificates.crt)"
```

Then do the same for `github.com` and `www.google.com`. For each: how many
certificates does the server send? Who is the root? Which certificate in the
chain expires first — and is that the one you'd monitor?

### 2. Decode a certificate (lab — required practice)

```bash
# Pull httpbin's leaf certificate and print the fields that matter.
openssl s_client -connect httpbin.org:443 -servername httpbin.org </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -fingerprint -sha256 \
      -ext subjectAltName,keyUsage,extendedKeyUsage,basicConstraints
```

Answer from your output: which names is it valid for? Would it be valid for
`api.httpbin.org`? For `eu.api.httpbin.org`? How many days was it issued for, and
why does that number matter now? Then replace the second command with `openssl
x509 -noout -text` and find the public key size and the signature algorithm in
the full dump.

<details>
<summary>Answers</summary>

`httpbin.org` and `*.httpbin.org`. `api.httpbin.org` — yes (one label).
`eu.api.httpbin.org` — no (two labels; a wildcard covers exactly one). Issued 19
June 2026 → 2 January 2027 ≈ 197 days: just under the new 200-day maximum for
public certificates issued since March 2026.

</details>

### 3. Watch the handshake (lab)

```bash
# curl narrates every handshake message, then what it verified.
curl -sv -o /dev/null https://httpbin.org/get 2>&1 \
  | grep -E 'TLS handshake|change cipher|SSL connection|ALPN, server|subject:|issuer:|verify ok|matched'
echo "--- a TLS 1.3 server, for comparison"
openssl s_client -connect 1.1.1.1:443 -servername one.one.one.one -brief </dev/null 2>&1 \
  | grep -E 'Protocol|Ciphersuite|Peer certificate'
```

Map each `TLS handshake` line onto the diagram in Part 3. Which message carries
the certificate? Which carries the signed key share? Then run the same `curl`
against `https://1.1.1.1/` — how many handshake lines are there now, and why are
some of them labelled differently? (Hint: TLS 1.3 encrypts most of its handshake.)

### 4. Your own CA, your own HTTPS server (lab — required practice)

The whole practice from the plan in one block: make a CA, issue a `localhost`
certificate, serve HTTPS, and watch `curl` reject it, then accept it two ways.
Everything happens in a temporary directory.

```bash
cd "$(mktemp -d)"

# 1. A private CA: a self-signed certificate with CA:TRUE
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout ca.key -out ca.crt -days 30 -subj "/CN=Learn Daily Dev CA" 2>/dev/null

# 2. A server key + CSR, then the CA signs it — valid ONLY for the name "localhost"
openssl req -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout server.key -out server.csr -subj "/CN=localhost" 2>/dev/null
printf 'subjectAltName=DNS:localhost\nextendedKeyUsage=serverAuth\nbasicConstraints=critical,CA:FALSE\n' > server.ext
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 30 -extfile server.ext 2>/dev/null
openssl verify -CAfile ca.crt server.crt

# 3. Serve HTTPS on 8443 with that certificate
python3 - <<'EOF' 2>/dev/null &
import http.server, ssl
srv = http.server.HTTPServer(("127.0.0.1", 8443), http.server.SimpleHTTPRequestHandler)
ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ctx.load_cert_chain("server.crt", "server.key")
srv.socket = ctx.wrap_socket(srv.socket, server_side=True)
srv.serve_forever()
EOF
server=$!
sleep 1

echo "--- 1. default trust store";       curl -sS -o /dev/null https://localhost:8443/ 2>&1 | head -1
echo "--- 2. -k (skip ALL verification)"; curl -sS -k -o /dev/null -w '%{http_code}\n' https://localhost:8443/
echo "--- 3. --cacert (trust our CA)";    curl -sS --cacert ca.crt -o /dev/null -w '%{http_code}\n' https://localhost:8443/
echo "--- 4. right CA, wrong name";       curl -sS --cacert ca.crt -o /dev/null https://127.0.0.1:8443/ 2>&1 | head -1
kill "$server"
```

Then:

1. Run `openssl x509 -in ca.crt -noout -subject -issuer -ext basicConstraints`
   (add it before `kill`). Why are subject and issuer the same?
2. Fix case 4 *properly*: change `server.ext` so the certificate is also valid for
   `127.0.0.1` (`subjectAltName=DNS:localhost,IP:127.0.0.1`), rerun, confirm.
   (On success, case 4 prints nothing — add `-w '%{http_code}\n'` to see the
   `200`.)
3. Explain why case 2 succeeding is a *problem*, not a feature.

### 5. mTLS: the server checks the client too (lab)

Same CA, now the server **requires** a client certificate signed by it and greets
the client by name.

```bash
cd "$(mktemp -d)"
ec() { openssl req -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes "$@" 2>/dev/null; }

# CA, server cert for localhost, client cert for "orders-service"
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout ca.key -out ca.crt -days 30 -subj "/CN=Learn Daily Dev CA" 2>/dev/null
ec -keyout server.key -out server.csr -subj "/CN=localhost"
printf 'subjectAltName=DNS:localhost\nextendedKeyUsage=serverAuth\n' > server.ext
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 30 -extfile server.ext 2>/dev/null
ec -keyout client.key -out client.csr -subj "/CN=orders-service"
printf 'extendedKeyUsage=clientAuth\n' > client.ext
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 30 -extfile client.ext 2>/dev/null

# A server that REQUIRES a client certificate from our CA
python3 - <<'EOF' &
import http.server, ssl
class Hello(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        subject = dict(field[0] for field in self.connection.getpeercert()["subject"])
        body = f"hello, {subject['commonName']}\n".encode()
        self.send_response(200)
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)
    def log_message(self, *args):
        pass
srv = http.server.HTTPServer(("127.0.0.1", 8444), Hello)
ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ctx.load_cert_chain("server.crt", "server.key")
ctx.verify_mode = ssl.CERT_REQUIRED          # <- this line is "mutual"
ctx.load_verify_locations("ca.crt")          # <- client certs must chain to this CA
srv.socket = ctx.wrap_socket(srv.socket, server_side=True)
srv.serve_forever()
EOF
server=$!
sleep 1

echo "--- no client certificate"
curl -sS --cacert ca.crt https://localhost:8444/ 2>&1 | head -1
echo "--- certificate from our CA"
curl -sS --cacert ca.crt --cert client.crt --key client.key https://localhost:8444/
echo "--- self-made certificate with the same name, not from our CA"
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout rogue.key -out rogue.crt -days 1 -subj "/CN=orders-service" 2>/dev/null
curl -sS --cacert ca.crt --cert rogue.crt --key rogue.key https://localhost:8444/ 2>&1 | head -1
kill "$server"
```

Questions: which line makes it *mutual*? Why is the rogue certificate rejected
even though its name is identical? (Notice the failures are curl exit 56 — an
error while *reading* — not 35: in TLS 1.3 the server checks the client
certificate after the client already thinks the handshake is done.)

### 6. SNI picks the certificate (lab)

```bash
# One IP, four answers — chosen by the name in the ClientHello.
for name in expired.badssl.com self-signed.badssl.com badssl.com; do
  printf '%-24s ' "$name"
  openssl s_client -connect 104.154.89.105:443 -servername "$name" </dev/null 2>/dev/null | openssl x509 -noout -subject
done
printf '%-24s ' "(no SNI)"
openssl s_client -connect 104.154.89.105:443 -noservername </dev/null 2>/dev/null | openssl x509 -noout -subject
```

Why did the "no SNI" connection still get *a* certificate? What would a client
that connected by IP have done with it?

### 7. An expiry monitor (lab)

```bash
# Days until each certificate expires; warn under 30.
check_expiry() {
  local host=$1 end days
  end=$(openssl s_client -connect "$host:443" -servername "$host" </dev/null 2>/dev/null \
        | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
  if [[ -z $end ]]; then printf '%-22s could not fetch certificate\n' "$host"; return 1; fi
  days=$(( ( $(date -d "$end" +%s) - $(date +%s) ) / 86400 ))
  printf '%-22s %4d days  (%s)%s\n' "$host" "$days" "$end" "$( (( days < 30 )) && echo '  <-- RENEW')"
}
for h in httpbin.org github.com google.com expired.badssl.com; do check_expiry "$h"; done
```

`expired.badssl.com` should come out negative. Then: `openssl x509 -checkend
2592000` exits non-zero if a certificate expires within 30 days (2,592,000 s) —
rewrite `check_expiry` to use it, so the function's *exit status* can drive a
cron alert (Day 5, Day 9).

### 8. TLS from Java: the chain, a failure, and the hostname trap (lab)

Runs on Windows' JDK, so it uses **the JDK's** trust store. Takes ~7 seconds.

```java
import java.net.InetSocketAddress;
import java.net.Socket;
import java.security.KeyStore;
import java.security.cert.Certificate;
import java.security.cert.X509Certificate;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import javax.net.ssl.SNIHostName;
import javax.net.ssl.SSLParameters;
import javax.net.ssl.SSLSession;
import javax.net.ssl.SSLSocket;
import javax.net.ssl.SSLSocketFactory;
import javax.net.ssl.TrustManagerFactory;
import javax.net.ssl.X509TrustManager;

public class TlsInspect {

    public static void main(String[] args) throws Exception {
        // 1. Which CAs does this JVM trust? (its own store, not Windows' and not WSL's)
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        tmf.init((KeyStore) null);                                   // null = the JDK's default cacerts
        X509TrustManager tm = (X509TrustManager) tmf.getTrustManagers()[0];
        System.out.println("JVM trusts " + tm.getAcceptedIssuers().length + " root CAs");

        // 2. A real handshake, and the chain the server sent
        System.out.println("\n== httpbin.org");
        try (SSLSocket s = connect("httpbin.org", "httpbin.org", true)) {
            SSLSession session = s.getSession();
            System.out.println("protocol " + session.getProtocol() + ", cipher " + session.getCipherSuite());
            Certificate[] chain = session.getPeerCertificates();
            for (int i = 0; i < chain.length; i++) {
                X509Certificate c = (X509Certificate) chain[i];
                long days = Duration.between(Instant.now(), c.getNotAfter().toInstant()).toDays();
                System.out.printf("  [%d] subject: %s%n      issuer:  %s%n      expires in %d days%n",
                        i, c.getSubjectX500Principal().getName(), c.getIssuerX500Principal().getName(), days);
                if (i == 0) System.out.println("      SANs:    " + c.getSubjectAlternativeNames());
            }
        }

        // 3. A certificate no trusted CA signed
        System.out.println("\n== self-signed.badssl.com");
        try (SSLSocket s = connect("self-signed.badssl.com", "self-signed.badssl.com", true)) {
            System.out.println("handshake OK?!");
        } catch (Exception e) {
            System.out.println("  " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        // 4. The hostname trap: we MEANT to reach api.example.com, but landed on httpbin.org
        //    (think: hijacked DNS). httpbin presents a perfectly valid certificate — for httpbin.org.
        System.out.println("\n== expecting api.example.com, actually talking to httpbin.org");
        for (boolean checkHostname : new boolean[] {false, true}) {
            try (SSLSocket s = connect("httpbin.org", "api.example.com", checkHostname)) {
                System.out.println("  hostname check " + (checkHostname ? "ON " : "OFF") + " -> handshake OK, talking to "
                        + ((X509Certificate) s.getSession().getPeerCertificates()[0]).getSubjectX500Principal().getName());
            } catch (Exception e) {
                System.out.println("  hostname check " + (checkHostname ? "ON " : "OFF") + " -> " + e.getClass().getSimpleName()
                        + ": " + e.getMessage());
            }
        }
    }

    /**
     * Opens TCP to connectHost:443, then runs TLS as if we were talking to expectedHost.
     * The certificate CHAIN is always verified against the trust store; the HOSTNAME only
     * when checkHostname is true — raw SSLSocket does not check it by default.
     */
    static SSLSocket connect(String connectHost, String expectedHost, boolean checkHostname) throws Exception {
        Socket tcp = new Socket();
        tcp.connect(new InetSocketAddress(connectHost, 443), 5_000);
        SSLSocket s = (SSLSocket) ((SSLSocketFactory) SSLSocketFactory.getDefault())
                .createSocket(tcp, expectedHost, 443, true);
        s.setSoTimeout(5_000);
        SSLParameters p = s.getSSLParameters();
        p.setServerNames(List.of(new SNIHostName(expectedHost)));     // SNI: the site we THINK we are reaching
        if (checkHostname) p.setEndpointIdentificationAlgorithm("HTTPS");
        s.setSSLParameters(p);
        s.startHandshake();
        return s;
    }
}
```

Your run:

```
JVM trusts 152 root CAs

== httpbin.org
protocol TLSv1.2, cipher TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  [0] subject: CN=httpbin.org
      issuer:  CN=Amazon RSA 2048 M01,O=Amazon,C=US
      expires in 86 days
      SANs:    [[2, httpbin.org], [2, *.httpbin.org]]
  [1] subject: CN=Amazon RSA 2048 M01,O=Amazon,C=US
  ...

== self-signed.badssl.com
  SSLHandshakeException: PKIX path building failed: ... unable to find valid certification path to requested target

== expecting api.example.com, actually talking to httpbin.org
  hostname check OFF -> handshake OK, talking to CN=httpbin.org
  hostname check ON  -> SSLHandshakeException: No subject alternative DNS name matching api.example.com found.
```

What to take from it:

- **152 vs 146**: the JVM's trust store is not WSL's (and not Windows'). Question
  1, measured.
- **`PKIX path building failed`** now has a concrete meaning: "I couldn't chain
  this certificate to any of my 152 roots."
- **The hostname trap.** With the check OFF, Java completed a fully "verified"
  handshake with a server holding someone else's valid certificate. A raw
  `SSLSocket` checks the chain but **not** the hostname unless you set
  `setEndpointIdentificationAlgorithm("HTTPS")`. `HttpClient`, `HttpsURLConnection`
  and Spring's clients do set it — the danger is custom socket code, old
  libraries, and anyone who "fixed" a certificate error by disabling hostname
  verification. Same bug as Day 15's `openssl s_client` without
  `-verify_hostname`.

(While building this, the check first appeared to pass even when ON: the JDK
verifies against the **SNI** name when one is set, and the demo was sending SNI
for `httpbin.org`. Sending SNI for the host you *meant* — as a real client
would — makes it fail correctly.)

---

## Gotchas

- **Java doesn't use the OS trust store.** `curl` works, Java says `PKIX` →
  the CA is missing from the JVM's trust configuration.
- **`-k` and trust-all `TrustManager`s disable authentication**, not just a
  warning. Never in code, scripts or health checks.
- **Hostname verification is a separate check from the chain.** `openssl
  s_client` needs `-verify_hostname`; raw `SSLSocket` needs
  `setEndpointIdentificationAlgorithm("HTTPS")`.
- **The CN is ignored; the SAN is what counts.** A certificate with
  `CN=api.example.com` and no SAN fails in modern clients.
- **Wildcards cover one label.** `*.example.com` does not cover `example.com` or
  `a.b.example.com`.
- **Connecting by IP** fails the name check (and sends no useful SNI) unless the
  certificate has an IP SAN.
- **Serve the full chain.** "Works in the browser, fails in Java/curl" is usually a
  missing intermediate.
- **Check clocks.** A machine with a wrong date sees valid certificates as expired
  or not yet valid.
- **Terminating at a load balancer means plaintext behind it** unless you
  re-encrypt.
- **Lifetimes are shrinking (200 → 100 → 47 days).** Automate renewal; monitor
  expiry anyway.
- **Private keys are secrets.** `server.key`, `client.key` and `ca.key` must never
  be committed to git or baked into an image; the CA key above all — whoever holds
  it can mint certificates every client trusts.

---

## Recap

- TLS = **confidentiality + integrity + authentication**. Without authentication,
  encryption protects you from nobody.
- **Asymmetric** crypto for the handshake (ECDHE key exchange + certificate
  signature), **symmetric** (AES-GCM) for the data; ephemeral keys → forward
  secrecy.
- **Handshake:** ClientHello (versions, suites, SNI, ALPN) → ServerHello,
  Certificate, signed key share → client verifies → key exchange → Finished. 1.2 =
  2 RTT, 1.3 = 1 RTT and encrypts the certificate.
- **Certificate:** SAN (what's checked), issuer, validity, key usage,
  `CA:FALSE`, signature.
- **Chain of trust:** leaf ← intermediate ← root in the **trust store**. Three
  stores on your machine: WSL (146), JDK (152), Windows.
- **Self-signed / private CA:** trust the CA explicitly (`--cacert`, SSL bundles,
  trust store); never disable verification.
- **SNI** chooses the certificate on shared IPs; it's plaintext.
- **Termination** at a proxy = plaintext behind it unless re-encrypted;
  passthrough keeps it end to end.
- **mTLS:** the client presents a certificate too — workload identity, the basis
  of Phase 16.
- **Expiry** causes outages; automate (ACM, cert-manager, ACME) and monitor.

---

## Say it out loud

1. What three things does TLS guarantee? Why is authentication the one you can't
   skip?
2. Why does TLS use both asymmetric and symmetric cryptography? What's forward
   secrecy?
3. Walk me through a TLS handshake, and what the client verifies about the
   certificate.
4. Explain the chain of trust from a server certificate to a root CA. Why does
   Java sometimes fail with `PKIX path building failed` when curl works?
5. What is SNI, and why does it exist?
6. What's the difference between terminating TLS at a load balancer and mTLS
   between services? When would you want each?

---

## Q&A

_Follow-up questions from reading this lesson go here._
