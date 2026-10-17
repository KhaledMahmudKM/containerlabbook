# Your First Lab

Time to deploy something. In this chapter you build the smallest useful lab — two Linux hosts connected by a single cable — then deploy it, look inside, send a ping across the link, and tear it down. This is the full ContainerLab loop in miniature.

---

## The topology

Two hosts, one link between them:

```
  h1  eth1 ─────────── eth1  h2
```

Create a file called `first.clab.yml`:

```yaml
name: first
topology:
  nodes:
    h1:
      kind: linux
      image: wbitt/network-multitool
    h2:
      kind: linux
      image: wbitt/network-multitool
  links:
    - endpoints: ["h1:eth1", "h2:eth1"]
```

That is a complete lab. Two nodes of kind `linux`, each running the `network-multitool` image (which includes `ping` and `ip`), joined by one link between their `eth1` interfaces.

---

## Deploy it

```bash
sudo containerlab deploy -t first.clab.yml
```

ContainerLab pulls the image if needed, starts the two containers, creates the management network, and wires the link. When it finishes it prints a summary of the nodes, similar to this:

| # | Name | Container ID | Image | Kind | State | IPv4 Address |
|---|------|--------------|-------|------|-------|--------------|
| 1 | clab-first-h1 | a1b2c3d4e5f6 | wbitt/network-multitool | linux | running | 172.20.20.2/24 |
| 2 | clab-first-h2 | f6e5d4c3b2a1 | wbitt/network-multitool | linux | running | 172.20.20.3/24 |

Those IPv4 addresses are on the **management** network (`eth0`), not the link you defined. The data link (`eth1` ↔ `eth1`) has no addresses yet — you will add them next.

---

## Inspect it

You can list the lab again at any time:

```bash
sudo containerlab inspect -t first.clab.yml
```

Or see every ContainerLab lab running on the host:

```bash
sudo containerlab inspect --all
```

---

## Look inside a node

Open a shell in `h1` using its container name, `clab-first-h1`:

```bash
docker exec -it clab-first-h1 /bin/bash
```

Inside, look at the interfaces:

```bash
ip address
```

You will see `eth0` (management, already addressed) and `eth1` (your data link, up but with no IP). Let's give the data link addresses.

---

## Address the link and ping

The `eth1` interfaces need IPs in a shared subnet. Use `10.0.0.0/30` for the point-to-point link.

On `h1`:

```bash
docker exec clab-first-h1 ip address add 10.0.0.1/30 dev eth1
```

On `h2`:

```bash
docker exec clab-first-h2 ip address add 10.0.0.2/30 dev eth1
```

Now ping across the link, from `h1` to `h2`:

```bash
docker exec clab-first-h1 ping -c 3 10.0.0.2
```

```
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.065 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.048 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.051 ms
```

Your two containers are talking over the virtual cable you defined. That is the whole idea of ContainerLab.

> **Why did the addresses disappear after I destroyed the lab?** They were added at runtime, by hand, so they live only as long as the container. Chapter 8 shows how to pre-load configuration so nodes come up addressed automatically.

---

## Destroy it

When you are finished, tear the lab down and clean up:

```bash
sudo containerlab destroy -t first.clab.yml --cleanup
```

`--cleanup` also removes the `clab-first` working directory that deploy created. Without it, that directory is left in place for next time.

---

## What just happened

In one short file and a few commands you:

1. **Described** a topology declaratively.
2. **Deployed** it — containers started, management network built, link wired.
3. **Inspected** the running nodes.
4. **Used** the lab — addressed an interface and pinged across it.
5. **Destroyed** it cleanly.

Everything from here builds on this loop: more nodes, real routing software, pre-loaded configs, and bigger topologies.

---

## Key Takeaways

- A minimal lab is two nodes and one link in a `.clab.yml` file.
- `containerlab deploy -t <file>` builds the lab; the output table shows management IPs.
- Data links come up without addresses — you configure them, by hand now, automatically later.
- `docker exec` into `clab-<lab>-<node>` to work inside a node.
- `containerlab destroy -t <file> --cleanup` removes everything.
