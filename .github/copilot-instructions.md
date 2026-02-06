<!-- Copilot instructions for Automated_loan_approval_system -->
# Quick Orientation for AI coding agents

This project contains a FastAPI Python backend with an ML model and a Next.js frontend. The repo is dockerized and CI builds/pushes images to GHCR.

- Big picture:
  - Backend: `backend/app.py` (FastAPI) loads `backend/model.joblib` and persists decision audits to `backend/audit.db` (SQLite).
  - Frontend: Next.js app in `frontend/` (see `frontend/package.json`).
  - Orchestration: `docker-compose.yml` builds `Dockerfile.backend` and `Dockerfile.frontend` and maps ports 8000 (api) and 3000 (frontend).
  - CI: `.github/workflows/docker-deploy.yml` builds and pushes both images to GHCR on push to `main`/`master`.

- Key files to inspect when changing behaviour:
  - Backend runtime + model: `backend/app.py`, `backend/requirements.txt`, `backend/model.joblib`
  - Backend image: `Dockerfile.backend`
  - Frontend app: `frontend/package.json`, `frontend/app/*`, `frontend/public`
  - Frontend image: `Dockerfile.frontend`
  - Compose + local deploy: `docker-compose.yml`, `DEPLOY.md`
  - CI publish: `.github/workflows/docker-deploy.yml`

- Concrete Docker/CI gotchas discovered (actionable details):
  1. `Dockerfile.frontend` currently does `COPY frontend/package.json frontend/package-lock.json ./` then `RUN npm ci`. There is no `frontend/package-lock.json` in the repo — this causes `docker build` to fail. Fix options:
     - Add and commit a `package-lock.json` (preferred for deterministic builds): run `npm install` locally then commit the lockfile.
     - Or modify the Dockerfile to copy only `package.json` and use `RUN npm install --legacy-peer-deps` (or `npm ci` only if lockfile present).
     Example quick patch: replace the COPY line with `COPY frontend/package.json ./` and change `RUN npm ci --legacy-peer-deps` to `RUN npm install --legacy-peer-deps`.

  2. `DEPLOY.md` contains an incorrect note saying the frontend build outputs to `build` — the actual Dockerfile copies the Next.js `.next` folder and `public`. Update docs to match `Dockerfile.frontend`.

  3. `Dockerfile.backend` base image is `python:3.11-slim` (works) but its header comment says 3.13 — minor inconsistency to correct. More important: `shap` (in `backend/requirements.txt`) can require native build tools. On a slim image you may need to install `build-essential`/`gcc` and system libs before `pip install` in the Dockerfile, otherwise CI or local builds may fail.

  4. `docker-compose.yml` mounts `./backend` into the container (`./backend:/app/backend`). Note: a host-mounted volume will mask files baked into the image. Ensure `backend/model.joblib` exists on the host (it does currently) or consider removing the bind mount for production containers.

  5. Start commands and ports:
     - Backend: `uvicorn app:app --host 0.0.0.0 --port 8000` (exposed 8000).
     - Frontend: `npm run start` after `next build` (exposed 3000). The Dockerfiles expect Node 20 (the frontend image uses `node:20-alpine`).

- How to test locally (commands):
```bash
# build and run locally (same as DEPLOY.md)
docker compose build
docker compose up

# API: http://localhost:8000
# Frontend: http://localhost:3000
```

- When editing CI or Dockerfiles, check:
  - The copy paths in Dockerfile frontend/backend (build context is repo root in CI).
  - That `frontend/package-lock.json` exists if `npm ci` is used.
  - That `Dockerfile.backend` installs system build deps if requirements include packages that need compilation (e.g., `shap`).
  - Volume mappings in `docker-compose.yml` and their effect on image contents.

- Suggested immediate fixes to make Docker deployable:
  1. Add `frontend/package-lock.json` (commit lockfile) OR update `Dockerfile.frontend` to use `npm install` when lockfile absent.
  2. Update `Dockerfile.backend` to install build tools when necessary, for example before `pip install`:
     ```Dockerfile
     RUN apt-get update && apt-get install -y build-essential gcc && rm -rf /var/lib/apt/lists/*
     ```
  3. Correct documentation in `DEPLOY.md` to match real Dockerfile behavior.

- Where to look for runtime problems after images build:
  - Backend logs: uvicorn output inside `api` service container.
  - SHAP/model initialization errors: startup prints in `backend/app.py` (model load / explainer initialization).
  - Frontend build errors: `npm run build` output in the `frontend` stage.

- Questions to ask the maintainer before automated edits:
  - Do you want me to: (A) add/commit `package-lock.json`, or (B) patch `Dockerfile.frontend` to use `npm install`? (Prefer A for stable builds.)
  - Should the `backend` image be hardened to install build tools for `shap`, or do you prefer using a prebuilt wheel-only dependency set?

If you want, I can apply the safe patches (fix Dockerfile.frontend copy+install, and add apt install step to `Dockerfile.backend`) and update `DEPLOY.md`. Reply with which fixes you prefer.
