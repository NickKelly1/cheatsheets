# DNS-Based Service Discovery (DNS-SD) Cheatsheet

## Table of Contents

- [Quick Reference](#quick-reference)
- [Mental Model](#mental-model)
- [DNS-SD and mDNS](#dns-sd-and-mdns)
- [Service Instance Names](#service-instance-names)
- [Core Record Set](#core-record-set)
- [Discovery Flow](#discovery-flow)
- [Common Service Types](#common-service-types)
- [Browse with Avahi](#browse-with-avahi)
- [Browse with `dns-sd`](#browse-with-dns-sd)
- [Resolve a Service](#resolve-a-service)
- [Publish with Avahi](#publish-with-avahi)
- [Publish with `dns-sd`](#publish-with-dns-sd)
- [Static Avahi Services](#static-avahi-services)
- [Inspect Raw Records](#inspect-raw-records)
- [Wide-Area DNS-SD](#wide-area-dns-sd)
- [TXT Record Guidelines](#txt-record-guidelines)
- [Service Subtypes](#service-subtypes)
- [Packet Capture](#packet-capture)
- [Troubleshooting Workflow](#troubleshooting-workflow)
- [Common Problems](#common-problems)
- [Security and Privacy](#security-and-privacy)
- [Avoid These Mistakes](#avoid-these-mistakes)
- [References](#references)

---

## Quick Reference

| Operation               | DNS record                                        |
| ----------------------- | ------------------------------------------------- |
| Enumerate instances     | `PTR`                                             |
| Find host and port      | `SRV`                                             |
| Read service metadata   | `TXT`                                             |
| Resolve the target host | `A` and `AAAA`                                    |
| Enumerate service types | `PTR` at `_services._dns-sd._udp.<domain>.`       |

Local zero-configuration discovery normally uses:

| Property      | Value                         |
| ------------- | ----------------------------- |
| Domain        | `local.`                      |
| Transport     | mDNS over UDP `5353`          |
| IPv4 group    | `224.0.0.251`                 |
| IPv6 group    | `FF02::FB`                    |
| Specification | RFC 6763                      |

Example service instance:

```text
Lab Dashboard._http._tcp.local.
```

| Part                | Value             |
| ------------------- | ----------------- |
| Instance name       | `Lab Dashboard`   |
| Application service | `_http`           |
| Transport protocol  | `_tcp`            |
| Discovery domain    | `local.`          |

---

## Mental Model

```text
Publisher
  │ advertises PTR + SRV + TXT + addresses
  ▼
DNS namespace
  ▲
  │ browse for a service type
Browser
  │
  ├── chooses a named instance
  ├── resolves its host, port, and metadata
  └── connects using the advertised application protocol
```

DNS-SD is not a port scan. It discovers services that deliberately publish records under a known service type.

The three main operations are:

1. **Publish** a named service instance.
2. **Browse** for instances of a service type.
3. **Resolve** one instance into a target host, port, addresses, and metadata.

---

## DNS-SD and mDNS

DNS-SD defines how DNS records describe and discover services. It does not require one specific DNS transport.

| Deployment          | Discovery domain | Query transport             | Typical use                       |
| ------------------- | ---------------- | --------------------------- | --------------------------------- |
| Local DNS-SD        | `local.`         | Multicast DNS               | Same-link zero configuration      |
| Wide-area DNS-SD    | A unicast domain | Conventional unicast DNS    | Routed or centrally managed sites |

Bonjour is Apple's implementation of zero-configuration networking. Avahi is a common Linux implementation. Both combine mDNS with DNS-SD for local discovery.

mDNS can answer:

```text
What address belongs to labhost.local.?
```

DNS-SD can answer:

```text
Which HTTP services are available?
Where is the selected instance listening?
Which metadata does it advertise?
```

See [mdns.md](mdns.md) for multicast transport, `.local.`, packet fields, and host-name resolution.

---

## Service Instance Names

The general form is:

```text
<Instance>.<Service>.<Domain>.
```

Expanded:

```text
<Instance>._<Application>._tcp.<Domain>.
<Instance>._<Application>._udp.<Domain>.
```

Example:

```text
Office Printer._ipp._tcp.local.
```

### Instance

The instance label is a user-visible name, not a host name:

```text
Office Printer
Alice's Music Library
Lab Dashboard
```

An instance label may contain spaces, punctuation, Unicode, and even literal dots. It is limited by the DNS label size of 63 bytes. Dots and backslashes must be escaped when a full instance name is written in DNS zone-file syntax.

### Service

The service consists of two labels:

```text
_<Application>._tcp
_<Application>._udp
```

The application service name is at most 15 characters, excluding the leading underscore. Use an IANA-registered service name when one exists.

`_tcp` or `_udp` describes the transport used by the discovered application protocol. It does not describe whether the DNS-SD query itself uses multicast or unicast DNS.

### Domain

Common examples:

```text
local.
example.com.
services.example.net.
```

The domain determines where the records live. It is not part of the user-visible instance name.

---

## Core Record Set

A complete instance normally looks like:

```dns
_http._tcp.local.                         PTR Lab\ Dashboard._http._tcp.local.
Lab\ Dashboard._http._tcp.local.          SRV 0 0 8080 labhost.local.
Lab\ Dashboard._http._tcp.local.          TXT "path=/status" "version=1"
labhost.local.                            A   192.168.1.50
labhost.local.                            AAAA 2001:db8::50
```

### `PTR`: Browse

```text
_http._tcp.local.
  → Lab Dashboard._http._tcp.local.
```

The `PTR` owner is the service type and domain. Each `PTR` target is one service instance. Multiple publishers can add targets to the same shared browse name.

### `SRV`: Endpoint

```text
Lab Dashboard._http._tcp.local.
  → priority 0, weight 0, port 8080, target labhost.local.
```

For the common single-endpoint case, priority and weight should both be `0`. The target must be a host name that resolves to `A` or `AAAA` records; it must not be a `CNAME` alias.

### `TXT`: Metadata

```text
"path=/status" "version=1"
```

The `TXT` record carries small service-type-specific key/value attributes. It must not duplicate the target host or port from the `SRV` record.

### `A` and `AAAA`: Addresses

```text
labhost.local. → 192.168.1.50 or 2001:db8::50
```

Clients resolve the `SRV` target and then connect to one of its usable addresses at the advertised port.

---

## Discovery Flow

```text
1. Browse
   PTR _http._tcp.local.?
        ↓
   Lab Dashboard._http._tcp.local.

2. Resolve the selected instance
   SRV + TXT Lab Dashboard._http._tcp.local.?
        ↓
   labhost.local.:8080, path=/status

3. Resolve the SRV target
   A + AAAA labhost.local.?
        ↓
   192.168.1.50, 2001:db8::50

4. Connect
   HTTP to a selected address on port 8080
```

mDNS responders commonly include the `SRV`, `TXT`, `A`, and `AAAA` records as additional answers so clients can complete discovery with fewer packets. Clients must still handle missing or independently refreshed records.

Browsing is a continuous operation. Instances may appear, change, or disappear while the browser is running.

---

## Common Service Types

| Type                | Typical purpose                         |
| ------------------- | --------------------------------------- |
| `_http._tcp`        | Human-viewable web interface            |
| `_ssh._tcp`         | Secure Shell                            |
| `_sftp-ssh._tcp`    | SFTP over SSH                           |
| `_ipp._tcp`         | Internet Printing Protocol              |
| `_ipps._tcp`        | IPP over TLS                            |
| `_printer._tcp`     | Line-printer spooler                    |
| `_smb._tcp`         | SMB file sharing                        |
| `_rtsp._tcp`        | Real-Time Streaming Protocol control    |

The service type describes the application protocol, not a device category. A printer may advertise `_ipp._tcp`, `_ipps._tcp`, `_printer._tcp`, and `_http._tcp` as separate services.

Consult the [IANA Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) before defining a new type.

---

## Browse with Avahi

Browse one service type continuously:

```bash
avahi-browse _http._tcp
```

Browse and resolve each instance:

```bash
avahi-browse --resolve _http._tcp
```

Return a current snapshot and exit:

```bash
avahi-browse --resolve --terminate _http._tcp
```

Browse all advertised service types:

```bash
avahi-browse --all
avahi-browse --all --resolve --terminate
```

Choose the discovery domain:

```bash
avahi-browse --domain=local _ipp._tcp
```

Script-friendly output:

```bash
avahi-browse --parsable --no-db-lookup \
  --resolve --terminate _http._tcp
```

Output prefixes:

| Prefix | Meaning                         |
| ------ | ------------------------------- |
| `+`    | Instance appeared               |
| `-`    | Instance disappeared            |
| `=`    | Instance was resolved           |

The interface and address family are significant. The same instance can appear on more than one interface or through both IPv4 and IPv6.

---

## Browse with `dns-sd`

On macOS, browse for HTTP services:

```bash
dns-sd -B _http._tcp local.
```

Browse for SSH services:

```bash
dns-sd -B _ssh._tcp local.
```

Enumerate advertised service types with the DNS-SD meta-query:

```bash
dns-sd -Q _services._dns-sd._udp.local. PTR
```

The browser remains active and reports additions and removals. Stop it with `Ctrl-C`.

Important output columns include:

| Column        | Meaning                                      |
| ------------- | -------------------------------------------- |
| `A/R`         | Add or remove event                          |
| `if`          | Interface index                              |
| Domain        | Discovery domain                             |
| Service Type  | Type being browsed                           |
| Instance Name | User-visible service instance                |

---

## Resolve a Service

First browse for an instance, then resolve the selected name.

### Avahi

```bash
avahi-browse --resolve --terminate _http._tcp
```

Resolved output includes the host name, address, port, and TXT attributes.

### macOS

```bash
dns-sd -L "Lab Dashboard" _http._tcp local.
```

### `systemd-resolved`

```bash
resolvectl service "Lab Dashboard" _http._tcp local
```

The three-argument form performs a DNS-SD-style `SRV` and `TXT` lookup for the named instance and normally resolves the target host's addresses as well.

Resolution is not the same as reachability. After resolving, verify the application endpoint:

```bash
curl http://labhost.local:8080/status
```

Use the application protocol implied by the service type rather than assuming every service is HTTP.

---

## Publish with Avahi

Publish a temporary service until the command exits:

```bash
avahi-publish-service \
  "Lab Dashboard" _http._tcp 8080 \
  path=/status version=1
```

Equivalent form:

```bash
avahi-publish --service \
  "Lab Dashboard" _http._tcp 8080 \
  path=/status version=1
```

Advertise a service running on another resolvable host:

```bash
avahi-publish-service \
  --host=labhost.local \
  "Lab Dashboard" _http._tcp 8080 \
  path=/status
```

Publish an additional subtype:

```bash
avahi-publish-service \
  --subtype=_admin._sub._http._tcp \
  "Lab Dashboard" _http._tcp 8080 \
  path=/status
```

Stop the command with `Ctrl-C` to withdraw the temporary registration.

Advertising a service does not start the server. Confirm that the process is listening on the advertised port and on an address reachable from clients.

---

## Publish with `dns-sd`

On macOS:

```bash
dns-sd -R \
  "Lab Dashboard" _http._tcp local. 8080 \
  path=/status version=1
```

The registration remains active until `dns-sd` exits. Stop it with `Ctrl-C`.

Test from another host:

```bash
dns-sd -B _http._tcp local.
```

Then resolve the instance:

```bash
dns-sd -L "Lab Dashboard" _http._tcp local.
```

---

## Static Avahi Services

Place XML service definitions in:

```text
/etc/avahi/services/*.service
```

Example `/etc/avahi/services/lab-dashboard.service`:

```xml
<?xml version="1.0" standalone='no'?>
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
  <name replace-wildcards="yes">Lab Dashboard on %h</name>
  <service>
    <type>_http._tcp</type>
    <port>8080</port>
    <txt-record>path=/status</txt-record>
    <txt-record>version=1</txt-record>
  </service>
</service-group>
```

Reload the definitions:

```bash
sudo systemctl reload avahi-daemon
```

If the service unit does not implement reload, restart it:

```bash
sudo systemctl restart avahi-daemon
```

Inspect registration errors:

```bash
journalctl -u avahi-daemon
```

---

## Inspect Raw Records

### Local mDNS with `dns-sd`

Browse records:

```bash
dns-sd -Q _http._tcp.local. PTR
```

For an instance without spaces in its label:

```bash
dns-sd -Q Lab-Dashboard._http._tcp.local. SRV
dns-sd -Q Lab-Dashboard._http._tcp.local. TXT
dns-sd -Q labhost.local. A
dns-sd -Q labhost.local. AAAA
```

### Legacy One-Shot mDNS with `dig`

```bash
dig @224.0.0.251 -p 5353 \
  _http._tcp.local PTR \
  +time=2 +tries=1
```

This can be useful for a quick packet-level check, but `dig` does not implement continuous DNS-SD browsing or full mDNS behavior.

### Unicast DNS-SD with `dig`

```bash
dig _http._tcp.example.com PTR
dig 'Lab\ Dashboard._http._tcp.example.com' SRV
dig 'Lab\ Dashboard._http._tcp.example.com' TXT
dig labhost.example.com A
dig labhost.example.com AAAA
```

Use a trailing `.` when search-domain expansion would be ambiguous.

---

## Wide-Area DNS-SD

DNS-SD can use an ordinary authoritative DNS zone and conventional routed DNS queries.

Example zone data:

```dns
$ORIGIN example.com.
$TTL 300

_services._dns-sd._udp IN PTR _http._tcp.example.com.

_http._tcp IN PTR Lab\ Dashboard._http._tcp.example.com.

Lab\ Dashboard._http._tcp IN SRV 0 0 8080 labhost.example.com.
Lab\ Dashboard._http._tcp IN TXT "path=/status" "version=1"

labhost IN A    192.0.2.50
labhost IN AAAA 2001:db8::50
```

Verify the chain:

```bash
dig _services._dns-sd._udp.example.com PTR
dig _http._tcp.example.com PTR
dig 'Lab\ Dashboard._http._tcp.example.com' SRV
dig 'Lab\ Dashboard._http._tcp.example.com' TXT
dig labhost.example.com A
dig labhost.example.com AAAA
```

Wide-area DNS-SD usually requires:

* An authoritative unicast DNS zone
* A way to create and remove `PTR`, `SRV`, `TXT`, `A`, and `AAAA` records
* Correct TTL and cache management
* DNS Update credentials when services register dynamically
* Client configuration or domain-enumeration records telling browsers where to search

DNS-SD defines special domain-enumeration names such as:

```text
b._dns-sd._udp.<domain>.   recommended browsing domains
db._dns-sd._udp.<domain>.  default browsing domain
r._dns-sd._udp.<domain>.   recommended registration domains
dr._dns-sd._udp.<domain>.  default registration domain
lb._dns-sd._udp.<domain>.  legacy browsing domains
```

Use wide-area DNS-SD or a standards-based discovery proxy when discovery must cross subnets. A broad mDNS reflector is not the only option.

---

## TXT Record Guidelines

Each DNS-SD service must publish a `TXT` record with the same owner name as its `SRV` record. When no metadata exists, the record contains one empty string.

Common forms:

```text
path=/status
version=1
tls
note=
```

These represent:

| Form        | Meaning                                  |
| ----------- | ---------------------------------------- |
| `key=value` | Attribute with a non-empty value         |
| `key`       | Boolean-style attribute with no value    |
| `key=`      | Attribute with an explicitly empty value |
| absent      | Attribute is not present                 |

Rules:

* Each key/value pair is one constituent DNS TXT string.
* Each constituent string can contain at most 255 bytes.
* Keys are printable US-ASCII except `=` and are compared case-insensitively.
* Keys should be nine characters or fewer.
* A key should appear at most once; clients use the first duplicate.
* Values may contain arbitrary bytes, although text is easiest to interoperate with.
* Unknown keys should be ignored so profiles can evolve compatibly.
* The target host and port belong in `SRV`, not `TXT`.

Size guidance:

| Total TXT size   | Guidance                                      |
| ---------------- | --------------------------------------------- |
| Up to 200 bytes  | Typical target                                |
| Under 400 bytes  | Usually fits comfortably in small DNS replies |
| Under 1300 bytes | Upper practical target                        |
| Over 1300 bytes  | Not recommended                               |

Keep metadata small, stable, and directly relevant to selecting or using the service. Fetch larger or rapidly changing data through the application protocol.

Never put passwords, bearer tokens, private keys, or other secrets in a `TXT` record.

---

## Service Subtypes

A subtype lets clients browse a narrower category without changing the primary service instance name.

Example:

```dns
_http._tcp.local.                    PTR Lab\ Dashboard._http._tcp.local.
_admin._sub._http._tcp.local.        PTR Lab\ Dashboard._http._tcp.local.
```

The instance remains:

```text
Lab Dashboard._http._tcp.local.
```

Browse the subtype with Avahi:

```bash
avahi-browse _admin._sub._http._tcp
```

Publish it with Avahi:

```bash
avahi-publish-service \
  --subtype=_admin._sub._http._tcp \
  "Lab Dashboard" _http._tcp 8080
```

Use subtypes only when the service profile defines meaningful selectable categories. Use `TXT` attributes for metadata that clients inspect after finding an instance.

---

## Packet Capture

Capture local DNS-SD over mDNS:

```bash
sudo tcpdump -i any -enn -vv 'udp port 5353'
```

Capture one interface to a file:

```bash
sudo tcpdump -i eth0 -s 0 -w dns-sd.pcap \
  'udp port 5353'
```

Wireshark display filters:

```text
mdns
mdns && dns.qry.type == 12
mdns && dns.qry.name == "_http._tcp.local"
mdns && dns.resp.type == 33
mdns && dns.resp.type == 16
```

Record type numbers:

| Type   | Number |
| ------ | -----: |
| `A`    | `1`    |
| `PTR`  | `12`   |
| `TXT`  | `16`   |
| `AAAA` | `28`   |
| `SRV`  | `33`   |

During a successful browse and resolution, look for:

1. A `PTR` question for the service type.
2. One or more instance `PTR` answers.
3. `SRV` and `TXT` records for an instance.
4. `A` or `AAAA` records for the `SRV` target.

Additional records may combine several steps into one response.

---

## Troubleshooting Workflow

### 1. Confirm the Application Is Listening

```bash
sudo ss -lntup
```

Verify the port and address family. A server bound only to `127.0.0.1` or `::1` is not reachable from peers even if DNS-SD advertises it.

### 2. Verify Local Publication

```bash
avahi-browse --resolve --terminate _http._tcp
```

On macOS:

```bash
dns-sd -B _http._tcp local.
```

If the publisher cannot see its own instance, inspect registration syntax and daemon logs.

### 3. Browse from a Second Host

Use a client on the same link. If local browsing works but the second host sees nothing, check multicast, firewall, and network isolation.

### 4. Capture on Both Hosts

```bash
sudo tcpdump -i eth0 -enn -vv 'udp port 5353'
```

Determine whether the browse query reaches the publisher and whether its response reaches the browser.

### 5. Resolve the Selected Instance

```bash
dns-sd -L "Lab Dashboard" _http._tcp local.
resolvectl service "Lab Dashboard" _http._tcp local
```

Confirm the `SRV` target, port, TXT metadata, and target addresses.

### 6. Test the Endpoint Directly

```bash
curl http://labhost.local:8080/status
```

Use the actual application client for non-HTTP service types.

### 7. Check Network Scope

If the hosts are on different VLANs, mDNS is not expected to cross the router. Check for an intentional discovery proxy, reflector, or wide-area DNS-SD deployment.

### 8. Compare Interface and Address Family

An instance may be published on the wrong interface or resolve only to an unusable IPv6 or IPv4 address. Avahi and `dns-sd` output includes the discovery interface.

---

## Common Problems

### The Instance Appears but Connections Fail

Common causes:

* The application is not running.
* The advertised port is wrong.
* The server is bound only to loopback.
* The `SRV` target resolves to a stale or unreachable address.
* A host firewall blocks the application port.
* The client is using a protocol different from the advertised service type.

### Browsing Works Only on the Publisher

The registration exists locally, but multicast is filtered or isolated. Check host firewalls, Wi-Fi client isolation, virtual networks, containers, and switch multicast behavior.

### The Service Is Missing from an All-Types Browse

Some implementations or devices may not correctly publish the `_services._dns-sd._udp` meta-query record. Browse the known service type directly before concluding the instance is absent.

### The Instance Has a Numeric Suffix

A name conflict occurred. A responder renamed the instance or host after probing found the original name already in use.

### TXT Values Are Missing or Truncated

Check shell quoting, per-string size, total record size, duplicate keys, XML escaping, and whether the client understands the service type's TXT profile.

### Wide-Area Browsing Returns Nothing

Verify the browsing domain, the type `PTR`, the instance `SRV` and `TXT`, the target addresses, authoritative DNS responses, DNSSEC state where used, and domain-enumeration configuration.

### It Works on One Subnet but Not Another

That is normal for mDNS-backed DNS-SD. Use wide-area DNS-SD or an explicitly designed cross-subnet discovery mechanism.

---

## Security and Privacy

* Local DNS-SD advertisements are visible to other participants on the link.
* Instance names, host names, service types, and TXT data can reveal device roles, user names, software, and capabilities.
* Discovery does not authenticate a service. A malicious host can advertise a convincing instance.
* Authenticate and encrypt the application connection independently.
* Treat all TXT values, host names, ports, and addresses as untrusted input.
* Never place credentials or secrets in instance names or TXT records.
* Reflectors and discovery proxies expose advertisements to additional links and should enforce deliberate policy.
* Operating systems may require explicit local-network permission for applications that browse or register services.

DNSSEC can authenticate wide-area DNS-SD records when clients validate them, but it does not replace authentication and authorization in the discovered application protocol.

---

## Avoid These Mistakes

* Do not confuse the instance name with the target host name.
* Do not omit the browse `PTR`, instance `SRV`, instance `TXT`, or target address records.
* Do not point an `SRV` target at a `CNAME`.
* Do not duplicate the target host or port in `TXT` metadata.
* Do not assume `_tcp` means TLS or `_udp` means the DNS query used UDP.
* Do not assume the service uses its conventionally assigned port; use the `SRV` port.
* Do not publish a service before the application is reachable.
* Do not expect local DNS-SD to cross routers by default.
* Do not invent a new service type when an interoperable registered type exists.
* Do not treat discovered records as trusted configuration.

---

## References

* [RFC 6763: DNS-Based Service Discovery](https://www.rfc-editor.org/rfc/rfc6763.html)
* [RFC 6762: Multicast DNS](https://www.rfc-editor.org/rfc/rfc6762.html)
* [RFC 2782: A DNS RR for Specifying the Location of Services](https://www.rfc-editor.org/rfc/rfc2782.html)
* [RFC 8882: DNS-SD Privacy and Security Requirements](https://www.rfc-editor.org/rfc/rfc8882.html)
* [IANA Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
* [Avahi command documentation](https://github.com/avahi/avahi/tree/master/man)
* [Apple DNS Service Discovery Programming Guide](https://developer.apple.com/library/archive/documentation/Networking/Conceptual/dns_discovery_api/Introduction.html)
