# Encrypted DNS and DNSSEC Cheatsheet

Encrypted DNS protects a transport path. DNSSEC authenticates DNS data. They
solve different problems and are often strongest when used together.

Last checked: 2026-07-18. Linux examples target Debian 13 and Ubuntu
24.04/26.04 LTS.

## Table of Contents

- [Practical exercise](#practical-exercise)
- [Security properties](#security-properties)
- [Threat boundaries](#threat-boundaries)
- [Plain DNS](#plain-dns)
- [DNS over TLS](#dns-over-tls)
- [DNS over HTTPS](#dns-over-https)
- [DNS over QUIC](#dns-over-quic)
- [Oblivious DNS over HTTPS](#oblivious-dns-over-https)
- [Transport comparison](#transport-comparison)
- [Discovery and bootstrap](#discovery-and-bootstrap)
- [Linux client support](#linux-client-support)
- [`systemd-resolved` with DoT](#systemd-resolved-with-dot)
- [Test encrypted DNS with `kdig`](#test-encrypted-dns-with-kdig)
- [`dig`, `delv`, OpenSSL, and curl](#dig-delv-openssl-and-curl)
- [DNSSEC mental model](#dnssec-mental-model)
- [DNSSEC record types](#dnssec-record-types)
- [Validation states](#validation-states)
- [DNSSEC flags](#dnssec-flags)
- [Validate and troubleshoot DNSSEC](#validate-and-troubleshoot-dnssec)
- [Authoritative operator overview](#authoritative-operator-overview)
- [Private and split DNS](#private-and-split-dns)
- [Choosing an approach](#choosing-an-approach)
- [Packet capture](#packet-capture)
- [Troubleshooting workflow](#troubleshooting-workflow)
- [Avoid these mistakes](#avoid-these-mistakes)
- [References](#references)

---

## Practical exercise

First discover what the installed tools can actually do:

```bash
dig -v
dig -h 2>&1 | rg '\+(tls|https|quic|dnssec)'

if command -v kdig >/dev/null; then
  kdig -V
  kdig -h 2>&1 | rg '\+(tls|https|quic)'
fi
```

Request DNSSEC records over ordinary DNS:

```bash
NAME='ietf.org'
dig "$NAME" A +dnssec +noall +answer +comments
```

If `kdig` from `knot-dnsutils` is installed, compare strict DoT and DoH to the
same resolver:

```bash
NAME='ietf.org'
RESOLVER_IP='1.1.1.1'
DOT_NAME='one.one.one.one'
DOH_NAME='cloudflare-dns.com'

kdig "@$RESOLVER_IP" +tls-ca \
  "+tls-hostname=$DOT_NAME" "$NAME" A

kdig "@$RESOLVER_IP" +https=/dns-query +tls-ca \
  "+tls-hostname=$DOH_NAME" "$NAME" A
```

The endpoint is an example, not a recommendation. Use a resolver whose policy
and certificate name you have verified.

```text
ordinary DNS:  client ── readable query ─────────────────────────► resolver
DoT / DoH:     client ══ encrypted, authenticated channel ═══════► resolver
DNSSEC:        client or resolver ◄── signed DNS data ── zone owner
```

| Observe | Meaning |
| --- | --- |
| DoT and DoH return the same RRset | The DNS message can be identical while its transport differs. |
| `RRSIG` appears after `+dnssec` | The server returned DNSSEC material; this alone does not prove local validation. |
| `ad` appears in a reply | The responding resolver claims it validated the answer. Trust that claim only over a trusted path. |
| Certificate validation fails | The encrypted channel did not authenticate the configured resolver identity. Do not hide the failure with an insecure mode. |

---

## Security properties

Separate these properties when evaluating a design:

| Property | Question |
| --- | --- |
| Confidentiality | Can an observer read the queried name or answer? |
| Transport integrity | Can an on-path party alter this client-to-resolver exchange undetected? |
| Resolver authentication | Did the client connect to the intended recursive resolver? |
| DNS data origin authentication | Can a validator prove the data was signed by the delegated zone? |
| Client unlinkability | Can one party associate the client address with the cleartext DNS question? |
| Availability | Will resolution still work when a transport or validator fails? |

No mechanism in this sheet provides every property.

```text
DoT / DoH / DoQ ─► protect one network hop to a resolver
ODoH            ─► separates client identity from query contents across two parties
DNSSEC          ─► authenticates DNS data from zone signer to validator
```

DNSSEC does not encrypt names. Encrypted DNS does not prove that an unsigned
answer originated with the domain owner.

---

## Threat boundaries

A typical encrypted stub-to-recursive path is:

```text
application ─► local stub ══ encrypted DNS ══► recursive resolver ─► authoritative DNS
                   ▲                                  ▲                       ▲
             host can see query              resolver sees query       usually plain DNS
```

Encrypted stub DNS can protect against an observer on Wi-Fi or the access
network. It does not prevent these parties from learning everything:

- The application and operating system already know the requested name.
- The chosen recursive resolver sees the client identity and query.
- Destination IP addresses and traffic timing can reveal contacted services.
- Recursive-to-authoritative traffic may remain unencrypted.
- Malware on either endpoint can see data before encryption or after decryption.

Choose the threat boundary before choosing a protocol.

---

## Plain DNS

Traditional DNS normally uses UDP port 53 and retries over TCP when needed.

```text
client ── UDP/53 query ──► resolver
client ◄─ UDP/53 answer ── resolver
```

Properties:

- No transport confidentiality.
- No authenticated resolver identity.
- DNS transaction IDs, source-port randomization, and DNS cookies are not encryption.
- DNSSEC can authenticate signed data even when transported in plaintext.
- Network policy and troubleshooting are simple because the traffic is visible.

```bash
dig @192.0.2.53 example.com A
dig @192.0.2.53 example.com A +tcp
```

---

## DNS over TLS

DNS over TLS (DoT) carries DNS wire messages in TLS over TCP, normally on port
853.

```text
client ── TCP handshake ──► resolver:853
client ══ TLS handshake ══► authenticated resolver
client ══ DNS messages over persistent TLS connection ══► resolver
```

Important properties:

- A dedicated port makes the protocol identifiable and easy to permit or block.
- Persistent connections and TLS resumption reduce handshake cost.
- Strict mode validates a configured resolver name or key.
- Opportunistic mode encrypts when possible but can permit downgrade or an unauthenticated peer.
- The resolver still sees the client address and cleartext question.

Inspect a DoT certificate without sending a DNS message:

```bash
openssl s_client \
  -connect '1.1.1.1:853' \
  -servername 'one.one.one.one' \
  -verify_hostname 'one.one.one.one' \
  -brief </dev/null
```

Successful TLS proves control of the configured certificate identity, not the
truth of every DNS answer.

---

## DNS over HTTPS

DNS over HTTPS (DoH) maps each DNS query and response to an HTTPS exchange,
normally on TCP port 443. HTTP/2 and HTTP/3 can multiplex requests.

```text
client ─► HTTPS origin `/dns-query` ─► HTTP request carrying DNS wire message
client ◄────────────────────────────── HTTP response `application/dns-message`
```

RFC 8484 supports:

- HTTP `POST` with the DNS wire message as the body.
- HTTP `GET` with a base64url-encoded `dns` query parameter.
- Media type `application/dns-message`.
- Normal HTTPS server authentication and HTTP cache semantics.

`application/dns-json` endpoints are provider-specific APIs, not the RFC 8484
wire format. A successful JSON query does not prove a standards-compliant DoH
client works.

Operational characteristics:

- Port 443 can traverse networks that block dedicated DNS transports.
- DoH traffic shares infrastructure and observability with general HTTPS.
- Browsers may select a DoH resolver independently of the operating system.
- HTTP proxying, authentication, redirects, content types, and status codes add failure modes.
- The DoH resolver still sees the client address and question.

---

## DNS over QUIC

DNS over QUIC (DoQ) carries DNS over dedicated QUIC connections, normally on
UDP port 853.

```text
client ══ QUIC + TLS 1.3 over UDP/853 ══► resolver
          stream 0: query A
          stream 4: query AAAA       independent loss recovery
```

Properties:

- QUIC provides encryption and resolver authentication using TLS 1.3.
- Separate streams avoid TCP-level head-of-line blocking between queries.
- Connection establishment and resumption can reduce latency.
- UDP port 853 is easy for a firewall to identify or block.
- NAT rebinding and network changes can be handled better than a fixed TCP connection.
- The resolver still sees the client address and question.

DoQ is not DoH over HTTP/3. Both use QUIC, but DoQ carries DNS directly on a
dedicated QUIC connection while HTTP/3 DoH retains HTTP semantics.

---

## Oblivious DNS over HTTPS

Oblivious DoH (ODoH) adds a relay and encrypts the DNS message for a target.
RFC 9230 is Experimental.

```text
                          encrypted ODoH message
client ── HTTPS ──► relay ─────────────────────────► target resolver
  │                   │                                  │
  │ knows query       │ sees client IP + target         │ sees query + relay IP
  │ and client IP     │ cannot decrypt DNS message      │ cannot see client IP
  └───────────────────┴──────────────────────────────────┘
             privacy requires relay and target not to collude
```

The client obtains the target's public configuration, encrypts the DNS query
with HPKE, and sends the encapsulated message through the relay. The target
decrypts and resolves it, then encrypts the response back to the client.

ODoH improves separation of identity and query contents, but:

- The relay and target together can reconstruct the association if they collude.
- Timing, message size, and traffic analysis remain relevant.
- Configuration discovery and key rotation add bootstrap dependencies.
- ODoH does not itself validate DNSSEC.
- Native Debian and Ubuntu system resolvers do not provide a general ODoH client.

Use an implementation-specific ODoH client and its own test suite. Do not
pretend a normal DoH request is an ODoH test.

---

## Transport comparison

| Mechanism | Normal transport | Resolver authenticated | Query hidden from access network | Resolver sees client + query | Authenticates DNS data |
| --- | --- | --- | --- | --- | --- |
| Plain DNS | UDP/TCP 53 | No | No | Yes | Only when DNSSEC is validated separately |
| DoT | TLS/TCP 853 | Yes in strict mode | Yes | Yes | No |
| DoH | HTTPS/TCP 443 or HTTP/3 | Yes | Yes | Yes | No |
| DoQ | QUIC/UDP 853 | Yes | Yes | Yes | No |
| ODoH | HTTPS relay plus encrypted message | Relay and target identities | Yes | No single non-colluding server | No |
| DNSSEC | Any DNS transport | Not a transport property | No | Not applicable | Yes, for signed data and a valid chain |

### Visibility by party

| Party | Plain DNS | DoT/DoH/DoQ | ODoH |
| --- | --- | --- | --- |
| Local application | Query and answer | Query and answer | Query and answer |
| Access network | Query, answer, endpoints | Resolver endpoint and traffic metadata | Relay endpoint and traffic metadata |
| Recursive resolver or ODoH target | Client + query | Client + query | Relay + query |
| ODoH relay | Not applicable | Not applicable | Client + target, not query contents |

---

## Discovery and bootstrap

An encrypted resolver needs more than an IP address:

```text
resolver address + authentication name + transport + port/path + policy
```

Bootstrap questions:

1. How does the client learn the resolver address and encrypted capability?
2. How does it resolve the resolver's own hostname without a loop?
3. Which certificate name must match?
4. Does failure stop resolution or fall back to plaintext?
5. How does a captive portal work before encrypted DNS is available?

Possible sources include manual configuration, device management, application
configuration, VPN policy, Discovery of Designated Resolvers (DDR), and
encrypted resolver information delivered by DHCP or Router Advertisements.
Support varies by operating system and network.

Prefer an explicit bootstrap IP paired with a certificate name when strict
identity is required. Document fallback behavior; silent plaintext fallback
can erase the expected security property.

---

## Linux client support

Check packages and compiled features instead of assuming:

```bash
. /etc/os-release
printf '%s %s\n' "$ID" "$VERSION_ID"

resolvectl --version 2>/dev/null || true
dig -v 2>/dev/null || true
kdig -V 2>/dev/null || true
curl --version
```

| Client | DoT | DoH | DoQ | ODoH | DNSSEC role |
| --- | --- | --- | --- | --- | --- |
| `systemd-resolved` in target releases | Yes | No general client | No | No | Optional local validation |
| `kdig` with required libraries | Yes | Yes | Yes | No | Requests records; not a validating resolver |
| BIND `dig` | Build/version dependent | Build/version dependent | Build/version dependent | No | Sets DNSSEC flags; does not normally validate locally |
| `delv` | DNS transport support depends on build | Build dependent | Build dependent | No | Performs local DNSSEC validation |
| `curl --doh-url` | No | Yes | No dedicated DoQ | No | Uses DoH to resolve curl destinations |

Package names on Debian and Ubuntu:

```bash
sudo apt update
sudo apt install bind9-dnsutils knot-dnsutils ca-certificates
```

Installing tools is optional. Do not replace the system resolver merely to
run a transport experiment.

---

## `systemd-resolved` with DoT

Inspect current behavior:

```bash
resolvectl status
resolvectl dnsovertls 2>/dev/null || true
systemd-analyze cat-config systemd/resolved.conf
```

Example strict global DoT configuration:

```ini
# /etc/systemd/resolved.conf.d/30-dot.conf
[Resolve]
DNS=1.1.1.1#one.one.one.one
DNSOverTLS=yes
```

The `#name` suffix supplies the certificate authentication name. An address
without a server name can prevent meaningful public-CA hostname validation.

Apply and verify:

```bash
sudo systemctl reload-or-restart systemd-resolved
resolvectl status
resolvectl query example.com
journalctl -u systemd-resolved --since '-5 minutes'
```

`reload-or-restart` works across the target releases: it reloads units that
support it and otherwise performs a brief restart.

Temporary per-link test when supported:

```bash
sudo resolvectl dns eth0 '1.1.1.1#one.one.one.one'
sudo resolvectl dnsovertls eth0 yes
resolvectl query example.com

# Restore network-manager-provided settings
sudo resolvectl revert eth0
```

Modes:

| `DNSOverTLS=` | Behavior |
| --- | --- |
| `no` | Do not use DoT. |
| `opportunistic` | Try DoT but permit downgrade; confidentiality is not guaranteed. |
| `yes` | Require DoT; resolution fails when TLS or authentication fails. |

Use the installed `resolved.conf(5)` as the authority because supported syntax
and per-link controls vary between systemd releases.

---

## Test encrypted DNS with `kdig`

### Feature check

```bash
kdig -V
kdig -h 2>&1 | rg '\+(tls|tls-ca|https|quic)'
```

Features such as DoH and DoQ depend on how Knot DNS was compiled.

### Strict DoT

```bash
kdig @1.1.1.1 +tls-ca \
  +tls-hostname=one.one.one.one \
  example.com A
```

`+tls` alone uses an opportunistic profile. `+tls-ca` with the correct
hostname tests certificate-chain and hostname validation.

### DoH

```bash
kdig @1.1.1.1 +https=/dns-query +tls-ca \
  +tls-hostname=cloudflare-dns.com \
  example.com A

kdig @1.1.1.1 +https=/dns-query +https-get +tls-ca \
  +tls-hostname=cloudflare-dns.com \
  example.com A
```

The first uses HTTP `POST`; the second uses `GET`.

### DoQ

Use an endpoint that explicitly documents DoQ support:

```bash
kdig @94.140.14.14 +quic +tls-ca \
  +tls-hostname=dns.adguard-dns.com \
  example.com A
```

If the package lacks QUIC libraries, `+quic` will not provide a valid test.

### Connection reuse and padding

```bash
kdig @1.1.1.1 +tls-ca \
  +tls-hostname=one.one.one.one \
  +keepopen +padding \
  example.com A example.net AAAA
```

Connection reuse amortizes handshakes. Padding reduces simple size correlation
but costs bytes and does not eliminate traffic analysis.

---

## `dig`, `delv`, OpenSSL, and curl

### `dig`

Portable DNSSEC request:

```bash
dig example.com A +dnssec
```

Probe encrypted options only when listed by the installed build:

```bash
dig -h 2>&1 | rg '\+(tls|https|quic)'
```

Do not copy encrypted `dig` flags from a newer BIND manual into an older Ubuntu
release without this check.

### `delv`

`delv` performs validation locally instead of merely trusting an upstream
resolver's `AD` bit:

```bash
delv ietf.org A
delv ietf.org DNSKEY
delv +vtrace ietf.org A
```

It needs a usable trust anchor and correct time. Read the installed `delv(1)`
for its default anchor location.

### OpenSSL

Inspect the DoT handshake and certificate identity:

```bash
openssl s_client \
  -connect '1.1.1.1:853' \
  -servername 'one.one.one.one' \
  -verify_hostname 'one.one.one.one' \
  -showcerts </dev/null
```

OpenSSL does not construct the DNS length prefix and query for you; this tests
TLS, not a complete DoT exchange.

### curl

Check whether curl supports a DoH resolver:

```bash
curl --version
curl --help all | rg -- '--doh-url'
```

Example web request whose destination is resolved through DoH:

```bash
curl --doh-url 'https://cloudflare-dns.com/dns-query' \
  --resolve 'cloudflare-dns.com:443:1.1.1.1' \
  --output /dev/null \
  --show-error \
  --verbose \
  'https://example.com/'
```

`--resolve` bootstraps the DoH server's address. This tests curl's own resolver,
not glibc NSS or the whole operating system.

---

## DNSSEC mental model

DNSSEC signs RRsets and builds a delegation chain from a configured trust
anchor.

```text
validator trusts root DNSKEY
          │ validates root signature
          ▼
root DS for `.org` ──digest match──► `.org` DNSKEY
          │
          ▼
`.org` DS for `example.org` ───────► child DNSKEY
          │
          ▼
child DNSKEY validates RRSIG over the requested RRset
```

The validator checks cryptographic signatures, delegation, algorithms, times,
and authenticated denial of existence. It does not contact a certificate
authority and does not use TLS certificates.

DNSSEC provides:

- Origin authentication of signed DNS data.
- Integrity for signed RRsets.
- Authenticated proof that signed data does not exist.

DNSSEC does not provide:

- Query or answer confidentiality.
- Protection from a malicious or compromised zone signer.
- Availability when servers, signatures, clocks, or delegations fail.
- Authentication of the application reached at the returned IP address.

---

## DNSSEC record types

| Record | Job |
| --- | --- |
| `DNSKEY` | Publishes a zone's DNSSEC public keys. |
| `DS` | Parent-side digest that delegates trust to a child's key. |
| `RRSIG` | Signature over one RRset. |
| `NSEC` | Proves nonexistence by linking existing names and record types. |
| `NSEC3` | Hashes names in authenticated denial records. |
| `CDS` / `CDNSKEY` | Signals desired parent DS changes where registry automation supports it. |

Inspect the chain material:

```bash
ZONE='ietf.org'

dig "$ZONE" DNSKEY +dnssec
dig "$ZONE" DS +dnssec
dig "$ZONE" A +dnssec
dig does-not-exist."$ZONE" A +dnssec
```

A `DS` record for a child is stored in the parent zone. Querying at the child
apex and reading the authority section helps keep that ownership model clear.

---

## Validation states

| State | Meaning | Typical client result |
| --- | --- | --- |
| Secure | A complete valid chain reaches a trust anchor. | Answer succeeds and a validating resolver may set `AD`. |
| Insecure | The delegation proves that the child is unsigned. | Answer succeeds without DNSSEC authentication. |
| Bogus | Signatures, keys, delegation, denial proof, or time checks fail. | Validating resolver normally returns `SERVFAIL`. |
| Indeterminate | The validator cannot determine a state, often because required data is unavailable. | Implementation-dependent failure or retry. |

Unsigned is not the same as bogus. A securely proven unsigned delegation is an
expected DNSSEC state.

---

## DNSSEC flags

| Flag or bit | Set by | Meaning |
| --- | --- | --- |
| `DO` | Client in EDNS | Send DNSSEC records when relevant. |
| `AD` | Validating resolver | The resolver considers the answer authenticated. |
| `CD` | Client | Disable checking at the recursive resolver for this query. |
| `RD` | Client | Recursion desired; unrelated to DNSSEC validation. |

Examples:

```bash
# Ask for DNSSEC records
dig @1.1.1.1 ietf.org A +dnssec

# Compare resolver validation with checking disabled
dig @1.1.1.1 ietf.org A +dnssec
dig @1.1.1.1 ietf.org A +dnssec +cdflag
```

`+dnssec` sets `DO`; `dig` does not thereby become a validator. An `AD` bit is
only trustworthy when the reply comes from a trusted validator over a local or
authenticated transport. A remote attacker on plaintext DNS can forge flags.

---

## Validate and troubleshoot DNSSEC

### Check time first

RRSIG records have inception and expiration times:

```bash
timedatectl status
date --utc
```

### Compare validating and non-checking responses

```bash
NAME='ietf.org'
RESOLVER='1.1.1.1'

dig "@$RESOLVER" "$NAME" A +dnssec
dig "@$RESOLVER" "$NAME" A +dnssec +cdflag
```

If the normal query returns `SERVFAIL` but `+cdflag` returns data, investigate
DNSSEC rather than assuming general connectivity failure.

### Walk keys and delegation

```bash
dig org. DNSKEY +dnssec
dig ietf.org DS +dnssec
dig ietf.org DNSKEY +dnssec
dig ietf.org A +dnssec
```

### Validate locally

```bash
delv +vtrace ietf.org A
```

For Unbound:

```bash
sudo unbound-host -C /etc/unbound/unbound.conf -v ietf.org
dig @127.0.0.1 ietf.org A +dnssec
```

`unbound-host` reads no configuration by default. `-C` supplies the packaged
trust anchor and resolver settings needed for this validation test.

### Inspect with `resolvectl`

```bash
resolvectl dnssec
resolvectl query --type=A ietf.org
```

Look for `Data is authenticated: yes`. The wording and available commands vary
by systemd version.

### Frequent causes of bogus answers

- Parent `DS` does not match any active child `DNSKEY`.
- RRSIG is expired, not yet valid, or uses an unsupported algorithm.
- A key rollover removed the old key or signatures too early.
- An authoritative server serves inconsistent signed data.
- A firewall or middlebox drops larger EDNS, fragmented, or TCP responses.
- The validator's clock is wrong.
- A private unsigned answer collides with a signed public delegation.
- A stale cache contains a previous key or delegation state.

---

## Authoritative operator overview

Operating a signed zone adds a lifecycle, not just extra records:

```text
generate keys ─► sign every RRset ─► publish DNSKEY ─► publish matching parent DS
      ▲                 │                    │                    │
      └──── roll keys + refresh signatures + monitor validation ─┘
```

Operational responsibilities:

- Keep signing keys protected and backed up.
- Refresh signatures before expiration with correct clocks.
- Coordinate DNSKEY and parent DS changes during rollover.
- Serve consistent signed data from every authoritative server.
- Monitor from validating resolvers outside the authoritative platform.
- Have a tested recovery process for a bad DS or lost signing key.

KSK and ZSK describe operational key roles; validators ultimately follow
DNSKEY, DS, and RRSIG relationships. Modern signers can automate much of the
lifecycle, but automation does not remove registry and monitoring dependencies.

This cheatsheet intentionally does not prescribe a production key ceremony or
rollover command sequence. Follow the authoritative server and registrar's
current operational guide.

---

## Private and split DNS

Strict validation needs an explicit model for private namespaces:

```text
public signed parent
       ├─ securely unsigned child: no DS, answer is insecure but valid
       └─ child with DS: signatures must validate or answer is bogus
```

For local zones:

- Use a namespace and delegation model you control.
- Do not assume encryption makes unsigned private data authentic.
- Configure trust anchors when an internal zone is signed with a private trust chain.
- Configure a narrowly scoped insecure exception only when the unsigned design is intentional.
- Avoid broad negative trust anchors that disable validation for unrelated names.
- Test both inside and outside the split-DNS environment.

`systemd-resolved`, Unbound, BIND, and filtering resolvers use different
configuration terms for private trust anchors and insecure domains. Document
the exception next to the zone ownership and removal condition.

---

## Choosing an approach

| Goal | Suitable starting point | Important caveat |
| --- | --- | --- |
| Protect DNS on untrusted Wi-Fi | Strict DoT, DoH, or DoQ to an authenticated resolver | Resolver still sees client and queries. |
| Make DNS blend with normal HTTPS | DoH | Harder network policy and troubleshooting. |
| Avoid TCP head-of-line blocking | DoQ | UDP/853 may be blocked. |
| Separate client identity from query | ODoH | Requires non-colluding relay and target plus client support. |
| Authenticate domain-owner data | Local or trusted DNSSEC validation | No confidentiality; unsigned zones remain insecure. |
| Network-wide policy and filtering | Controlled local resolver with a documented encrypted upstream | Applications may bypass it with their own DoH. |
| Validate and minimize dependence on public recursive resolvers | Local Unbound recursion with DNSSEC | Recursive-to-authoritative traffic is generally visible. |

See [resolvers.md](resolvers.md) for native resolver deployments and
[linux.md](linux.md) for how Linux applications reach the configured resolver.

---

## Packet capture

Capture conventional DNS and dedicated encrypted transports separately:

```bash
sudo tcpdump -ni any 'udp port 53 or tcp port 53'
sudo tcpdump -ni any 'tcp port 853'
sudo tcpdump -ni any 'udp port 853'
```

Interpretation:

| Capture | Likely protocol |
| --- | --- |
| UDP/TCP 53 with visible questions | Plain DNS |
| TCP 853 followed by TLS records | DoT |
| UDP 853 with QUIC handshake and encrypted payload | DoQ |
| TCP 443 | Could be DoH or any other HTTPS application |
| UDP 443 | Could be HTTP/3 DoH or unrelated QUIC traffic |

DoH payloads are encrypted and cannot be identified reliably by a simple port
filter. Endpoint addresses, TLS names where visible, application logs, or
managed endpoint telemetry are needed.

---

## Troubleshooting workflow

### 1. Verify ordinary reachability and time

```bash
ip route
ip -6 route
timedatectl status
```

### 2. Confirm the client feature and exact endpoint

```bash
kdig -V 2>/dev/null || true
dig -v
curl --version
```

Record IP address, certificate name, port, DoH path, and strict/fallback mode.

### 3. Test bootstrap resolution separately

```bash
getent ahosts one.one.one.one
dig one.one.one.one A
```

If the encrypted resolver depends on its own hostname, use the configured
bootstrap mechanism rather than creating a resolution loop.

### 4. Test transport authentication

```bash
openssl s_client \
  -connect '1.1.1.1:853' \
  -servername 'one.one.one.one' \
  -verify_hostname 'one.one.one.one' \
  -brief </dev/null
```

Check CA roots, hostname, certificate time, system time, and TLS interception.

### 5. Send one DNS query with verbose output

```bash
kdig -d @1.1.1.1 +tls-ca \
  +tls-hostname=one.one.one.one \
  example.com A
```

### 6. Separate transport failure from DNSSEC failure

```bash
dig @1.1.1.1 ietf.org A +dnssec
dig @1.1.1.1 ietf.org A +dnssec +cdflag
```

### 7. Inspect resolver and firewall logs

```bash
journalctl -u systemd-resolved --since '-10 minutes'
sudo tcpdump -ni any 'port 53 or port 853'
```

### 8. Test another network without weakening policy permanently

A captive portal, blocked UDP/853, TLS interception, or path MTU problem may be
network-specific. Preserve the failed strict configuration and compare rather
than silently switching to plaintext.

---

## Avoid these mistakes

- Treating `dig +dnssec` as local validation.
- Trusting an `AD` flag received over an unauthenticated network path.
- Calling DoT secure while using opportunistic mode without authentication.
- Using an IP endpoint without the required certificate name.
- Describing a provider's DNS JSON API as RFC 8484 DoH.
- Assuming DoH hides the destination services contacted after resolution.
- Calling HTTP/3 DoH and DoQ the same protocol.
- Assuming ODoH protects privacy when relay and target are the same operator without a separation policy.
- Enabling strict DNSSEC without testing private zones, clocks, and upstream behavior.
- Disabling validation globally to work around one broken private domain.
- Interpreting blocked encrypted DNS as proof that ordinary DNS is broken.
- Allowing a browser's private DoH configuration to bypass required local policy unnoticed.

---

## References

- [RFC 7858: DNS over TLS](https://www.rfc-editor.org/rfc/rfc7858)
- [RFC 8310: Usage Profiles for DNS over TLS and DTLS](https://www.rfc-editor.org/rfc/rfc8310)
- [RFC 8484: DNS over HTTPS](https://www.rfc-editor.org/rfc/rfc8484)
- [RFC 9250: DNS over Dedicated QUIC Connections](https://www.rfc-editor.org/rfc/rfc9250)
- [RFC 9230: Oblivious DNS over HTTPS](https://www.rfc-editor.org/rfc/rfc9230)
- [RFC 9462: Discovery of Designated Resolvers](https://www.rfc-editor.org/rfc/rfc9462)
- [RFC 9463: DHCP and Router Advertisement Options for Encrypted DNS](https://www.rfc-editor.org/rfc/rfc9463)
- [RFC 4033: DNSSEC Introduction and Requirements](https://www.rfc-editor.org/rfc/rfc4033)
- [RFC 4034: DNSSEC Resource Records](https://www.rfc-editor.org/rfc/rfc4034)
- [RFC 4035: DNSSEC Protocol Modifications](https://www.rfc-editor.org/rfc/rfc4035)
- [RFC 9364 / BCP 237: DNSSEC](https://www.rfc-editor.org/rfc/rfc9364)
- [`kdig` manual](https://www.knot-dns.cz/docs/latest/html/man_kdig.html)
- [`systemd-resolved`](https://www.freedesktop.org/software/systemd/man/systemd-resolved.service.html)
- [Cloudflare 1.1.1.1 DoT endpoint](https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-tls/)
- [Cloudflare 1.1.1.1 DoH endpoint](https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-https/make-api-requests/)
- [AdGuard public DNS endpoints](https://adguard-dns.io/kb/general/dns-providers/)
- [Ubuntu DNSSEC documentation](https://documentation.ubuntu.com/server/explanation/dnssec/dnssec/)
