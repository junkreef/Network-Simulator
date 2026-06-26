# Network Simulator

> 🇯🇵 日本語版はこちら → [README.ja.md](./README.ja.md)

**A web-based network topology simulation environment — draw, configure, and deploy real containerized networks right from your browser.**

![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-009688?logo=fastapi&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-required-2496ED?logo=docker&logoColor=white)
![Containerlab](https://img.shields.io/badge/Containerlab-latest-00BFA5)
![FRR](https://img.shields.io/badge/FRR-Free_Range_Routing-4CAF50)

> 📸 Screenshot / demo coming soon

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) — container runtime
- [Containerlab](https://containerlab.dev/install/) — network lab orchestration
- [Node.js](https://nodejs.org/) ≥ 18 — frontend build toolchain
- [Python](https://www.python.org/downloads/) ≥ 3.10 — backend runtime
- [Git](https://git-scm.com/) — with submodule support

---

## Quick Start

1. **Clone the repository** (including submodules):

   ```bash
   git clone --recurse-submodules <repo-url> && cd network-simulator
   ```

2. **Terminal 1 — Start the frontend:**

   ```bash
   cd frontend && npm install && npm run dev
   ```

3. **Terminal 2 — Start the backend:**

   ```bash
   cd backend && python -m venv .venv && pip install -r requirements.txt && .venv/bin/uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload
   ```

4. **Open your browser** → [http://localhost:5173](http://localhost:5173)

---

## Documentation

| Document | Description |
|---|---|
| [Getting Started](./docs/getting-started.md) | Full setup guide with troubleshooting |
| [Architecture](./docs/architecture/index.md) | System design & data flow diagrams |
| [User Guide](./docs/user-guide.md) | How to build and run network topologies |
| [Contributing](./docs/contributing.md) | Development workflow & contribution guide |

---

## Repository Structure

This repository uses Git submodules to separate frontend and backend concerns:

```
network-simulator/          ← main repo (this repo)
├── frontend/               ← submodule: React + TypeScript + React Flow + Xterm.js
├── backend/                ← submodule: FastAPI + Containerlab + FRR
├── examples/               ← example topology configurations
├── docs/                   ← project-wide documentation
├── README.md
├── README.ja.md
└── .gitmodules
```

- **Frontend** ([`frontend/`](./frontend/)): React 18, TypeScript, React Flow (topology canvas), Xterm.js (web terminal), Zustand state management, Vite, Vitest, Playwright.
- **Backend** ([`backend/`](./backend/)): FastAPI, Pydantic, Docker SDK, Containerlab CLI, FRR (Free Range Routing), Jinja2 templating.

---

## License

License information coming soon.
