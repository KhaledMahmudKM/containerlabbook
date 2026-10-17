# Installation and Setup

ContainerLab needs two things: a Linux host and Docker. This chapter installs both, then ContainerLab itself, and verifies the result with a one-line version check.

---

## Prerequisites

- A Linux host (bare metal, VM, WSL2, or a cloud instance). Ubuntu and Debian are used in the examples; other distributions work too.
- `sudo` / root access.
- Internet access to pull packages and container images.

ContainerLab is a single Go binary with no runtime dependencies of its own, so installation is quick.

---

## Step 1 — Install Docker

ContainerLab uses Docker as its container runtime. If Docker is already installed, skip to Step 2.

The simplest method is Docker's official convenience script:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Verify Docker is running:

```bash
sudo docker version
sudo systemctl status docker
```

To run Docker (and ContainerLab) without `sudo`, add your user to the `docker` group and start a new session:

```bash
sudo usermod -aG docker $USER
# log out and back in, or:
newgrp docker
docker run --rm hello-world
```

> **Security note.** Membership in the `docker` group grants root-equivalent access to the host. That is fine on a personal lab machine; be deliberate about it on shared systems.

---

## Step 2 — Install ContainerLab

ContainerLab also provides a convenience script that detects your platform and installs the latest release:

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
```

This installs the `containerlab` binary (with a short alias, `clab`) into your `PATH`.

### Alternative: package managers

On Debian/Ubuntu you can install from ContainerLab's apt repository, and on RHEL-family systems from the yum repository, which makes upgrades go through your normal package manager. The install script prints the exact commands for your distribution, and the official docs list them under *Installation*.

---

## Step 3 — Verify the installation

```bash
containerlab version
```

You should see the installed version, commit, and build date. The alias works identically:

```bash
clab version
```

From here on, this book uses `containerlab` in full for clarity, but `clab` is interchangeable.

---

## Do you need sudo?

ContainerLab creates network namespaces and virtual interfaces, which require elevated privileges. In practice:

- If your user is in the `docker` group, many commands work without `sudo`.
- Some operations still need root. If a deploy fails with a permissions error, re-run it with `sudo`.

This book shows `sudo` where it is commonly required. Using it when it is not needed does no harm.

---

## Upgrading

ContainerLab can upgrade itself:

```bash
sudo containerlab version upgrade
```

Or re-run the install script, or use your package manager if you installed that way. Check your version against the latest periodically — the project moves quickly.

---

## Uninstalling

To remove the binary installed by the script:

```bash
sudo rm -f $(which containerlab)
```

If you installed via apt/yum, use `sudo apt remove containerlab` or `sudo yum remove containerlab`. Docker is separate; remove it independently if you no longer need it.

---

## Pre-pulling the lab images (optional)

The labs in this book use two free images. Pulling them now means your first deploy is instant:

```bash
docker pull quay.io/frrouting/frr:latest
docker pull wbitt/network-multitool:latest
```

- `quay.io/frrouting/frr` — the FRRouting stack (zebra, ospfd, bgpd, …), used for routers.
- `wbitt/network-multitool` — a small Linux image with `ping`, `ip`, `curl`, `traceroute`, and friends preinstalled, used for hosts. Plain `alpine:latest` also works if you install tools yourself.

> **Pin for reproducibility.** `:latest` always works, but for a lab you will revisit, pin a version — for example `quay.io/frrouting/frr:10.0` — so results do not drift when a new image is published. Check the image's registry page for current tags.

---

## Key Takeaways

- ContainerLab needs a Linux host and Docker.
- Install Docker with its convenience script, then ContainerLab with `get.containerlab.dev`.
- `containerlab` and `clab` are the same command.
- Some operations need `sudo`; being in the `docker` group covers most.
- Pre-pull `frrouting/frr` and `network-multitool` to make the first lab instant.
