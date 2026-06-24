# Project Run & Test Guide

## 1. Project Overview & Architecture
This project is a web-based network topology simulation and learning environment that runs entirely on container-based virtualization (Docker & Containerlab).

### Core Components:
- **Frontend** (`/frontend`): Built with React, React Flow (for topology canvas), and Xterm.js (for terminal interface).
- **Backend** (`/backend`): Built with FastAPI, communicating with Containerlab on the host system to orchestrate container configurations.
- **State Management**: Fully stateless architecture. The React Flow UI maintains the full configuration state and POSTs the entire topology JSON to the backend during deployment.

### Key Workflows:
- **Topology Sync & Zero-Downtime Config**:
  - The backend dynamically renders Jinja2 templates to generate Containerlab's `topology.clab.yml`.
  - For FRR routers, it generates `frr.conf` and executes `frr-reload.py` inside the Alpine containers to apply OSPF, RIP, and BGP rules without rebooting.
  - L2 Switches are configured by creating a VLAN-filtering-enabled bridge (`br0`) inside Alpine containers and dynamically mapping ports to Access or Trunk modes via CLI commands.
- **Web Terminal Proxy**:
  - WebSocket connections from the frontend's Xterm.js are proxied by FastAPI directly to the Docker container's `/bin/sh` shell process (stdin/stdout bidirectional pipe).

## 2. System Components & Port Mapping
- **Frontend**: React + React Flow + Xterm.js (running on `http://localhost:5173`)
- **Backend**: FastAPI + Containerlab + FRR (running on `http://localhost:8000`)

## 3. Command Reference

### Frontend (`/frontend`)
- **Start Dev Server**: `npm run dev`
- **Build**: `npm run build`
- **Unit / Integration Tests**: `npm test` (Vitest)
- **E2E Tests**: `npm run test:e2e` (Playwright)

### Backend (`/backend`)
- **Start Server**: `.venv/bin/uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload`
- **Run pytest**: `.venv/bin/pytest` (runs unit & integration tests)

## 4. Past Troubleshooting & Gotchas (Important)
- **UI Key Input & Focus Conflicts**:
  Xterm.js focus state may occasionally conflict with normal text inputs (such as inputting IP addresses, interface names, or the CIDR slash `/` character). Ensure input components correctly capture focus without terminal listeners swallowing key events.
- **WebSocket & Terminal Integration**:
  Verifying terminal behavior via `docker exec` in testing is not equivalent to WebSocket validation. The actual frontend-backend WebSocket path must be verified using the websocket terminal proxy tests.
- **Terminal Default Gateway**:
  Ensure default gateway configuration correctly translates to routing tables inside host containers.
- **Test-First Requirement**:
  Always define the test verification steps *before* initiating changes, following: Define Steps -> [Execute -> Fix if failed] loop.

## 5. Detailed Manual E2E Verification
Refer to [e2e_manual_test_procedure.md](file:///home/junkreef/.gemini/antigravity-cli/brain/94ac0d8a-cd86-4715-81ee-b512f2cae4be/e2e_manual_test_procedure.md) for step-by-step E2E setup, Switch node addition, port reconnection, VLAN configuration, deployment, and runtime status checking.
