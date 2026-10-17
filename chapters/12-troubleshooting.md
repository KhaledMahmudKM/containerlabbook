# Troubleshooting, Tips, and Best Practices

Labs go wrong — a typo in YAML, an image that will not pull, a protocol that will not come up. This chapter collects the errors you are most likely to hit, how to diagnose them, and the habits that keep labs healthy.

---

## Deploy fails immediately

### YAML / schema errors

ContainerLab parses the file before doing anything. A syntax error stops it with a message pointing near the problem. Common causes:

- Inconsistent indentation (YAML is whitespace-sensitive — use spaces, never tabs).
- A missing `kind` or `image` on a node.
- A malformed link — `endpoints` must be a two-item list of `"node:iface"` strings.

Editors with the ContainerLab/YAML extension catch most of these as you type. When in doubt, compare against a known-good file.

### Permission errors

Creating namespaces and `veth` pairs needs privilege. If deploy fails with a permissions error, run it with `sudo`. If Docker itself is unreachable, confirm the daemon is running (`sudo systemctl status docker`) and that your user is in the `docker` group.

### Image cannot be pulled

```
Error response from daemon: pull access denied ... / manifest unknown
```

Check the image name and tag. A wrong tag (for example a version that does not exist) gives `manifest unknown`; pre-pull the image manually to see the real error:

```bash
docker pull quay.io/frrouting/frr:latest
```

---

## A node will not start or keeps restarting

Read its logs first:

```bash
docker logs clab-<lab>-<node>
```

For FRR, a crash loop often means a bad `frr.conf` or `daemons` file. Check that:

- `daemons` has `zebra=yes` and the protocol daemon you need (`ospfd`/`bgpd`) set to `yes`.
- `frr.conf` is valid — a single bad line can stop a daemon.

> **Remember the restart caveat.** If a `linux` node exits and Docker restarts it, it comes back **without its data interfaces**. If a node loses its links after a crash, redeploy the lab (`destroy` then `deploy`) rather than restarting the one container.

---

## Links or interfaces missing inside a node

If `ip address` inside a node does not show the `eth1`/`eth2` you expected:

- Confirm the interface is actually referenced in a `links:` entry — ContainerLab only creates interfaces that appear in a link.
- Confirm you did not collide with `eth0` (management); data links start at `eth1`.
- If the node restarted, see the caveat above.

---

## Routing protocol will not come up

A methodical path for OSPF/BGP that will not peer:

1. **Layer 1/2 — is the link up and addressed?**
   ```bash
   docker exec clab-<lab>-r1 vtysh -c "show interface brief"
   ```
2. **Can the two routers reach each other on the link subnet?**
   ```bash
   docker exec clab-<lab>-r1 ping -c 2 10.0.0.2
   ```
3. **Is the daemon even running?** Check `daemons` has the protocol enabled, then:
   ```bash
   docker exec clab-<lab>-r1 vtysh -c "show ip ospf neighbor"
   docker exec clab-<lab>-r1 vtysh -c "show bgp summary"
   ```
4. **Do the configs agree?** Mismatched areas (OSPF) or wrong `remote-as` / neighbor addresses (BGP) are the usual culprits. `show running-config` shows what the device actually loaded.

Work bottom-up: a BGP session cannot form if the two routers cannot ping each other on the link.

---

## Cleaning up

### A normal teardown

```bash
sudo containerlab destroy -t mylab.clab.yml --cleanup
```

### When a lab is "stuck"

If a deploy was interrupted and leaves containers behind, destroy by name or clean everything:

```bash
sudo containerlab destroy -t mylab.clab.yml --cleanup
# or, list and remove leftover containers:
docker ps -a | grep clab-
```

Leftover `clab-*` working directories can be removed by hand if `--cleanup` did not catch them.

---

## Resource tips

ContainerLab labs are light, but they are not free:

- **RAM** is the usual limit. FRR and host containers are small (tens of MB each), so even a laptop runs sizeable fabrics — but VM-based nodes (vrnetlab) are heavy; size accordingly.
- **Disk** fills with images over time. `docker image prune` and `docker system prune` reclaim space.
- **Many labs at once** share the host's management subnets; give each lab a distinct `name` so container names and the `clab-<name>` directories do not collide.

---

## Versioning and reproducibility

- **Pin image tags.** `:latest` drifts; a pinned tag (e.g. `quay.io/frrouting/frr:10.0`) makes a lab reproduce identically next year.
- **Keep ContainerLab current**, but note the version in your lab's README so others match it.
- **Commit everything** — topology and per-node config — so a clone of the repo deploys the same lab.

---

## Good habits, in short

- Start small; add nodes once the basics work.
- Name nodes by role, keep lab names short and dash-free.
- Comment links with the two roles they join.
- Use `defaults`/`kinds` to avoid repetition.
- `write memory` good config back to files, then commit.
- Destroy labs you are not using — free the RAM for the next experiment.

---

## Key Takeaways

- Deploy failures are usually YAML errors, missing `sudo`, or a bad image tag — the message and `docker logs` point the way.
- A crash-looping FRR node almost always has a bad `daemons` or `frr.conf`.
- Troubleshoot protocols bottom-up: link up → ping across → daemon running → configs agree.
- `destroy --cleanup` is the clean teardown; prune images to reclaim disk.
- Pin image tags and commit topology + config for reproducible labs.
