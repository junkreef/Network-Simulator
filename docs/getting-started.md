# Getting Started

> 🇯🇵 日本語版はこちら → [getting-started.ja.md](./getting-started.ja.md)

This guide walks you through installing prerequisites, cloning the repository, and running the frontend and backend locally.

---

## Prerequisites

Install each of the following before proceeding:

| Tool | Version | Install |
|---|---|---|
| Docker | Latest | [Docker Desktop](https://docs.docker.com/get-docker/) |
| Containerlab | Latest | [Containerlab docs](https://containerlab.dev/install/) |
| Node.js | ≥ 18 | [nodejs.org](https://nodejs.org/en/download) |
| Python | ≥ 3.10 | [python.org](https://www.python.org/downloads/) |
| Git | Any | [git-scm.com](https://git-scm.com/) |

Verify your installations:

```bash
docker --version
containerlab version
node --version
python3 --version
git --version
```

---

## Clone the Repository

This project uses **Git submodules** — the frontend and backend are separate repositories linked from the main repo. The `--recurse-submodules` flag is required to clone them together:

```bash
git clone --recurse-submodules <repo-url>
cd network-simulator
```

> **What are submodules?** A Git submodule is a reference to a specific commit in another repository. The `frontend/` and `backend/` directories are independent repos included here at pinned versions.

If you already cloned without `--recurse-submodules`, run:

```bash
git submodule update --init --recursive
```

---

## Frontend Setup

```bash
cd frontend
npm install
npm run dev
```

Expected output:

```
  VITE v5.x.x  ready in XXX ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
```

The React development server is now running on **http://localhost:5173**.

---

## Backend Setup

```bash
cd backend
python -m venv .venv
```

Activate the virtual environment:

- **Linux / macOS:**
  ```bash
  source .venv/bin/activate
  ```
- **Windows:**
  ```bat
  .venv\Scripts\activate
  ```

Install dependencies and start the server:

```bash
pip install -r requirements.txt
.venv/bin/uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload
```

Expected output:

```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Started reloader process
```

The FastAPI server is now running on **http://localhost:8000**.

---

## Verification

With both servers running, confirm everything is working:

1. **Open the UI** → [http://localhost:5173](http://localhost:5173)
   You should see the topology canvas with a node palette on the left.

2. **Check the backend API:**
   ```bash
   curl http://localhost:8000/api/v1/topology/status
   ```
   Expected response:
   ```json
   {"status": "stopped"}
   ```

---

## Troubleshooting

### Docker is not running

**Symptom:** Backend starts but deployment fails with a Docker connection error.

**Fix:** Ensure Docker Desktop (or the Docker daemon) is running:
```bash
docker ps
```
If this errors, start Docker and retry.

---

### Port conflicts (5173 or 8000 already in use)

**Symptom:** Address already in use error on startup.

**Fix:** Find and stop the conflicting process:
```bash
# Find what's using port 5173
lsof -i :5173

# Find what's using port 8000
lsof -i :8000
```

Alternatively, change the port:
- Frontend: `npm run dev -- --port 3000`
- Backend: change `--port 8000` in the uvicorn command

---

### Submodules not initialized

**Symptom:** `frontend/` or `backend/` directories are empty.

**Fix:**
```bash
git submodule update --init --recursive
```

---

### `pip install` fails on requirements

**Symptom:** Dependency resolution errors or missing system libraries.

**Fix:** Ensure you're using Python ≥ 3.10 and have activated the virtual environment before running `pip install`.

---

## Next Steps

- ← [Back to README](../README.md)
- → [User Guide](./user-guide.md) — learn how to build and deploy topologies
