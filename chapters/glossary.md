# Glossary

Key terms used throughout this book and in the ContainerLab ecosystem.

---

## A

**Alpine**
A tiny Linux distribution often used as a base image for host nodes.

**Ansible inventory**
A file listing hosts for Ansible to manage. ContainerLab writes one (`ansible-inventory.yml`) into each lab's working directory after deploy.

**ASN (Autonomous System Number)**
A number identifying a BGP speaker. In a CLOS fabric, each device usually gets its own ASN.

---

## B

**BGP (Border Gateway Protocol)**
A path-vector routing protocol. The standard choice for the underlay of a leaf–spine fabric, with eBGP between tiers.

**bind (bind mount)**
Mounting a host file or directory into a container. In ContainerLab, the `binds` node property — the main way to deliver config such as `frr.conf`.

**bridge**
A virtual Layer 2 switch. ContainerLab's management network is a Docker bridge.

---

## C

**cEOS**
Arista's containerized EOS. Runs under the `arista_ceos` kind; free but requires an Arista account to download.

**CLOS fabric**
A multi-tier, fully-meshed topology (leaf–spine) that provides redundant equal-cost paths. Named after Charles Clos.

**clab**
The short command alias for `containerlab`.

**containerlab (clab)**
The tool this book is about: it builds network labs from containers described in a YAML topology file.

---

## D

**daemons file**
FRR's `/etc/frr/daemons`, which selects the routing daemons to run (zebra, ospfd, bgpd, …) and enables `vtysh`.

**deploy**
The ContainerLab action that reads a topology file and builds the lab (containers, links, management network).

**destroy**
The action that stops and removes a lab; `--cleanup` also deletes its working directory.

**Docker**
The container runtime ContainerLab uses to run nodes.

---

## E

**eBGP (external BGP)**
BGP between different ASNs. Used between leaves and spines in a fabric.

**ECMP (Equal-Cost Multi-Path)**
Load-sharing traffic across multiple equal-cost next-hops — e.g. a leaf reaching another leaf through both spines.

**endpoints**
The two ends of a link, written `["node:interface", "node:interface"]`.

**eth0**
The reserved management interface of a node. Data links start at `eth1`.

---

## F

**FRR (FRRouting)**
An open-source routing stack (OSPF, BGP, IS-IS, and more), the successor to Quagga. Runs under `kind: linux`; this book's router.

**frr.conf**
FRR's integrated configuration file (`/etc/frr/frr.conf`), holding interfaces, addresses, and routing config.

---

## G

**generate**
The ContainerLab command that scaffolds a multi-tier fabric topology file from a short description.

**graph**
The ContainerLab command that renders a diagram of a topology.

---

## I

**image**
The container image a node runs, e.g. `quay.io/frrouting/frr:latest`.

**inspect**
The ContainerLab command that lists a lab's nodes, kinds, states, and management IPs.

---

## K

**kind**
The type of a node (`linux`, `nokia_srlinux`, `arista_ceos`, …). It controls how ContainerLab starts and manages the node and how interfaces are named.

---

## L

**leaf**
An access-tier switch in a CLOS fabric; hosts connect to leaves.

**link**
A virtual cable (`veth` pair) between two node interfaces.

**linux (kind)**
The generic kind for any container; used here for both FRR routers and Linux hosts.

---

## M

**macvlan**
An extended link type that binds a node interface to a physical host NIC, placing the node on the real LAN.

**management network**
The dedicated Docker bridge ContainerLab builds so you can reach every node (via `eth0`); separate from data links.

---

## N

**node**
One device in a lab — a container with a name, kind, and image.

**NOS (Network Operating System)**
The software a network device runs (FRR, SR Linux, cEOS, SONiC, …).

---

## O

**OSPF (Open Shortest Path First)**
A link-state interior routing protocol; used in this book's first routed lab.

---

## P

**prefix**
(1) A network/mask such as `172.16.1.0/24`. (2) The ContainerLab top-level `prefix` key controlling container naming.

---

## S

**save**
The ContainerLab command that snapshots nodes' running configuration (for supported kinds).

**spine**
A core-tier switch in a CLOS fabric; every leaf connects to every spine.

**SR Linux**
Nokia's container-native NOS (`nokia_srlinux` kind); free and ContainerLab's flagship.

---

## T

**topology file**
The YAML file (`*.clab.yml`) describing a lab's nodes and links.

---

## V

**veth (virtual Ethernet pair)**
Two linked virtual interfaces; the mechanism behind a ContainerLab point-to-point link.

**vrnetlab**
A project that wraps VM-based network OS images in containers so ContainerLab can run them.

**vtysh**
FRR's integrated CLI, used to configure and inspect routers (`docker exec -it <node> vtysh`).

---

## Z

**zebra**
FRR's core daemon; it programs routes into the kernel and must always be enabled.
