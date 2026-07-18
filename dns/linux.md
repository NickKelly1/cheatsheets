# Linux Name Resolution Cheatsheet

Name resolution on Linux is a pipeline, not a single DNS lookup. This guide
focuses on Debian 13 and Ubuntu 24.04/26.04 LTS.

Last checked: 2026-07-18.

## Table of Contents

- [Practical exercise](#practical-exercise)
- [Resolution path](#resolution-path)
- [Three tools, three questions](#three-tools-three-questions)
- [NSS and `/etc/nsswitch.conf`](#nss-and-etcnsswitchconf)
- [`getent`](#getent)
- [Resolver files and ownership](#resolver-files-and-ownership)
- [`systemd-resolved`](#systemd-resolved)
- [`resolvectl`](#resolvectl)
- [`systemd-networkd` and `networkctl`](#systemd-networkd-and-networkctl)
- [NetworkManager and `nmcli`](#networkmanager-and-nmcli)
- [Netplan on Ubuntu](#netplan-on-ubuntu)
- [Debian and Ubuntu differences](#debian-and-ubuntu-differences)
- [Search domains and split DNS](#search-domains-and-split-dns)
- [Avahi and multicast DNS](#avahi-and-multicast-dns)
- [Name resolution for libvirt VMs](#name-resolution-for-libvirt-vms)
- [Applications that bypass NSS](#applications-that-bypass-nss)
- [Caches](#caches)
- [Troubleshooting workflow](#troubleshooting-workflow)
- [Common symptoms](#common-symptoms)
- [Safe change workflow](#safe-change-workflow)
- [References](#references)

---

## Practical exercise

Compare the system resolver with a direct DNS query. All commands are
read-only:

```bash
NAME='example.com'

getent ahosts "$NAME"
resolvectl query "$NAME"
dig "$NAME" A +noall +answer

rg '^hosts:' /etc/nsswitch.conf
readlink -f /etc/resolv.conf
resolvectl status
```

If `resolvectl` is unavailable or reports that `systemd-resolved` is not
running, that observation is part of the result; continue with `getent` and
`dig`.

```text
application-like lookup                    DNS-only diagnostic
getent ─► glibc/NSS ─┬─► /etc/hosts        dig ───────────────► chosen DNS server
                     ├─► systemd-resolved
                     ├─► unicast DNS
                     ├─► Avahi/mDNS
                     └─► libvirt NSS
```

| Compare | What it proves |
| --- | --- |
| `getent` succeeds, `dig` fails | A non-DNS NSS source, such as `/etc/hosts`, may have answered. |
| `dig` succeeds, `getent` fails | DNS works, but the application's NSS path, address family, or policy does not. |
| `resolvectl` and `dig` use different servers | `dig` may be bypassing per-link routing or the local stub. |
| All three agree | The common path is healthy; an affected application may have its own cache or resolver. |

Repeat with `localhost`, the output of `hostname`, and a name from an internal
domain. The differences are more useful than a single successful answer.

Addresses in `192.0.2.0/24` and `2001:db8::/32` below are documentation-only.
Replace them before making a configuration change.

---

## Resolution path

Most dynamically linked applications call `getaddrinfo(3)`. glibc then follows
the `hosts:` database in `/etc/nsswitch.conf`.

```text
┌─────────────┐   getaddrinfo()   ┌───────────────────────────────┐
│ application ├──────────────────►│ glibc Name Service Switch     │
└─────────────┘                   │ `/etc/nsswitch.conf`: hosts:  │
                                  └──────┬────────────────────────┘
                                         │ in configured order
          ┌──────────────┬───────────────┼───────────────┬──────────────┐
          ▼              ▼               ▼               ▼              ▼
     `/etc/hosts`   systemd-resolved   nss_dns       Avahi/mDNS   libvirt leases
       `files`         `resolve`         `dns`       `mdns*`      `libvirt*`
```

The path can include sources that are not DNS:

| NSS service | Typical source |
| --- | --- |
| `files` | `/etc/hosts` |
| `resolve` | `systemd-resolved` over its NSS interface |
| `dns` | DNS servers described by `/etc/resolv.conf` |
| `mdns`, `mdns4_minimal` | Avahi multicast DNS, commonly for `.local` |
| `myhostname` | Local hostname and selected local addresses |
| `mymachines` | Local systemd container names |
| `libvirt`, `libvirt_guest` | libvirt dnsmasq leases for NATed guests |

NSS does not force every program to use this path. Browsers, VPN clients,
containers, statically linked programs, and applications using libraries such
as c-ares may perform their own DNS queries.

---

## Three tools, three questions

| Tool | Question answered | Sources consulted |
| --- | --- | --- |
| `getent` | What will an NSS-aware application probably receive? | The configured `hosts:` NSS chain |
| `resolvectl` | What does `systemd-resolved` know and which link will it use? | resolved's cache, policy, and per-link DNS scopes |
| `dig` | What does this DNS server return for this record type? | DNS only; no `/etc/hosts` or general NSS walk |

`ping`, `ssh`, and `curl` are application tests, not clean DNS diagnostics.
They may select one address, apply search suffixes, use a proxy, or use a
resolver library different from glibc.

---

## NSS and `/etc/nsswitch.conf`

Inspect the active rule rather than copying one from another machine:

```bash
rg '^[[:space:]]*hosts:' /etc/nsswitch.conf
getent -s files hosts localhost
```

Illustrative configurations include:

```text
# Traditional files followed by DNS
hosts: files dns

# Avahi handles `.local` before conventional DNS
hosts: files mdns4_minimal [NOTFOUND=return] dns

# systemd-resolved is the main resolver service
hosts: mymachines resolve [!UNAVAIL=return] files myhostname dns
```

Do not replace the local `hosts:` line blindly. Packages and the distribution
may have installed modules needed for containers, mDNS, LDAP, or libvirt.

### NSS status actions

An NSS module returns a status, and an optional bracket expression changes what
happens next:

| Status | Meaning |
| --- | --- |
| `SUCCESS` | A result was found. |
| `NOTFOUND` | The source was reached but has no matching name. |
| `UNAVAIL` | The source or module is unavailable. |
| `TRYAGAIN` | A temporary failure occurred. |

```text
[NOTFOUND=return]     stop when this module says the name does not exist
[!UNAVAIL=return]    stop for every result except an unavailable module
```

Ordering and these actions explain why adding one module can make later
sources unreachable.

### Confirm installed modules

```bash
find /lib /usr/lib -name 'libnss_*.so.2' 2>/dev/null | sort
dpkg-query -W 'libnss-*' 2>/dev/null
```

---

## `getent`

`getent` asks a glibc NSS database. For host troubleshooting, prefer `ahosts`
when you want all socket-ready addresses.

```bash
getent hosts example.com
getent ahosts example.com
getent ahostsv4 example.com
getent ahostsv6 example.com

getent hosts 192.0.2.10
getent hosts "$(hostname)"
```

Restrict a lookup to one NSS service when supported by the installed glibc:

```bash
getent -s files hosts localhost
getent -s dns ahosts example.com
getent -s resolve ahosts example.com
```

| Command | Important behavior |
| --- | --- |
| `getent hosts` | Uses the historical `hosts` database and may present addresses differently. |
| `getent ahosts` | Calls `getaddrinfo()` and includes socket types. |
| `getent ahostsv4` | Limits results to IPv4. |
| `getent ahostsv6` | Limits results to IPv6. |
| `getent -s SERVICE ...` | Isolates one NSS module instead of following the full chain. |

`getent` does not show DNS TTLs, authoritative servers, DNSSEC records, or a
full DNS response. Use `dig` for those.

---

## Resolver files and ownership

Inspect these files and their owners before changing anything:

```text
/etc/hostname
/etc/hosts
/etc/nsswitch.conf
/etc/resolv.conf
/etc/systemd/resolved.conf
/etc/systemd/resolved.conf.d/*.conf
/etc/systemd/network/*.network
/etc/netplan/*.yaml
/etc/NetworkManager/system-connections/*
/etc/avahi/avahi-daemon.conf
```

### `/etc/hosts`

`/etc/hosts` is normally consumed by the `files` NSS module:

```text
127.0.0.1       localhost
127.0.1.1       workstation
192.0.2.20      app.example.test app
```

Test it through NSS:

```bash
getent hosts workstation
getent hosts app.example.test
```

`dig app.example.test` will not read `/etc/hosts`.

### `/etc/resolv.conf`

Determine whether it is generated:

```bash
ls -l /etc/resolv.conf
readlink -f /etc/resolv.conf
sed -n '1,120p' /etc/resolv.conf
```

Common systemd-resolved targets:

| Target | Effect for programs reading `/etc/resolv.conf` |
| --- | --- |
| `/run/systemd/resolve/stub-resolv.conf` | Sends queries to the local stub at `127.0.0.53`. |
| `/run/systemd/resolve/resolv.conf` | Lists known upstream servers and bypasses the local stub. |
| A regular file | Is managed statically or by another network component. |

Typical directives:

```text
nameserver 127.0.0.53
search corp.example test.example
options edns0 trust-ad
```

Do not edit a generated `/etc/resolv.conf`; change the component that owns it.

---

## `systemd-resolved`

`systemd-resolved` provides a cache, a local stub listener, an NSS interface,
and per-link DNS routing. NetworkManager, Netplan, or `systemd-networkd` can
feed DNS servers and domains into it.

```text
DHCP / static network config / VPN
                 │ per-link DNS servers + domains
                 ▼
         ┌──────────────────┐
NSS ────►│ systemd-resolved │────► best matching DNS scope
stub ───►│ cache + policy   │      Wi-Fi / Ethernet / VPN
         └──────────────────┘
```

Inspect the service and sockets:

```bash
systemctl status systemd-resolved --no-pager
resolvectl status
ss -lntup '( sport = :53 )'
journalctl -u systemd-resolved --since today
```

Configuration precedence follows systemd conventions: vendor files are lower
priority than local drop-ins under `/etc/systemd/resolved.conf.d/`.

Example global drop-in:

```ini
# /etc/systemd/resolved.conf.d/20-dns.conf
[Resolve]
DNS=192.0.2.53
FallbackDNS=2001:db8::53
Domains=~.
```

`~.` makes the configured scope a preferred route for all DNS names. Do not
deploy it casually on a machine that also needs VPN or internal split DNS.

Validate the effective configuration before applying it:

```bash
systemd-analyze cat-config systemd/resolved.conf
sudo systemctl reload-or-restart systemd-resolved
resolvectl status
```

Ubuntu 24.04 may restart the service because its systemd-resolved unit does not
support an in-place reload; newer units can reload. Either operation can briefly
interrupt lookups.

---

## `resolvectl`

### Inspect state

```bash
resolvectl status
resolvectl status eth0
resolvectl dns
resolvectl domain
resolvectl default-route
resolvectl statistics
```

### Query through resolved

```bash
resolvectl query example.com
resolvectl query --type=MX example.com
resolvectl query 192.0.2.10
resolvectl service _https._tcp example.com
```

Constrain the query when debugging a multi-link host:

```bash
resolvectl --interface=eth0 query app.corp.example
resolvectl --protocol=dns query example.com
resolvectl --protocol=mdns query printer.local
```

### Observe and clear state

```bash
resolvectl monitor
sudo resolvectl flush-caches
sudo resolvectl reset-statistics
sudo resolvectl reset-server-features
```

### Temporary per-link configuration

```bash
sudo resolvectl dns eth0 192.0.2.53 2001:db8::53
sudo resolvectl domain eth0 '~corp.example' 'example.test'
sudo resolvectl default-route eth0 yes

# Remove all runtime settings assigned with resolvectl for this link
sudo resolvectl revert eth0
```

Runtime settings disappear when the link or service is recreated. Make lasting
changes in NetworkManager, Netplan, or a `.network` file.

Some subcommands, such as `dnsovertls` and `dnssec`, depend on the installed
systemd version. Confirm with `resolvectl --help` before documenting automation.

---

## `systemd-networkd` and `networkctl`

`networkctl` inspects and controls links managed by `systemd-networkd`. It does
not perform name lookups; pair it with `resolvectl`.

```bash
systemctl is-active systemd-networkd
networkctl list
networkctl status
networkctl status eth0
networkctl status --all
```

Look for DNS servers, search domains, DHCP state, routes, and which `.network`
file matched the interface.

Example persistent configuration:

```ini
# /etc/systemd/network/20-wired.network
[Match]
Name=eth0

[Network]
DHCP=yes
DNS=192.0.2.53
Domains=example.test ~corp.example
DNSDefaultRoute=yes

[DHCPv4]
UseDNS=no
```

`example.test` is both a search suffix and routing domain. `~corp.example` is
routing-only and is not appended to single-label names.

Apply only from a console or with a rollback path:

```bash
sudo networkctl reload
sudo networkctl reconfigure eth0
resolvectl status eth0
```

Reconfiguration may drop addresses, routes, and the management connection.

---

## NetworkManager and `nmcli`

Confirm that NetworkManager owns the interface:

```bash
systemctl is-active NetworkManager
nmcli device status
nmcli connection show --active
nmcli device show eth0 | rg 'IP[46]\.(DNS|DOMAIN)'
```

Inspect one profile:

```bash
CON='Wired connection 1'
nmcli connection show "$CON" | rg 'ipv[46]\.(dns|dns-search|ignore-auto-dns)'
```

Example persistent IPv4 DNS configuration:

```bash
CON='Wired connection 1'
sudo nmcli connection modify "$CON" \
  ipv4.ignore-auto-dns yes \
  ipv4.dns '192.0.2.53 192.0.2.54' \
  ipv4.dns-search 'example.test ~corp.example'

sudo nmcli connection up "$CON"
resolvectl status
```

Record the current profile first:

```bash
nmcli connection show "$CON"
```

Bringing a profile up can interrupt connectivity. Use `nmcli device reapply`
only when the installed NetworkManager supports applying the changed property
without a reconnect.

---

## Netplan on Ubuntu

Netplan renders YAML into NetworkManager or `systemd-networkd` configuration.
Find the renderer and generated state before editing:

```bash
sudo netplan get
rg 'renderer:|nameservers:|use-dns:' /etc/netplan
networkctl status 2>/dev/null || true
nmcli device status 2>/dev/null || true
```

Example using DHCP for addressing but static DNS:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false
      nameservers:
        addresses: [192.0.2.53, 2001:db8::53]
        search: [example.test, "~corp.example"]
```

Validate with automatic rollback:

```bash
sudo netplan generate
sudo netplan try
resolvectl status eth0
```

Prefer `netplan try` over an unattended `netplan apply` on a remote host.

---

## Debian and Ubuntu differences

| Area | Debian 13 | Ubuntu 24.04/26.04 LTS |
| --- | --- | --- |
| `systemd-resolved` | Separate, optional package; not the default resolver | Normally installed and integrated by default |
| `/etc/resolv.conf` | May be static or managed by several possible components | Normally links to the resolved stub file |
| Network configuration | ifupdown, NetworkManager, or networkd are all common | Netplan renders to NetworkManager or networkd |
| Recommended first check | Identify the actual owner and running services | Inspect Netplan renderer, resolved, and per-link state |

Identify the machine instead of assuming its defaults:

```bash
. /etc/os-release
printf '%s %s\n' "$ID" "$VERSION_ID"

dpkg-query -W systemd-resolved resolvconf network-manager 2>/dev/null
systemctl is-active systemd-resolved NetworkManager systemd-networkd
ls -l /etc/resolv.conf
```

On Debian 13, installing `systemd-resolved` deliberately transfers control of
`/etc/resolv.conf`; plan the transition rather than installing it merely to
obtain `resolvectl`.

On Ubuntu, edit Netplan or the owning NetworkManager connection instead of
making a temporary edit beneath `/run/systemd/resolve/`.

---

## Search domains and split DNS

A search domain expands a single-label query:

```text
search corp.example

ssh database
      └────────► database.corp.example
```

A route-only domain selects a DNS scope without changing the queried name:

```text
VPN link:       DNS=10.20.0.53   Domains=~corp.example
Wi-Fi link:     DNS=192.0.2.53   Domains=~.

db.corp.example ─► VPN DNS
www.example.com ─► Wi-Fi DNS
```

Inspect both concepts separately:

```bash
resolvectl domain
resolvectl dns
resolvectl default-route
```

Important rules:

- `example.test` is a search domain and a routing domain.
- `~example.test` is routing-only.
- `~.` is the best possible match for every DNS name.
- The most specific matching route wins; equal matches can query multiple links.
- Search expansion can leak short internal names to unintended resolvers.
- `.local` is reserved for mDNS and is a poor choice for a private unicast zone.

---

## Avahi and multicast DNS

Avahi commonly provides mDNS on Debian and Ubuntu. Three paths can produce
different results:

```bash
NAME='printer.local'

avahi-resolve-host-name "$NAME"
resolvectl --protocol=mdns query "$NAME"
getent ahosts "$NAME"
```

```text
direct Avahi tool ─────────────────────────► avahi-daemon
resolvectl --protocol=mdns ────────────────► systemd-resolved mDNS
getent ─► NSS (`mdns*` or `resolve`) ──────► whichever stack NSS selects
```

Inspect integration:

```bash
systemctl status avahi-daemon --no-pager
avahi-daemon --check
resolvectl mdns
rg '^hosts:' /etc/nsswitch.conf
dpkg-query -W avahi-daemon avahi-utils libnss-mdns 2>/dev/null
```

If direct Avahi lookup works but `getent` does not, debug NSS ordering and the
installed `libnss-mdns` module rather than multicast packets first.

See [mds.md](mds.md) for the complete mDNS guide and [sd.md](sd.md) for DNS
service discovery.

---

## Name resolution for libvirt VMs

### Guest-to-network resolution

A libvirt NAT network normally runs a dedicated dnsmasq instance. DHCP gives
the guest the bridge address as its DNS server:

```text
VM application ─► guest NSS ─► guest `/etc/resolv.conf`
                                      │ DNS from DHCP
                                      ▼
                             192.168.122.1 / dnsmasq
                                ├─► DHCP lease names
                                ├─► configured local records
                                └─► upstream DNS
```

Inspect the default network from the host:

```bash
virsh net-info default
virsh net-dumpxml default
virsh net-dhcp-leases default
ps -ef | rg '[d]nsmasq.*libvirt'
```

Inspect one guest:

```bash
GUEST='vm1'
virsh domiflist "$GUEST"
virsh domifaddr "$GUEST" --source lease
```

Inside the guest:

```bash
ip route
cat /etc/resolv.conf
getent hosts host.example.test
```

### Host-to-guest resolution with NSS

Debian and Ubuntu package libvirt's NSS modules as `libnss-libvirt`:

```bash
sudo apt update
sudo apt install libnss-libvirt
```

Merge the modules into the existing `hosts:` line; preserve other services:

```text
hosts: files libvirt libvirt_guest resolve [!UNAVAIL=return] dns
```

Test through NSS:

```bash
getent hosts vm1
getent ahosts vm1
LIBVIRT_NSS_DEBUG=1 getent hosts vm1
```

| Module | Name source |
| --- | --- |
| `libvirt` | Hostname the guest reported while obtaining a DHCP lease |
| `libvirt_guest` | libvirt domain name mapped to its leased address |

The modules inspect leases from libvirt-managed NAT networks. They do not make
bridged guests, static addresses, user-mode networking, or arbitrary remote
hypervisors automatically resolvable.

### Network DNS records

Libvirt network XML can provide a local domain and explicit records:

```xml
<network>
  <name>lab</name>
  <domain name='lab.example' localOnly='yes'/>
  <dns>
    <host ip='192.168.122.20'>
      <hostname>db.lab.example</hostname>
      <hostname>db</hostname>
    </host>
  </dns>
</network>
```

Before editing a network, save its XML and note that restarting the network
disconnects attached guests:

```bash
virsh net-dumpxml default
virsh net-list --all
```

Use `virsh net-edit` only with a maintenance and rollback plan. Validate from a
guest with `getent` or `dig` against the bridge address, then test the host NSS
path separately.

### Common libvirt misunderstandings

- A guest hostname in a DHCP lease is not automatically published to LAN DNS.
- The host's `/etc/resolv.conf` does not normally point at every libvirt dnsmasq.
- `dig vm1` bypasses the libvirt NSS module; `getent hosts vm1` exercises it.
- A VM using a static address may have no lease for the NSS module to inspect.
- Changing the network XML does not update external authoritative DNS.

---

## Applications that bypass NSS

Possible alternate paths include:

- Browser or application DNS over HTTPS.
- c-ares or another asynchronous resolver library.
- A statically linked binary with its own resolver implementation.
- A container-specific `/etc/resolv.conf` and embedded DNS proxy.
- A language runtime configured to prefer its native resolver.
- A proxy that performs resolution on a remote host.

Useful observations:

```bash
getent ahosts example.com
strace -f -e trace=file,network getent ahosts example.com 2>&1
ss -ntup | rg ':53|:853|:443'
sudo tcpdump -ni any 'port 53 or port 853'
```

Encrypted DNS on port 443 cannot be identified reliably by port alone. Inspect
the application's settings and logs before concluding that no query occurred.

---

## Caches

Potential caches exist at several layers:

```text
application → language runtime → NSS daemon → systemd-resolved → recursive resolver
```

Inspect before flushing:

```bash
resolvectl statistics
systemctl is-active systemd-resolved nscd sssd dnsmasq unbound
```

Targeted cache operations:

```bash
sudo resolvectl flush-caches
sudo nscd --invalidate=hosts
sudo unbound-control flush_zone example.com
```

Only run the command for the cache actually in use. Restarting networking or a
resolver can interrupt unrelated clients and does not clear browser caches.

---

## Troubleshooting workflow

### 1. Capture the exact input and application error

```bash
NAME='app.corp.example'
printf '<%s>\n' "$NAME"
```

Check for spelling, Unicode lookalikes, a missing trailing dot, and whether the
application supplied a port or URL instead of a hostname.

### 2. Test the application-like NSS path

```bash
getent ahosts "$NAME"
getent ahostsv4 "$NAME"
getent ahostsv6 "$NAME"
rg '^hosts:' /etc/nsswitch.conf
```

### 3. Identify resolver ownership

```bash
readlink -f /etc/resolv.conf
cat /etc/resolv.conf
systemctl is-active systemd-resolved NetworkManager systemd-networkd
resolvectl status
```

### 4. Compare resolved with direct DNS

```bash
resolvectl query "$NAME"
dig "$NAME" A +noall +answer +comments
dig "$NAME" AAAA +noall +answer +comments
```

Query a known server only to isolate the configured resolver; it is not a
universal fix:

```bash
dig @192.0.2.53 "$NAME" A
```

### 5. Inspect the chosen link and split-DNS policy

```bash
resolvectl dns
resolvectl domain
resolvectl default-route
networkctl status 2>/dev/null || true
nmcli device show 2>/dev/null | rg 'IP[46]\.(DNS|DOMAIN)' || true
```

### 6. Check services, sockets, and logs

```bash
ss -lntup '( sport = :53 )'
journalctl -u systemd-resolved --since '-10 minutes'
journalctl -u NetworkManager --since '-10 minutes'
journalctl -u avahi-daemon --since '-10 minutes'
```

### 7. Capture only after selecting an interface

```bash
sudo tcpdump -ni eth0 -vv 'udp port 53 or tcp port 53'
sudo tcpdump -ni eth0 -vv 'udp port 5353'
```

No port-53 packet can be normal when a local cache answers. A packet on another
interface can be normal when split DNS selects a VPN.

### 8. Test from another host

This separates a client-specific NSS problem from an upstream DNS or zone
problem. Record the resolver, interface, address family, and exact query on
both systems.

---

## Common symptoms

| Symptom | Likely layer | First comparison |
| --- | --- | --- |
| `dig` works; browser fails | Application cache, proxy, address selection, or browser DoH | `getent`, browser DNS settings, `curl -v` |
| `getent` works; `dig` fails | `/etc/hosts`, mDNS, libvirt, or another NSS source | `getent -s SERVICE` and `hosts:` order |
| `resolvectl` works; `getent` fails | Missing or misordered `resolve` NSS module | `getent -s resolve` and package state |
| Public names work; private names fail | Split-DNS domain or VPN routing | `resolvectl domain` and `dns` |
| Short name fails; FQDN works | Missing search suffix or excessive `ndots` behavior | `resolvectl domain` and `/etc/resolv.conf` |
| `.local` works with Avahi only | NSS or resolved mDNS integration | `avahi-resolve`, `getent`, `hosts:` |
| VM has DNS; host cannot resolve VM | Guest DHCP DNS and host NSS are separate paths | leases, `libnss-libvirt`, `getent` |
| IPv4 works; application waits | Unusable IPv6 answer or route | `getent ahostsv4/ahostsv6`, `ip -6 route` |
| Changes vanish after reconnect | Runtime configuration changed at the wrong layer | Identify NetworkManager, Netplan, or networkd owner |

---

## Safe change workflow

```text
observe ─► identify owner ─► save current state ─► change source config
   ▲                                                   │
   └──────── verify NSS + resolver + application ◄─────┘
```

1. Keep a console or second session open.
2. Save `/etc/resolv.conf` ownership, the active connection, and per-link DNS.
3. Change the persistent owner, not a generated file.
4. Prefer `netplan try` or another automatic rollback mechanism.
5. Verify `getent`, `resolvectl`, `dig`, and the real application.
6. Revert immediately if routing, search domains, or management connectivity changes unexpectedly.

---

## References

- [Debian Reference: hostname resolution](https://www.debian.org/doc/manuals/debian-reference/ch05.en.html#_the_hostname_resolution)
- [Debian 13 release notes](https://www.debian.org/releases/trixie/release-notes/)
- [Ubuntu Server: configuring networks and name resolution](https://ubuntu.com/server/docs/explanation/networking/configuring-networks/)
- [`systemd-resolved`](https://www.freedesktop.org/software/systemd/man/systemd-resolved.service.html)
- [`resolvectl`](https://www.freedesktop.org/software/systemd/man/resolvectl.html)
- [`networkctl`](https://www.freedesktop.org/software/systemd/man/networkctl.html)
- [libvirt NSS module](https://libvirt.org/nss.html)
- [libvirt virtual network XML](https://libvirt.org/formatnetwork.html)
- [Avahi documentation](https://github.com/avahi/avahi/tree/master/man)
