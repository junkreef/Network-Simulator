> 🇯🇵 日本語版はこちら → [system-overview.ja.md](./system-overview.ja.md)

# System Components & Technology Stack

## Component Diagram

```
┌─────────────────────────────────────────────────────────┐
│                      Browser                             │
│  ┌──────────────────────────────────────────────────┐   │
│  │              React Frontend (:5173)               │   │
│  │  ┌──────────────┐  ┌──────────┐  ┌───────────┐  │   │
│  │  │  React Flow  │  │ Zustand  │  │ Xterm.js  │  │   │
│  │  │   (Canvas)   │  │  Store   │  │(Terminal) │  │   │
│  │  └──────┬───────┘  └────┬─────┘  └─────┬─────┘  │   │
│  └─────────┼───────────────┼──────────────┼─────────┘   │
│            │ REST/JSON     │              │ WebSocket    │
└────────────┼───────────────┼──────────────┼─────────────┘
             ▼               ▼              ▼
┌─────────────────────────────────────────────────────────┐
│              FastAPI Backend (:8000)                      │
│  ┌──────────────────────┐   ┌────────────────────────┐  │
│  │   REST API Router    │   │  WebSocket Router      │  │
│  │  /api/v1/topology/*  │   │  /ws/terminal/{node}   │  │
│  │  /api/v1/nodes/*     │   └──────────┬─────────────┘  │
│  └──────────┬───────────┘              │                 │
│             ▼                          ▼                 │
│  ┌──────────────────────────────────────────────────┐   │
│  │               Orchestrator                        │   │
│  │  deploy_topology() │ configure_node()             │   │
│  │  get_runtime_info()│ get_topology_status()        │   │
│  └─────┬──────────────┴──────────────────┬──────────┘   │
│        │                                  │              │
│        ▼ Jinja2                           ▼ Docker SDK   │
│  ┌──────────────┐              ┌──────────────────────┐  │
│  │  Templates   │              │    Docker Daemon      │  │
│  │frr.conf.j2   │              └──────────┬───────────┘  │
│  │topology.clab │                         │              │
│  │  .yml.j2     │              ┌──────────▼───────────┐  │
│  └──────────────┘              │  Containerlab CLI    │  │
│        │                       └──────────┬───────────┘  │
└────────┼───────────────────────────────────┼─────────────┘
         │ writes files                      │ manages
         ▼                                  ▼
┌──────────────────────┐   ┌─────────────────────────────┐
│    data/ directory   │   │    Docker Containers        │
│  topology.clab.yml   │   │  ┌─────────┐ ┌──────────┐  │
│  sim-network/        │   │  │alpine-  │ │alpine-   │  │
│    r1/frr.conf       │   │  │frr      │ │switch    │  │
│    r1/daemons        │   │  │(Router) │ │(Switch)  │  │
│    r1/vtysh.conf     │   │  └────┬────┘ └────┬─────┘  │
│  topology_state.json │   │       │veth pairs  │         │
└──────────────────────┘   │  ┌───▼──────┐     │         │
                            │  │alpine-   │     │         │
                            │  │terminal  │◄────┘         │
                            │  │(Host PC) │               │
                            │  └──────────┘               │
                            └─────────────────────────────┘
```

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Frontend UI | React 18 + TypeScript | Component framework |
| Topology Canvas | React Flow | Visual node/edge graph — drag, connect, pan, zoom |
| State Management | Zustand | Lightweight client-side state store with subscriptions |
| Web Terminal | Xterm.js | Full-featured browser terminal emulator |
| Build Tool | Vite | Frontend dev server with Hot Module Replacement (HMR) |
| Testing (FE) | Vitest + Playwright | Unit/integration tests + end-to-end browser tests |
| API Server | FastAPI (Python) | REST endpoints and WebSocket handlers |
| Data Validation | Pydantic | Request/response schema validation and serialization |
| Container Runtime | Docker | Container lifecycle management via Python SDK |
| Network Topology | Containerlab | Declarative container-based network topology orchestration |
| Routing Software | FRR (Free Range Routing) | OSPF, RIP, BGP routing protocol daemons inside containers |
| Templating | Jinja2 | Rendering `topology.clab.yml` and `frr.conf` from templates |
| Testing (BE) | pytest | Unit and integration tests for backend logic |

## Port & URL Mapping

| Service | URL | Note |
|---|---|---|
| Frontend Dev Server | http://localhost:5173 | Vite HMR — changes reflect immediately without page reload |
| Backend API (REST) | http://localhost:8000/api/v1 | FastAPI — all REST endpoints under this prefix |
| WebSocket Terminal | ws://localhost:8000/ws/terminal/{node_name} | One WebSocket connection per node |
| Backend API Docs | http://localhost:8000/docs | FastAPI auto-generated Swagger UI |

## Container Images

Each node type in the simulator runs a purpose-built Docker image:

| Image | Node Type | Key Software |
|---|---|---|
| `alpine-frr:latest` | Router | FRR daemons (zebra, bgpd, ospfd, ripd), vtysh, bash |
| `alpine-switch:latest` | L2 Switch | iproute2 (bridge, ip), bash — kernel bridge for VLAN switching |
| `alpine-terminal:latest` | Terminal / Host PC | bash, iproute2, ping, curl, traceroute |

All images are based on Alpine Linux and kept minimal. Bash is installed in all images to support the WebSocket terminal proxy (which spawns `/bin/bash`).

## Configuration Directory Layout (`data/`)

The `data/` directory lives at the root of the repository and is the shared persistence layer between the backend and the running containers. Router containers mount subdirectories from `data/` at startup to receive their initial FRR configuration.

```
data/
├── topology.clab.yml              # Rendered Containerlab topology definition
│                                  # Consumed by: containerlab deploy / inspect / destroy
│
├── topology_state.json            # Saved React Flow UI state (nodes + edges)
│                                  # Written by: saveState(deployed=false)
│                                  # Read by: loadState() on page load
│
├── topology_deployed_state.json   # Snapshot of UI state at the last successful deploy
│                                  # Written by: saveState(deployed=true) after deploy
│                                  # Used to compute hasChanges (diff vs current state)
│
└── sim-network/                   # Per-topology config dir (name = topology name field)
    ├── r1/                        # Per-router config dir (name = node label)
    │   ├── daemons                # FRR daemon enable/disable list — read at container start
    │   ├── frr.conf               # FRR routing config — persisted by configure_node()
    │   │                          # (also used as startup config via frr.conf.import mount)
    │   └── vtysh.conf             # vtysh integration config (service integrated-vtysh-config)
    ├── r2/
    │   └── ...
    └── r3/
        └── ...
```

### Mount Points in Router Containers

Each router container (`alpine-frr:latest`) has three read-only bind mounts configured in `topology.clab.yml`:

| Host Path | Container Path | Purpose |
|---|---|---|
| `data/{topo}/{node}/daemons` | `/etc/frr/daemons` | Tells FRR which daemons to start |
| `data/{topo}/{node}/frr.conf` | `/etc/frr/frr.conf.import` | Initial routing config loaded at startup |
| `data/{topo}/{node}/vtysh.conf` | `/etc/frr/vtysh.conf` | Enables `vtysh` integrated configuration mode |

> **Note**: The host-side `frr.conf` is mounted as `/etc/frr/frr.conf.import` (not `/etc/frr/frr.conf`) at startup. After the container is running, `configure_node()` writes new config directly to `/etc/frr/frr.conf` inside the container via `docker exec`, and also updates the host-side `frr.conf` for persistence across container restarts.

## FRR Daemon Configuration

Every router node is pre-configured with the following FRR daemon states (written to the `daemons` file during `deploy_topology()`):

| Daemon | State | Protocol |
|---|---|---|
| `zebra` | **yes** | Kernel routing table manager (required by all other daemons) |
| `bgpd` | **yes** | BGP (Border Gateway Protocol) |
| `ospfd` | **yes** | OSPFv2 (Open Shortest Path First for IPv4) |
| `ripd` | **yes** | RIPv2 (Routing Information Protocol) |
| `ospf6d` | no | OSPFv3 (IPv6) |
| `ripngd` | no | RIPng (IPv6) |
| `isisd` | no | IS-IS |
| `pimd` | no | PIM (Multicast) |
| `ldpd` | no | LDP (MPLS Label Distribution) |
| `nhrpd` | no | NHRP (Next Hop Resolution) |
| `eigrpd` | no | EIGRP |
| `babeld` | no | Babel routing protocol |
| `sharpd` | no | SHARP (testing daemon) |
| `pbrd` | no | PBR (Policy-Based Routing) |
| `bfdd` | no | BFD (Bidirectional Forwarding Detection) |
| `fabricd` | no | OpenFabric |
| `vrrpd` | no | VRRP (Virtual Router Redundancy) |
| `pathd` | no | SRTE Path Management |

All files and directories under `data/` are created with `chmod 777` to ensure the container processes (which run as non-root users inside Alpine) can read them.

---

*Navigation: [Architecture Index](./index.md) · [Data Flows](./data-flows.md) · [State Management](./state-management.md)*
