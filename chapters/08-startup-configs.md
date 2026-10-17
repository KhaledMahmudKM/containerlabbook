# Startup Configs and Node Customization

In the first lab you addressed interfaces by hand, and they vanished when the lab was destroyed. Real labs should come up *already configured*. This chapter shows how: bind-mount FRR configuration so routers boot with their addresses and routing protocols running, and use `exec` to prepare hosts. By the end you will have a fully routed two-router lab that works the moment it deploys.

---

## How FRR loads configuration

The FRR container reads two files under `/etc/frr`:

- **`daemons`** — which routing daemons to start (zebra, ospfd, bgpd, …).
- **`frr.conf`** — the actual configuration (interfaces, addresses, routing).

If you bind-mount both into the container, FRR starts the right daemons and loads your config at boot. No manual steps.

### The daemons file

Copy FRR's standard `daemons` file and enable what you need. The important lines for an OSPF lab:

```ini
zebra=yes
ospfd=yes
bgpd=no
# ... other daemons default to no ...

vtysh_enable=yes
```

`zebra` is the core (it talks to the kernel's routing table and must always be on). `ospfd` runs OSPF. `vtysh_enable=yes` lets you use the integrated `vtysh` CLI. Turn on `bgpd=yes` instead when you build a BGP lab.

---

## The lab

Two routers across a backbone, each with a host behind it. OSPF in area 0 ties it together so the hosts can reach each other.

```
h1 ── r1 ════ r2 ── h2
   .10  │ 10.0.0.0/30 │  .10
 192.168.1.0/24   192.168.2.0/24
```

### Directory layout

Keep each router's files in its own folder next to the topology:

```
ospf-lab/
├── ospf.clab.yml
├── r1/
│   ├── daemons
│   └── frr.conf
└── r2/
    ├── daemons
    └── frr.conf
```

Both `daemons` files are identical (zebra + ospfd, as above). The `frr.conf` files differ per router.

### r1/frr.conf

```
frr version 10.0
frr defaults traditional
hostname r1
service integrated-vtysh-config
!
interface eth1
 ip address 10.0.0.1/30
!
interface eth2
 ip address 192.168.1.1/24
!
router ospf
 network 10.0.0.0/30 area 0
 network 192.168.1.0/24 area 0
!
```

### r2/frr.conf

```
frr version 10.0
frr defaults traditional
hostname r2
service integrated-vtysh-config
!
interface eth1
 ip address 10.0.0.2/30
!
interface eth2
 ip address 192.168.2.1/24
!
router ospf
 network 10.0.0.0/30 area 0
 network 192.168.2.0/24 area 0
!
```

Each router addresses its backbone link (`eth1`) and its host-facing link (`eth2`), then advertises both subnets into OSPF area 0.

### ospf.clab.yml

```yaml
name: ospf
topology:
  defaults:
    kind: linux
  nodes:
    r1:
      image: quay.io/frrouting/frr:latest
      binds:
        - r1/daemons:/etc/frr/daemons
        - r1/frr.conf:/etc/frr/frr.conf
    r2:
      image: quay.io/frrouting/frr:latest
      binds:
        - r2/daemons:/etc/frr/daemons
        - r2/frr.conf:/etc/frr/frr.conf
    h1:
      image: wbitt/network-multitool
      exec:
        - ip address add 192.168.1.10/24 dev eth1
        - ip route replace default via 192.168.1.1
    h2:
      image: wbitt/network-multitool
      exec:
        - ip address add 192.168.2.10/24 dev eth1
        - ip route replace default via 192.168.2.1
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]   # backbone
    - endpoints: ["r1:eth2", "h1:eth1"]   # h1 behind r1
    - endpoints: ["r2:eth2", "h2:eth1"]   # h2 behind r2
```

The routers load their config from the binds; the hosts configure their address and default route with `exec`, which runs after the links are wired.

---

## Deploy and verify

```bash
sudo containerlab deploy -t ospf.clab.yml
```

Check that OSPF formed a neighbor relationship across the backbone:

```bash
docker exec clab-ospf-r1 vtysh -c "show ip ospf neighbor"
```

```
Neighbor ID     Pri State           Up Time  Dead Time Address        Interface
10.0.0.2          1 Full/DR         30.123s    39.456s 10.0.0.2       eth1:10.0.0.1
```

Look at r1's routing table — it should have learned `192.168.2.0/24` via OSPF (marked `O`):

```bash
docker exec clab-ospf-r1 vtysh -c "show ip route ospf"
```

```
O>* 192.168.2.0/24 [110/20] via 10.0.0.2, eth1, weight 1, 00:00:25
```

Now the real test — `h1` pings `h2`, across two routers:

```bash
docker exec clab-ospf-h1 ping -c 3 192.168.2.10
```

```
64 bytes from 192.168.2.10: icmp_seq=1 ttl=62 time=0.21 ms
```

`ttl=62` (down from 64) confirms the packets crossed two routed hops. You built a working routed network from a few text files.

---

## Editing config at runtime

You can also configure FRR live, the way you would a real router:

```bash
docker exec -it clab-ospf-r1 vtysh
```

```
r1# configure terminal
r1(config)# interface eth2
r1(config-if)# description to-h1
r1(config-if)# end
r1# write memory
```

Because `/etc/frr/frr.conf` is bind-mounted from the host, `write memory` saves straight back to your `r1/frr.conf` file — so live changes survive a redeploy. That two-way link between the running device and a file on disk is the essence of lab-as-code.

---

## Other customization hooks

- **`exec`** — run commands in a node at startup (used above for the hosts).
- **`env`** — pass environment variables the image understands.
- **`binds`** — mount any file, not just FRR config: scripts, keys, data.
- **`cmd`** — override the container's start command for special cases.

---

## Key Takeaways

- Bind-mount `daemons` and `frr.conf` so FRR boots configured.
- `daemons` enables zebra + the protocol daemons; `frr.conf` holds the config.
- Give each router its own folder of config files next to the topology.
- Use `exec` to address and route Linux hosts at startup.
- `write memory` saves back to the mounted file, so live edits persist.
