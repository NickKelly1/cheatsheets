# Native DNS Resolvers and Filters Cheatsheet

This guide compares and deploys Unbound, AdGuard Home, and Pi-hole directly on
Debian 13 and Ubuntu 24.04/26.04 LTS. It does not use containers.

Last checked: 2026-07-18.

## Table of Contents

- [Practical exercise](#practical-exercise)
- [Choose the role first](#choose-the-role-first)
- [Common architectures](#common-architectures)
- [Preflight inventory](#preflight-inventory)
- [Port 53 conflicts](#port-53-conflicts)
- [Network and firewall safety](#network-and-firewall-safety)
- [Unbound](#unbound)
- [Install Unbound](#install-unbound)
- [Standalone recursive resolver](#standalone-recursive-resolver)
- [Loopback Unbound for a filtering frontend](#loopback-unbound-for-a-filtering-frontend)
- [Unbound forwarding over TLS](#unbound-forwarding-over-tls)
- [Validate and operate Unbound](#validate-and-operate-unbound)
- [Local records in Unbound](#local-records-in-unbound)
- [AdGuard Home](#adguard-home)
- [Install AdGuard Home natively](#install-adguard-home-natively)
- [AdGuard Home initial configuration](#adguard-home-initial-configuration)
- [AdGuard Home upstreams](#adguard-home-upstreams)
- [AdGuard Home filtering and local names](#adguard-home-filtering-and-local-names)
- [AdGuard Home encrypted endpoints](#adguard-home-encrypted-endpoints)
- [Operate and update AdGuard Home](#operate-and-update-adguard-home)
- [Pi-hole](#pi-hole)
- [Install Pi-hole natively](#install-pi-hole-natively)
- [Pi-hole v6 configuration](#pi-hole-v6-configuration)
- [Pi-hole with Unbound](#pi-hole-with-unbound)
- [Pi-hole local names and filtering](#pi-hole-local-names-and-filtering)
- [Operate and update Pi-hole](#operate-and-update-pi-hole)
- [Service ordering for a stacked resolver](#service-ordering-for-a-stacked-resolver)
- [Roll out to clients](#roll-out-to-clients)
- [IPv6 and DNS bypass](#ipv6-and-dns-bypass)
- [End-to-end verification](#end-to-end-verification)
- [Troubleshooting workflow](#troubleshooting-workflow)
- [Choosing between the three](#choosing-between-the-three)
- [Avoid these mistakes](#avoid-these-mistakes)
- [References](#references)

---

## Practical exercise

Map the DNS services already present before installing another one:

```bash
LAN_IF='eth0'

ip -brief address show "$LAN_IF"
ss -lntup | rg ':(53|80|443|853|3000|5335)\b'
systemctl --no-pager --type=service --state=running | \
  rg 'resolved|unbound|dnsmasq|named|AdGuardHome|pihole|libvirt'
readlink -f /etc/resolv.conf
resolvectl status 2>/dev/null || true
```

If a resolver is already listening, query each layer directly:

```bash
dig @127.0.0.53 example.com A +time=2 +tries=1
dig @127.0.0.1 -p 5335 example.com A +time=2 +tries=1
```

Failures are expected when those example listeners do not exist.

```text
client ─► LAN address:53 ─► filter/cache ─► 127.0.0.1:5335 ─► Unbound recursion
                             │
                             ├─ local record or block: answer immediately
                             └─ allowed public name: forward to validation layer
```

| Observation | Consequence |
| --- | --- |
| `0.0.0.0:53` or `[::]:53` is occupied | A second wildcard DNS listener cannot start. |
| `127.0.0.53:53` is occupied | This may only be the systemd-resolved stub; a service bound to one LAN address can still coexist. |
| `192.168.122.1:53` is occupied | libvirt dnsmasq is serving its virtual bridge; avoid wildcard binding. |
| Port `5335` is free | It is a conventional, non-privileged choice for local Unbound behind a filter. |

Do not stop the current resolver yet. First decide the final listener addresses,
client path, and rollback DNS.

---

## Choose the role first

| Product | Primary role | Recursive by itself | Network filtering | Administration |
| --- | --- | --- | --- | --- |
| Unbound | Caching validating recursive resolver or forwarder | Yes | Policy zones and local data, but not a consumer filtering UI | Files, CLI, systemd |
| AdGuard Home | Filtering DNS proxy with local services | Normally forwards to upstreams | Yes | Web UI, YAML, API, CLI |
| Pi-hole | Filtering resolver built around FTL/dnsmasq behavior | Normally forwards to upstreams | Yes | Web UI, TOML, CLI, API |

Common goals lead to different designs:

- Choose Unbound alone for local recursion, DNSSEC validation, caching, and no filtering dashboard.
- Choose AdGuard Home for filtering plus native encrypted DNS client/server features.
- Choose Pi-hole for its filtering ecosystem, FTL tooling, and local-network integration.
- Put AdGuard Home or Pi-hole in front of Unbound when filtering and independent recursion are both required.

A filter and a validator are separate roles even when one product can perform
parts of both.

---

## Common architectures

### Unbound recursion

```text
LAN clients ─► Unbound:53 ─► root ─► TLD ─► authoritative servers
                   │
                   └─ cache + DNSSEC validation
```

### Filtering forwarder

```text
LAN clients ─► AdGuard Home or Pi-hole:53 ─► selected upstream resolver
                   │
                   ├─ filter lists
                   ├─ local records
                   └─ client/query reporting
```

### Filtering plus local recursion

```text
LAN clients ─► filter:53 ─► Unbound:127.0.0.1:5335 ─► authoritative DNS
                   │                    │
              policy decision      cache + validation
```

In the stacked design, only the filtering frontend listens on the LAN. Unbound
accepts queries from loopback and is not an alternate path around policy.

---

## Preflight inventory

Define the intended network explicitly:

```bash
LAN_IF='eth0'
LAN_ADDR='192.168.50.53'
LAN_CIDR='192.168.50.0/24'
ROUTER_ADDR='192.168.50.1'

ip -brief address show "$LAN_IF"
ip route
ip -6 route
timedatectl status
```

Replace every example value. The resolver needs a stable address supplied by a
static configuration or DHCP reservation.

Record current DNS and listeners:

```bash
cat /etc/resolv.conf
readlink -f /etc/resolv.conf
resolvectl status 2>/dev/null || true
ss -lntup
sudo nft list ruleset
```

Check resources and versions:

```bash
. /etc/os-release
printf '%s %s\n' "$ID" "$VERSION_ID"
uname -m
df -h / /var
free -h
```

Before changing DHCP or the host's own resolver, retain a tested alternate DNS
path and console access.

---

## Port 53 conflicts

DNS uses UDP and TCP port 53. Both must be available on every address a new
service will bind.

```bash
sudo ss -lntup '( sport = :53 )'
sudo lsof -nP -iUDP:53 -iTCP:53 2>/dev/null
```

| Existing listener | Typical address | Safe response |
| --- | --- | --- |
| systemd-resolved stub | `127.0.0.53:53` | Bind the LAN resolver only to its LAN address, or deliberately redesign resolved integration. |
| libvirt dnsmasq | Bridge address such as `192.168.122.1:53` | Preserve the bridge listener and avoid `0.0.0.0:53`. |
| Local dnsmasq/BIND | Loopback, LAN, or wildcard | Decide which service owns each role and address. |
| Unbound behind a filter | `127.0.0.1:5335` | Keep it loopback-only and point the frontend at that port. |

Binding rules:

```text
0.0.0.0:53       means every IPv4 address; conflicts with address-specific listeners
[::]:53          may also accept IPv4, depending on IPV6_V6ONLY
192.168.50.53:53 means only that LAN address
127.0.0.1:5335   means local callers on a nonstandard port
```

Do not disable systemd-resolved or libvirt merely because their process names
appear. Identify the exact socket collision first.

---

## Network and firewall safety

A LAN resolver should not become a public recursive resolver.

```text
Internet / WAN ──X──► resolver
trusted LAN ─────────► UDP + TCP 53
admin subnet ────────► web UI and SSH
```

Example UFW rules after replacing variables:

```bash
LAN_IF='eth0'
LAN_ADDR='192.168.50.53'
LAN_CIDR='192.168.50.0/24'

sudo ufw allow in on "$LAN_IF" from "$LAN_CIDR" \
  to "$LAN_ADDR" port 53 proto udp
sudo ufw allow in on "$LAN_IF" from "$LAN_CIDR" \
  to "$LAN_ADDR" port 53 proto tcp
sudo ufw status verbose
```

Do not add a WAN port forward for 53. Restrict web administration separately;
DNS client access does not imply admin access.

IPv6 needs its own listener, ACL, and firewall decision. An IPv4-only firewall
does not protect a globally reachable IPv6 socket.

---

## Unbound

Unbound can recurse from the DNS root, validate DNSSEC, cache answers, serve
local data, or forward to selected upstreams.

```text
stub client ─► Unbound modules: iterator ─► validator ─► cache ─► response
```

Decide between these mutually exclusive upstream modes:

| Mode | Configuration | Dependency |
| --- | --- | --- |
| Full recursion | No catch-all `forward-zone` | Root and authoritative servers reachable on UDP/TCP 53 |
| Forwarding | `forward-zone` for `.` | Chosen recursive upstream |
| Forwarding over TLS | `forward-tls-upstream: yes` | Upstream DoT, CA bundle, correct certificate name |

---

## Install Unbound

Use distribution packages:

```bash
sudo apt update
sudo apt install unbound dns-root-data

unbound -V
sudo unbound-checkconf
systemctl status unbound --no-pager
```

Debian and Ubuntu package configuration normally includes files under:

```text
/etc/unbound/unbound.conf
/etc/unbound/unbound.conf.d/*.conf
/var/lib/unbound/root.key
```

Inspect packaged configuration before adding a trust anchor or root hints:

```bash
sudo unbound-checkconf -o auto-trust-anchor-file
sudo unbound-checkconf -o root-hints
dpkg -L dns-root-data | sort
```

Do not download a second root-hints file or overwrite a managed root key unless
the package configuration actually requires it.

---

## Standalone recursive resolver

This example listens on loopback and one LAN address. Replace the network:

```conf
# /etc/unbound/unbound.conf.d/20-lan.conf
server:
    interface: 127.0.0.1
    interface: 192.168.50.53
    port: 53

    access-control: 127.0.0.0/8 allow
    access-control: 192.168.50.0/24 allow
    access-control: 0.0.0.0/0 refuse
    access-control: ::0/0 refuse

    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    qname-minimisation: yes
    aggressive-nsec: yes
    hide-identity: yes
    hide-version: yes
    prefetch: yes
```

The absence of a catch-all `forward-zone` means Unbound performs recursion.
Enable IPv6 listening and an IPv6 ACL only when the host has a planned IPv6
address and firewall policy.

Validate before restart:

```bash
sudo unbound-checkconf
sudo systemctl restart unbound
systemctl status unbound --no-pager
sudo ss -lntup '( sport = :53 )'
```

---

## Loopback Unbound for a filtering frontend

Use this instead of the standalone listener when AdGuard Home or Pi-hole owns
LAN port 53:

```conf
# /etc/unbound/unbound.conf.d/20-filter-upstream.conf
server:
    interface: 127.0.0.1
    port: 5335

    access-control: 127.0.0.0/8 allow
    access-control: 0.0.0.0/0 refuse
    access-control: ::0/0 refuse

    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    qname-minimisation: yes
    aggressive-nsec: yes
    hide-identity: yes
    hide-version: yes
    prefetch: yes
```

Validate the isolated layer:

```bash
sudo unbound-checkconf
sudo systemctl restart unbound
dig @127.0.0.1 -p 5335 example.com A
dig @127.0.0.1 -p 5335 ietf.org A +dnssec
```

Do not also leave a different local Unbound drop-in binding `0.0.0.0:53`.

---

## Unbound forwarding over TLS

Forwarding changes Unbound from independent recursion to reliance on an
upstream. Add this only when that is the intended design:

```conf
# /etc/unbound/unbound.conf.d/30-forward-tls.conf
server:
    tls-cert-bundle: "/etc/ssl/certs/ca-certificates.crt"

forward-zone:
    name: "."
    forward-tls-upstream: yes
    forward-addr: 1.1.1.1@853#one.one.one.one
    forward-addr: 1.0.0.1@853#one.one.one.one
```

The `#name` authenticates the TLS certificate. Confirm the installed Unbound
supports the syntax:

```bash
unbound -V
sudo unbound-checkconf
```

Test strict failure as well as success. If the firewall blocks TCP 853 or the
certificate cannot validate, the resolver should not silently become plain DNS.

See [security.md](security.md) for DoT guarantees and downgrade behavior.

---

## Validate and operate Unbound

### Query and DNSSEC tests

```bash
PORT='5335'

dig @127.0.0.1 -p "$PORT" example.com A
dig @127.0.0.1 -p "$PORT" ietf.org A +dnssec
dig @127.0.0.1 -p "$PORT" dnssec.works A +dnssec
dig @127.0.0.1 -p "$PORT" fail01.dnssec.works A +dnssec
```

The maintained bogus test name should return `SERVFAIL`; the valid test should
return an answer with `ad`. External test domains can change, so pair them with
logs and a known signed domain.

### Status and logs

```bash
systemctl status unbound --no-pager
journalctl -u unbound --since '-10 minutes'
sudo unbound-control status
sudo unbound-control stats_noreset
```

`unbound-control` requires remote control to be enabled by the package or local
configuration.

### Targeted cache operations

```bash
sudo unbound-control lookup example.com
sudo unbound-control flush example.com
sudo unbound-control flush_zone example.com
sudo unbound-control reload
```

Prefer targeted flushing over restarting the service.

### Package upgrades

```bash
apt list --upgradable 2>/dev/null | rg '^unbound/'
sudo apt install --only-upgrade unbound
sudo unbound-checkconf
systemctl status unbound --no-pager
```

Keep local drop-ins and their rationale under configuration management.

---

## Local records in Unbound

Use `home.arpa` for a home network rather than inventing a public suffix or
using `.local`, which belongs to mDNS.

```conf
# /etc/unbound/unbound.conf.d/40-local-data.conf
server:
    local-zone: "home.arpa." static
    local-data: "router.home.arpa. 300 IN A 192.168.50.1"
    local-data: "nas.home.arpa. 300 IN A 192.168.50.20"
    local-data-ptr: "192.168.50.1 router.home.arpa."
    local-data-ptr: "192.168.50.20 nas.home.arpa."
```

```bash
sudo unbound-checkconf
sudo unbound-control reload
dig @127.0.0.1 -p 5335 nas.home.arpa A
dig @127.0.0.1 -p 5335 -x 192.168.50.20
```

`static` returns `NXDOMAIN` for unknown data beneath the zone. Use another
local-zone type only after reading its exact Unbound semantics.

---

## AdGuard Home

AdGuard Home is a filtering DNS proxy with a web interface, client policies,
local rewrites, encrypted upstream support, and encrypted client endpoints.

```text
clients ─► AdGuard Home ─┬─ blocked/local ─► immediate answer
                         └─ allowed ───────► upstream DNS or local Unbound
```

It is not a replacement for an independent recursive validator merely because
it can request DNSSEC data. Pair it with a validating upstream when DNSSEC
assurance is required.

---

## Install AdGuard Home natively

The official project supplies an installation script. Download and inspect it
instead of piping a network response directly into a privileged shell:

```bash
cd /tmp
curl --fail --show-error --location \
  --output install-adguard-home.sh \
  'https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh'

sha256sum install-adguard-home.sh
less install-adguard-home.sh
sudo sh ./install-adguard-home.sh -v
```

For stricter change control, download a release archive from the official
GitHub release, verify its published checksum, unpack it, and run:

```bash
sudo ./AdGuardHome -s install
```

Typical scripted installation location:

```text
/opt/AdGuardHome/AdGuardHome
/opt/AdGuardHome/AdGuardHome.yaml
/opt/AdGuardHome/data/
```

Discover the actual service rather than assuming:

```bash
sudo /opt/AdGuardHome/AdGuardHome -s status
systemctl status AdGuardHome --no-pager
systemctl cat AdGuardHome
journalctl -u AdGuardHome --since '-10 minutes'
```

---

## AdGuard Home initial configuration

The first-run wizard normally listens on port 3000:

```text
http://192.168.50.53:3000/
```

During the wizard:

1. Bind DNS to the intended LAN address, not every public interface.
2. Bind the admin UI to a trusted management address.
3. Set a unique administrator password.
4. Choose an upstream that does not point back to AdGuard Home.
5. Finish setup before changing DHCP clients.

Verify sockets and one direct query:

```bash
sudo ss -lntup | rg 'AdGuardHome|:(53|80|443|853|3000)\b'
dig @192.168.50.53 example.com A
```

If port 53 fails to bind, return to the listener inventory. Do not solve it by
turning off unrelated services without identifying their address.

---

## AdGuard Home upstreams

AdGuard Home accepts several upstream forms:

```text
# Plain DNS
192.0.2.53
192.0.2.53:53

# Local Unbound
127.0.0.1:5335

# DNS over TLS
tls://resolver.example

# DNS over HTTPS
https://resolver.example/dns-query

# DNS over QUIC
quic://resolver.example

# Only for a private domain
[/corp.example/]10.20.0.53
```

Configure bootstrap resolvers when an encrypted upstream is named by hostname.
Bootstrap DNS must not depend on the same unresolved hostname or create a loop.

For AdGuard Home in front of Unbound:

```text
Upstream DNS servers: 127.0.0.1:5335
Fallback DNS servers: leave empty unless a deliberate policy permits bypass
```

Test both layers:

```bash
dig @127.0.0.1 -p 5335 example.com A
dig @192.168.50.53 example.com A
```

If the first works and the second fails, troubleshoot AdGuard policy or its
upstream configuration rather than Unbound recursion.

---

## AdGuard Home filtering and local names

Use the web UI for validated configuration where possible:

- **Filters → DNS blocklists**: subscribe to curated lists sparingly.
- **Filters → Custom filtering rules**: add narrow allow or block rules.
- **Filters → DNS rewrites**: define local A, AAAA, CNAME, or other responses.
- **Settings → Client settings**: assign policies to known clients.
- **Settings → DNS settings**: configure private reverse DNS and cache behavior.
- **Query Log**: observe which rule or upstream produced a result.

Test a rule without relying on a browser cache:

```bash
# Replace these reserved example names with names matched by your test rules.
dig @192.168.50.53 blocked.example A
dig @192.168.50.53 allowed.example A
```

For local reverse names, configure the router or authoritative local resolver
as the private reverse DNS upstream. Do not forward RFC 1918 PTR questions to a
public resolver.

The `enable_dnssec` configuration sends DNSSEC-aware upstream requests; use a
validating upstream and verify `AD` end to end when validation is a requirement.

---

## AdGuard Home encrypted endpoints

AdGuard Home can serve DoH, DoT, and DoQ when configured with a DNS name and a
valid certificate.

```text
DoH ─► HTTPS port 443 and `/dns-query`
DoT ─► TCP port 853
DoQ ─► UDP port 853
```

Before enabling an endpoint:

1. Create public or private DNS for the server name.
2. Obtain a certificate clients trust for that exact name.
3. Automate certificate renewal and reload.
4. Restrict the admin UI independently from the DNS endpoint.
5. Decide whether the endpoint is LAN-only, VPN-only, or internet-facing.
6. Apply rate limits and firewall policy before public exposure.

Do not expose plain recursive DNS to the internet merely because encrypted DNS
is also enabled. See [security.md](security.md) for protocol differences and
client tests.

---

## Operate and update AdGuard Home

### Service and logs

```bash
sudo /opt/AdGuardHome/AdGuardHome -s status
sudo /opt/AdGuardHome/AdGuardHome -s restart
journalctl -u AdGuardHome --since today
```

### Validate a manual configuration change

Stop the service before editing YAML; a running instance can overwrite manual
changes:

```bash
sudo /opt/AdGuardHome/AdGuardHome -s stop
sudo cp --archive \
  /opt/AdGuardHome/AdGuardHome.yaml \
  /opt/AdGuardHome/AdGuardHome.yaml.before-change

sudo /opt/AdGuardHome/AdGuardHome \
  --check-config \
  --config /opt/AdGuardHome/AdGuardHome.yaml \
  --work-dir /opt/AdGuardHome

sudo /opt/AdGuardHome/AdGuardHome -s start
```

Prefer the UI or API for normal changes because they validate and persist the
current schema.

### Backup

Back up the YAML, data directory, certificate references, and service unit:

```bash
sudo /opt/AdGuardHome/AdGuardHome -s stop
sudo tar --create --gzip \
  --file="/var/backups/adguard-home-$(date +%F).tar.gz" \
  --directory=/opt AdGuardHome
sudo /opt/AdGuardHome/AdGuardHome -s start
```

Copy the archive off the resolver and test restoration on a non-production
address.

### Update

The UI can perform a stable update and retains a backup. CLI update:

```bash
sudo /opt/AdGuardHome/AdGuardHome --update
sudo /opt/AdGuardHome/AdGuardHome -s status
dig @192.168.50.53 example.com A
```

Review release notes first and keep the previous executable and configuration.

### Uninstall

Move clients to a working resolver before unregistering the service:

```bash
sudo /opt/AdGuardHome/AdGuardHome -s uninstall
```

The installation directory may remain for deliberate backup or removal. Do not
delete it until client DNS and rollback have been verified.

---

## Pi-hole

Pi-hole v6 combines FTL's DNS, DHCP, API, and web services with filtering and
local-network visibility.

```text
clients ─► pihole-FTL:53 ─┬─ gravity/custom rule ─► block or rewrite
                           ├─ local DNS/DHCP data ─► local answer
                           └─ allowed query ───────► upstream
```

Current configuration is centered on `/etc/pihole/pihole.toml`; prefer the web
interface or `pihole-FTL --config` over legacy v5 file edits.

---

## Install Pi-hole natively

Give the host a stable address and complete the port inventory first. Download
and inspect the official installer:

```bash
cd /tmp
curl --fail --show-error --location \
  --output basic-install.sh \
  'https://install.pi-hole.net'

sha256sum basic-install.sh
less basic-install.sh
sudo bash ./basic-install.sh
```

After installation:

```bash
pihole version
pihole status
systemctl status pihole-FTL --no-pager
sudo ss -lntup | rg 'pihole-FTL|:(53|80|443)\b'
dig @127.0.0.1 example.com A
```

Set the web password interactively so it does not enter shell history:

```bash
sudo pihole setpassword
```

Do not change router DHCP until a second client can query the Pi-hole address.

---

## Pi-hole v6 configuration

Inspect configuration through the supported CLI:

```bash
sudo pihole-FTL --config dns
sudo pihole-FTL --config misc.privacylevel
sudo sed -n '1,200p' /etc/pihole/pihole.toml
```

Use CLI setters for validation:

```bash
sudo pihole-FTL --config dns.interface 'eth0'
sudo pihole-FTL --config dns.listeningMode 'LOCAL'
sudo pihole-FTL --config dns.domain.name 'home.arpa'
sudo pihole-FTL --config dns.domain.local true
```

Listening modes have security consequences:

| Mode | Use |
| --- | --- |
| `LOCAL` | Accept clients on directly connected local subnets; safe default. |
| `SINGLE` | Accept all origins on the configured interface; add firewall policy. |
| `BIND` | Bind only selected addresses; useful with libvirt or another local port-53 service. |
| `ALL` | Accept all origins; dangerous without strict firewalling. |
| `NONE` | Advanced manual dnsmasq-style configuration. |

After changes:

```bash
sudo systemctl restart pihole-FTL
pihole status
journalctl -u pihole-FTL --since '-5 minutes'
```

---

## Pi-hole with Unbound

Configure Unbound on `127.0.0.1:5335` first and prove it works. Then set Pi-hole
to use only that upstream:

```bash
dig @127.0.0.1 -p 5335 example.com A

sudo pihole-FTL --config dns.upstreams \
  '[ "127.0.0.1#5335" ]'
```

When Unbound is the validating layer, avoid redundant Pi-hole validation:

```bash
sudo pihole-FTL --config dns.dnssec false
sudo systemctl restart pihole-FTL
```

Verify the path:

```bash
dig @127.0.0.1 example.com A
dig @127.0.0.1 ietf.org A +dnssec
pihole tail
```

The Pi-hole log should show allowed queries forwarded to `127.0.0.1#5335`.
Do not configure Unbound to forward back to Pi-hole.

---

## Pi-hole local names and filtering

### Local records

```bash
sudo pihole-FTL --config dns.hosts \
  '[ "192.168.50.1 router.home.arpa router", "192.168.50.20 nas.home.arpa nas" ]'
```

Test forward and reverse behavior:

```bash
dig @127.0.0.1 nas.home.arpa A
dig @127.0.0.1 -x 192.168.50.20
```

### Reverse server for router-managed DHCP

When the router owns DHCP and local names:

```bash
sudo pihole-FTL --config dns.revServers \
  '[ "true,192.168.50.0/24,192.168.50.1,home.arpa" ]'
```

Ensure the target router actually answers those forward and reverse zones.
Avoid a loop where the router forwards the same question back to Pi-hole.

### Filtering operations

```bash
pihole query example.com
pihole tail
sudo pihole updateGravity

# Temporarily disable blocking for five minutes
sudo pihole disable 5m
sudo pihole enable
```

Use temporary disablement to diagnose policy, not as a permanent workaround.

### Query-log privacy

```bash
sudo pihole-FTL --config misc.privacylevel 1
```

Higher levels hide more client and domain data but reduce diagnostic detail.
Choose retention and privacy settings intentionally.

---

## Operate and update Pi-hole

### Health and logs

```bash
pihole status
pihole version
pihole tail
journalctl -u pihole-FTL --since today
```

### Backup and restore

Use **Settings → Teleporter** in the Pi-hole web interface to export supported
configuration. Store the archive away from the resolver and test import on a
non-production address.

Also record:

```bash
sudo pihole-FTL --config dns
pihole version
ip -brief address
```

### Update

Read release notes and create a Teleporter backup first:

```bash
sudo pihole updatePihole
pihole version
pihole status
dig @127.0.0.1 example.com A
```

`pihole -up` is the established short form.

### Repair and debug

```bash
sudo pihole debug
sudo pihole repair
```

Review debug output for secrets before sharing it.

### Uninstall

Move DHCP clients and the host itself to a working resolver first:

```bash
sudo pihole uninstall
```

Renew a client lease and verify DNS after removal.

---

## Service ordering for a stacked resolver

The filtering frontend should start after Unbound:

```text
network-online ─► unbound:5335 ─► AdGuard Home or pihole-FTL:53
```

AdGuard Home override:

```ini
# systemctl edit AdGuardHome.service
[Unit]
Wants=unbound.service
After=unbound.service
```

Pi-hole override:

```ini
# systemctl edit pihole-FTL.service
[Unit]
Wants=unbound.service
After=unbound.service
```

Apply and inspect ordering:

```bash
sudo systemctl daemon-reload
systemctl list-dependencies AdGuardHome.service 2>/dev/null || true
systemctl list-dependencies pihole-FTL.service 2>/dev/null || true
```

`Wants=` improves startup ordering but does not make the frontend continuously
healthy when Unbound later fails. Monitor both layers.

---

## Roll out to clients

### Stage 1: direct tests

From a second LAN host:

```bash
SERVER='192.168.50.53'

dig "@$SERVER" example.com A
dig "@$SERVER" ietf.org A +dnssec
dig "@$SERVER" nas.home.arpa A
```

Test allowed, blocked, local, reverse, signed, and nonexistent names.

### Stage 2: one client

Set one disposable client's DNS to the new resolver. Renew its connection and
test through the system path:

```bash
getent ahosts example.com
resolvectl status 2>/dev/null || cat /etc/resolv.conf
```

### Stage 3: DHCP

Configure the router's DHCP option 6 to advertise the resolver's LAN address.
Then renew one lease before changing the whole network.

```text
DHCP server ── option 6: 192.168.50.53 ──► clients
```

If two DNS server addresses are advertised, clients may use either; they do not
necessarily treat the second as standby. Use two resolvers with equivalent
policy or advertise only the intended resolver.

### Stage 4: resolver host

Change the resolver host's own DNS last. Keep package downloads and emergency
access working if the local DNS service is stopped.

---

## IPv6 and DNS bypass

IPv6 clients can receive DNS through Router Advertisements (RDNSS) or DHCPv6
independently of IPv4 DHCP:

```text
IPv4 DHCP DNS ─► Pi-hole
IPv6 RDNSS ────► router or ISP resolver   ← filtering bypass
```

Inspect a client:

```bash
resolvectl status
nmcli device show 2>/dev/null | rg 'IP[46]\.DNS' || true
```

Decide whether to:

- Give the resolver a stable IPv6 address and advertise it.
- Stop advertising an unintended IPv6 resolver.
- Keep the resolver IPv4-only and verify clients do not learn another path.

Other bypasses include browser DoH, Android Private DNS, VPN-provided DNS,
hard-coded device resolvers, and proxies that resolve remotely. Blocking all
encrypted DNS is neither simple nor always appropriate; define the policy and
manage capable clients.

---

## End-to-end verification

For a stacked resolver, test from the inside outward:

```bash
LAN_SERVER='192.168.50.53'

# Recursive layer
dig @127.0.0.1 -p 5335 example.com A

# Filtering layer on the server
dig @127.0.0.1 example.com A

# LAN listener
dig "@$LAN_SERVER" example.com A

# System resolver path
getent ahosts example.com
```

Acceptance matrix:

| Test | Expected result |
| --- | --- |
| Allowed public name | Answer returned and logged through the intended upstream. |
| Blocked name | Documented block response and matching rule. |
| Local forward name | Correct private address without upstream leakage. |
| Local reverse name | Correct PTR or an intentional negative answer. |
| Signed valid name | Successful answer with validation evidence at the chosen layer. |
| Bogus DNSSEC name | `SERVFAIL` from the validating layer. |
| Unknown name | Correct `NXDOMAIN`, not timeout or fabricated address. |
| TCP query | Works as well as UDP. |
| Second LAN client | Uses the advertised resolver, not a hidden alternate. |

Watch traffic and logs during the tests:

```bash
sudo tcpdump -ni eth0 'udp port 53 or tcp port 53'
journalctl -f -u unbound -u AdGuardHome -u pihole-FTL
```

---

## Troubleshooting workflow

### 1. Identify the failed layer

```bash
dig @127.0.0.1 -p 5335 example.com A
dig @127.0.0.1 example.com A
dig @192.168.50.53 example.com A
```

Stop at the first failure.

### 2. Confirm sockets and ACLs

```bash
sudo ss -lntup '( sport = :53 or sport = :5335 )'
sudo nft list ruleset
```

An address-specific query cannot reach a loopback-only listener. `REFUSED`
usually points to an ACL or listening mode; a timeout often points to routing or
firewalling.

### 3. Check service configuration and logs

```bash
sudo unbound-checkconf
systemctl --no-pager status unbound AdGuardHome pihole-FTL
journalctl -u unbound -u AdGuardHome -u pihole-FTL --since '-10 minutes'
```

Missing units are harmless when that product is not installed.

### 4. Separate DNSSEC failure

```bash
dig @127.0.0.1 -p 5335 ietf.org A +dnssec
dig @127.0.0.1 -p 5335 ietf.org A +dnssec +cdflag
timedatectl status
```

Normal `SERVFAIL` plus `+cdflag` success points toward validation.

### 5. Separate filtering policy

Temporarily disable blocking through the product's supported UI or timed CLI,
then retry the exact name. Inspect the matching rule before allowlisting.

### 6. Check local-name loops

```bash
dig @192.168.50.53 router.home.arpa A
dig @192.168.50.53 -x 192.168.50.1
```

Make sure Pi-hole or AdGuard does not forward to a router that forwards the
same private question straight back.

### 7. Verify the client did not bypass the resolver

```bash
resolvectl status 2>/dev/null || cat /etc/resolv.conf
sudo tcpdump -ni any 'port 53 or port 853'
```

Inspect browser, VPN, IPv6, and mobile private-DNS settings.

### 8. Restore known-good DNS before repair

If the network loses resolution, restore DHCP and the resolver host to the
recorded upstream first. Repair the server without keeping every client broken.

---

## Choosing between the three

| Requirement | Unbound | AdGuard Home | Pi-hole |
| --- | --- | --- | --- |
| Full recursive resolution | Strong fit | Use upstream Unbound | Use upstream Unbound |
| DNSSEC validation | Core capability | Prefer validating upstream | Built in or upstream Unbound |
| Consumer blocklists | Manual policy only | Strong fit | Strong fit |
| Web dashboard | No | Yes | Yes |
| Native DoT/DoH/DoQ upstreams | DoT forwarding | Yes | Use a separate encrypted proxy/resolver |
| Native encrypted client endpoints | DoT possible with advanced Unbound config | DoT, DoH, and DoQ | Not the primary role |
| DHCP service | No | Optional | Integrated |
| File-first administration | Strong fit | YAML available; UI preferred | TOML available; CLI/UI preferred |
| Small independent resolver | Strong fit | More application features | More application features |

Choose the smallest design that meets the requirement. Stacking products adds
failure modes, caches, logs, and upgrade ordering as well as capability.

---

## Avoid these mistakes

- Installing a resolver before checking every existing port-53 listener.
- Binding to `0.0.0.0` and breaking systemd-resolved or libvirt dnsmasq.
- Exposing recursion to the WAN or global IPv6 internet.
- Advertising a public secondary DNS that bypasses filtering whenever a client chooses it.
- Pointing a filtering resolver and its upstream at each other.
- Changing all DHCP clients before testing one client and a rollback path.
- Using `.local` for unicast local DNS instead of mDNS.
- Editing generated `/etc/resolv.conf` or live AdGuard YAML as if it were the source of truth.
- Copying Pi-hole v5 configuration advice into a Pi-hole v6 installation.
- Enabling DNSSEC at several layers without knowing which layer returns `SERVFAIL`.
- Treating a blocklist as malware protection or application authentication.
- Keeping detailed query logs indefinitely without a privacy decision.
- Updating the DNS server without a backup and temporary client fallback.

---

## References

- [Unbound documentation](https://unbound.docs.nlnetlabs.nl/)
- [Unbound configuration reference](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.conf.html)
- [Unbound DNSSEC trust-anchor guidance](https://nlnetlabs.nl/documentation/unbound/howto-anchor/)
- [AdGuard Home project](https://github.com/AdguardTeam/AdGuardHome)
- [AdGuard Home configuration](https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration)
- [AdGuard Home encryption](https://adguard-dns.io/kb/adguard-home/running-securely/)
- [Pi-hole installation](https://docs.pi-hole.net/main/basic-install/)
- [Pi-hole v6 configuration reference](https://docs.pi-hole.net/ftldns/configfile/)
- [Pi-hole command reference](https://docs.pi-hole.net/main/pihole-command/)
- [Pi-hole with Unbound](https://docs.pi-hole.net/guides/dns/unbound/)
- [RFC 8375: Special-Use Domain `home.arpa`](https://www.rfc-editor.org/rfc/rfc8375)
