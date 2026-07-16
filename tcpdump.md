# tcpdump Cheatsheet

## Table of Contents

- [Command Structure](#command-structure)
- [Interfaces](#interfaces)
- [Common Options](#common-options)
- [Host Filters](#host-filters)
- [Network Filters](#network-filters)
- [Port Filters](#port-filters)
- [Protocol Filters](#protocol-filters)
- [Combining Filters](#combining-filters)
- [TCP Flags](#tcp-flags)
- [Packet Size](#packet-size)
- [Ethernet Filters](#ethernet-filters)
- [VLAN Traffic](#vlan-traffic)
- [Common Protocol Recipes](#common-protocol-recipes)
- [Capture Files](#capture-files)
- [Capture Rotation](#capture-rotation)
- [Remote Capture](#remote-capture)
- [Pipe-Friendly Output](#pipe-friendly-output)
- [Useful Diagnostic Commands](#useful-diagnostic-commands)
- [Common Gotchas](#common-gotchas)

---

## Command Structure

```bash
sudo tcpdump [options] '[filter]'
```

Useful default:

```bash
sudo tcpdump -i any -nn -tttt
```

Full-packet capture:

```bash
sudo tcpdump -i any -nn -s 0
```

---

## Interfaces

```bash
# List capture interfaces
tcpdump -D

# Capture on one interface
sudo tcpdump -i eth0

# Capture on all Linux interfaces
sudo tcpdump -i any

# Disable promiscuous mode
sudo tcpdump -p -i eth0
```

`any` uses Linux cooked capture, so Ethernet headers may not be available.

---

## Common Options

| Option              | Description                               |
| ------------------- | ----------------------------------------- |
| `-i eth0`           | Capture from interface                    |
| `-D`                | List interfaces                           |
| `-n`                | Do not resolve hostnames                  |
| `-nn`               | Do not resolve hostnames or service names |
| `-v`, `-vv`, `-vvv` | Increase verbosity                        |
| `-c 100`            | Stop after 100 packets                    |
| `-s 0`              | Capture complete packets                  |
| `-A`                | Print payload as ASCII                    |
| `-X`                | Print payload as hex and ASCII            |
| `-XX`               | Include link-layer headers in hex output  |
| `-e`                | Display link-layer headers                |
| `-q`                | Less verbose output                       |
| `-l`                | Line-buffered output                      |
| `-tttt`             | Human-readable timestamps                 |
| `-S`                | Show absolute TCP sequence numbers        |
| `-Q in`             | Capture inbound packets where supported   |
| `-Q out`            | Capture outbound packets where supported  |
| `-w file.pcap`      | Write packets to a file                   |
| `-r file.pcap`      | Read packets from a file                  |

---

## Host Filters

```bash
# Traffic to or from a host
sudo tcpdump -nn 'host 10.0.0.10'

# Source host
sudo tcpdump -nn 'src host 10.0.0.10'

# Destination host
sudo tcpdump -nn 'dst host 10.0.0.10'

# Exclude a host
sudo tcpdump -nn 'not host 10.0.0.10'
```

IPv6:

```bash
sudo tcpdump -nn 'ip6 and host 2001:db8::10'
```

---

## Network Filters

```bash
sudo tcpdump -nn 'net 10.0.0.0/24'
sudo tcpdump -nn 'src net 10.0.0.0/24'
sudo tcpdump -nn 'dst net 10.0.0.0/24'
```

---

## Port Filters

```bash
# Source or destination port
sudo tcpdump -nn 'port 443'

# Source port
sudo tcpdump -nn 'src port 443'

# Destination port
sudo tcpdump -nn 'dst port 443'

# Port range
sudo tcpdump -nn 'portrange 8000-9000'

# Multiple ports
sudo tcpdump -nn 'port 80 or port 443'

# Exclude a port
sudo tcpdump -nn 'not port 22'
```

---

## Protocol Filters

```bash
sudo tcpdump -nn 'tcp'
sudo tcpdump -nn 'udp'
sudo tcpdump -nn 'icmp'
sudo tcpdump -nn 'icmp6'
sudo tcpdump -nn 'arp'
sudo tcpdump -nn 'ip'
sudo tcpdump -nn 'ip6'
```

---

## Combining Filters

Use `and`, `or`, `not`, and parentheses.

```bash
sudo tcpdump -nn 'tcp and port 443'

sudo tcpdump -nn \
  'host 10.0.0.10 and (port 80 or port 443)'

sudo tcpdump -nn \
  'net 10.0.0.0/24 and not tcp port 22'

sudo tcpdump -nn \
  '(src host 10.0.0.10 and dst port 443) or icmp'
```

Quote the entire filter to prevent shell interpretation.

---

## TCP Flags

### SYN packets without ACK

```bash
sudo tcpdump -nn \
  'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'
```

### SYN-ACK packets

```bash
sudo tcpdump -nn \
  'tcp[tcpflags] & (tcp-syn|tcp-ack) == (tcp-syn|tcp-ack)'
```

### RST packets

```bash
sudo tcpdump -nn \
  'tcp[tcpflags] & tcp-rst != 0'
```

### FIN packets

```bash
sudo tcpdump -nn \
  'tcp[tcpflags] & tcp-fin != 0'
```

### PSH packets

```bash
sudo tcpdump -nn \
  'tcp[tcpflags] & tcp-push != 0'
```

### Any SYN, FIN, or RST

```bash
sudo tcpdump -nn \
  'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'
```

---

## Packet Size

```bash
# Packets smaller than 128 bytes
sudo tcpdump -nn 'less 128'

# Packets larger than 1500 bytes
sudo tcpdump -nn 'greater 1500'

# TCP packets larger than 1000 bytes
sudo tcpdump -nn 'tcp and greater 1000'
```

---

## Ethernet Filters

```bash
# MAC address
sudo tcpdump -e -nn \
  'ether host 00:11:22:33:44:55'

# Source MAC
sudo tcpdump -e -nn \
  'ether src 00:11:22:33:44:55'

# Destination MAC
sudo tcpdump -e -nn \
  'ether dst 00:11:22:33:44:55'

# Broadcast traffic
sudo tcpdump -e -nn 'broadcast'

# Multicast traffic
sudo tcpdump -e -nn 'multicast'
```

---

## VLAN Traffic

```bash
sudo tcpdump -i eth0 -e -nn 'vlan'

sudo tcpdump -i eth0 -e -nn \
  'vlan 100'

sudo tcpdump -i eth0 -e -nn \
  'vlan and host 10.0.0.10'
```

---

## Common Protocol Recipes

### DNS

```bash
sudo tcpdump -i any -nn \
  'udp port 53 or tcp port 53'
```

Queries only:

```bash
sudo tcpdump -i any -nn \
  'dst port 53'
```

Responses only:

```bash
sudo tcpdump -i any -nn \
  'src port 53'
```

### HTTP

```bash
sudo tcpdump -i any -nn -A \
  'tcp port 80'
```

### HTTPS / TLS

```bash
sudo tcpdump -i any -nn \
  'tcp port 443'
```

tcpdump cannot decrypt TLS payloads without additional tooling and session keys.

### SSH

```bash
sudo tcpdump -i any -nn \
  'tcp port 22'
```

### DHCP

```bash
sudo tcpdump -i any -nn \
  'udp and (port 67 or port 68)'
```

### NTP

```bash
sudo tcpdump -i any -nn \
  'udp port 123'
```

### ICMP

```bash
sudo tcpdump -i any -nn 'icmp'
sudo tcpdump -i any -nn 'icmp6'
```

### ARP

```bash
sudo tcpdump -i any -e -nn 'arp'
```

### PostgreSQL

```bash
sudo tcpdump -i any -nn \
  'tcp port 5432'
```

---

## Capture Files

### Write a capture

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -w capture.pcap
```

### Read a capture

```bash
tcpdump -nn -r capture.pcap
```

### Filter while reading

```bash
tcpdump -nn -r capture.pcap \
  'host 10.0.0.10 and tcp port 443'
```

### Display payloads from a capture

```bash
tcpdump -nn -A -r capture.pcap
tcpdump -nn -X -r capture.pcap
```

---

## Capture Rotation

### Rotate by size

Create up to ten approximately 100 MB files:

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -C 100 \
  -W 10 \
  -w capture.pcap
```

### Rotate by time

Create one file every five minutes and stop after twelve files:

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -G 300 \
  -W 12 \
  -w 'capture-%Y%m%d-%H%M%S.pcap'
```

---

## Remote Capture

Capture remotely and write locally:

```bash
ssh server \
  "sudo tcpdump -i eth0 -nn -s 0 -U -w - 'not port 22'" \
  > capture.pcap
```

Open a remote capture directly in Wireshark:

```bash
ssh server \
  "sudo tcpdump -i eth0 -nn -s 0 -U -w - 'not port 22'" \
  | wireshark -k -i -
```

Excluding port 22 prevents the SSH connection from capturing itself.

---

## Pipe-Friendly Output

```bash
sudo tcpdump -i any -nn -l \
  | grep '10.0.0.10'
```

```bash
sudo tcpdump -i any -nn -l \
  'tcp port 443' \
  | tee tcpdump.log
```

Use a BPF filter instead of `grep` whenever possible.

---

## Useful Diagnostic Commands

### Watch a connection

```bash
sudo tcpdump -i any -nn -tttt \
  'host 10.0.0.10 and tcp port 443'
```

### Watch outbound connections

```bash
sudo tcpdump -i eth0 -nn \
  'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'
```

### Watch failed or reset connections

```bash
sudo tcpdump -i any -nn \
  'tcp[tcpflags] & tcp-rst != 0'
```

### Capture everything except SSH and DNS

```bash
sudo tcpdump -i any -nn \
  'not port 22 and not port 53'
```

### Capture traffic between two hosts

```bash
sudo tcpdump -i any -nn \
  'host 10.0.0.10 and host 10.0.0.20'
```

### Capture IPv4 fragments

```bash
sudo tcpdump -i any -nn \
  'ip[6:2] & 0x3fff != 0'
```

---

## Common Gotchas

* Use `-nn` unless DNS and service-name resolution are explicitly useful.
* Use `-s 0` when packet payloads must not be truncated.
* `-w` writes binary PCAP data, not readable terminal output.
* Outgoing packets may appear to have invalid checksums because of NIC checksum offloading.
* Capturing on `any` may omit Ethernet MAC information.
* Filters are applied before packets reach userspace and are more efficient than piping through `grep`.
* tcpdump cannot reliably identify TCP retransmissions using a capture filter alone. Analyze the PCAP with Wireshark or tshark.
* Avoid capturing secrets from unencrypted protocols on shared systems.
