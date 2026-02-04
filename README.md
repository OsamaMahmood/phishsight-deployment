# PhishSight Deployment

Deployment and orchestration repo for the PhishSight platform -- a multi-tenant SaaS phishing email analysis tool. This repo does **not** contain application source code. It provides Docker Compose configurations and environment templates to run all PhishSight services together for development or production.

The actual application code lives in separate repositories that are cloned into this directory by the setup script.

## Repository Layout

```
PhishSight/                        # This repo (deployment)
├── dev-deploy/                    # Development docker-compose + env template
│   ├── docker-compose.yml
│   └── .env.example
├── prod-deploy/                   # Production docker-compose + env template
│   ├── docker-compose.yml
│   └── .env.example
├── deployment-setup.sh            # Clones/pulls all app repos into this directory
│
│  ── Cloned by deployment-setup.sh ──
├── brain/                         # Backend API (Express.js + Prisma)
├── app/                           # Frontend dashboard (Next.js 14)
└── phishsight.com/                # Marketing site (Next.js 14)
```

## Services

| Service         | Description                    | Dev Port | Prod Port          |
|-----------------|--------------------------------|----------|---------------------|
| Backend API     | Express.js REST API            | 3001     | 3001 (via Traefik)  |
| Frontend App    | Next.js dashboard              | 3002     | 3000 (via Traefik)  |
| Marketing Site  | Next.js landing/marketing page | 3003     | 3000 (via Traefik)  |
| PostgreSQL      | Primary database               | 5432     | Internal only       |
| Redis           | Cache + session store          | 6379     | Internal only       |

---

## Quick Start (Development)

### Prerequisites

- Docker and Docker Compose
- Git with SSH access to the PhishSight GitHub org

### 1. Clone the app repositories

This deployment repo does not include the application source code. Run the setup script to clone them:

```bash
chmod +x deployment-setup.sh
./deployment-setup.sh
```

This clones three repos into the current directory:

| Repo | Cloned To | Description |
|------|-----------|-------------|
| `PhishSight/brain` | `brain/` | Backend API |
| `PhishSight/app` | `app/` | Frontend dashboard |
| `PhishSight/phishsight.com` | `phishsight.com/` | Marketing site |

If the directories already exist, the script pulls the latest changes instead. Use `--clone-only` to skip pulling.

### 2. Configure environment

```bash
cd dev-deploy
cp .env.example .env
```

Edit `.env` and fill in:
- **Required**: `POSTGRES_PASSWORD`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `SUPER_ADMIN_EMAIL`, `SUPER_ADMIN_PASSWORD`
- **Optional**: `GOOGLE_CLIENT_ID` (Google OAuth), `VIRUSTOTAL_API_KEY` (threat intel), `OPENROUTER_API_KEY` (AI analysis), `ZEPTOMAIL_API_KEY` (transactional email)

Most services work without optional keys -- features that depend on them degrade gracefully.

### 3. Start all services

```bash
cd dev-deploy
docker compose up --build
```

This starts PostgreSQL, Redis, the backend API, the frontend app, and the marketing site. On first run, the backend automatically:
1. Waits for PostgreSQL and Redis to be healthy
2. Runs Prisma migrations
3. Seeds the database (creates super admin user)
4. Starts the dev server with hot-reload

### 4. Access the services

| Service        | URL                          |
|----------------|------------------------------|
| Frontend App   | http://localhost:3002         |
| Marketing Site | http://localhost:3003         |
| Backend API    | http://localhost:3001         |
| API Health     | http://localhost:3001/health  |

Login with the `SUPER_ADMIN_EMAIL` / `SUPER_ADMIN_PASSWORD` you configured.

### Hot-Reload

The dev docker-compose mounts source code as volumes. Changes to files in `src/` (backend), `app/`, `components/`, `lib/` (frontend/site) are picked up automatically without rebuilding containers.

---

## Production Deployment

Use `prod-deploy/` to build production Docker images and test them locally, or deploy to a server.

### Prerequisites

- Docker and Docker Compose
- For server deployment: a domain pointed to your server + Traefik reverse proxy

### 1. Clone app repos (if not done already)

```bash
./deployment-setup.sh
```

### 2. Configure environment

```bash
cd prod-deploy
cp .env.example .env
```

Fill in **ALL** required values. Key differences from dev:
- Use strong, randomly generated secrets (`openssl rand -hex 64` for JWT secrets)
- Set real domain URLs for `FRONTEND_URL`, `APP_URL`, `MARKETING_SITE_URL`
- Configure `ZEPTOMAIL_API_KEY` for transactional emails
- Set `REDIS_PASSWORD` (dev uses no password)
- Configure AWS S3 credentials for file storage

### 3. Build and deploy

```bash
cd prod-deploy
CACHEBUST=$(date +%s) docker compose up --build -d
```

The `CACHEBUST` variable ensures Docker doesn't serve stale cached source code layers. Always pass it when deploying updates.

### 4. Post-deployment

- After the first successful deployment, set `RUN_SEED=false` in `.env` to prevent re-seeding on container restarts
- Monitor logs: `docker compose logs -f api`
- Check health: `curl https://api.yourdomain.com/health`

### Production Architecture

```
Internet
    |
    v
 Traefik (SSL termination + routing)
    |
    |-- app.phishsight.com  --> Frontend App (port 3000)
    |-- api.phishsight.com  --> Backend API (port 3001)
    '-- phishsight.com      --> Marketing Site (port 3000, optional)
            |
            v
     +----------------+
     |  PostgreSQL     |  (internal network only)
     |  Redis          |
     +----------------+
```

Production services are NOT exposed to the host directly. Traefik routes traffic via the `traefik-frontend` Docker network.

---

## Build-Time vs Runtime Environment Variables

### Backend (Express.js)

All backend env vars are **runtime** -- set them in docker-compose `environment` section or `.env` file. Changes take effect on container restart.

### Frontend / Marketing Site (Next.js)

`NEXT_PUBLIC_*` variables are **baked into the JavaScript bundle at build time**. They are declared as `ARG` + `ENV` in each Dockerfile's builder stage.

**If you change a `NEXT_PUBLIC_*` variable, you MUST rebuild the Docker image:**

```bash
CACHEBUST=$(date +%s) docker compose up --build -d
```

Setting them only at runtime has no effect.

#### Frontend App build-time vars:
- `NEXT_PUBLIC_API_URL` -- Backend API URL (internal Docker DNS in prod: `http://api:3001`)
- `NEXT_PUBLIC_APP_URL` -- Public app URL for SEO
- `NEXT_PUBLIC_SITE_URL` -- Marketing site URL for cross-linking
- `TURNSTILE_SITEKEY` -- Cloudflare bot protection
- `NEXT_PUBLIC_GOOGLE_CLIENT_ID` -- Google OAuth
- `NEXT_PUBLIC_GA_MEASUREMENT_ID` -- Google Analytics
- `NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION` -- Search Console

#### Marketing Site build-time vars:
- `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_APP_URL`, `NEXT_PUBLIC_SITE_URL`
- `NEXT_PUBLIC_TURNSTILE_SITEKEY`, `NEXT_PUBLIC_GA_MEASUREMENT_ID`, `NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION`

---

## Database Migrations

Migrations run automatically on container start via the backend entrypoint script.

To create a new migration during development:

```bash
cd brain
npx prisma migrate dev --name descriptive_name
npx prisma generate
```

In production, `npx prisma migrate deploy` runs on every container start.

---

## Common Tasks

### Rebuild a single service

```bash
cd dev-deploy  # or prod-deploy
docker compose up --build -d api    # just the backend
docker compose up --build -d app    # just the frontend
docker compose up --build -d site   # just the marketing site
```

### View logs

```bash
docker compose logs -f api          # backend logs
docker compose logs -f app          # frontend logs
docker compose logs -f postgres     # database logs
```

### Access the database

```bash
# Dev (password from .env POSTGRES_PASSWORD)
docker compose exec postgres psql -U phishsight -d phishsight

# Or use Prisma Studio
cd brain
npx prisma studio
```

### Reset the database (dev only)

```bash
cd dev-deploy
docker compose down -v              # removes volumes (all data)
docker compose up --build
```

### Pull latest code for all repos

```bash
./deployment-setup.sh
```

The script detects existing repos and pulls the latest changes. It stashes any uncommitted local changes before pulling and restores them after.

---

## App Repository Documentation

Each cloned repo has its own detailed README:
- `brain/README.md` -- Backend API architecture, routes, services
- `app/README.md` -- Dashboard app structure, components
- `phishsight.com/README.md` -- Marketing site pages, SEO
