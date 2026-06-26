> 🇯🇵 日本語版はこちら → [index.ja.md](./index.ja.md)

# Architecture Overview

Network Simulator is a web-based learning environment that lets users design network topologies visually and deploy them as real, running Docker containers — all from a browser. The guiding philosophy is **"real networking, zero friction"**: every router, switch, and terminal is a live container running actual routing software (FRR), connected by real virtual Ethernet (veth) pairs, accessible through a browser-native terminal.

The system is designed around a **stateless backend**. The React frontend owns all topology configuration state and sends the complete topology description to the backend on every deploy or configure action. The backend is a pure executor — it renders configs, invokes Containerlab, runs `docker exec` commands, and returns results. This design means the frontend is always the single source of truth, and partial re-configurations can be applied to individual nodes without redeploying the whole topology.

A key architectural feature is **zero-downtime configuration**. Router settings (OSPF, RIP, BGP, static routes, IP addresses) can be changed and applied to running containers without restarting any daemon. This is achieved by using FRR's `frr-reload.py` tool, which diffs the old and new configurations and applies only the delta via `vtysh`.

## Navigation

| Document | Description |
|---|---|
| [system-overview.md](./system-overview.md) | Detailed component diagram, technology stack table, port map, container images, and `data/` directory layout |
| [data-flows.md](./data-flows.md) | Step-by-step walkthroughs of all four major data flows: topology deployment, router configuration, L2 switch configuration, and WebSocket terminal proxy |
| [state-management.md](./state-management.md) | How the Zustand store owns topology state, how state is persisted to files, and why the backend is stateless |

## High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                          Browser                             │
│                                                             │
│   Draws topology, configures nodes, opens web terminals     │
└───────────────────┬─────────────────────────────────────────┘
                    │  REST/JSON  (HTTP POST/GET)
                    │  WebSocket  (terminal I/O)
                    ▼
┌─────────────────────────────────────────────────────────────┐
│              React Frontend  (:5173)                         │
│                                                             │
│  React Flow canvas · Zustand state store · Xterm.js         │
└───────────────────┬─────────────────────────────────────────┘
                    │  HTTP + WebSocket
                    ▼
┌─────────────────────────────────────────────────────────────┐
│              FastAPI Backend  (:8000)                        │
│                                                             │
│  REST API routers · WebSocket proxy · Pydantic validation   │
└───────────────────┬─────────────────────────────────────────┘
                    │  Python function calls
                    ▼
┌─────────────────────────────────────────────────────────────┐
│                    Orchestrator                              │
│                                                             │
│  Jinja2 template rendering · Docker SDK · subprocess calls  │
└──────────┬────────────────────────────────┬─────────────────┘
           │ writes YAML + frr.conf          │ Docker API / CLI
           ▼                                ▼
    data/ directory              ┌──────────────────────────┐
    (configs, state)             │     Containerlab CLI      │
                                 └──────────┬───────────────┘
                                            │ creates/manages
                                            ▼
                                 ┌──────────────────────────┐
                                 │    Docker Containers      │
                                 │  (Routers, Switches,      │
                                 │   Terminals) + veth pairs │
                                 └──────────────────────────┘
```

---

*See also: [../../README.md](../../README.md)*
