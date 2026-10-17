# The Topology File in Depth

The topology file is the heart of ContainerLab. You met its basic shape in the first lab; this chapter covers the pieces you will actually reach for: the top-level keys, the `nodes`/`kinds`/`defaults` hierarchy, node properties, and the two forms of links.

---

## Top-level structure

```yaml
name: mylab            # required — names the lab
prefix: ""             # optional — controls container naming
mgmt:                  # optional — management network settings
  network: clab-mgmt
  ipv4-subnet: 172.20.20.0/24
topology:              # required — the lab itself
  defaults: { }        # values applied to every node
  kinds: { }           # values applied per kind
  nodes: { }           # the devices
  links: [ ]           # the cables
```

Only `name` and `topology` (with `nodes`) are required. Everything else is optional refinement.

### name and prefix

`name` distinguishes this lab from others on the host and becomes part of every container name. By default containers are named `clab-<name>-<node>`. Setting `prefix: ""` removes the `clab-` prefix; most people leave the default.

### mgmt

The `mgmt` block customizes the management network — its Docker bridge name and its subnet. You need this only when the default `172.20.20.0/24` clashes with something, or when you want predictable management IPs.

---

## The three levels: defaults, kinds, nodes

ContainerLab lets you set a property in three places, from most general to most specific. More specific wins.

```yaml
topology:
  defaults:
    kind: linux                      # every node is linux unless it says otherwise
  kinds:
    linux:
      image: wbitt/network-multitool # every linux node uses this image by default
  nodes:
    h1: {}                           # inherits kind=linux, image=network-multitool
    h2: {}                           # same
    r1:
      image: quay.io/frrouting/frr:latest   # overrides just the image
```

This keeps big topologies DRY: declare the common image once under `kinds`, and only the nodes that differ carry their own settings. In this example `h1` and `h2` need no body at all — they inherit everything.

---

## Node properties

A node supports many properties. The ones you will use most with `linux`-kind nodes:

| Property | Purpose |
|---|---|
| `kind` | Device type (here, `linux`) |
| `image` | Container image to run |
| `binds` | Mount host files/dirs into the container |
| `env` | Environment variables |
| `ports` | Publish container ports to the host |
| `cmd` | Override the container's command |
| `exec` | Run commands inside the node after it starts |
| `labels` | Arbitrary metadata |

### binds

`binds` mount host paths into the container — the main way you deliver configuration files (you will use this heavily for FRR):

```yaml
    r1:
      kind: linux
      image: quay.io/frrouting/frr:latest
      binds:
        - r1/frr.conf:/etc/frr/frr.conf
        - r1/daemons:/etc/frr/daemons
```

Paths on the left are relative to the topology file; paths on the right are inside the container.

### env, ports, exec

```yaml
    web:
      kind: linux
      image: nginx
      env:
        MY_VAR: hello
      ports:
        - "8080:80"            # host:container
      exec:
        - "echo started > /tmp/ready"
```

- `env` sets environment variables inside the node.
- `ports` publishes a container port to the host, handy for reaching a web UI from your browser.
- `exec` runs one or more commands inside the node right after it starts — useful for small runtime tweaks.

---

## Links: brief and extended forms

### Brief form

The common case — a cable between two node interfaces:

```yaml
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]
    - endpoints: ["r1:eth2", "r3:eth1"]
```

Each entry is one link. `eth0` is management, so data interfaces start at `eth1`. Number them however you like; many people match interface numbers to the neighbor for readability.

### Extended form

For special link types, the extended form names a `type` and its parameters:

```yaml
  links:
    - type: veth
      endpoints:
        - node: r1
          interface: eth1
        - node: r2
          interface: eth1
    - type: mgmt-net
      endpoint:
        node: r1
        interface: eth9
      host-interface: clab-r1-mgmt
    - type: host
      endpoint:
        node: r1
        interface: eth8
      host-interface: r1-host
```

Supported types include `veth` (the default point-to-point cable), `mgmt-net`, `macvlan`, `host`, `vxlan`, and `vxlan-stitched`. For almost all labs the brief `veth` form is all you need; the others connect a node to the host, to a physical interface, or across machines. Chapter 7 returns to links in practice.

---

## A fuller example

Putting the pieces together — two FRR routers and a host, with the common image declared once:

```yaml
name: triangle
topology:
  defaults:
    kind: linux
  kinds:
    linux:
      image: wbitt/network-multitool
  nodes:
    r1:
      image: quay.io/frrouting/frr:latest
      binds:
        - r1/frr.conf:/etc/frr/frr.conf
        - r1/daemons:/etc/frr/daemons
    r2:
      image: quay.io/frrouting/frr:latest
      binds:
        - r2/frr.conf:/etc/frr/frr.conf
        - r2/daemons:/etc/frr/daemons
    h1: {}
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]   # r1 — r2 backbone
    - endpoints: ["r1:eth2", "h1:eth1"]   # host hangs off r1
```

`h1` inherits the multitool image; the routers override it with FRR and mount their configs. The next chapters fill in those config files and bring the routers to life.

---

## Validating before you deploy

A quick way to catch YAML mistakes is to let ContainerLab parse the file and show the graph it would build:

```bash
sudo containerlab inspect -t triangle.clab.yml   # after deploy
```

Editors with the ContainerLab/YAML extension also flag schema errors as you type, which is the fastest feedback loop.

---

## Key Takeaways

- The file has a `name` and a `topology`; `mgmt` and `prefix` are optional.
- Set properties at three levels — `defaults`, `kinds`, `nodes` — with the most specific winning.
- Key node properties: `image`, `binds`, `env`, `ports`, `exec`.
- `binds` mount host files into a node and are how you deliver config.
- Links are usually the brief `endpoints: ["a:eth1","b:eth1"]` form; the extended form adds special link types.
