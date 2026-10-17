# Introduction to ContainerLab

ContainerLab is an open-source tool that builds network labs out of containers. You write a short YAML file describing your devices and the cables between them, run `containerlab deploy`, and within seconds you have a running topology you can log into, configure, and test.

---

## The problem it solves

Network engineers have always needed labs — somewhere to try a config, reproduce a bug, or study for a certification without touching production. Traditional options are heavy:

- **Physical gear** is expensive, noisy, and hard to share.
- **VM-based emulators** (GNS3, EVE-NG, VIRL/CML) run a full virtual machine per device. A six-node lab can consume many gigabytes of RAM and take minutes to boot.

ContainerLab takes a different approach. Many modern network operating systems — and plenty of routing stacks like FRR — ship as container images. Containers share the host kernel, so they start in seconds and use a fraction of the memory. A lab that would stress a laptop in EVE-NG often runs comfortably in ContainerLab.

---

## What ContainerLab actually does

ContainerLab is an *orchestrator*. It does not emulate anything itself. Given a topology file, it:

1. Pulls the container images for each node (if not already present).
2. Starts a container for every node.
3. Creates virtual cables (`veth` pairs) between the interfaces you listed.
4. Builds a dedicated management network so you can reach every node.
5. Optionally loads a startup configuration into each device.

When you are done, one command tears it all down and cleans up.

> **Mental model.** Think of ContainerLab as the "docker compose of network labs": a declarative file plus a CLI that makes the file real. The difference is that it also wires point-to-point data links between containers, which compose does not do.

---

## ContainerLab vs GNS3 / EVE-NG

| | ContainerLab | GNS3 / EVE-NG |
|---|---|---|
| Unit of a device | Container | Virtual machine |
| Startup time | Seconds | Tens of seconds to minutes |
| Memory per node | Tens to hundreds of MB | Often 512 MB – several GB |
| Lab definition | A text file (YAML) | A GUI project |
| Version control | Natural — it's just text | Harder — binary project files |
| Best for | Container-native NOS, FRR, Linux, CI | Classic IOS/vendor VM images |

These tools are not strictly competitors. ContainerLab shines when your devices are container-native (Nokia SR Linux, Arista cEOS, FRR, SONiC) or plain Linux. For images that only exist as full VMs, EVE-NG/CML remain convenient — though ContainerLab can run some VM images too, through a helper called `vrnetlab` (covered briefly later).

---

## Why "lab as code" matters

Because a ContainerLab topology is just a YAML file, your entire lab becomes code:

- **Reproducible** — commit it to Git and anyone can deploy the identical lab.
- **Reviewable** — changes show up as diffs in a pull request.
- **Shareable** — send a colleague a file, not a 4 GB appliance export.
- **Automatable** — spin labs up and down in CI to test configs.

This is the same shift that configuration management brought to servers, applied to lab infrastructure. Your other books on Ansible and GitHub fit naturally alongside it.

---

## Typical use cases

- **Learning and certification** — practice OSPF, BGP, VXLAN, or segment routing hands-on.
- **Config validation** — test a change against a copy of your topology before production.
- **Interoperability testing** — connect NOSes from different vendors on one host.
- **Automation development** — a cheap, fast target for Ansible/Nornir playbooks.
- **CI pipelines** — deploy a lab, run tests, tear it down, all unattended.

---

## What you need

To follow along you need a Linux host (a VM or cloud instance is fine) with:

- **Docker** installed and running
- **ContainerLab** installed
- Enough RAM for your labs — the FRR and Linux labs in this book are light, a few hundred MB total

The next chapter walks through installing both.

> **A note on platforms.** ContainerLab runs natively on Linux. On Windows and macOS you run it inside a Linux VM or WSL2, because it relies on Linux networking features (network namespaces, `veth` pairs) to wire the labs.

---

## Key Takeaways

- ContainerLab builds network labs from containers described in a YAML file.
- It orchestrates containers and virtual links; it does not emulate devices itself.
- Compared to VM-based emulators it is dramatically faster and lighter.
- Its file-driven design makes labs reproducible, reviewable, and automatable.
- It runs on Linux and needs Docker; this book's labs use only free images.
