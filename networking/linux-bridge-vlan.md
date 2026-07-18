# Linux Bridge VLAN Cheatsheet

## Table of Contents

- [Practical exercise](#practical-exercise)
- [Mental model](#mental-model)
- [Check whether VLAN filtering is active](#check-whether-vlan-filtering-is-active)
- [Read `bridge vlan show`](#read-bridge-vlan-show)
- [Access port example](#access-port-example)
- [Trunk port example](#trunk-port-example)
- [Access-to-trunk forwarding](#access-to-trunk-forwarding)
- [The bridge-local or CPU port](#the-bridge-local-or-cpu-port)
- [`self` versus `master`](#self-versus-master)
- [Inspect configuration](#inspect-configuration)
- [Configure an access port cleanly](#configure-an-access-port-cleanly)
- [Configure a trunk cleanly](#configure-a-trunk-cleanly)
- [Hybrid or native-VLAN trunk](#hybrid-or-native-vlan-trunk)
- [Understanding `tcpdump`](#understanding-tcpdump)
- [VLAN hardware offloading](#vlan-hardware-offloading)
- [Troubleshooting checklist](#troubleshooting-checklist)
- [Common mistakes](#common-mistakes)
- [Quick reference](#quick-reference)
- [Minimal working example](#minimal-working-example)

---

## Practical exercise

Start with a read-only inventory. Replace `br0` with a bridge reported by the
first two commands:

```bash
ip -brief link
ip -details link show type bridge

BR='br0'
bridge link show master "$BR"
bridge vlan show
bridge fdb show br "$BR" | head -n 20
```

Now choose one bridge port from the output and trace an untagged ingress frame:

```text
access host ── untagged frame ──► access port ── PVID 10 ──► br0 FDB lookup
                                                                       │
                         another VLAN 10 access port ◄── untagged ─────┤
                                  VLAN 10 trunk port ◄── tagged 10 ────┘
```

| Output clue | Physical behavior to picture |
| --- | --- |
| `PVID` | An untagged ingress frame is assigned to this VLAN. |
| `Egress Untagged` | The tag is removed before the frame leaves that port. |
| A VLAN listed without flags | Membership exists; egress is normally tagged. |
| A learned MAC in the FDB | The bridge has learned which port reaches that source. |

If no bridge exists, keep reading through the minimal namespace example near
the end before changing a production host. VLAN changes can cut off your own
management connection, so make them from a console or disposable lab first.

---

## Mental model

A VLAN-aware Linux bridge behaves like a managed Ethernet switch.

```text
Physical interfaces
        │
        ▼
┌───────────────────┐
│ Linux bridge br0  │
│                   │
│ VLAN forwarding   │
│ MAC learning      │
│ tag/untag actions │
└───────────────────┘
```

Each bridge port can be:

* An **access port**: one VLAN, normally untagged
* A **trunk port**: multiple VLANs, normally tagged
* A **hybrid port**: tagged VLANs plus one untagged/native VLAN

---

## Check whether VLAN filtering is active

```bash
cat /sys/class/net/br0/bridge/vlan_filtering
```

Equivalent:

```bash
ip -d link show br0
```

### `vlan_filtering = 0`

The bridge is VLAN-unaware.

* VLAN membership is not enforced
* PVID settings are not applied
* Tags are not inserted or removed by the bridge
* Frames are generally forwarded unchanged

Example:

```text
enp3s0 receives untagged frame
        ↓
br0 forwards unchanged
        ↓
enp1s0 sends untagged frame
```

### `vlan_filtering = 1`

The bridge is VLAN-aware.

* Ingress frames are assigned to VLANs
* VLAN membership is enforced
* Tags can be added or removed on egress
* Broadcasts remain inside their VLAN

Enable it:

```bash
sudo ip link set dev br0 type bridge vlan_filtering 1
```

Do this from a console when possible. Enabling filtering can immediately interrupt connectivity.

---

## Read `bridge vlan show`

Example:

```text
port      vlan-id
enp1s0    10
          20
          30
enp3s0    20 PVID Egress Untagged
br0       1 PVID Egress Untagged
```

### No flags

```text
enp1s0  20
```

Meaning:

* Port is a member of VLAN 20
* VLAN 20 normally leaves the port tagged
* Untagged ingress is not automatically assigned to VLAN 20

This is typical for a trunk port.

### `PVID`

```text
enp3s0  20 PVID
```

Port VLAN ID.

Untagged frames arriving on the port are internally assigned to VLAN 20.

```text
untagged ingress
        ↓
classified as VLAN 20
```

A port can have only one PVID.

### `Egress Untagged`

```text
enp3s0  20 Egress Untagged
```

Frames belonging to VLAN 20 leave this port without an 802.1Q tag.

### `PVID Egress Untagged`

```text
enp3s0  20 PVID Egress Untagged
```

Typical access-port configuration:

* Untagged ingress becomes VLAN 20
* VLAN 20 egress becomes untagged

---

## Access port example

```bash
bridge vlan add dev enp3s0 vid 20 pvid untagged
```

Behavior:

```text
Host sends untagged frame
        ↓
enp3s0 assigns VLAN 20
        ↓
bridge forwards inside VLAN 20
        ↓
frames returning through enp3s0 are untagged
```

---

## Trunk port example

```bash
bridge vlan add dev enp1s0 vid 10
bridge vlan add dev enp1s0 vid 20
bridge vlan add dev enp1s0 vid 30
```

Because `untagged` is absent, these VLANs leave tagged.

```text
VLAN 20 frame
      ↓
enp1s0 egress
      ↓
802.1Q tag containing VLAN ID 20
```

---

## Access-to-trunk forwarding

Given:

```text
enp3s0  20 PVID Egress Untagged
enp1s0  20
```

And VLAN filtering enabled:

```text
Untagged frame enters enp3s0
        ↓
PVID classifies it as VLAN 20
        ↓
bridge forwards it within VLAN 20
        ↓
frame exits enp1s0 tagged as VLAN 20
```

Return traffic:

```text
Tagged VLAN 20 frame enters enp1s0
        ↓
bridge forwards it within VLAN 20
        ↓
frame exits enp3s0 untagged
```

---

## The bridge-local or CPU port

The bridge device itself represents the Linux host’s connection to the bridge.

```text
br0  1 PVID Egress Untagged
```

This means the host-side bridge interface participates in untagged VLAN 1.

It does not automatically mean the host participates in VLANs 10, 20, or 30.

Add host participation in a tagged VLAN:

```bash
sudo bridge vlan add dev br0 vid 20 self
```

Create a VLAN interface for Layer 3:

```bash
sudo ip link add link br0 name br0.20 type vlan id 20
sudo ip link set br0.20 up
sudo ip address add 10.0.20.1/24 dev br0.20
```

Typical arrangement:

```text
br0
├── br0.10
├── br0.20
└── br0.30
```

Use the VLAN interfaces for IP addresses. Keep the physical bridge ports without IP addresses.

---

## `self` versus `master`

### `self`

Apply the VLAN to the bridge device itself:

```bash
bridge vlan add dev br0 vid 20 self
```

This controls host or CPU participation.

### `master`

Apply the VLAN to the bridge port’s bridge master.

For ordinary bridge port configuration, this is usually implicit:

```bash
bridge vlan add dev enp3s0 vid 20 pvid untagged
```

---

## Inspect configuration

```bash
bridge vlan show
```

More detail:

```bash
bridge -d vlan show
```

Show bridge properties:

```bash
ip -d link show br0
```

Show bridge membership:

```bash
bridge link show
```

Show forwarding database:

```bash
bridge fdb show br br0
```

Show VLAN-specific forwarding entries:

```bash
bridge fdb show br br0 vlan 20
```

---

## Configure an access port cleanly

Remove the default VLAN first:

```bash
sudo bridge vlan del dev enp3s0 vid 1
```

Add VLAN 20:

```bash
sudo bridge vlan add dev enp3s0 vid 20 pvid untagged
```

Verify:

```bash
bridge vlan show dev enp3s0
```

Expected:

```text
enp3s0  20 PVID Egress Untagged
```

---

## Configure a trunk cleanly

Remove the default VLAN:

```bash
sudo bridge vlan del dev enp1s0 vid 1
```

Add tagged VLANs:

```bash
sudo bridge vlan add dev enp1s0 vid 10
sudo bridge vlan add dev enp1s0 vid 20
sudo bridge vlan add dev enp1s0 vid 30
```

Or add a range:

```bash
sudo bridge vlan add dev enp1s0 vid 10-30
```

Verify:

```bash
bridge vlan show dev enp1s0
```

Entries without `Egress Untagged` are tagged on egress.

---

## Hybrid or native-VLAN trunk

Example:

```text
VLAN 10 untagged/native
VLAN 20 tagged
VLAN 30 tagged
```

Configuration:

```bash
sudo bridge vlan add dev enp1s0 vid 10 pvid untagged
sudo bridge vlan add dev enp1s0 vid 20
sudo bridge vlan add dev enp1s0 vid 30
```

Behavior:

* Untagged ingress becomes VLAN 10
* VLAN 10 leaves untagged
* VLANs 20 and 30 leave tagged

---

## Understanding `tcpdump`

Use Ethernet output:

```bash
sudo tcpdump -i enp1s0 -enn
```

### Untagged ARP

```text
ethertype ARP (0x0806)
```

### Tagged ARP

```text
ethertype 802.1Q (0x8100), vlan 20, ethertype ARP
```

### Capture only VLAN traffic

```bash
sudo tcpdump -i enp1s0 -enn vlan
```

Specific VLAN:

```bash
sudo tcpdump -i enp1s0 -enn 'vlan 20'
```

Capture direction:

```bash
sudo tcpdump -i enp1s0 -Q in -enn
sudo tcpdump -i enp1s0 -Q out -enn
```

`-Q out` means outbound packets only. It does not guarantee capture after NIC hardware offloading.

---

## VLAN hardware offloading

NIC offloading can make tagged traffic appear untagged in `tcpdump`.

Check offload settings:

```bash
sudo ethtool -k enp1s0 | grep -i vlan
```

Important fields:

```text
rx-vlan-offload
tx-vlan-offload
rx-vlan-filter
```

Temporarily disable VLAN offloading:

```bash
sudo ethtool -K enp1s0 rxvlan off txvlan off
```

Then capture again:

```bash
sudo tcpdump -i enp1s0 -Q out -enn
```

Restore it afterward:

```bash
sudo ethtool -K enp1s0 rxvlan on txvlan on
```

The most reliable verification is a capture on the peer device or another physical capture point.

---

## Troubleshooting checklist

### Traffic leaves a trunk untagged

Check:

```bash
cat /sys/class/net/br0/bridge/vlan_filtering
```

It must return:

```text
1
```

Then check:

```bash
bridge vlan show dev enp1s0
```

The VLAN must not contain `Egress Untagged`.

Also check TX VLAN offloading:

```bash
ethtool -k enp1s0 | grep tx-vlan
```

### Access-port traffic does not pass

Verify the port has both flags:

```text
20 PVID Egress Untagged
```

Also verify that the destination or trunk port is a member of VLAN 20.

### Tagged traffic is accepted on the wrong port

Check whether the port has additional VLAN memberships:

```bash
bridge vlan show dev enp3s0
```

A pure VLAN 20 access port should normally only contain VLAN 20.

### VLANs can communicate unexpectedly

Possible causes:

* `vlan_filtering` is `0`
* A router is routing between VLANs
* A firewall allows inter-VLAN forwarding
* Both ports belong to the same VLAN
* Untagged traffic is entering through an unexpected PVID

### Host cannot reach a VLAN

Check the bridge-local port:

```bash
bridge vlan show dev br0
```

Then verify the host has an appropriate VLAN interface:

```bash
ip -d link show br0.20
ip address show br0.20
```

---

## Common mistakes

### Assuming `bridge vlan show` means filtering is active

It does not.

Always check:

```bash
cat /sys/class/net/br0/bridge/vlan_filtering
```

### Putting an IP directly on an access-port interface

Bridge slave interfaces should normally have no IP addresses.

Use:

```text
br0
br0.<vlan>
```

### Forgetting the default VLAN 1

When ports are added to a bridge, they may initially receive VLAN 1.

Remove it when building explicit access or trunk configurations:

```bash
bridge vlan del dev enp1s0 vid 1
```

### Treating IP subnets as VLAN identifiers

The bridge does not understand IP addressing.

```text
10.0.20.0/24
```

does not automatically mean VLAN 20.

The VLAN assignment is based on:

* 802.1Q tags
* PVID
* Bridge VLAN membership

### Trusting local `tcpdump` as the on-wire truth

Hardware offloading may hide or remove VLAN headers at the capture point.

Capture on the peer side for definitive results.

---

## Quick reference

| Configuration          | Ingress                  | Egress         |
| ---------------------- | ------------------------ | -------------- |
| `vid 20`               | Tagged VLAN 20 accepted  | Tagged VLAN 20 |
| `vid 20 pvid`          | Untagged becomes VLAN 20 | Tagged VLAN 20 |
| `vid 20 untagged`      | Tagged VLAN 20 accepted  | Untagged       |
| `vid 20 pvid untagged` | Untagged becomes VLAN 20 | Untagged       |

---

## Minimal working example

```bash
sudo ip link add br0 type bridge vlan_filtering 1
sudo ip link set br0 up

sudo ip link set enp1s0 master br0
sudo ip link set enp3s0 master br0

sudo ip link set enp1s0 up
sudo ip link set enp3s0 up

sudo bridge vlan del dev enp1s0 vid 1
sudo bridge vlan del dev enp3s0 vid 1

# Trunk port
sudo bridge vlan add dev enp1s0 vid 20

# Access port
sudo bridge vlan add dev enp3s0 vid 20 pvid untagged
```

Expected behavior:

```text
enp3s0: untagged VLAN 20 access port
enp1s0: tagged VLAN 20 trunk port
```

Verify:

```bash
cat /sys/class/net/br0/bridge/vlan_filtering
bridge vlan show
sudo tcpdump -i enp1s0 -enn
```
