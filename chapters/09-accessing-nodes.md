# Accessing and Operating Nodes

Once a lab is running you need to get inside the devices — to configure them, read their state, watch logs, and copy files in and out. This chapter collects the everyday ways to reach and operate nodes.

---

## Three ways in

| Method | Good for |
|---|---|
| `docker exec` | Quick commands and shells into any node |
| `vtysh` (FRR) | The router CLI, interactive or one-shot |
| `ssh` | Devices that run an SSH server (SR Linux, cEOS, some images) |

For FRR and Linux containers, `docker exec` is the workhorse.

---

## docker exec

Every node is a container named `clab-<lab>-<node>`. Open an interactive shell:

```bash
docker exec -it clab-ospf-r1 /bin/bash     # or /bin/sh on minimal images
```

Run a single command without a shell:

```bash
docker exec clab-ospf-h1 ip address
docker exec clab-ospf-h1 ping -c 3 192.168.2.10
```

The `-it` flags (interactive + TTY) are for interactive sessions; omit them for one-shot commands you script.

---

## The FRR CLI: vtysh

`vtysh` is FRR's integrated shell — a Cisco-like CLI across all the routing daemons. Enter it interactively:

```bash
docker exec -it clab-ospf-r1 vtysh
```

```
r1# show ip route
r1# show ip ospf neighbor
r1# configure terminal
r1(config)# ...
```

Or run one command and return, which is ideal for scripting and quick checks:

```bash
docker exec clab-ospf-r1 vtysh -c "show ip route"
docker exec clab-ospf-r1 vtysh -c "show ip ospf neighbor"
```

### Useful FRR show commands

| Command | Shows |
|---|---|
| `show interface brief` | Interfaces and their addresses/state |
| `show ip route` | The routing table |
| `show ip ospf neighbor` | OSPF adjacencies |
| `show ip ospf interface` | OSPF per-interface detail |
| `show bgp summary` | BGP peers and prefix counts |
| `show running-config` | The live configuration |

---

## SSH (for NOSes that support it)

Container-native NOSes typically run an SSH server, so you can treat them like real gear. ContainerLab even lets you SSH by node name:

```bash
ssh admin@clab-mylab-srl1
```

The default credentials depend on the NOS (for SR Linux, `admin`). FRR and the `network-multitool` image used here do not run SSH by default, which is why this book uses `docker exec` instead — but it is worth knowing the option exists when you move to SR Linux or cEOS.

---

## Reading logs

When a node misbehaves, its container logs are the first place to look:

```bash
docker logs clab-ospf-r1
docker logs -f clab-ospf-r1      # follow, like tail -f
```

For FRR specifically, the daemons also log inside the container (commonly under `/var/log/frr`), which you can read with `docker exec`.

---

## Copying files in and out

Use `docker cp` to move files between the host and a node:

```bash
# host → node
docker cp ./newfrr.conf clab-ospf-r1:/etc/frr/frr.conf

# node → host
docker cp clab-ospf-r1:/var/log/frr/frr.log ./frr-r1.log
```

For FRR config, remember the cleaner path from Chapter 8: because `frr.conf` is bind-mounted, you can just edit the file on the host and reload, or edit live and `write memory`.

---

## Inspecting the whole lab

`containerlab inspect` summarizes a running lab — names, kinds, states, and management IPs:

```bash
sudo containerlab inspect -t ospf.clab.yml
```

Add `--all` to see every ContainerLab lab on the host. This is the fastest way to get the container names and management addresses you will feed to `docker exec` and `ssh`.

---

## Running a command on every node

The CLI can execute a command across all nodes of a lab at once, which is handy for quick audits:

```bash
sudo containerlab exec -t ospf.clab.yml --cmd "ip -br address"
```

This runs `ip -br address` inside each node and prints the output per node — a one-shot way to confirm every interface is addressed as you expect.

---

## A quick operating loop

A comfortable rhythm once a lab is up:

1. `containerlab inspect` to get names and IPs.
2. `docker exec ... vtysh -c "show ..."` to check protocol state.
3. `docker exec -it ... vtysh` to configure interactively.
4. `docker logs -f ...` in another terminal when something is wrong.
5. `write memory` to persist good changes back to your config files.

---

## Key Takeaways

- `docker exec` into `clab-<lab>-<node>` is the main way into FRR and Linux nodes.
- `vtysh` is FRR's CLI; use `vtysh -c "..."` for scriptable one-shot commands.
- Container-native NOSes also accept `ssh <user>@clab-<lab>-<node>`.
- `docker logs [-f]` reads a node's logs; `docker cp` moves files.
- `containerlab inspect` lists the lab; `containerlab exec` runs a command on every node.
