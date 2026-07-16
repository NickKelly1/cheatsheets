# Nmap Cheatsheet

Nmap discovers hosts, scans ports, identifies services and operating systems,
and runs selected checks through the Nmap Scripting Engine (NSE).

Last checked: 2026-07-16. Examples target Nmap 7.99.

Only scan systems you own or have explicit permission to test. Define the
allowed addresses, ports, methods, rate, and maintenance window before running
a scan.

## Table of Contents

- [Safety and scope](#safety-and-scope)
- [Install and verify](#install-and-verify)
- [Example variables](#example-variables)
- [Command anatomy](#command-anatomy)
- [Target specification](#target-specification)
- [Host discovery](#host-discovery)
- [Port selection](#port-selection)
- [Scan techniques](#scan-techniques)
- [Privileges](#privileges)
- [Port states](#port-states)
- [Service and version detection](#service-and-version-detection)
- [OS detection and aggressive scan](#os-detection-and-aggressive-scan)
- [Nmap Scripting Engine](#nmap-scripting-engine)
- [Output](#output)
- [Timing and rate control](#timing-and-rate-control)
- [DNS, interfaces, and routes](#dns-interfaces-and-routes)
- [IPv6](#ipv6)
- [Practical scan workflows](#practical-scan-workflows)
- [Web and TLS inventory](#web-and-tls-inventory)
- [Firewall diagnostics](#firewall-diagnostics)
- [Runtime controls](#runtime-controls)
- [Automation and comparison](#automation-and-comparison)
- [Troubleshooting](#troubleshooting)
- [Quick reference](#quick-reference)
- [References](#references)

---

## Safety and scope

Before scanning:

1. Get written authorization.
2. Record the exact target ranges and exclusions.
3. Agree on allowed scan types, NSE scripts, rates, and hours.
4. Identify fragile systems and monitoring contacts.
5. Save the command line and machine-readable output.

Start with target validation and host discovery. Add port scanning, version
detection, OS detection, and NSE only when each is needed.

Nmap results describe what the scanner observed from one network location at
one time. Firewalls, load balancers, NAT, rate limiting, and packet loss can
change the result.

NSE labels such as `safe` reduce risk but are not guarantees. The official
documentation also considers the default script set potentially intrusive.
Review every selected script before using it in production.

The example addresses `192.0.2.0/24` and `2001:db8::/32` are reserved for
documentation. Replace them with an authorized scope before running commands.

---

## Install and verify

Debian or Ubuntu:

```bash
sudo apt update
sudo apt install -y nmap
```

Fedora:

```bash
sudo dnf install nmap
```

macOS with Homebrew:

```bash
brew install nmap
```

For Windows, use the official installer, which includes the Npcap packet
capture driver.

Verify the installation:

```bash
nmap --version
nmap --help
nmap --iflist
```

Service and OS fingerprints improve between releases. Check the installed
version when results look incomplete or inaccurate.

---

## Example variables

The examples use Bash variables to make the target obvious:

```bash
export TARGET='192.0.2.10'
export SUBNET='192.0.2.0/24'
export IPV6_TARGET='2001:db8::10'
export OUTDIR='./nmap-results'

mkdir -p "$OUTDIR"
```

Always quote target variables:

```bash
nmap "$TARGET"
```

Do not put a free-form, untrusted string into a shell variable and pass it to
Nmap. Target and option injection can expand the scope or alter the scan.

---

## Command anatomy

```text
nmap [scan technique] [discovery and scan options] [output options] target
```

Typical phases:

```text
Target expansion
    ↓
Host discovery
    ↓
Port scan
    ↓
Optional service, OS, traceroute, and NSE checks
    ↓
Human and machine-readable output
```

A plain scan:

```bash
nmap "$TARGET"
```

normally performs host discovery and scans the 1,000 most common TCP ports. A
privileged Unix user normally gets a TCP SYN scan; an unprivileged user gets a
TCP connect scan.

Show why Nmap assigned each state:

```bash
nmap --reason "$TARGET"
```

---

## Target specification

Single targets:

```bash
nmap '192.0.2.10'
nmap 'host.example.net'
```

When a hostname resolves to several addresses, Nmap scans the first address by
default. Scan every resolved address with:

```bash
nmap --resolve-all 'host.example.net'
```

Several targets:

```bash
nmap '192.0.2.10' '192.0.2.20' 'host.example.net'
```

CIDR and octet ranges:

```bash
nmap '192.0.2.0/24'
nmap '192.0.2.10-20'
```

Input file:

```text
# targets.txt
192.0.2.10
192.0.2.20
192.0.2.64/28
host.example.net
```

```bash
nmap -iL targets.txt
```

Exclude sensitive or out-of-scope systems:

```bash
nmap \
  --exclude '192.0.2.1,192.0.2.50' \
  "$SUBNET"

nmap \
  --excludefile excludes.txt \
  -iL targets.txt
```

Prevent overlapping target entries from scanning an address more than once:

```bash
nmap --unique -iL targets.txt
```

Preview the expanded scope without sending probes directly to targets:

```bash
nmap -sL -n -iL targets.txt --excludefile excludes.txt
```

`-sL` lists targets instead of scanning them. It performs reverse DNS by
default, so add `-n` when the preview must avoid DNS traffic too.

---

## Host discovery

Discover hosts without a port scan:

```bash
sudo nmap -sn "$SUBNET"
```

On a local Ethernet network, privileged Nmap normally uses ARP for IPv4 or
Neighbor Discovery for IPv6. Across routed networks, the privileged defaults
combine ICMP echo, TCP SYN to 443, TCP ACK to 80, and ICMP timestamp probes.
Unprivileged Unix discovery uses TCP connect probes to ports 80 and 443.

Explicit discovery methods:

| Option | Probe |
| --- | --- |
| `-PR` | ARP discovery on local IPv4 Ethernet |
| `-PE` | ICMP echo request |
| `-PP` | ICMP timestamp request |
| `-PS22,80,443` | TCP SYN discovery on selected ports |
| `-PA80,443` | TCP ACK discovery on selected ports |
| `-PU53,161` | UDP discovery on selected ports |

Examples:

```bash
sudo nmap -sn -PR "$SUBNET"
sudo nmap -sn -PE "$SUBNET"
sudo nmap -sn -PS22,80,443 "$SUBNET"
sudo nmap -sn -PE -PS22,443 -PA80 "$SUBNET"
```

There is no space between `-PS`, `-PA`, or `-PU` and its port list.

Skip discovery and treat every target as online:

```bash
sudo nmap -Pn "$TARGET"
```

Use `-Pn` when an authorized host blocks discovery probes. Avoid it on large,
sparsely populated ranges: Nmap will perform the requested scan against every
address and may take much longer.

On local Ethernet, Nmap may still use ARP or IPv6 Neighbor Discovery with
`-Pn`. Add `--disable-arp-ping` when you specifically need to suppress that
local discovery behavior.

Proxy ARP can make every local address appear online. Disable implicit ARP or
Neighbor Discovery when diagnosing that specific problem:

```bash
sudo nmap -sn --disable-arp-ping "$SUBNET"
```

---

## Port selection

| Option | Ports scanned |
| --- | --- |
| No port option | 1,000 most common ports for each scanned protocol |
| `-F` | 100 most common ports |
| `--top-ports 100` | Requested number of most common ports |
| `-p 22` | One port |
| `-p 22,80,443` | A list |
| `-p 1-1024` | A range |
| `-p-` | Ports 1 through 65535 |
| `-p http,https` | Ports by names in `nmap-services` |

Examples:

```bash
nmap -p 22,80,443 "$TARGET"
nmap --top-ports 100 "$TARGET"
nmap -p- "$TARGET"
```

Select different TCP and UDP ports in a combined scan:

```bash
sudo nmap \
  -sS -sU \
  -p 'T:22,80,443,U:53,123,161' \
  "$TARGET"
```

Exclude ports:

```bash
nmap --exclude-ports '9100,515' "$TARGET"
```

Scan ports sequentially instead of Nmap's usual randomized order:

```bash
nmap -r -p 1-1024 "$TARGET"
```

Quote complex port expressions so the shell cannot expand wildcard characters.

---

## Scan techniques

| Option | Technique | Notes |
| --- | --- | --- |
| `-sS` | TCP SYN | Fast, does not complete the TCP connection; needs raw-packet privileges |
| `-sT` | TCP connect | Uses the OS `connect()` call; works unprivileged and is more likely to be logged |
| `-sU` | UDP | Often slow; no response commonly means `open|filtered` |
| `-sA` | TCP ACK | Maps filtering; does not determine whether a port is open |

TCP SYN:

```bash
sudo nmap -sS -p 1-1024 --reason "$TARGET"
```

TCP connect:

```bash
nmap -sT -p 22,80,443 --reason "$TARGET"
```

UDP:

```bash
sudo nmap -sU --top-ports 50 --reason "$TARGET"
```

ACK firewall reachability check:

```bash
sudo nmap -sA -p 22,80,443 --reason "$TARGET"
```

Only one TCP scan technique can be active at a time. A TCP technique can be
combined with `-sU`.

This cheatsheet intentionally omits packet spoofing, decoys, idle scanning,
fragmentation, and IDS-evasion recipes.

---

## Privileges

On Unix-like systems, raw-packet scans such as `-sS`, `-sU`, `-sA`, and OS
detection normally require root privileges or explicitly granted network
capabilities.

Use `sudo` only for the Nmap command:

```bash
sudo nmap -sS "$TARGET"
```

Without raw-packet privileges, use TCP connect:

```bash
nmap -sT "$TARGET"
```

Force Nmap to behave as unprivileged when testing a command:

```bash
nmap --unprivileged -sT "$TARGET"
```

Privilege changes affect Nmap's default discovery and TCP scan technique, so
record whether the scan was privileged when comparing results.

On Windows, install and start Npcap. An elevated terminal is recommended for
raw-packet features.

---

## Port states

Nmap reports observations, not permanent properties:

| State | Meaning |
| --- | --- |
| `open` | An application accepted the probe |
| `closed` | The host responded, but nothing is listening |
| `filtered` | Filtering prevented Nmap from deciding whether the port is open |
| `unfiltered` | ACK probes reached the port, but `-sA` cannot tell open from closed |
| `open|filtered` | No response; common with UDP and some specialized scan types |
| `closed|filtered` | Nmap cannot distinguish closed from filtered |

Show the evidence used:

```bash
nmap --reason -p 22,80,443 "$TARGET"
```

Show only open or possibly open ports:

```bash
nmap --open "$TARGET"
```

`--open` is convenient for inventory, but hides closed and filtered results
that may be important when diagnosing a firewall.

A service name beside a port is initially inferred from the port number. It
does not prove that the named service is actually running; use `-sV`.

---

## Service and version detection

Probe discovered ports to identify the actual protocol, product, version, and
sometimes CPE:

```bash
sudo nmap -sS -sV -p 22,80,443 "$TARGET"
```

Control probe intensity:

| Option | Effect |
| --- | --- |
| `--version-light` | Intensity 2; faster, fewer probes |
| `--version-intensity 0-9` | Explicit intensity; default is 7 |
| `--version-all` | Intensity 9; try every probe |
| `--version-trace` | Debug version-detection activity |

Start light on production services:

```bash
sudo nmap \
  -sS \
  -sV --version-light \
  -p 22,80,443 \
  "$TARGET"
```

Increase intensity only when a service remains unidentified:

```bash
sudo nmap \
  -sS \
  -sV --version-intensity 7 \
  -p 8443 \
  "$TARGET"
```

Version detection sends application-level probes and can appear in service
logs. Reported versions may also be incomplete or misleading when vendors
backport fixes without changing version strings.

---

## OS detection and aggressive scan

Enable remote OS fingerprinting:

```bash
sudo nmap -O "$TARGET"
```

OS detection is most effective when Nmap finds at least one open and one closed
TCP port.

Limit attempts to promising targets:

```bash
sudo nmap -O --osscan-limit "$SUBNET"
```

Request less certain guesses:

```bash
sudo nmap -O --osscan-guess "$TARGET"
```

Treat guesses as hypotheses and check their confidence. NAT, proxies, virtual
networking, and unusual TCP/IP stacks can distort fingerprints.

Aggressive scan:

```bash
sudo nmap -A "$TARGET"
```

`-A` currently enables:

- OS detection: `-O`
- Service/version detection: `-sV`
- Default NSE scripts: `-sC`
- Traceroute: `--traceroute`

It does not enable a timing template. Because default NSE scripts can be
intrusive, prefer selecting each required feature explicitly.

---

## Nmap Scripting Engine

Inspect help before running a script:

```bash
nmap --script-help 'http-title'
nmap --script-help 'safe'
```

Run the default script set:

```bash
sudo nmap -sV -sC -p 22,80,443 "$TARGET"
```

`-sC` is equivalent to `--script=default`. Use it only when the scope permits
the default category.

Run named scripts:

```bash
sudo nmap \
  -sV \
  -p 22,80,443 \
  --script 'ssh-hostkey,http-title,http-headers,ssl-cert' \
  "$TARGET"
```

Run a category:

```bash
sudo nmap -sV --script 'safe' "$TARGET"
```

Relevant categories:

| Category | Purpose and caution |
| --- | --- |
| `safe` | Designed to avoid exploitation or heavy resource use; still review |
| `default` | Curated useful set; may include intrusive behavior |
| `discovery` | Collects additional host or service information |
| `broadcast` | Uses local broadcasts and may discover additional targets |
| `auth` | Authentication and access checks |
| `malware` | Looks for signs of malware or backdoors |
| `vuln` | Checks for known vulnerabilities |
| `intrusive` | Higher risk of disruption or privacy impact |
| `brute` | Credential guessing |
| `dos`, `exploit`, `fuzzer` | Potentially destructive or disruptive |
| `external` | Sends information to a third-party service |
| `version` | Extends `-sV`; Nmap selects these scripts automatically |

Do not run `brute`, `dos`, `exploit`, `fuzzer`, `intrusive`, or broad `vuln`
sets without specific authorization and operational review.

Pass script arguments:

```bash
nmap \
  --script 'SCRIPT_NAME' \
  --script-args 'name=value' \
  "$TARGET"
```

Read the script's NSEDoc page for accepted arguments and side effects.

Refresh the local script index after adding or removing scripts:

```bash
sudo nmap --script-updatedb
```

NSE scripts are not sandboxed. Only use scripts shipped by a trusted Nmap
package or third-party scripts you have audited.

---

## Output

Create the output directory first:

```bash
mkdir -p "$OUTDIR"
```

| Option | Output |
| --- | --- |
| `-oN FILE` | Normal human-readable file |
| `-oX FILE` | XML for programs and imports |
| `-oG FILE` | Deprecated grepable format |
| `-oA BASENAME` | Normal, XML, and grepable files |

Save all common formats:

```bash
sudo nmap \
  -sS \
  --reason \
  -oA "$OUTDIR/tcp-initial" \
  "$TARGET"
```

This creates:

```text
nmap-results/tcp-initial.nmap
nmap-results/tcp-initial.xml
nmap-results/tcp-initial.gnmap
```

Prefer XML over parsing normal or deprecated grepable output.

Write XML to standard output:

```bash
nmap -oX - "$TARGET"
```

Useful display controls:

| Option | Effect |
| --- | --- |
| `-v`, `-vv` | Increase verbosity |
| `-d`, `-dd` | Increase debugging detail |
| `--reason` | Explain host and port states |
| `--open` | Show only open or possibly open ports |
| `--packet-trace` | Show sent and received packets |

Use packet tracing only with a narrow target and port set:

```bash
sudo nmap \
  -sS \
  -p 443 \
  --packet-trace \
  "$TARGET"
```

Resume a scan from one of its saved output files:

```bash
sudo nmap --resume "$OUTDIR/tcp-full.nmap"
```

Do not add new scan arguments to `--resume`; Nmap restores them from the
output file.

`--append-output` appends instead of overwriting, but appended XML is generally
not a valid single XML document. Prefer unique output names.

---

## Timing and rate control

Timing templates:

| Option | Name | Use |
| --- | --- | --- |
| `-T2` | Polite | Very slow; reduces bandwidth and target load |
| `-T3` | Normal | Default; good conservative starting point |
| `-T4` | Aggressive | Fast and reliable networks |
| `-T5` | Insane | Sacrifices retries and accuracy; rarely appropriate |

Start with `-T3`:

```bash
sudo nmap -sS -T3 "$TARGET"
```

Use `-T4` only when the authorized network is modern, fast, and reliable:

```bash
sudo nmap -sS -T4 "$TARGET"
```

Cap packet rate to protect a sensitive network:

```bash
sudo nmap \
  -sS \
  --max-rate 100 \
  "$TARGET"
```

Other controls:

| Option | Purpose |
| --- | --- |
| `--min-rate N` | Try to send at least N packets per second |
| `--max-rate N` | Limit average packets per second |
| `--max-retries N` | Cap probe retransmissions |
| `--host-timeout 15m` | Give up on one host after the limit |
| `--script-timeout 2m` | Limit each NSE script instance |
| `--scan-delay 100ms` | Wait at least this long between probes to one host |

Time values accept `ms`, `s`, `m`, and `h` suffixes.

Higher minimum rates, fewer retries, short timeouts, and `-T5` can miss hosts
or ports. Narrowing the target and port set is usually safer than forcing a
higher rate.

---

## DNS, interfaces, and routes

Disable reverse DNS:

```bash
nmap -n "$TARGET"
```

Resolve every target, even if it appears offline:

```bash
nmap -R "$TARGET"
```

Scan every address returned for a hostname:

```bash
nmap --resolve-all 'host.example.net'
```

Use the operating system resolver:

```bash
nmap --system-dns 'host.example.net'
```

Specify approved DNS servers:

```bash
nmap --dns-servers '192.0.2.53,192.0.2.54' "$SUBNET"
```

Inspect Nmap's interface and route selection:

```bash
nmap --iflist
```

Select an interface when automatic routing chooses incorrectly:

```bash
sudo nmap -e 'eth0' -sS "$TARGET"
```

Trace the network path after scanning:

```bash
sudo nmap --traceroute "$TARGET"
```

Nmap traceroute is not supported with TCP connect scans.

---

## IPv6

Enable IPv6:

```bash
nmap -6 "$IPV6_TARGET"
```

IPv6 CIDR:

```bash
sudo nmap -6 -sn '2001:db8:1234::/64'
```

Version detection:

```bash
sudo nmap \
  -6 \
  -sS -sV \
  -p 22,80,443 \
  "$IPV6_TARGET"
```

Link-local addresses require a zone identifier:

```bash
sudo nmap -6 -sn 'fe80::1234%eth0'
```

Nmap uses IPv6 Neighbor Discovery for local Ethernet targets. Octet-range
syntax is IPv4-only.

---

## Practical scan workflows

### 1. Validate scope

```bash
nmap \
  -sL -n \
  -iL targets.txt \
  --excludefile excludes.txt
```

Review the complete list before sending probes.

### 2. Discover live hosts

```bash
sudo nmap \
  -sn -n \
  --reason \
  -iL targets.txt \
  --excludefile excludes.txt \
  -oA "$OUTDIR/discovery"
```

### 3. Initial TCP inventory

```bash
sudo nmap \
  -sS -T3 \
  --top-ports 1000 \
  --reason \
  --open \
  -oA "$OUTDIR/tcp-initial" \
  "$TARGET"
```

Remove `--open` when closed and filtered states matter.

### 4. Full TCP port scan

```bash
sudo nmap \
  -sS -T3 \
  -p- \
  --reason \
  -oA "$OUTDIR/tcp-full" \
  "$TARGET"
```

Run this only after the initial scan and within the approved window.

### 5. Targeted service follow-up

```bash
sudo nmap \
  -sS \
  -sV --version-light \
  -p 22,80,443 \
  --script 'ssh-hostkey,http-title,http-headers,ssl-cert' \
  --reason \
  -oA "$OUTDIR/service-followup" \
  "$TARGET"
```

Replace the port list with ports actually found during discovery.

### 6. Focused UDP inventory

```bash
sudo nmap \
  -sU -T3 \
  --top-ports 50 \
  -sV --version-light \
  --reason \
  -oA "$OUTDIR/udp-top50" \
  "$TARGET"
```

UDP scanning can take much longer because silence is ambiguous and ICMP errors
are often rate-limited.

### 7. Combined selected TCP and UDP ports

```bash
sudo nmap \
  -sS -sU \
  -p 'T:22,80,443,U:53,123,161' \
  -sV --version-light \
  --reason \
  -oA "$OUTDIR/tcp-udp-selected" \
  "$TARGET"
```

---

## Web and TLS inventory

Identify common web endpoints:

```bash
sudo nmap \
  -sS -sV \
  -p 80,443,8000,8080,8443 \
  --script 'http-title,http-headers,ssl-cert' \
  "$TARGET"
```

Inspect TLS protocol and cipher support:

```bash
sudo nmap \
  -sS -sV \
  -p 443,8443 \
  --script 'ssl-cert,ssl-enum-ciphers' \
  "$TARGET"
```

`ssl-enum-ciphers` makes many TLS connections and may be noisy. Confirm it is
allowed before use.

HTTP NSE scripts issue real application requests. They can affect access logs,
rate limits, and application monitoring even when categorized as safe.

---

## Firewall diagnostics

Show the response behind each decision:

```bash
sudo nmap \
  -sS \
  -p 22,80,443 \
  --reason \
  "$TARGET"
```

If discovery is blocked but the host is known to exist:

```bash
sudo nmap \
  -Pn \
  -sS \
  -p 22,80,443 \
  --reason \
  "$TARGET"
```

Map whether ACK probes are filtered:

```bash
sudo nmap \
  -sA \
  -p 22,80,443 \
  --reason \
  "$TARGET"
```

ACK results mean:

| State | Interpretation |
| --- | --- |
| `unfiltered` | The ACK probe reached the host and elicited a reset |
| `filtered` | No qualifying reply arrived, or an ICMP filtering error arrived |

An ACK scan does not tell whether a service is listening.

Compare authorized scans from inside and outside the firewall. A port can be
`open` from one vantage point and `filtered` from another.

Use a narrow packet trace when the reason remains unclear:

```bash
sudo nmap \
  -sS \
  -p 443 \
  --reason \
  --packet-trace \
  "$TARGET"
```

---

## Runtime controls

During an interactive scan:

| Key | Action |
| --- | --- |
| Any ordinary key | Print current status and estimated completion |
| `v` / `V` | Increase / decrease verbosity |
| `d` / `D` | Increase / decrease debugging |
| `p` / `P` | Enable / disable packet tracing |
| `?` | Show runtime help |
| `Ctrl+C` | Stop the scan |

Disable keyboard interaction for background automation:

```bash
nmap \
  --noninteractive \
  -oA "$OUTDIR/background-scan" \
  "$TARGET"
```

Always save output before starting a long scan so `--resume` is available.

---

## Automation and comparison

Use target and exclusion files rather than generating a long command:

```bash
sudo nmap \
  --noninteractive \
  -sS -T3 \
  -iL targets.txt \
  --excludefile excludes.txt \
  --reason \
  -oA "$OUTDIR/inventory-current"
```

Create timestamped output:

```bash
stamp="$(date -u +%Y%m%dT%H%M%SZ)"

sudo nmap \
  --noninteractive \
  -sS -T3 \
  -iL targets.txt \
  --excludefile excludes.txt \
  -oA "$OUTDIR/inventory-$stamp"
```

For programmatic processing, consume XML:

```bash
nmap -oX "$OUTDIR/scan.xml" "$TARGET"
```

Do not parse terminal-formatted output; it can change with verbosity, options,
and Nmap releases.

Compare two XML scans with `ndiff`:

```bash
ndiff \
  "$OUTDIR/baseline.xml" \
  "$OUTDIR/current.xml"
```

Changes are observations, not automatically incidents. Confirm changes from
the same scanner location and with the same Nmap version, privilege level,
targets, options, and timing.

---

## Troubleshooting

### Nmap reports zero hosts up

Check the target first:

```bash
nmap -sL -n "$TARGET"
```

If the host is known to exist and discovery probes are blocked:

```bash
sudo nmap -Pn -sS -p 22,80,443 --reason "$TARGET"
```

Use `-Pn` only for the known authorized target, not an unnecessarily large
range.

### The requested scan requires privileges

Use `sudo` for raw-packet features or switch to TCP connect:

```bash
nmap -sT -p 22,80,443 "$TARGET"
```

### Every port is filtered

Check:

- Correct source network and route.
- Host and perimeter firewalls.
- Cloud security groups or ACLs.
- VPN policy.
- Packet loss and rate limiting.
- Whether discovery or the chosen probe type is allowed.

Then retry a narrow port list with `--reason` from an approved vantage point.

### UDP ports remain `open|filtered`

This is expected when neither a UDP reply nor an ICMP error arrives. Add
targeted version detection:

```bash
sudo nmap \
  -sU -sV \
  -p 53,123,161 \
  --reason \
  "$TARGET"
```

Application-specific NSE scripts may help, but review them before use.

### The scan is too slow

Narrow the work first:

```bash
nmap -n --top-ports 100 "$TARGET"
```

Then consider:

- `--version-light` instead of full version detection.
- Fewer NSE scripts.
- `-T4` on a fast, reliable network.
- A sensible `--host-timeout`.
- Scanning TCP and UDP in separate passes.

Filtered ports and UDP rate limiting can legitimately take time.

### Results are inconsistent

Use conservative timing and restore retries:

```bash
sudo nmap -sS -T3 --reason "$TARGET"
```

Remove aggressive `--min-rate`, short timeouts, and low retry limits. Compare
from the same network location and privilege level.

### Local discovery says every address is up

Proxy ARP may be answering for unused addresses:

```bash
sudo nmap -sn --disable-arp-ping "$SUBNET"
```

Confirm behavior with the network administrator before changing the discovery
method.

### Names or addresses are surprising

```bash
nmap -sL -n "$TARGET"
nmap --resolve-all 'host.example.net'
nmap --iflist
```

Check DNS, address family, duplicate targets, routes, and interface selection.

### NSE script cannot be found

Inspect the installed data directory and refresh the script database:

```bash
nmap --version
sudo nmap --script-updatedb
nmap --script-help 'SCRIPT_NAME'
```

Reinstall or upgrade Nmap if its script and data files are incomplete.

---

## Quick reference

| Task | Command |
| --- | --- |
| Version | `nmap --version` |
| Interfaces and routes | `nmap --iflist` |
| Preview targets | `nmap -sL -n TARGETS` |
| Discover hosts only | `sudo nmap -sn SUBNET` |
| Skip discovery | `sudo nmap -Pn TARGET` |
| Default TCP scan | `nmap TARGET` |
| TCP SYN scan | `sudo nmap -sS TARGET` |
| TCP connect scan | `nmap -sT TARGET` |
| UDP scan | `sudo nmap -sU TARGET` |
| Selected ports | `nmap -p 22,80,443 TARGET` |
| Full port range | `nmap -p- TARGET` |
| Top 100 ports | `nmap --top-ports 100 TARGET` |
| Service detection | `nmap -sV TARGET` |
| Light version scan | `nmap -sV --version-light TARGET` |
| OS detection | `sudo nmap -O TARGET` |
| Default NSE scripts | `nmap -sC TARGET` |
| Named NSE scripts | `nmap --script 'NAME1,NAME2' TARGET` |
| Explain states | `nmap --reason TARGET` |
| Show open ports | `nmap --open TARGET` |
| Disable reverse DNS | `nmap -n TARGET` |
| IPv6 | `nmap -6 TARGET` |
| Save all formats | `nmap -oA BASENAME TARGET` |
| XML output | `nmap -oX FILE TARGET` |
| Resume | `nmap --resume OUTPUT_FILE` |
| Normal timing | `nmap -T3 TARGET` |
| Limit packet rate | `nmap --max-rate 100 TARGET` |

---

## References

- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [Nmap 7.99 download page](https://nmap.org/download.html)
- [Target specification](https://nmap.org/book/man-target-specification.html)
- [Host discovery](https://nmap.org/book/man-host-discovery.html)
- [Port scanning basics](https://nmap.org/book/man-port-scanning-basics.html)
- [Port scanning techniques](https://nmap.org/book/man-port-scanning-techniques.html)
- [Port specification and scan order](https://nmap.org/book/man-port-specification.html)
- [Service and version detection](https://nmap.org/book/man-version-detection.html)
- [OS detection](https://nmap.org/book/man-os-detection.html)
- [Nmap Scripting Engine](https://nmap.org/book/man-nse.html)
- [NSEDoc script reference](https://nmap.org/nsedoc/)
- [Timing and performance](https://nmap.org/book/man-performance.html)
- [Output formats](https://nmap.org/book/man-output.html)
