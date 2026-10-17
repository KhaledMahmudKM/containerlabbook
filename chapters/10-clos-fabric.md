# A CLOS Fabric Example

This is the capstone lab: a small leaf–spine (CLOS) fabric — the topology that underpins modern data centers — built entirely from FRR routers and two Linux hosts. You will run eBGP between the tiers, the way real fabrics do, and watch a host on one leaf reach a host on another across redundant spines.

---

## The design

Two spines, two leaves, a host behind each leaf. Every leaf connects to every spine, so there are two equal paths between any pair of leaves.

```
        spine1        spine2
        /    \        /    \
       /      \      /      \
   leaf1       leaf2
     |            |
    h1           h2
```

### eBGP addressing plan

In a CLOS fabric the standard choice is **eBGP with one AS number per device**. Every leaf–spine link is a `/30`.

| Device | ASN | Role |
|---|---|---|
| spine1 | 65001 | spine |
| spine2 | 65002 | spine |
| leaf1 | 65101 | leaf (hosts 172.16.1.0/24) |
| leaf2 | 65102 | leaf (hosts 172.16.2.0/24) |

| Link | Subnet | Addresses |
|---|---|---|
| leaf1 ↔ spine1 | 10.1.1.0/30 | leaf1 .1 / spine1 .2 |
| leaf1 ↔ spine2 | 10.1.2.0/30 | leaf1 .1 / spine2 .2 |
| leaf2 ↔ spine1 | 10.2.1.0/30 | leaf2 .1 / spine1 .2 |
| leaf2 ↔ spine2 | 10.2.2.0/30 | leaf2 .1 / spine2 .2 |

Leaves originate their host subnets into BGP; spines transit between leaves.

---

## The daemons file

All four routers use the same `daemons` file, this time with **bgpd** on:

```ini
zebra=yes
bgpd=yes
vtysh_enable=yes
```

---

## The router configs

### leaf1/frr.conf

```
hostname leaf1
service integrated-vtysh-config
!
interface eth1
 ip address 10.1.1.1/30
interface eth2
 ip address 10.1.2.1/30
interface eth3
 ip address 172.16.1.1/24
!
router bgp 65101
 bgp router-id 10.1.1.1
 neighbor 10.1.1.2 remote-as 65001
 neighbor 10.1.2.2 remote-as 65002
 address-family ipv4 unicast
  network 172.16.1.0/24
 exit-address-family
!
```

### leaf2/frr.conf

```
hostname leaf2
service integrated-vtysh-config
!
interface eth1
 ip address 10.2.1.1/30
interface eth2
 ip address 10.2.2.1/30
interface eth3
 ip address 172.16.2.1/24
!
router bgp 65102
 bgp router-id 10.2.1.1
 neighbor 10.2.1.2 remote-as 65001
 neighbor 10.2.2.2 remote-as 65002
 address-family ipv4 unicast
  network 172.16.2.0/24
 exit-address-family
!
```

### spine1/frr.conf

```
hostname spine1
service integrated-vtysh-config
!
interface eth1
 ip address 10.1.1.2/30
interface eth2
 ip address 10.2.1.2/30
!
router bgp 65001
 bgp router-id 10.1.1.2
 neighbor 10.1.1.1 remote-as 65101
 neighbor 10.2.1.1 remote-as 65102
 address-family ipv4 unicast
 exit-address-family
!
```

### spine2/frr.conf

```
hostname spine2
service integrated-vtysh-config
!
interface eth1
 ip address 10.1.2.2/30
interface eth2
 ip address 10.2.2.2/30
!
router bgp 65002
 bgp router-id 10.1.2.2
 neighbor 10.1.2.1 remote-as 65101
 neighbor 10.2.2.1 remote-as 65102
 address-family ipv4 unicast
 exit-address-family
!
```

The spines carry no `network` statements — they do not originate routes, they relay what the leaves advertise.

---

## The topology file

```yaml
name: clos
topology:
  defaults:
    kind: linux
  kinds:
    linux:
      image: quay.io/frrouting/frr:latest
  nodes:
    spine1: { binds: [spine1/daemons:/etc/frr/daemons, spine1/frr.conf:/etc/frr/frr.conf] }
    spine2: { binds: [spine2/daemons:/etc/frr/daemons, spine2/frr.conf:/etc/frr/frr.conf] }
    leaf1:  { binds: [leaf1/daemons:/etc/frr/daemons,  leaf1/frr.conf:/etc/frr/frr.conf] }
    leaf2:  { binds: [leaf2/daemons:/etc/frr/daemons,  leaf2/frr.conf:/etc/frr/frr.conf] }
    h1:
      image: wbitt/network-multitool
      exec:
        - ip address add 172.16.1.10/24 dev eth1
        - ip route replace default via 172.16.1.1
    h2:
      image: wbitt/network-multitool
      exec:
        - ip address add 172.16.2.10/24 dev eth1
        - ip route replace default via 172.16.2.1
  links:
    - endpoints: ["leaf1:eth1", "spine1:eth1"]
    - endpoints: ["leaf1:eth2", "spine2:eth1"]
    - endpoints: ["leaf2:eth1", "spine1:eth2"]
    - endpoints: ["leaf2:eth2", "spine2:eth2"]
    - endpoints: ["leaf1:eth3", "h1:eth1"]
    - endpoints: ["leaf2:eth3", "h2:eth1"]
```

Note how `kinds` sets the FRR image once for all four routers, and only the hosts override it. The `binds` are written inline here to keep the file compact; the multi-line form from Chapter 8 is equivalent.

The directory holds a folder per router (`spine1/`, `spine2/`, `leaf1/`, `leaf2/`), each with its `daemons` and `frr.conf`.

---

## Deploy and verify

```bash
sudo containerlab deploy -t clos.clab.yml
```

**BGP sessions** — leaf1 should have two established peers (one per spine):

```bash
docker exec clab-clos-leaf1 vtysh -c "show bgp summary"
```

```
Neighbor        V         AS   MsgRcvd   MsgSent   State/PfxRcd
10.1.1.2        4      65001        12        12              1
10.1.2.2        4      65002        12        12              1
```

The `State/PfxRcd` column showing a number (not `Active`/`Idle`) means the session is up and that many prefixes were received.

**Routing table** — leaf1 should reach `172.16.2.0/24` via *both* spines (equal-cost multipath):

```bash
docker exec clab-clos-leaf1 vtysh -c "show ip route bgp"
```

```
B>* 172.16.2.0/24 [20/0] via 10.1.1.2, eth1, weight 1
  *                      via 10.1.2.2, eth2, weight 1
```

Two next-hops for the same prefix — traffic is load-shared across the spines, exactly as a fabric should behave.

**End to end** — h1 reaches h2 across the fabric:

```bash
docker exec clab-clos-h1 ping -c 3 172.16.2.10
```

---

## Experiments to try

The fabric is a great sandbox. A few things to explore:

- **Fail a link.** `sudo containerlab exec -t clos.clab.yml --node leaf1 --cmd "ip link set eth1 down"` and watch BGP reconverge onto the remaining spine — the `172.16.2.0/24` route drops to a single next-hop.
- **Watch convergence.** Re-run `show ip route bgp` before and after.
- **Add a leaf.** Copy the leaf1 pattern as leaf3 with a new ASN and host subnet, add its two links, and redeploy.
- **Trace the path.** `docker exec clab-clos-h1 traceroute 172.16.2.10` to see which leaf and spine the packets cross.

---

## Key Takeaways

- A leaf–spine fabric connects every leaf to every spine for redundant, equal-cost paths.
- The standard underlay is eBGP with one AS per device; leaves originate host subnets, spines transit.
- `kinds` sets the common FRR image once so node definitions stay short.
- `show bgp summary` confirms peering; two next-hops in the route table confirm multipath.
- Failing a link and watching BGP reconverge is the payoff of a redundant fabric.
