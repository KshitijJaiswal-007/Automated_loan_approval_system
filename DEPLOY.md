# Deployment (Docker & GitHub)

This document describes how the repository builds Docker images and how CI publishes them to GitHub Container Registry (GHCR).

Build & publish using GitHub Actions
- On push to `main` or `master`, the workflow `.github/workflows/docker-deploy.yml` builds two images and pushes them to GHCR:
  - backend -> `ghcr.io/<OWNER>/automated-loan-approval-backend:latest`
  - frontend -> `ghcr.io/<OWNER>/automated-loan-approval-frontend:latest`

Local testing with Docker Compose

1. Build and run locally:

```bash
docker compose build
docker compose up
```

2. API will be exposed at `http://localhost:8000` (FastAPI/Uvicorn).
   Frontend served via Nginx at `http://localhost:3000`.

Notes & gotchas
- The backend Dockerfile expects `backend/model.joblib` and copies `backend/requirements.txt`.
- The frontend Dockerfile produces the Next.js `.next` output and serves `public/`. The `Dockerfile.frontend` copies `.next` and `public` into the runner image. If your Next.js output or custom server differs, update `Dockerfile.frontend` accordingly.
- CI pushes images to GHCR using `secrets.GITHUB_TOKEN` and `docker/login-action`.

Quick GHCR publish (manual)

```bash
# Authenticate
echo $GITHUB_TOKEN | docker login ghcr.io -u <USERNAME> --password-stdin

# Build images
docker build -f Dockerfile.backend -t ghcr.io/<OWNER>/automated-loan-approval-backend:local .
docker build -f Dockerfile.frontend -t ghcr.io/<OWNER>/automated-loan-approval-frontend:local .

# Push if desired
docker push ghcr.io/<OWNER>/automated-loan-approval-backend:local
docker push ghcr.io/<OWNER>/automated-loan-approval-frontend:local
```

Replace `<OWNER>` with your GitHub user or organization name.
