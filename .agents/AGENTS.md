# Project Run & Test Guide

## 1. System Components & Port Mapping
- **Frontend**: React + React Flow + Xterm.js (running on `http://localhost:5173`)
- **Backend**: FastAPI + Containerlab + FRR (running on `http://localhost:8000`)

## 2. Command Reference

### Frontend (`/frontend`)
- **Start Dev Server**: `npm run dev`
- **Build**: `npm run build`
- **Unit / Integration Tests**: `npm test` (Vitest)
- **E2E Tests**: `npm run test:e2e` (Playwright)

### Backend (`/backend`)
- **Start Server**: `.venv/bin/uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload`
- **Run pytest**: `.venv/bin/pytest` (runs unit & integration tests)

## 3. Past Troubleshooting & Gotchas (Important)
- **UI Key Input & Focus Conflicts**:
  Xterm.js focus state may occasionally conflict with normal text inputs (such as inputting IP addresses, interface names, or the CIDR slash `/` character). Ensure input components correctly capture focus without terminal listeners swallowing key events.
- **WebSocket & Terminal Integration**:
  Verifying terminal behavior via `docker exec` in testing is not equivalent to WebSocket validation. The actual frontend-backend WebSocket path must be verified using the websocket terminal proxy tests.
- **Terminal Default Gateway**:
  Ensure default gateway configuration correctly translates to routing tables inside host containers.
- **Test-First Requirement**:
  Always define the test verification steps *before* initiating changes, following: Define Steps -> [Execute -> Fix if failed] loop.

## 4. Detailed Manual E2E Verification
Refer to [e2e_manual_test_procedure.md](file:///home/junkreef/.gemini/antigravity-cli/brain/94ac0d8a-cd86-4715-81ee-b512f2cae4be/e2e_manual_test_procedure.md) for step-by-step E2E setup, Switch node addition, port reconnection, VLAN configuration, deployment, and runtime status checking.
