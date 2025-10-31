# Agent Hub - Claude Code Instructions

## Project Structure

Multi-platform chat system with Kubernetes deployment:

- **`/frontend`** - Vue 3 + TypeScript + Vuetify chat interface
- **`/backend`** - FastAPI + SQLModel + Celery for real-time messaging
- **`/frontend_pos`** - Vue 3 + Vuetify POS interface
- **`/backend_pos`** - FastAPI + SQLModel POS API
- **`/kubernetes`** - K8s manifests for complete deployment

## Architecture Overview

**Base domain**: `@deployment_vars.env INGRESS_HOST`

### Ingress & Routing

- **Single Ingress** handles all HTTP/HTTPS traffic with SSL certificates
- **Main routes**:
  - `/` → frontend (main system)
  - `/api/` → backend (main API)
  - `/pos/` → frontend_pos (POS interface)
  - `/pos/api/` → backend_pos (POS API)
- **Subdomains**:
  - `n8n.kareninahub.nomada.dev` → n8n workflow automation

### Main Systems

**1. Main System (Chat/Agent Hub)**

- **Frontend**: Vue 3 + Vuetify at `/`
- **Backend**: FastAPI + SQLModel at `/api/`
- **Database**: Shared PostgreSQL
- **Purpose**: Multi-channel chat system with agents

**2. POS System (Point of Sale)**

- **Frontend POS**: Vue 3 + Vuetify at `/pos/`
- **Backend POS**: FastAPI + SQLModel at `/pos/api/`
- **Database**: Same PostgreSQL (tables: customer, product, sale)
- **Purpose**: Point of sale system

**3. n8n Workflow Automation**

- **URL**: https://n8n.kareninahub.nomada.dev (subdomain)
- **Database**: Same PostgreSQL (database: n8n_db, user: n8n_user)
- **Redis**: Same Redis (db: 1)
- **Purpose**: Workflow automation and integrations

### Shared Authentication Architecture

- **Storage**: localStorage with key `auth_token`
- **Shared domain**: Both systems access same localStorage
- **Flow**:
  1. User authenticates in main system (`/`)
  2. Token saved in localStorage
  3. POS (`/pos/`) automatically uses same token
  4. Both backends validate against same tables (`user`, `token`, `agent`)

### Unified Database

- **PostgreSQL**: `@deployment_vars.env POSTGRES_DB` DB with user `@deployment_vars.env POSTGRES_USER`
- **Shared tables**: `user`, `token`, `agent` (authentication)
- **Main tables**: `chat`, `message`, `channel`, etc.
- **POS tables**: `customer`, `product`, `sale`, etc.
- **n8n database**: `n8n_db` with user `n8n_user` (isolated)

### Support Services

- **Redis**: Cache and task queue
- **Celery**: Asynchronous processing

## Key Files

- **`AGENT_DEPLOY_KUBERNETES.md`** - Step-by-step deployment guide for agents
- **`frontend/src/config/index.ts`** - Environment configuration with `VITE_PROJECT_DOMAIN`
- **`kubernetes/postgres-secret.yaml`** - Database credentials (change before production)

## Deployment

Follow `AGENT_DEPLOY_KUBERNETES.md` for complete Kubernetes deployment with private Docker registry.

### GitHub Actions Deployment Workflow

Each subproject uses GitHub Actions for automated Docker builds and deployments.

**Tag Naming Convention**: `vX.Y-alpha` (e.g., v0.1-alpha, v0.2-alpha, v0.3-alpha)

**Standard Deployment Process**:

1. **Local Build & Test** (MANDATORY before pushing):

   ```bash
   # For frontends
   cd frontend_<project>
   npm run build # just for check if everthing works

   # For backends
   cd backend_<project>
   # Run tests if available
   ```

2. **Commit Changes**:

   ```bash
   git add -A
   git commit -m "Description of changes

   🤖 Generated with [Claude Code](https://claude.com/claude-code)

   Co-Authored-By: Claude <noreply@anthropic.com>"
   git push
   ```

3. **Create and Push Tag**:

   ```bash
   git tag vX.Y-alpha
   git push origin vX.Y-alpha
   ```

4. **Wait for GitHub Actions Build**:

   ```bash
   # Check build status
   gh run list --repo <owner>/<repo> --limit 1

   # Wait for completion (if needed)
   sleep 45 && gh run list --repo <owner>/<repo> --limit 1
   ```

5. **Deploy to Kubernetes**:

   ```bash
   # Restart deployment to pull new image
   kubectl rollout restart deployment/<deployment-name> -n karenina-hub

   # Wait for rollout completion
   kubectl rollout status deployment/<deployment-name> -n karenina-hub --timeout=120s

   # Verify pod is running
   kubectl get pods -n karenina-hub | grep <deployment-name>
   ```

**Project-Specific Image Names**:

- Main system backend: `ghcr.io/darkkaze/chat-agent-hub-backend:latest`
- Main system frontend: `ghcr.io/darkkaze/chat-agent-hub-frontend:latest`
- POS backend: `ghcr.io/darkkaze/chat-agent-hub-pos-backend:latest`
- POS frontend: `ghcr.io/darkkaze/chat-agent-hub-pos-frontend:latest`
- Staff backend: `ghcr.io/darkkaze/chat-agent-hub-staff-backend:latest`
- Staff frontend: `ghcr.io/darkkaze/chat-agent-hub-staff-frontend:latest`

**Important Notes**:

- ALWAYS build locally before pushing to catch TypeScript/build errors early
- GitHub Actions triggers on tag push matching `v*` pattern
- Image tag is always `latest` (pulled on deployment restart)
- Verify build success before restarting Kubernetes deployment

## Development

Each subproject (frontend/backend) has independent Git repos, virtual environments, and development cycles. This root level is for deployment orchestration only.

## n8n Deployment

n8n is deployed as a workflow automation tool using the shared PostgreSQL and Redis infrastructure.

### Configuration

All n8n configuration is in `deployment_vars.env`:

- `N8N_HOST`: n8n.kareninahub.nomada.dev
- `N8N_DB_NAME`: n8n_db
- `N8N_DB_USER`: n8n_user
- `N8N_DB_PASSWORD`: Database password
- `N8N_ENCRYPTION_KEY`: 32+ character encryption key for credentials

### Deployment Process

**1. Apply Secret, ConfigMap, and PVC:**

```bash
source deployment_vars.env && \
NAMESPACE="$NAMESPACE" N8N_HOST="$N8N_HOST" N8N_DB_NAME="$N8N_DB_NAME" \
N8N_DB_USER="$N8N_DB_USER" N8N_DB_PASSWORD="$N8N_DB_PASSWORD" \
N8N_ENCRYPTION_KEY="$N8N_ENCRYPTION_KEY" \
envsubst < kubernetes/n8n-secret.yaml | kubectl apply -f - && \
envsubst < kubernetes/n8n-configmap.yaml | kubectl apply -f - && \
envsubst < kubernetes/n8n-pvc.yaml | kubectl apply -f -
```

**2. Initialize n8n Database:**

```bash
source deployment_vars.env && \
export NAMESPACE N8N_DB_NAME && \
envsubst '$NAMESPACE $N8N_DB_NAME' < kubernetes/n8n-init-job.yaml | kubectl apply -f - && \
kubectl wait --for=condition=complete job/n8n-init -n karenina-hub --timeout=120s
```

**3. Deploy n8n Application:**

```bash
source deployment_vars.env && \
export NAMESPACE && \
envsubst '$NAMESPACE' < kubernetes/n8n-deployment.yaml | kubectl apply -f - && \
envsubst '$NAMESPACE' < kubernetes/n8n-service.yaml | kubectl apply -f - && \
kubectl rollout status deployment/n8n -n karenina-hub --timeout=180s
```

**4. Update Ingress for SSL:**

```bash
source deployment_vars.env && \
export NAMESPACE INGRESS_HOST N8N_HOST TLS_SECRET_NAME && \
envsubst '$NAMESPACE $INGRESS_HOST $N8N_HOST $TLS_SECRET_NAME' < kubernetes/ingress.yaml | kubectl apply -f -
```

### Important Notes

- **Encryption Key**: Never change `N8N_ENCRYPTION_KEY` after initial setup - it's required to decrypt stored credentials
- **Subdomain Required**: n8n must run on a subdomain (not a path) due to WebSocket and asset loading requirements
- **Database Isolation**: n8n uses its own database (`n8n_db`) within the shared PostgreSQL instance
- **Resources**: Configured with 256Mi/100m requests and 1Gi/1000m limits to handle workflow bursts

### Verification

```bash
kubectl get pods -n karenina-hub -l app=n8n
kubectl get svc -n karenina-hub n8n
kubectl get ingress -n karenina-hub agent-hub-ingress
```

Access n8n at: https://n8n.kareninahub.nomada.dev

## Common Issues

### envsubst Variable Substitution

**CRITICAL**: When using `envsubst` with Kubernetes manifests, you MUST specify which variables to substitute. Otherwise, `envsubst` will replace ALL variables, including those meant for Kubernetes to interpret (like `${POSTGRES_USER}` in scripts).

**Wrong** (replaces everything):
```bash
envsubst < kubernetes/file.yaml | kubectl apply -f -
```

**Correct** (only replaces specified variables):
```bash
source deployment_vars.env && \
export NAMESPACE N8N_DB_NAME && \
envsubst '$NAMESPACE $N8N_DB_NAME' < kubernetes/file.yaml | kubectl apply -f -
```

This prevents:
- Variables becoming empty strings
- Resources created in wrong namespaces
- Scripts failing due to missing environment variables
