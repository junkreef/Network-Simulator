# Contributing

> 🇯🇵 日本語版はこちら → [contributing.ja.md](./contributing.ja.md)

Thank you for your interest in contributing to Network Simulator! This guide covers the repository structure, development workflow, testing requirements, and contribution standards.

---

## Repository Structure

The project is organized as a **main repository with two Git submodules**:

```
network-simulator/                    ← main repo
├── frontend/                         ← submodule (git@github.com:junkreef/Network-Simulator-frontend.git)
├── backend/                          ← submodule (git@github.com:junkreef/Network-Simulator-backend.git)
├── examples/                         ← example topology configurations
├── docs/                             ← project-wide documentation
├── README.md
├── README.ja.md
└── .gitmodules
```

Each submodule is an independent Git repository:

- **Frontend submodule** (`frontend/`) — React 18, TypeScript, React Flow, Xterm.js, Zustand, Vite, Vitest, Playwright. Hosted at `git@github.com:junkreef/Network-Simulator-frontend.git`.
- **Backend submodule** (`backend/`) — FastAPI, Pydantic, Docker SDK, Containerlab CLI, FRR, Jinja2. Hosted at `git@github.com:junkreef/Network-Simulator-backend.git`.

> Changes to `frontend/` or `backend/` must be committed in the submodule first, then the submodule reference in the main repo must be updated.

---

## Development Setup

Clone with submodules:

```bash
git clone --recurse-submodules <repo-url>
cd network-simulator
```

**Frontend:**

```bash
cd frontend
npm install
npm run dev
```

**Backend:**

```bash
cd backend
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
.venv/bin/uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload
```

Refer to [Getting Started](./getting-started.md) for full setup details and troubleshooting.

---

## Test-First Requirement

**Always define your test verification steps _before_ making code changes.**

The required workflow is:

```
Define Test Steps → Execute Changes → Run Tests → Fix if Failed → Repeat
```

This means:
1. Read the relevant test files to understand existing coverage.
2. Write or identify the test(s) that verify your change.
3. Confirm the tests fail (red) before your change.
4. Implement the change.
5. Confirm the tests pass (green).

Do **not** skip this process for "trivial" changes — this discipline prevents regressions and keeps the test suite trustworthy.

---

## Running Tests

### Frontend — Unit / Integration (Vitest)

```bash
cd frontend
npm test
```

### Frontend — End-to-End (Playwright)

```bash
cd frontend
npm run test:e2e
```

> Playwright E2E tests require both the frontend and backend to be running. Start both servers before running E2E tests.

### Backend — Unit / Integration (pytest)

```bash
cd backend
.venv/bin/pytest
```

For verbose output:

```bash
.venv/bin/pytest -v
```

For a specific test file:

```bash
.venv/bin/pytest tests/test_topology.py -v
```

---

## Code Style

### Frontend (TypeScript)

- **TypeScript strict mode** is enabled — no implicit `any`, all types must be explicit.
- **ESLint** is configured — run before committing:
  ```bash
  cd frontend
  npx eslint src/
  ```

### Backend (Python)

- Follow standard Python conventions (PEP 8).
- **pylint** is available for linting:
  ```bash
  cd backend
  .venv/bin/pylint src/
  ```

---

## Commit Guidelines

- Write **clear, concise commit messages** that describe *what* changed and *why*.
- Use the imperative mood for the subject line: `Add OSPF area type support`, not `Added OSPF area types`.
- Keep the subject line under 72 characters; add a body if more context is needed.
- Do **NOT** add `Co-authored-by:` trailers or auto-generated footers to commit messages.

### Committing to Submodules

When your change touches `frontend/` or `backend/`:

1. Commit your changes inside the submodule:
   ```bash
   cd frontend   # or backend
   git add .
   git commit -m "Your message"
   git push
   ```

2. Update the submodule reference in the main repo:
   ```bash
   cd ..   # back to network-simulator/
   git add frontend   # or backend
   git commit -m "Update frontend submodule to include <description>"
   git push
   ```

---

## Pull Request Checklist

Before opening a pull request, verify all of the following:

- [ ] All relevant tests pass (`npm test`, `npm run test:e2e`, `.venv/bin/pytest`)
- [ ] New behavior is covered by tests
- [ ] Documentation is updated if the change affects user-facing behavior or developer setup
- [ ] No secrets, credentials, or API keys are committed
- [ ] TypeScript types are explicit (no `any` suppressions without justification)
- [ ] Commit messages are clear and do not include auto-generated trailers
- [ ] Submodule reference in the main repo is updated if a submodule changed

---

## Navigation

- ← [Back to README](../README.md)
- ← [Getting Started](./getting-started.md)
- ← [User Guide](./user-guide.md)
