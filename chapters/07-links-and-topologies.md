# Links, Interfaces, and Topologies

Links are the cables of your lab. This chapter goes deeper into how to wire nodes together: point-to-point links, building out multi-node shapes, connecting a node to the host or the outside world, and the conventions that keep larger topologies readable.

---

## The point-to-point link

The everyday link connects one interface on one node to one interface on another:

```yaml
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]
```

This is a `veth` pair — a virtual Ethernet cable with two ends. Traffic that `r1` sends on `eth1` arrives on `r2`'s `eth1`, and vice versa. There is no switch in between; it is a direct link, exactly like a cable between two physical ports.

---

## Interface numbering conventions

`eth0` is always management. Data links start at `eth1`. Beyond that, the numbers are yours to choose — but a convention saves pain in bigger labs. Two common ones:

- **Sequential:** just number each node's links `eth1`, `eth2`, `eth3`… in the order you add them.
- **Neighbor-aligned:** pick numbers that hint at the neighbor, e.g. the link from `leaf1` toward `spine1` is `eth1`, toward `spine2` is `eth2`.

Whatever you choose, be consistent. The interface name is how you will refer to the link when you address it and configure routing on it.

---

## Building shapes

Larger topologies are just more links. A few canonical shapes:

### Chain

```yaml
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]
    - endpoints: ["r2:eth2", "r3:eth1"]
```

```
r1 ── r2 ── r3
```

### Triangle (ring of three)

```yaml
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]
    - endpoints: ["r2:eth2", "r3:eth1"]
    - endpoints: ["r3:eth2", "r1:eth2"]
```

```
    r1
   /  \
  r3 ── r2
```

A triangle is the smallest topology with a redundant path, which makes it perfect for watching a routing protocol pick a best path and then reconverge when you fail a link.

### Hub and spoke

```yaml
  links:
    - endpoints: ["hub:eth1", "s1:eth1"]
    - endpoints: ["hub:eth2", "s2:eth1"]
    - endpoints: ["hub:eth3", "s3:eth1"]
```

The leaf–spine fabric in Chapter 10 is a generalization of this idea.

---

## Connecting hosts to routers

A routed lab usually has endpoints hanging off routers so you can test real traffic:

```yaml
name: routed
topology:
  defaults:
    kind: linux
  nodes:
    r1:
      image: quay.io/frrouting/frr:latest
    r2:
      image: quay.io/frrouting/frr:latest
    h1:
      image: wbitt/network-multitool
    h2:
      image: wbitt/network-multitool
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]   # router backbone
    - endpoints: ["h1:eth1", "r1:eth2"]   # h1 behind r1
    - endpoints: ["h2:eth1", "r2:eth2"]   # h2 behind r2
```

```
h1 ── r1 ──── r2 ── h2
```

Once the routers run a protocol (next chapter), `h1` can reach `h2` across the backbone. The hosts just need an address and a default route pointing at their local router.

---

## Links to the outside: host and macvlan

Sometimes a node must reach beyond the lab — to the internet, or to a service on your host. The extended link form provides several types:

- **`host`** — connect a node interface straight into the host's networking.
- **`mgmt-net`** — attach an extra interface to the management bridge.
- **`macvlan`** — bind a node interface to a physical host interface, so the node appears on the real LAN.
- **`vxlan` / `vxlan-stitched`** — stretch a link across different machines over VXLAN.

```yaml
  links:
    - type: macvlan
      endpoint:
        node: r1
        interface: eth3
      host-interface: eno1       # a physical NIC on the host
```

Most labs never need these — the self-contained `veth` links cover learning and testing. Reach for them when a lab must touch the real network.

---

## Giving a node internet access

A quick pattern for letting a node reach the internet is to publish nothing and instead rely on the management network, which is a normal Docker bridge with NAT to the host's uplink. Management interfaces (`eth0`) can usually reach outside already, so to `apt`/`apk` install something inside a node you often just use `eth0` — no special link required. For data-plane internet access, the `host` or `macvlan` link types above are the tools.

---

## Keeping big topologies readable

As labs grow, a few habits help:

- **Comment every link** with the two roles it connects (`# leaf1 → spine1`).
- **Group links** in the file by tier or function, not randomly.
- **Use `kinds`/`defaults`** so node bodies stay short and the links are easy to scan.
- **Name nodes by role** — `leaf1`, `spine2`, `edge1` — not `r1`, `r2`, once the lab has structure.

A topology file you can read is a topology you can debug.

---

## Key Takeaways

- The default link is a `veth` point-to-point cable between two `node:interface` endpoints.
- `eth0` is management; number data links from `eth1` with a consistent convention.
- Build chains, rings, and hub-and-spoke shapes by adding links; a triangle is the simplest redundant topology.
- Hang Linux hosts off routers to test end-to-end traffic.
- The extended link form (`host`, `macvlan`, `vxlan`, …) connects a lab to the host or the real network.
