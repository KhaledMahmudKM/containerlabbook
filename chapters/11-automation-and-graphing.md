# Automation, Graphing, and Lab-as-Code

Because a lab is just a file, it fits naturally into automation. This chapter covers the tools that make ContainerLab more than a manual process: drawing topology graphs, generating fabrics, feeding automation tools like Ansible, and running labs in CI.

---

## Drawing the topology: graph

ContainerLab can render a picture of your topology so you (and your students) can see the shape of a lab at a glance:

```bash
sudo containerlab graph -t clos.clab.yml
```

By default this starts a small local web server and serves an interactive diagram of the nodes and links in your browser. It is generated straight from the topology file, so the picture is always in sync with the lab — no hand-drawn diagrams to keep updated. It is ideal for documentation and for explaining a lab before deploying it.

---

## Generating fabrics: generate

Typing out a large CLOS fabric by hand is tedious. The `generate` command builds a topology file for you from a short description of the tiers:

```bash
sudo containerlab generate \
  --name fabric \
  --kind linux \
  --image quay.io/frrouting/frr:latest \
  --nodes 2,4
```

`--nodes 2,4` means a two-tier fabric with 2 nodes in the first tier (spines) and 4 in the second (leaves), fully meshed between tiers. The command prints — or, with the right flag, writes — a ready `.clab.yml` you can then customize with addresses and configs. It is the fastest way to scaffold a big fabric before filling in the details from Chapter 10.

---

## Feeding Ansible

When you deploy a lab, ContainerLab writes an **Ansible inventory** into the lab's working directory (`clab-<lab>/ansible-inventory.yml`). It groups nodes by kind and lists their management addresses, so you can point a playbook at your lab with no manual inventory:

```bash
ansible-playbook -i clab-ospf/ansible-inventory.yml configure.yml
```

This closes a nice loop with your Ansible book: ContainerLab builds the topology, and Ansible configures the devices in it. For a classroom, students can practice real automation against a disposable fabric that costs nothing to rebuild.

---

## Saving configurations: save

For supported node kinds, ContainerLab can snapshot the running configuration of every node at once:

```bash
sudo containerlab save -t clos.clab.yml
```

For FRR specifically, the cleaner path you already know is `write memory` inside `vtysh`, which — because `frr.conf` is bind-mounted — saves straight back to your config file on the host. Either way, the goal is the same: capture good config so the next deploy reproduces it.

---

## Lab as code: the workflow

Put it together and a lab becomes a Git repository:

```
clos-fabric/
├── clos.clab.yml          # the topology
├── spine1/ … leaf2/       # per-node config
└── README.md              # how to run it
```

A healthy workflow:

1. **Edit** the topology or a config file.
2. **Deploy** and test.
3. **`write memory`** to persist good config back to the files.
4. **Commit** the change — it shows up as a readable diff.
5. **Share** the repo; anyone runs `containerlab deploy` and gets the identical lab.

This is the same discipline your GitHub and Ansible books teach, applied to network labs: versioned, reviewable, reproducible.

---

## Running labs in CI

Because deploy and destroy are single commands, a CI pipeline can stand up a lab, test it, and tear it down automatically. A sketch of a pipeline job:

```bash
# 1. bring the lab up
sudo containerlab deploy -t clos.clab.yml

# 2. run checks (exit non-zero on failure)
docker exec clab-clos-h1 ping -c 3 -W 2 172.16.2.10
docker exec clab-clos-leaf1 vtysh -c "show bgp summary" | grep -q 65001

# 3. always clean up
sudo containerlab destroy -t clos.clab.yml --cleanup
```

Wire that into GitHub Actions (on a Linux runner with Docker) and every change to a config file is validated automatically — the network equivalent of unit tests. Tests like the ping and the BGP check above catch a broken config before it is merged.

---

## Templating and reuse

Topology files are plain text, so the usual reuse tricks apply:

- **`defaults` and `kinds`** keep common settings in one place (you used these in Chapter 10).
- **Environment variables** can be referenced in a topology file and expanded at deploy time, so one file can target different images or counts.
- **Per-node folders** of config files keep large labs organized and diff-friendly.

Start simple — a plain file in Git is already "lab as code." Reach for generation and templating when a lab grows large enough to earn it.

---

## Key Takeaways

- `containerlab graph` renders an always-accurate topology diagram in your browser.
- `containerlab generate` scaffolds large fabrics from a tiers description.
- Deploy writes an Ansible inventory, so playbooks can configure the lab with no manual setup.
- `containerlab save` (or FRR's `write memory`) persists running config.
- Single-command deploy/destroy makes labs testable in CI; commit topology and config to Git for true lab-as-code.
