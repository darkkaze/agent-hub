# Agent Hub

Multi-platform chat and management system with shared authentication, deployed on Kubernetes.

## Overview

Agent Hub is a modular system that combines multiple web applications under a single domain with unified authentication. The platform currently includes:

- **Main System**: Multi-channel chat system with AI agents
- **POS System**: Point of Sale interface for retail operations
- **Staff Timetable**: Employee scheduling and time tracking

All modules share authentication tokens via localStorage and connect to a unified PostgreSQL database.

## Architecture

### Modular Design

The project follows a **multi-module architecture** where each module is an independent Vue 3 + FastAPI application with its own repository:

```
agent_hub/
├── frontend/                    # Main chat system (Vue 3 + Vuetify)
├── backend/                     # Main API (FastAPI + SQLModel)
├── frontend_pos/                # POS interface (Vue 3 + Vuetify)
├── backend_pos/                 # POS API (FastAPI + SQLModel)
├── frontend_staff_timetable/    # Staff scheduling interface
├── backend_staff_timetable/     # Staff scheduling API
└── kubernetes/                  # Deployment manifests
```

Each frontend/backend pair is a **separate Git repository** with independent GitHub Actions for CI/CD.

### Adding New Modules

To add a new section (e.g., "Inventory" or "Analytics"):

1. **Create repositories**:
   - `chat-agent-hub-<module>-frontend`
   - `chat-agent-hub-<module>-backend`

2. **Clone into project**:
   ```bash
   cd agent_hub/
   git clone https://github.com/darkkaze/chat-agent-hub-<module>-frontend.git frontend_<module>
   git clone https://github.com/darkkaze/chat-agent-hub-<module>-backend.git backend_<module>
   ```

3. **Configure routes in `kubernetes/ingress.yaml`**:
   ```yaml
   - path: /<module>/
     pathType: Prefix
     backend:
       service:
         name: frontend-<module>
         port:
           number: 80
   - path: /<module>/api/
     pathType: Prefix
     backend:
       service:
         name: backend-<module>
         port:
           number: 8000
   ```

4. **Set `VITE_PROJECT_DOMAIN` in frontend config**:
   ```typescript
   // frontend_<module>/src/config/index.ts
   export const API_BASE_URL = `${VITE_PROJECT_DOMAIN}/<module>/api`
   ```

5. **Deploy Kubernetes resources** (see deployment guides below)

### Shared Infrastructure

All modules share:

- **PostgreSQL Database**: Unified database with shared auth tables (`user`, `token`, `agent`)
- **Redis**: Cache and Celery task queue
- **Ingress**: Single NGINX ingress with SSL for all routes
- **Authentication**: Token stored in `localStorage` accessible across all modules

### Domain-Based Authentication

The system uses **path-based routing** under a single domain to share authentication:

```
https://kareninahub.nomada.dev/              → Main frontend
https://kareninahub.nomada.dev/api/          → Main backend
https://kareninahub.nomada.dev/pos/          → POS frontend
https://kareninahub.nomada.dev/pos/api/      → POS backend
https://kareninahub.nomada.dev/staff/        → Staff frontend
https://kareninahub.nomada.dev/staff/api/    → Staff backend
```

**Authentication Flow**:

1. User logs in via main system (`/`)
2. Backend generates JWT token and returns it
3. Frontend stores token in `localStorage.setItem('auth_token', token)`
4. When user navigates to `/pos/` or `/staff/`:
   - Same domain = same localStorage
   - Frontend reads `localStorage.getItem('auth_token')`
   - Sends token in `Authorization: Bearer <token>` header
   - Backend validates against shared `user` and `token` tables

**Key Configuration**:

- **Ingress**: Single NGINX ingress controller routes paths to different services
- **CORS**: Not needed (same domain)
- **SSL**: One certificate covers all paths
- **Session Persistence**: localStorage persists across page refreshes

## Deployment

### Prerequisites

- Kubernetes cluster (Minikube, GKE, EKS, etc.)
- `kubectl` configured
- Docker registry access (GitHub Container Registry)

### Deployment Guides

Follow these guides in order for first-time setup:

1. **[Main System Deployment](AGENT_DEPLOY_KUBERNETES.md)**: Deploy core chat system, database, and ingress
2. **[POS System Deployment](AGENT_DEPLOY_POS_KUBERNETES.md)**: Add POS module to existing infrastructure
3. **[Staff Timetable Deployment](STAFF_TIMETABLE_DEPLOY.md)**: Add employee scheduling module

### Quick Deploy (After Initial Setup)

Update a specific module:

```bash
# 1. Build locally (MANDATORY)
cd frontend_<module>
npm run build

# 2. Commit and tag
git add -A
git commit -m "Update <feature>"
git tag v0.X-alpha
git push && git push origin v0.X-alpha

# 3. Wait for GitHub Actions build
gh run list --repo darkkaze/chat-agent-hub-<module>-frontend --limit 1

# 4. Restart deployment
kubectl rollout restart deployment/frontend-<module> -n karenina-hub
kubectl rollout status deployment/frontend-<module> -n karenina-hub --timeout=120s
```

## Development

Each module is developed independently:

```bash
# Frontend development
cd frontend_<module>
npm install
npm run dev  # Runs on http://localhost:5173

# Backend development
cd backend_<module>
python -m venv venv
source venv/bin/activate  # or `venv\Scripts\activate` on Windows
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000
```

### Environment Variables

Each module uses `deployment_vars.env` for configuration:

```bash
source deployment_vars.env
echo $POSTGRES_USER  # Check loaded variables
```

## Technology Stack

- **Frontend**: Vue 3 + TypeScript + Vuetify 3
- **Backend**: FastAPI + SQLModel + Celery
- **Database**: PostgreSQL
- **Cache**: Redis
- **Deployment**: Kubernetes + NGINX Ingress
- **CI/CD**: GitHub Actions + GitHub Container Registry

## License

This project is licensed under the Beerware License (Revision 42).

You can do whatever you want with this code. If we meet some day, and you think this stuff is worth it, you can buy me a beer (or a coffee) in return.

☕ Support via Ko-fi: [ko-fi.com/darkkaze](https://ko-fi.com/darkkaze)

See [LICENSE](LICENSE) for the full license text.

---

**Author**: Kaze (martin@nomada.dev)
**Repository**: https://github.com/darkkaze/agent_hub
