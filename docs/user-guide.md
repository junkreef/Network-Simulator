# User Guide

> 🇯🇵 日本語版はこちら → [user-guide.ja.md](./user-guide.ja.md)

This guide explains how to use the Network Simulator to build, configure, and run containerized network topologies.

---

## Interface Overview

The application is divided into four main areas:

| Area | Location | Purpose |
|---|---|---|
| **Header (Toolbar)** | Top bar | Deploy / Destroy buttons, global actions |
| **Canvas** | Center (main area) | Draw and connect topology nodes |
| **Property Panel** | Right sidebar | Configure the selected node |
| **Web Terminal** | Bottom panel | In-browser shell into a running container |

---

## Adding Nodes

Nodes represent physical (or virtual) network devices. There are three node types:

- **Router** — an FRR-based Alpine container supporting OSPF, RIP, and BGP
- **L2 Switch** — an Alpine container with a Linux VLAN bridge
- **Terminal** — a plain Alpine container acting as an end-host

To add a node, **drag it from the node palette** (left sidebar) onto the canvas. You can also right-click an empty area of the canvas to open a context menu with node options.

Each node displays its name and a status badge (shown after deployment).

---

## Connecting Nodes

Connections represent physical cables between device interfaces.

1. Hover over a node — connection **handles** (small dots) appear on its edges.
2. Click and drag from one handle to a handle on another node.
3. The resulting edge represents a point-to-point link (veth pair in the container world).

Edges can be deleted by selecting them and pressing `Delete` or `Backspace`.

---

## Configuring Routers

Select a Router node to open its configuration in the Property Panel.

### Interfaces

Set the IP address for each interface (`eth1` through `eth4`):

- Enter the address in **CIDR notation**: `192.168.1.1/24`
- Each connected edge maps to one interface, in connection order.

### VLAN Subinterfaces

Create dot1q subinterfaces (e.g., `eth1.10`):

1. Click **Add Subinterface**.
2. Set **Parent Interface** (e.g., `eth1`), **VLAN ID** (e.g., `10`), and **IP Address** (CIDR).

### OSPF

1. Enable the **OSPF toggle**.
2. Set **Router ID** (e.g., `1.1.1.1`).
3. Add **Areas**:
   - **Area ID** (e.g., `0` for backbone, `1` for stub, etc.)
   - **Networks** in CIDR notation (e.g., `192.168.1.0/24`)
   - **Area Type**: `normal`, `stub`, `totally-stub`, `nssa`, or `totally-nssa`
4. Optional — **Route Aggregation (range)**: summarize networks within an area.
5. Optional — **Redistribute**: inject routes from `connected`, `static`, `rip`, or `bgp` into OSPF.
6. Optional — **default-information originate**: advertise a default route into OSPF.

### RIP

1. Enable the **RIP toggle**.
2. Add **Networks** (CIDR) to include in RIP advertisements.
3. Optional — **Redistribute**: inject `connected`, `static`, `ospf`, or `bgp` routes.

### BGP

1. Enable the **BGP toggle**.
2. Set **AS Number** and **Router ID**.
3. Add **Neighbors**:
   - **IP Address** of the peer
   - **Remote AS** number
4. Optional — **Redistribute**: inject `connected`, `static`, `ospf`, or `rip` routes.

### Static Routes

Add one or more static routes:

- **Destination**: CIDR notation (e.g., `10.0.0.0/8`)
- **Next Hop**: IP address of the gateway (e.g., `192.168.1.254`)

### Default Gateway

Enter the IP address of the default gateway for this router. This populates a `0.0.0.0/0` static route.

---

## Configuring L2 Switches

Select an L2 Switch node to configure its ports. Each port corresponds to a connected edge.

| Mode | Description |
|---|---|
| **None** | Port is inactive (no VLAN configuration applied) |
| **Access** | Port carries traffic for a single VLAN; set one **VLAN ID** (1–4094) |
| **Trunk** | Port carries multiple VLANs; set a comma-separated list of **VLAN IDs** |

Example trunk VLANs: `10,20,30`

> Internally, the switch uses a Linux bridge (`br0`) with VLAN filtering enabled inside an Alpine container.

---

## Configuring Terminals

Select a Terminal node. Configure:

- **IP Address** — in CIDR notation (e.g., `192.168.1.100/24`)
- **Default Gateway** — IP address of the upstream router interface (e.g., `192.168.1.1`)

---

## Deploying the Topology

When you are satisfied with your topology:

1. Click **Deploy** in the header toolbar.
2. The frontend sends the full topology JSON to `POST /api/v1/topology/deploy`.
3. The backend:
   - Renders a `topology.clab.yml` from a Jinja2 template.
   - Calls `containerlab deploy` to create Docker containers and veth pairs.
4. Node status badges update to show **up** (green) or **down** (red) once containers are running.

> Deployment creates real Docker containers. Ensure Docker and Containerlab are running beforehand.

---

## Applying Node Configuration

After deployment, node configuration (IP addresses, routing protocols) is applied separately:

1. Select a node in the canvas.
2. In the Property Panel, click **Apply**.
3. The backend applies the configuration inside the running container **without restarting it**:
   - **Routers**: renders `frr.conf` and runs `frr-reload.py` inside the container.
   - **Switches**: configures the Linux bridge and VLAN port mappings via shell commands.
   - **Terminals**: sets the IP address and default route.

You can modify configuration and re-apply at any time while the topology is deployed.

---

## Web Terminal

Every running container can be accessed through an in-browser terminal:

1. Click the **terminal icon** (📟) on a node.
2. The bottom panel opens an **Xterm.js** terminal connected to `/bin/sh` inside the container.
3. You have full interactive shell access.

Useful commands:

```bash
# Check interface IP addresses
ip addr

# View the routing table
ip route

# Open the FRR routing CLI (routers only)
vtysh

# Ping another node
ping 192.168.1.1

# Show ARP table
ip neigh
```

To close the terminal, click the **×** on the terminal panel.

---

## Viewing Runtime Status

Select a deployed node to view live runtime information in the Property Panel:

| Tab | Information |
|---|---|
| **Routing Table** | Active routes (`ip route show`) |
| **ARP Table** | Neighbor MAC/IP mappings (`ip neigh`) |
| **OSPF Neighbors** | OSPF adjacency state (`vtysh -c "show ip ospf neighbor"`) |
| **BGP Summary** | BGP session state and prefix counts |
| **RIP Status** | RIP routing table and neighbor status |

---

## Destroying the Topology

To remove all running containers and clean up network interfaces:

1. Click **Destroy** in the header toolbar.
2. The backend calls `containerlab destroy`, removing all containers and veth pairs.
3. Node status badges reset to **stopped**.

> Destroying the topology does **not** delete your canvas configuration. You can redeploy at any time.

---

## Navigation

- ← [Getting Started](./getting-started.md)
- → [Architecture](./architecture/index.md)
- → [Contributing](./contributing.md)
