# Core Concepts

Before building labs, it helps to know the handful of concepts ContainerLab is built on: nodes, kinds, links, the management network, naming, and the lab lifecycle. Everything else is detail on top of these.

---

## The topology file

A lab is described by a single YAML file whose name ends in `.clab.yml`. At the top level it has a lab **name** and a **topology** block:

```yaml
name: mylab
topology:
  nodes:
    # devices go here
  links:
    # cables go here
```

The filename convention matters: editors with the ContainerLab extension give autocompletion and validation for files ending in `.clab.yml`.

---

## Nodes

A **node** is one device in your lab — a router, a switch, a host. Each node is a container. You give it a name (the map key) and, at minimum, a `kind` and an `image`:

```yaml
topology:
  nodes:
    r1:
      kind: linux
      image: quay.io/frrouting/frr:latest
    h1:
      kind: linux
      image: wbitt/network-multitool
```

Here `r1` and `h1` are node names you choose. They appear in container names and in link definitions.

---

## Kinds

A **kind** tells ContainerLab *what type of device* a node is, so it knows how to start and manage it. Different kinds have different startup logic, default users, and interface-naming rules.

Common kinds include:

| Kind | Used for |
|---|---|
| `linux` | Any generic container: FRR, Alpine hosts, traffic tools |
| `nokia_srlinux` | Nokia SR Linux |
| `arista_ceos` | Arista cEOS |
| `cisco_xrd` | Cisco XRd |
| `sonic-vs` | SONiC |

This book uses `linux` throughout, because FRR routers and Linux hosts both run as generic containers. Chapter 6 surveys the other kinds.

> **Why kinds exist.** A Nokia SR Linux node needs different handling than an Alpine container — different management user, different interface names, a different way to push config. The `kind` lets ContainerLab apply the right behavior automatically.

---

## Links

A **link** is a virtual cable between two node interfaces. In the common "brief" form, a link is a pair of endpoints written as `node:interface`:

```yaml
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]
```

This connects interface `eth1` on `r1` to `eth1` on `r2`. Under the hood ContainerLab creates a `veth` pair and places each end in the right container.

A few rules worth knowing now:

- `eth0` is reserved for the **management** network. Your data links start at `eth1`.
- Interface names are per-node; `r1:eth1` and `r2:eth1` are different interfaces that happen to share a name.
- Each endpoint appears in exactly one link.

---

## The management network

Every lab gets a dedicated **management network** — a Docker bridge that ContainerLab creates automatically. Each node's `eth0` attaches to it and receives a management IP address.

This network is how you reach your nodes: `docker exec`, SSH, and `containerlab inspect` all use it. It is separate from the data-plane links you define, exactly like a real device's dedicated management port. You can customize its subnet with a top-level `mgmt:` block, but the defaults are fine to start.

---

## Node naming

ContainerLab names each container predictably:

```
clab-<lab-name>-<node-name>
```

So a node `r1` in a lab named `mylab` becomes the container `clab-mylab-r1`. You will use these names constantly — to open a shell, read logs, or copy files:

```bash
docker exec -it clab-mylab-r1 vtysh
```

> **Tip.** Keep lab names short and avoid dashes in them. ContainerLab uses the dash to separate the lab name from the node name, so a dash inside the lab name makes container names harder to read.

---

## The lab lifecycle

Working with a lab is a simple loop of four verbs:

```
deploy   →   inspect   →   (use the lab)   →   destroy
```

- **deploy** — read the topology file, start containers, wire links, build the management network.
- **inspect** — list the running nodes, their kinds, and their management IPs.
- **destroy** — stop and remove the containers and clean up the links.

```bash
sudo containerlab deploy -t mylab.clab.yml
sudo containerlab inspect -t mylab.clab.yml
sudo containerlab destroy -t mylab.clab.yml --cleanup
```

The next chapter runs this loop end-to-end on a real two-node lab.

---

## The lab directory

When you deploy, ContainerLab creates a working directory next to your topology file, named `clab-<lab-name>`. It holds per-node runtime files, generated TLS certificates, and other state. You normally leave it alone; `destroy --cleanup` removes it.

---

## Key Takeaways

- A lab is one `.clab.yml` file with a `name` and a `topology` of `nodes` and `links`.
- A **node** is a container; its **kind** tells ContainerLab how to run it.
- A **link** is a `veth` cable between two `node:interface` endpoints; `eth0` is reserved for management.
- Every lab gets an automatic **management network** for reaching nodes.
- Containers are named `clab-<lab>-<node>`.
- The workflow is **deploy → inspect → destroy**.
