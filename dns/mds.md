# Multicast DNS (mDNS) Cheatsheet

## Table of Contents

- [Practical exercise](#practical-exercise)
- [Quick Reference](#quick-reference)
- [Mental Model](#mental-model)
- [mDNS and DNS-SD](#mdns-and-dns-sd)
- [mDNS versus Unicast DNS](#mdns-versus-unicast-dns)
- [The `.local.` Namespace](#the-local-namespace)
- [Resolve Host Names](#resolve-host-names)
- [Linux with Avahi](#linux-with-avahi)
- [Linux with `systemd-resolved`](#linux-with-systemd-resolved)
- [macOS](#macos)
- [Publish Host Names with Avahi](#publish-host-names-with-avahi)
- [Packet Capture](#packet-capture)
- [Reading mDNS Packets](#reading-mdns-packets)
- [Firewalls and Network Boundaries](#firewalls-and-network-boundaries)
- [Troubleshooting Workflow](#troubleshooting-workflow)
- [Common Problems](#common-problems)
- [Security Notes](#security-notes)
- [Avoid These Mistakes](#avoid-these-mistakes)
- [References](#references)

---

## Practical exercise

Use two terminals on a Linux host. In terminal A, watch mDNS traffic:

```bash
sudo tcpdump -ni any -vv 'udp port 5353'
```

In terminal B, resolve your own advertised name or replace it with a known
device such as `printer.local`:

```bash
TARGET="$(hostname -s).local"
printf 'Looking for %s\n' "$TARGET"
avahi-resolve-host-name "$TARGET"
getent hosts "$TARGET"
```

Stop the capture with `Ctrl-C` after the response appears.

```text
terminal B                    local link                         device owning the name
resolver ── query ──► 224.0.0.251:5353 / [FF02::FB]:5353 ◄── A or AAAA response
terminal A ───────────────────────── observes both directions ───────────────────────
```

| Look for | Meaning |
| --- | --- |
| Destination port `5353` | This is mDNS, not a query to a conventional DNS server. |
| `A` or `AAAA` answer | The owner is mapping its `.local.` name to an address. |
| No answer | Try a known `.local` device, then check Avahi and multicast reachability. |

This exercise changes no network configuration. It lets you feel the key mDNS
idea first: every participant asks and answers directly on the local link.

---

## Quick Reference

| Property       | Value                                  |
| -------------- | -------------------------------------- |
| Transport      | UDP                                    |
| Port           | `5353`                                 |
| IPv4 multicast | `224.0.0.251`                          |
| IPv6 multicast | `FF02::FB`                             |
| Scope          | Local link                             |
| Primary suffix | `.local.`                              |
| Specification  | RFC 6762                               |

Typical host lookup:

```text
printer.local.  IN A     192.168.1.50
printer.local.  IN AAAA  2001:db8::50
```

mDNS performs DNS-like lookups without requiring a conventional DNS server. Participants on the link cooperatively answer for records they own.

---

## Mental Model

```text
Querier
  │
  │  Who has printer.local. A?
  ▼
224.0.0.251:5353 or [FF02::FB]:5353
  │
  ├── laptop
  ├── phone
  ├── printer  ──► printer.local. A 192.168.1.50
  └── media server
```

There is no single authoritative mDNS server for the link. Each responder is authoritative for the records it publishes.

An mDNS namespace has no `NS` or `SOA` records, delegation, or zone transfers. It behaves like a distributed local-link DNS zone rather than a centrally administered zone.

---

## mDNS and DNS-SD

mDNS and DNS-SD are related but distinct:

| Technology | Primary job                                                |
| ---------- | ---------------------------------------------------------- |
| mDNS       | Carry DNS queries and records on the local link            |
| DNS-SD     | Advertise, browse, and resolve named service instances     |

mDNS can resolve a host name:

```text
printer.local. → 192.168.1.50
```

DNS-SD can discover a service and its endpoint:

```text
Office Printer._ipp._tcp.local.
  → printer.local.:631
```

DNS-SD commonly uses mDNS for zero-configuration discovery, but DNS-SD can also use conventional unicast DNS. See [sd.md](sd.md) for service discovery.

---

## mDNS versus Unicast DNS

| Property          | mDNS                                      | Unicast DNS                         |
| ----------------- | ----------------------------------------- | ----------------------------------- |
| Destination       | A link-local multicast group              | A configured DNS server             |
| Port              | UDP `5353`                                | UDP or TCP `53`                     |
| Administration    | Distributed among participating hosts     | Centrally managed zones             |
| Normal scope      | One link                                  | Routed networks and the internet    |
| Typical namespace | `.local.`                                 | Delegated DNS names                 |
| Authority records | No `NS` or `SOA`                          | `NS` and `SOA`                      |
| Zone transfers    | Not supported                             | Possible with AXFR or IXFR          |
| Name conflicts    | Detected through probing                  | Prevented by zone administration    |

Both protocols use the DNS message format and familiar record types such as `A`, `AAAA`, `PTR`, `SRV`, and `TXT`.

---

## The `.local.` Namespace

Every fully qualified name ending in `.local.` has link-local mDNS semantics.

```text
printer.local.
nas.local.
labhost.local.
```

For host names, the recommended form is a single label followed by `.local.`:

```text
<host>.local.
```

Important properties:

* The name is meaningful only on the link where it is advertised.
* The same name may resolve differently on another link.
* A responder probes before claiming a unique name and renames on conflict.
* The final `.` marks a fully qualified name and avoids search-domain expansion.
* Names below `.local.` may contain more labels, particularly for DNS-SD records.

Do not use `.local.` as a conventional private unicast DNS zone. Doing so creates ambiguous resolution behavior because mDNS-aware clients treat the suffix specially.

---

## Resolve Host Names

### Avahi

Resolve a host name:

```bash
avahi-resolve-host-name printer.local
avahi-resolve --name printer.local
```

Restrict the address family:

```bash
avahi-resolve -4 --name printer.local
avahi-resolve -6 --name printer.local
```

Reverse lookup:

```bash
avahi-resolve-address 192.168.1.50
avahi-resolve --address 192.168.1.50
```

### System Name Service

```bash
getent ahosts printer.local
getent hosts printer.local
ping printer.local
```

These commands use the host's configured name-service stack. Success depends on Avahi, `systemd-resolved`, or another mDNS integration being enabled.

### One-Shot Query with `dig`

```bash
dig @224.0.0.251 -p 5353 printer.local A +time=2 +tries=1
```

This is only a legacy one-shot multicast query. `dig` is not a full mDNS browser and does not implement continuous browsing, probing, cache management, or known-answer suppression. Prefer the platform mDNS tools for normal use.

`nslookup` and a plain `dig printer.local` normally query configured unicast DNS and are poor tests of the system's mDNS path.

---

## Linux with Avahi

Common components:

| Component            | Purpose                                      |
| -------------------- | -------------------------------------------- |
| `avahi-daemon`       | mDNS responder, querier, and cache           |
| `avahi-resolve`      | Resolve host names and addresses             |
| `avahi-browse`       | Browse DNS-SD services                       |
| `avahi-publish`      | Publish temporary hosts or services          |
| `nss-mdns`           | Integrate mDNS with libc host-name lookups   |

Check the daemon:

```bash
systemctl status avahi-daemon
avahi-daemon --check
journalctl -u avahi-daemon
```

Primary paths:

```text
/etc/avahi/avahi-daemon.conf
/etc/avahi/hosts
/etc/avahi/services/*.service
```

Example `/etc/avahi/avahi-daemon.conf` settings:

```ini
[server]
host-name=labhost
domain-name=local
use-ipv4=yes
use-ipv6=yes
allow-interfaces=eth0,wlan0

[publish]
publish-addresses=yes
```

`host-name` is the label without `.local.`. Restricting `allow-interfaces` is useful on hosts connected to both trusted and untrusted networks.

After changing the daemon configuration:

```bash
sudo systemctl restart avahi-daemon
```

### NSS Integration

See [linux.md](linux.md) for the complete Linux NSS resolution path and for
diagnosing differences between `getent`, `resolvectl`, and `dig`.

A common `nss-mdns` configuration looks like:

```text
hosts: files mdns4_minimal [NOTFOUND=return] dns mdns4
```

The exact `hosts:` line is distribution-specific. Do not copy it blindly over an existing `resolve`, `myhostname`, LDAP, or other NSS setup.

If direct Avahi resolution works but `getent` does not, inspect:

```bash
getent ahosts printer.local
rg '^hosts:' /etc/nsswitch.conf
```

---

## Linux with `systemd-resolved`

Inspect global and per-link mDNS state:

```bash
resolvectl status
resolvectl mdns
resolvectl mdns eth0
```

Force an mDNS lookup:

```bash
resolvectl --protocol=mdns query printer.local
resolvectl --protocol=mdns-ipv4 query printer.local
resolvectl --protocol=mdns-ipv6 query printer.local
```

Select an interface:

```bash
resolvectl --interface=eth0 --protocol=mdns query printer.local
```

Temporarily enable full mDNS support on a link:

```bash
sudo resolvectl mdns eth0 yes
```

Allow resolution without registering the local host name:

```bash
sudo resolvectl mdns eth0 resolve
```

Undo temporary per-link changes:

```bash
sudo resolvectl revert eth0
```

Global persistent configuration:

```ini
# /etc/systemd/resolved.conf
[Resolve]
MulticastDNS=yes
```

Then restart the resolver:

```bash
sudo systemctl restart systemd-resolved
```

Per-link settings supplied by NetworkManager or `systemd-networkd` can override the global mode.

---

## macOS

Resolve IPv4 and IPv6 addresses using Bonjour:

```bash
dns-sd -G v4v6 printer.local.
```

Query specific records:

```bash
dns-sd -Q printer.local. A
dns-sd -Q printer.local. AAAA
```

Test the system resolver:

```bash
ping printer.local
dscacheutil -q host -a name printer.local
```

`dns-sd` operations are normally continuous. Stop them with `Ctrl-C` after observing the result.

For service browsing and registration with `dns-sd`, see [sd.md](sd.md).

---

## Publish Host Names with Avahi

Publish an address record until the process exits:

```bash
avahi-publish-address labhost.local 192.168.1.50
```

Equivalent form:

```bash
avahi-publish --address labhost.local 192.168.1.50
```

Publish an IPv6 address:

```bash
avahi-publish-address labhost.local 2001:db8::50
```

Static mappings can be placed in `/etc/avahi/hosts`:

```text
192.168.1.50 labhost.local
2001:db8::50 labhost.local
```

Reload static host and service definitions:

```bash
sudo avahi-daemon --reload
```

Publishing a record does not configure the address on an interface or make a service reachable. The advertised address must actually belong to the intended host and be usable from the link.

---

## Packet Capture

Capture all mDNS traffic:

```bash
sudo tcpdump -i any -enn -vv 'udp port 5353'
```

Capture on one interface:

```bash
sudo tcpdump -i eth0 -enn -vv 'udp port 5353'
```

IPv4 multicast only:

```bash
sudo tcpdump -i eth0 -enn -vv \
  'udp port 5353 and host 224.0.0.251'
```

IPv6 mDNS:

```bash
sudo tcpdump -i eth0 -enn -vv \
  'ip6 and udp port 5353'
```

Write a packet capture:

```bash
sudo tcpdump -i eth0 -s 0 -w mdns.pcap \
  'udp port 5353'
```

Wireshark display filter:

```text
mdns
```

Useful narrower filters:

```text
mdns && dns.qry.name == "printer.local"
mdns && dns.flags.response == 1
```

---

## Reading mDNS Packets

### Query and Response Ports

A fully compliant continuous querier sends from UDP source port `5353` and listens on the multicast group. A simple legacy one-shot querier may use an ephemeral source port and receive a unicast response.

All valid mDNS responses use UDP source port `5353`.

### Important Fields and Behaviors

| Feature                  | Meaning                                                       |
| ------------------------ | ------------------------------------------------------------- |
| Query ID `0`             | Normal for multicast queries and responses                    |
| `AA` in a response       | Responder is authoritative for the records it publishes       |
| `QM` question            | Request a multicast response                                  |
| `QU` question            | A unicast response is acceptable                              |
| Known-answer list        | Suppresses answers the querier already has                    |
| Probe                    | Tests whether a proposed unique name is already in use        |
| Announcement             | Unsolicited response publishing new or changed records        |
| Cache-flush bit          | Replaces older data for a unique record set                   |
| Goodbye with TTL `0`     | Requests prompt removal of a record from peer caches          |

### Startup Sequence

```text
Probe proposed unique records
        ↓
Resolve any conflicts
        ↓
Send unsolicited announcements
        ↓
Answer queries and refresh records
```

Typical address-record TTLs are `120` seconds. The protocol recommends `75` minutes for many other record types. Clients age cached records normally and process announcements, cache-flush bits, and goodbye packets to keep caches coherent.

mDNS responses should use an IP TTL or IPv6 hop limit of `255`. Receivers must ensure responses originated on the local link.

---

## Firewalls and Network Boundaries

mDNS requires local inbound and outbound UDP `5353` traffic for:

```text
224.0.0.251
FF02::FB
```

On systems using `firewalld`, the predefined service can be enabled on a trusted zone:

```bash
sudo firewall-cmd --zone=home --add-service=mdns
sudo firewall-cmd --zone=home --add-service=mdns --permanent
```

mDNS is link-local and conventional routers do not forward it. Discovery across VLANs or routed subnets requires an explicitly configured discovery proxy, gateway, or reflector.

Before extending mDNS across boundaries, consider:

* Which service types should cross the boundary
* Whether both directions are necessary
* Whether guest or untrusted clients will learn internal host and service names
* Whether duplicate names exist on the connected links
* Whether the gateway preserves protocol requirements such as local-link validation

Wi-Fi client isolation, multicast filtering, broken IGMP or MLD snooping, and power-saving behavior can also block or delay mDNS on a single subnet.

---

## Troubleshooting Workflow

### 1. Confirm the Interface Is Up and Multicast-Capable

```bash
ip link show dev eth0
ip address show dev eth0
ip maddress show dev eth0
```

Look for `UP` and `MULTICAST` on the interface.

### 2. Identify the Active mDNS Stack

Avahi:

```bash
systemctl is-active avahi-daemon
avahi-daemon --check
```

`systemd-resolved`:

```bash
resolvectl status eth0
resolvectl mdns eth0
```

Know which daemon is resolving and which daemon is publishing. Conflicting configurations can produce confusing results.

### 3. Query Through the mDNS-Aware Tool

```bash
avahi-resolve-host-name printer.local
resolvectl --interface=eth0 --protocol=mdns query printer.local
```

If this works but the application fails, inspect NSS or application-specific resolver behavior.

### 4. Capture on the Requesting Host

```bash
sudo tcpdump -i eth0 -enn -vv 'udp port 5353'
```

Confirm that a query leaves on the expected interface and that a response returns.

### 5. Capture on the Responding Host

If the responder never sees the query, investigate the network path. If it sees the query but sends no answer, inspect its published records, interface restrictions, and daemon logs.

```bash
journalctl -u avahi-daemon
```

### 6. Compare IPv4 and IPv6

```bash
avahi-resolve -4 --name printer.local
avahi-resolve -6 --name printer.local
```

One address family may be disabled, filtered, or advertised with an unusable address.

### 7. Check the Local Resolver Path

```bash
getent ahosts printer.local
rg '^hosts:' /etc/nsswitch.conf
```

Direct Avahi success combined with `getent` failure usually points to NSS integration rather than an on-wire mDNS failure.

---

## Common Problems

### Avahi Resolves but Applications Do Not

Check `/etc/nsswitch.conf`, the presence of `nss-mdns`, and whether the application bypasses the system resolver.

### Queries Go to Unicast DNS

The resolver may not have mDNS enabled, the link may be configured with mDNS disabled, or `.local.` may have been incorrectly deployed as a unicast DNS suffix.

### It Works Only on the Same Host

Local publication is working, but multicast may be blocked by a host firewall, access point, switch, container network, or virtual-machine bridge.

### It Works on Ethernet but Not Wi-Fi

Check wireless client isolation, multicast-to-unicast conversion, multicast rate limiting, power-saving behavior, and whether the daemon is allowed on the Wi-Fi interface.

### It Works on One VLAN Only

That is the normal mDNS scope. Use unicast DNS, wide-area DNS-SD, or a deliberately scoped discovery proxy when cross-subnet discovery is required.

### The Host Name Changes to Include a Number

Another responder already claimed the name. mDNS probing detected the conflict and the losing responder selected a unique alternative such as `labhost-2.local.`.

### Stale Addresses Remain Temporarily

A device that crashes or disconnects cannot send a goodbye packet. Peers retain its records until their TTLs expire or another cache-coherency mechanism removes them.

---

## Security Notes

* mDNS does not make local names trustworthy. Any on-link attacker may observe queries, spoof records, or race name claims.
* Authenticate the application protocol independently with TLS, SSH host keys, credentials, or another appropriate mechanism.
* Host names and DNS-SD metadata reveal information about users, devices, software, and network roles.
* Do not publish credentials, tokens, secrets, or sensitive identifiers in names or TXT records.
* Reflectors and discovery proxies expand both the multicast domain and the information exposed across trust boundaries.
* Limit mDNS to interfaces and firewall zones where local discovery is intended.

---

## Avoid These Mistakes

* Do not expect mDNS to cross a router by default.
* Do not configure `.local.` as an ordinary unicast DNS zone.
* Do not use plain `nslookup` as proof that mDNS is broken.
* Do not confuse a host-name lookup with DNS-SD service discovery.
* Do not assume an advertised address is configured or reachable.
* Do not open UDP `5353` on untrusted interfaces without considering the exposure.
* Do not treat a discovered name or address as authenticated.
* Do not deploy a reflector merely to hide an incorrect VLAN or DNS design.

---

## References

* [RFC 6762: Multicast DNS](https://www.rfc-editor.org/rfc/rfc6762.html)
* [RFC 6763: DNS-Based Service Discovery](https://www.rfc-editor.org/rfc/rfc6763.html)
* [Avahi command documentation](https://github.com/avahi/avahi/tree/master/man)
* [`resolvectl` documentation](https://www.freedesktop.org/software/systemd/man/devel/resolvectl.html)
* [Apple DNS Service Discovery Programming Guide](https://developer.apple.com/library/archive/documentation/Networking/Conceptual/dns_discovery_api/Introduction.html)
