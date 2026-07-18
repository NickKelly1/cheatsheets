# DNS Cheatsheet

## Table of Contents

- [Practical exercise](#practical-exercise)
- [DNS Resolution](#dns-resolution)
- [Common Record Types](#common-record-types)
- [`dig`](#dig)
- [Other Query Tools](#other-query-tools)
- [Reading a `dig` Response](#reading-a-dig-response)
- [Zone File Example](#zone-file-example)
- [Validate BIND Configuration](#validate-bind-configuration)
- [Local Resolver Configuration](#local-resolver-configuration)
- [Cache and TTL](#cache-and-ttl)
- [Delegation Checks](#delegation-checks)
- [DNSSEC Checks](#dnssec-checks)
- [Reverse DNS](#reverse-dns-1)
- [Email DNS](#email-dns)
- [Split-Horizon DNS](#split-horizon-dns)
- [Troubleshooting Workflow](#troubleshooting-workflow)
- [Packet Capture](#packet-capture)
- [Useful One-Liners](#useful-one-liners)
- [Common Problems](#common-problems)
- [Avoid These Mistakes](#avoid-these-mistakes)

---

## Practical exercise

Run these read-only queries and compare what each view adds:

```bash
NAME='example.com'

dig "$NAME" A +short
dig "$NAME" A +noall +answer +stats
dig "$NAME" NS +short
dig +trace "$NAME" A
```

```text
you (`dig`) ──► recursive resolver ──► root ──► .com ──► authoritative server
             ◄──────────── cached or final A record, TTL, and response status ────
```

| Do | Notice |
| --- | --- |
| Repeat the `+noall +answer +stats` query | The answer should stay stable while query time may fall after caching. |
| Compare `A`, `NS`, and `MX` | One name can expose different record sets with different jobs. |
| Read the last lines of `+trace` | Delegation walks from the root to the authoritative server. |

Change `NAME` to a domain you operate and repeat the exercise. Querying is safe;
do not treat an answer from one resolver as proof that every resolver sees it yet.

---

## DNS Resolution

```text
Application
  ↓
Stub resolver
  ↓
Recursive resolver
  ↓
Root nameserver
  ↓
TLD nameserver
  ↓
Authoritative nameserver
```

| Service        |  Port |
| -------------- | ----: |
| DNS over UDP   |  `53` |
| DNS over TCP   |  `53` |
| DNS over TLS   | `853` |
| DNS over HTTPS | `443` |

UDP handles most queries. TCP is used for zone transfers, truncated responses, and large responses.

---

## Common Record Types

| Type             | Purpose                           | Example                                                    |
| ---------------- | --------------------------------- | ---------------------------------------------------------- |
| `A`              | IPv4 address                      | `example.com. 300 IN A 192.0.2.10`                         |
| `AAAA`           | IPv6 address                      | `example.com. 300 IN AAAA 2001:db8::10`                    |
| `CNAME`          | Alias to another hostname         | `www.example.com. IN CNAME example.com.`                   |
| `MX`             | Mail server                       | `example.com. IN MX 10 mail.example.com.`                  |
| `TXT`            | Arbitrary text, SPF, verification | `example.com. IN TXT "v=spf1 -all"`                        |
| `NS`             | Authoritative nameserver          | `example.com. IN NS ns1.example.net.`                      |
| `SOA`            | Zone metadata                     | Serial, timers, primary NS                                 |
| `PTR`            | Reverse DNS                       | `10.2.0.192.in-addr.arpa. IN PTR host.example.com.`        |
| `SRV`            | Service location                  | `_sip._tcp.example.com. IN SRV 10 5 5060 sip.example.com.` |
| `CAA`            | Allowed certificate authorities   | `example.com. IN CAA 0 issue "letsencrypt.org"`            |
| `NAPTR`          | Service discovery and rewriting   | Common in SIP and telecom                                  |
| `TLSA`           | DANE TLS certificate binding      | Used with DNSSEC                                           |
| `DNSKEY`         | DNSSEC public key                 | Published in the signed zone                               |
| `DS`             | DNSSEC delegation signer          | Published in the parent zone                               |
| `RRSIG`          | DNSSEC signature                  | Signature for an RRset                                     |
| `NSEC` / `NSEC3` | Authenticated denial              | Proves a name does not exist                               |
| `SVCB`           | Service binding                   | Alternative endpoints and parameters                       |
| `HTTPS`          | HTTP service binding              | HTTP endpoint metadata                                     |

### Important Rules

* DNS names in zone files usually end with `.` when absolute.
* A `CNAME` cannot coexist with other records at the same name.
* Standard DNS does not allow a `CNAME` at the zone apex.
* Provider-specific `ALIAS`, `ANAME`, or flattening records are not standard DNS records.
* `MX` and `NS` targets should be hostnames with `A` or `AAAA` records, not aliases.
* Lower `MX` and `SRV` priority values are preferred.
* Multiple records of the same type form an unordered RRset.

---

## `dig`

### Basic Queries

```bash
dig example.com
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com TXT
dig example.com NS
dig example.com SOA
dig example.com CAA
```

### Concise Output

```bash
dig +short example.com
dig +short example.com MX
```

### Query a Specific Resolver

```bash
dig @1.1.1.1 example.com
dig @8.8.8.8 example.com AAAA
```

### Query an Authoritative Server

```bash
dig @ns1.example.com example.com SOA
dig @ns1.example.com www.example.com A +norecurse
```

### Reverse DNS

```bash
dig -x 192.0.2.10
dig -x 2001:db8::10
```

### Trace Delegation

```bash
dig +trace example.com
```

### Force TCP

```bash
dig +tcp example.com
```

### DNSSEC

```bash
dig +dnssec example.com
dig +dnssec example.com DNSKEY
dig +dnssec example.com DS
```

### Inspect Response Details

```bash
dig example.com +comments
dig example.com +stats
dig example.com +multiline
```

### Disable Search Domains

```bash
dig example.com +nosearch
```

### Zone Transfer

```bash
dig AXFR example.com @ns1.example.com
dig IXFR=2026071601 example.com @ns1.example.com
```

Zone transfers should only succeed for authorized clients.

### Query Many Names

```bash
dig -f domains.txt
```

Example `domains.txt`:

```text
example.com A
example.net MX
example.org TXT
```

### Minimal Script-Friendly Output

```bash
dig +short +time=2 +tries=1 example.com A
```

---

## Other Query Tools

```bash
host example.com
host -t MX example.com
host 192.0.2.10

nslookup example.com
nslookup -type=MX example.com

resolvectl query example.com
getent ahosts example.com
getent hosts example.com
```

`getent` follows the system's NSS configuration, so it reflects `/etc/hosts`, DNS, and other configured sources.

---

## Reading a `dig` Response

```text
;; flags: qr rd ra
;; QUESTION SECTION:
;; ANSWER SECTION:
;; AUTHORITY SECTION:
;; ADDITIONAL SECTION:
```

### Common Flags

| Flag | Meaning                   |
| ---- | ------------------------- |
| `qr` | Query response            |
| `aa` | Authoritative answer      |
| `tc` | Response was truncated    |
| `rd` | Recursion desired         |
| `ra` | Recursion available       |
| `ad` | DNSSEC data was validated |
| `cd` | Disable DNSSEC validation |

### Common Response Codes

| Code       | Meaning                             |
| ---------- | ----------------------------------- |
| `NOERROR`  | Query succeeded                     |
| `FORMERR`  | Invalid query format                |
| `SERVFAIL` | Server could not complete the query |
| `NXDOMAIN` | Name does not exist                 |
| `NOTIMP`   | Operation not supported             |
| `REFUSED`  | Server refused the query            |

`NOERROR` with an empty answer is different from `NXDOMAIN`. It usually means the name exists but has no record of the requested type.

---

## Zone File Example

```dns
$ORIGIN example.com.
$TTL 3600

@ IN SOA ns1.example.com. hostmaster.example.com. (
  2026071601 ; serial
  3600       ; refresh
  900        ; retry
  1209600    ; expire
  300        ; negative cache TTL
)

  IN NS ns1.example.net.
  IN NS ns2.example.net.

  IN MX 10 mail.example.com.

@    IN A     192.0.2.10
@    IN AAAA  2001:db8::10
www  IN CNAME @
mail IN A     192.0.2.20

@ IN TXT "v=spf1 mx -all"
@ IN CAA 0 issue "letsencrypt.org"

_sip._tcp IN SRV 10 5 5060 sip.example.com.
sip       IN A 192.0.2.30
```

### Zone File Shorthand

| Syntax             | Meaning                        |
| ------------------ | ------------------------------ |
| `@`                | Current zone origin            |
| Empty owner        | Reuse the previous owner name  |
| `www`              | Relative to `$ORIGIN`          |
| `www.example.com.` | Absolute name                  |
| `$TTL`             | Default TTL                    |
| `$ORIGIN`          | Base domain for relative names |

### SOA Serial Convention

A common convention is:

```text
YYYYMMDDnn
```

Example:

```text
2026071601
```

The only protocol requirement is that the serial increases according to DNS serial arithmetic.

---

## Validate BIND Configuration

```bash
named-checkconf
named-checkzone example.com /etc/bind/zones/db.example.com
named-checkzone -D example.com /etc/bind/zones/db.example.com
```

Reload:

```bash
sudo rndc reload
sudo rndc reload example.com
```

Inspect status:

```bash
sudo rndc status
sudo journalctl -u bind9
```

---

## Local Resolver Configuration

For the full application-to-network path on Debian and Ubuntu—including NSS,
`getent`, systemd-resolved, NetworkManager, Netplan, Avahi, and libvirt—see
[linux.md](linux.md).

### Files

```text
/etc/resolv.conf
/etc/hosts
/etc/nsswitch.conf
/etc/systemd/resolved.conf
```

Typical `/etc/nsswitch.conf` entry:

```text
hosts: files systemd dns
```

### systemd-resolved

```bash
resolvectl status
resolvectl dns
resolvectl domain
resolvectl query example.com
resolvectl statistics
sudo resolvectl flush-caches
sudo resolvectl reset-statistics
```

### Check `/etc/resolv.conf`

```bash
readlink -f /etc/resolv.conf
cat /etc/resolv.conf
```

Typical content:

```text
nameserver 127.0.0.53
search internal.example.com
options edns0 trust-ad
```

---

## Cache and TTL

For native Unbound, AdGuard Home, and Pi-hole cache and deployment examples,
see [resolvers.md](resolvers.md).

```text
TTL = how long a resolver may cache an answer
```

Check remaining TTL:

```bash
dig example.com A
```

A resolver usually decrements the returned TTL while the record is cached.

Negative answers are also cached. Their TTL is derived from the zone's SOA record.

Flush common caches:

```bash
sudo resolvectl flush-caches
sudo rndc flush
sudo systemctl restart dnsmasq
sudo systemctl restart unbound
```

Browser and application DNS caches may remain after the operating-system cache is cleared.

---

## Delegation Checks

```bash
dig example.com NS
dig com NS
dig @a.gtld-servers.net example.com NS
dig @ns1.example.net example.com SOA +norecurse
```

Check parent-side glue:

```bash
dig @a.gtld-servers.net example.com NS +additional
```

A delegation is lame when the parent delegates to a server that is not authoritative for the child zone.

---

## DNSSEC Checks

For validation states, record-chain debugging, and the distinction between
DNSSEC and DoT, DoH, DoQ, or ODoH, see [security.md](security.md).

```bash
dig example.com A +dnssec
dig example.com DNSKEY +dnssec
dig example.com DS +dnssec
dig @1.1.1.1 example.com A +dnssec
```

Look for:

```text
flags: ... ad ...
```

The basic chain is:

```text
Root trust anchor
  → parent DS
  → child DNSKEY
  → child RRSIG
  → validated record
```

Debug validation:

```bash
delv example.com
delv +rtrace example.com
```

A missing `ad` flag does not necessarily mean DNSSEC failed. The resolver may not validate, or checking may have been disabled.

---

## Reverse DNS

IPv4 reverse zones use `in-addr.arpa`:

```text
192.0.2.10
→ 10.2.0.192.in-addr.arpa
```

IPv6 reverse zones use nibble-reversed `ip6.arpa`.

Query:

```bash
dig -x 192.0.2.10
```

Forward and reverse records are independent. Creating an `A` record does not automatically create a `PTR` record.

---

## Email DNS

### MX

```bash
dig +short example.com MX
```

### SPF

```bash
dig +short example.com TXT
```

Example:

```dns
@ IN TXT "v=spf1 mx ip4:192.0.2.0/24 -all"
```

Only one SPF policy record should exist for a domain.

### DKIM

```bash
dig +short selector1._domainkey.example.com TXT
```

### DMARC

```bash
dig +short _dmarc.example.com TXT
```

Example:

```dns
_dmarc IN TXT "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
```

---

## Split-Horizon DNS

The same name may return different answers depending on the querying client or resolver.

```text
Internal client → private address
External client → public address
```

Compare resolvers:

```bash
dig @internal-resolver.example.com app.example.com
dig @1.1.1.1 app.example.com
```

---

## Troubleshooting Workflow

```bash
# 1. Check local name-service resolution
getent ahosts example.com

# 2. Query the configured resolver
resolvectl query example.com

# 3. Query a known public resolver
dig @1.1.1.1 example.com

# 4. Trace the delegation
dig +trace example.com

# 5. Find authoritative servers
dig example.com NS

# 6. Query an authoritative server directly
dig @ns1.example.net example.com A +norecurse

# 7. Check SOA serials across nameservers
for ns in $(dig +short example.com NS); do
  printf '%s ' "$ns"
  dig +short "@$ns" example.com SOA
done
```

### Compare All Authoritative Servers

```bash
for ns in $(dig +short example.com NS); do
  echo "== $ns =="
  dig +short "@$ns" example.com A
done
```

### Test UDP and TCP

```bash
dig @1.1.1.1 example.com
dig @1.1.1.1 example.com +tcp
```

If TCP succeeds but UDP fails, investigate firewalls, MTU, fragmentation, or EDNS handling.

### Test Without EDNS

```bash
dig example.com +noedns
```

### Check Connectivity

```bash
nc -zvu 1.1.1.1 53
nc -zv 1.1.1.1 53
```

---

## Packet Capture

```bash
sudo tcpdump -ni any port 53
sudo tcpdump -ni any 'udp port 53 or tcp port 53'
sudo tcpdump -ni any -vvv -s0 port 53
```

Filter a resolver:

```bash
sudo tcpdump -ni any host 1.1.1.1 and port 53
```

Save a capture:

```bash
sudo tcpdump -ni any -s0 -w dns.pcap port 53
```

---

## Useful One-Liners

### Resolve Several Record Types

```bash
for type in A AAAA MX NS TXT CAA SOA; do
  echo "== $type =="
  dig +short example.com "$type"
done
```

### Find the Public IP Seen by a DNS Resolver

```bash
dig +short myip.opendns.com @resolver1.opendns.com
```

### Check Propagation Across Resolvers

```bash
for resolver in 1.1.1.1 8.8.8.8 9.9.9.9; do
  printf '%-15s ' "$resolver"
  dig +short "@$resolver" example.com A
done
```

### Check SOA Serial

```bash
dig +short example.com SOA | awk '{print $3}'
```

### Extract Nameservers

```bash
dig +short example.com NS | sort
```

---

## Common Problems

| Symptom                           | Likely cause                                                         |
| --------------------------------- | -------------------------------------------------------------------- |
| `NXDOMAIN`                        | Name does not exist or incorrect search suffix                       |
| `SERVFAIL`                        | DNSSEC failure, upstream failure, broken delegation, or server error |
| Different answers                 | Caching, geo-DNS, load balancing, split-horizon DNS                  |
| Old answer                        | TTL has not expired or application cache                             |
| Works with `dig`, not application | NSS, `/etc/hosts`, search domains, application cache                 |
| Works externally, not internally  | Split DNS, firewall, internal resolver configuration                 |
| UDP fails, TCP works              | Firewall, MTU, fragmentation, or EDNS                                |
| One NS differs                    | Zone replication or stale secondary                                  |
| Missing DNSSEC `ad` flag          | Non-validating resolver or validation disabled                       |
| Intermittent failure              | Broken authoritative server or inconsistent RRsets                   |

---

## Avoid These Mistakes

* Do not use `ANY` queries as a reliable zone dump.
* Do not assume DNS changes are globally instantaneous.
* Do not put an IP address directly in an `MX`, `NS`, or `CNAME` target.
* Do not create circular `CNAME` chains.
* Do not forget the trailing `.` on absolute names in zone files.
* Do not expose unrestricted recursive DNS to the internet.
* Do not expose unrestricted AXFR.
* Do not assume a successful `ping` proves DNS is correct.
* Do not rely on only one recursive resolver when troubleshooting.
* Do not confuse `NXDOMAIN` with an existing name that lacks the requested record type.
