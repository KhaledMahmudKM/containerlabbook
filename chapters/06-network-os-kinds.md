# Network Operating Systems and Node Kinds

ContainerLab can run many network operating systems, each represented by a **kind**. This chapter maps the landscape — what runs natively, what needs a license, and what needs a VM wrapper — and then focuses on the two free building blocks this book uses: FRR routers and Linux hosts.

---

## Kinds, revisited

A node's `kind` tells ContainerLab how to start and manage it. Each kind encodes a device's quirks: its management user, how interfaces are named, and how configuration is pushed. Choosing the right kind is usually just a matter of matching it to your image.

Kinds fall into three practical groups:

1. **Container-native NOS** — ship as container images, run directly. Fast and light.
2. **Generic Linux** — any container at all, via `kind: linux`.
3. **VM-based NOS** — only exist as virtual machines, wrapped in a container with `vrnetlab`.

---

## Container-native network OSes

These run as plain containers and are ContainerLab's sweet spot:

| NOS | Kind | Image availability |
|---|---|---|
| Nokia SR Linux | `nokia_srlinux` | Free, public (`ghcr.io/nokia/srlinux`) |
| Arista cEOS | `arista_ceos` | Free, but requires an Arista account to download |
| SONiC (virtual) | `sonic-vs` | Free, public |
| Cisco XRd | `cisco_xrd` | Requires a Cisco account/entitlement |
| Juniper cRPD | `juniper_crpd` | Requires a Juniper account |

> **Nokia SR Linux** is ContainerLab's flagship — the tool and SR Linux come from the same team, and its image is freely pullable. If you later want a vendor-grade NOS with no licensing friction, it is the natural next step beyond FRR. This book stays on FRR so every lab is runnable with zero sign-ups, but the topology mechanics are identical: swap the `kind` and `image`, and the links and workflow are unchanged.

---

## VM-based network OSes (vrnetlab)

Some images — classic Cisco IOS XE/IOSv, Juniper vMX/vSRX, Palo Alto, and others — only exist as full virtual machines. ContainerLab can still run them using **vrnetlab**, which packages a VM inside a container so ContainerLab can treat it like any other node.

This is powerful but heavier: you supply the vendor image, build the container, and accept VM-level memory and boot times. It is out of scope for this book's free, lightweight labs, but it is why people say ContainerLab can run "almost anything."

---

## The `linux` kind — our workhorse

`kind: linux` runs any ordinary container and wires it into the topology. We use it for two roles.

### FRR routers

**FRRouting (FRR)** is a mature open-source routing stack — the successor to Quagga — implementing OSPF, BGP, IS-IS, RIP, and more. It ships as a container image and runs beautifully under `kind: linux`:

```yaml
    r1:
      kind: linux
      image: quay.io/frrouting/frr:latest
      binds:
        - r1/frr.conf:/etc/frr/frr.conf
        - r1/daemons:/etc/frr/daemons
```

FRR gives you real routing protocols with a Cisco-like CLI (`vtysh`) and zero licensing. Chapter 8 configures it properly; Chapter 10 builds a routed fabric with it.

### Linux hosts

For endpoints — the "PCs" that send traffic through your network — use a small Linux image:

```yaml
    h1:
      kind: linux
      image: wbitt/network-multitool
```

`wbitt/network-multitool` comes with `ping`, `traceroute`, `curl`, `ip`, `tcpdump`, and more, so you can test connectivity immediately. Plain `alpine:latest` works too if you install tools with `apk add`.

---

## How interfaces are named on linux nodes

For `linux` nodes, the interface you name in a link is the interface that appears in the container. `eth0` is the management interface; your data links use `eth1`, `eth2`, and so on:

```yaml
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]   # becomes eth1 in each container
```

Container-native NOSes name interfaces their own way — SR Linux uses `e1-1`, `e1-2`; cEOS uses `eth1`, `eth2`. The `kind` is what makes ContainerLab use the correct names, which is one more reason the kind matters.

---

## A restart caveat for linux nodes

If a `linux` container exits with an error, Docker's restart policy brings it back — but the restarted container **loses its data-plane interfaces**, because those `veth`s were injected by ContainerLab and are not recreated automatically. In practice this is rare for FRR and host images, but if a node seems to lose its links after a crash, redeploy the lab rather than fighting the single container.

---

## Choosing for your lab

- **Learning routing protocols, no licenses?** FRR (`linux`) + hosts (`linux`). This book.
- **Want a vendor NOS, still free?** Add Nokia SR Linux (`nokia_srlinux`).
- **Need a specific vendor image you own?** cEOS (`arista_ceos`) or a VM via vrnetlab.

The beauty of the topology file is that switching is a two-line change — `kind` and `image` — so you can start on FRR and graduate to SR Linux without relearning ContainerLab.

---

## Key Takeaways

- A node's **kind** matches its image and controls startup, users, and interface names.
- Container-native NOSes (SR Linux, cEOS, SONiC) run directly; SR Linux is free and the flagship.
- VM-only NOSes run via **vrnetlab**, at VM weight.
- `kind: linux` runs FRR routers and Linux hosts — this book's two free building blocks.
- On linux nodes, `eth0` is management and data links start at `eth1`.
