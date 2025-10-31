# Staff Timetable System - Deployment Guide

Sistema de gestión de horarios semanales del personal que extiende al Agent Hub principal.

## Arquitectura

### Backend (`backend_staff_timetable/`)
- **Puerto**: 8002
- **Base path**: `/staff-timetable/api`
- **Base de datos**: PostgreSQL compartida (tabla `staff`)
- **Autenticación**: Token JWT del sistema principal

### Frontend (`frontend_staff_timetable/`)
- **Base path**: `/staff-timetable/`
- **Autenticación**: Token desde localStorage (compartido con sistema principal)
- **UI**: Vue 3 + Vuetify 3 con tema pinkCandy

---

## Características

### CRUD de Personal
- ✅ Crear nuevo personal
- ✅ Editar nombre del personal
- ✅ Borrado lógico (is_active=False)
- ✅ Filtrar por estado (activo/inactivo)
- ✅ Búsqueda por nombre

### Editor de Horarios
- ✅ 7 días de la semana (Lunes - Domingo)
- ✅ Múltiples turnos por día
- ✅ Input type="time" nativo del navegador
- ✅ Agregar/eliminar turnos dinámicamente
- ✅ Guardar cambios con indicador visual
- ✅ Validación de horarios

### Autenticación
- ✅ Token compartido con sistema principal
- ✅ Router guard automático
- ✅ Redirección a `/` si no hay token
- ✅ Backend valida token en cada request

---

## Deployment en Kubernetes

### 1. Build y Push de Imágenes Docker

#### Backend
```bash
cd backend_staff_timetable

# Build
docker build -t ghcr.io/darkkaze/chat-agent-hub-staff-backend:latest .

# Push
docker push ghcr.io/darkkaze/chat-agent-hub-staff-backend:latest
```

#### Frontend
```bash
cd frontend_staff_timetable

# Install dependencies
npm install

# Build
docker build -t ghcr.io/darkkaze/chat-agent-hub-staff-frontend:latest .

# Push
docker push ghcr.io/darkkaze/chat-agent-hub-staff-frontend:latest
```

### 2. Deploy en Kubernetes

**IMPORTANTE**: La tabla `staff` ya existe en la base de datos (creada por el sistema POS). El init job la creará automáticamente si no existe usando `checkfirst=True`.

```bash
# Load environment variables
source deployment_vars.env

# Apply ConfigMap
NAMESPACE="$NAMESPACE" envsubst < kubernetes/backend-staff-timetable-configmap.yaml | kubectl apply -f -

# Apply Services
NAMESPACE="$NAMESPACE" envsubst < kubernetes/backend-staff-timetable-service.yaml | kubectl apply -f -
NAMESPACE="$NAMESPACE" envsubst < kubernetes/frontend-staff-timetable-service.yaml | kubectl apply -f -

# Run init job (creates staff table if needed)
NAMESPACE="$NAMESPACE" envsubst < kubernetes/staff-timetable-init-job.yaml | kubectl apply -f -

# Wait for job to complete
kubectl wait --for=condition=complete --timeout=60s job/staff-timetable-init-db -n $NAMESPACE

# Apply Deployments
NAMESPACE="$NAMESPACE" envsubst < kubernetes/backend-staff-timetable-deployment.yaml | kubectl apply -f -
NAMESPACE="$NAMESPACE" envsubst < kubernetes/frontend-staff-timetable-deployment.yaml | kubectl apply -f -

# Update Ingress (already contains staff-timetable routes)
NAMESPACE="$NAMESPACE" INGRESS_HOST="$INGRESS_HOST" TLS_SECRET_NAME="$TLS_SECRET_NAME" \
  envsubst < kubernetes/ingress.yaml | kubectl apply -f -
```

### 3. Verificar Deployment

```bash
# Check pods
kubectl get pods -n $NAMESPACE | grep staff-timetable

# Check services
kubectl get svc -n $NAMESPACE | grep staff-timetable

# Check ingress
kubectl get ingress -n $NAMESPACE

# Check logs
kubectl logs -n $NAMESPACE deployment/backend-staff-timetable
kubectl logs -n $NAMESPACE deployment/frontend-staff-timetable
```

### 4. Acceder al Sistema

```
https://<INGRESS_HOST>/staff-timetable/
```

**Nota**: Debes estar autenticado en el sistema principal primero (token en localStorage).

---

## Desarrollo Local

### Backend

```bash
cd backend_staff_timetable

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export DB_BACKEND=postgres
export POSTGRES_HOST=localhost
export POSTGRES_PORT=5432
export POSTGRES_DB=agent_hub
export POSTGRES_USER=your_user
export POSTGRES_PASSWORD=your_password

# Initialize database
python manage.py init_db

# Run server
fastapi dev main.py --port 8002
```

**Health check**: http://localhost:8002/staff-timetable/api/health

**API docs**: http://localhost:8002/staff-timetable/api/docs

### Frontend

```bash
cd frontend_staff_timetable

# Install dependencies
npm install

# Run dev server
npm run dev
```

**Dev server**: http://localhost:5174/staff-timetable/

---

## API Endpoints

Base URL: `/staff-timetable/api`

### Staff CRUD

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/staff/` | Listar todo el personal |
| GET | `/staff/?is_active=true` | Filtrar por estado |
| POST | `/staff/` | Crear personal |
| GET | `/staff/{staff_id}` | Obtener personal específico |
| PUT | `/staff/{staff_id}` | Actualizar personal |
| DELETE | `/staff/{staff_id}` | Borrado lógico |

### Formato de Horario (JSON)

```json
{
  "monday": [
    {"start": "09:00", "end": "13:00"},
    {"start": "14:00", "end": "21:00"}
  ],
  "tuesday": [{"start": "09:00", "end": "21:00"}],
  "wednesday": [],
  "thursday": [{"start": "10:00", "end": "14:00"}],
  "friday": [{"start": "09:00", "end": "13:00"}, {"start": "14:00", "end": "21:00"}],
  "saturday": [{"start": "10:00", "end": "18:00"}],
  "sunday": []
}
```

---

## Troubleshooting

### Backend no inicia
```bash
# Check pod logs
kubectl logs -n $NAMESPACE deployment/backend-staff-timetable

# Check if PostgreSQL is ready
kubectl get pods -n $NAMESPACE | grep postgres

# Check database connection
kubectl exec -it -n $NAMESPACE deployment/backend-staff-timetable -- python manage.py check_db
```

### Frontend muestra error 401
- Verificar que el usuario esté autenticado en el sistema principal (`/`)
- Revisar que el token exista en localStorage: `localStorage.getItem('auth_token')`
- Verificar que el backend esté validando correctamente el token

### Tabla `staff` no existe
```bash
# Re-run init job
kubectl delete job staff-timetable-init-db -n $NAMESPACE
NAMESPACE="$NAMESPACE" envsubst < kubernetes/staff-timetable-init-job.yaml | kubectl apply -f -
```

### Ingress no rutea correctamente
```bash
# Verify ingress configuration
kubectl describe ingress agent-hub-ingress -n $NAMESPACE

# Check nginx ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

---

## Arquitectura de Autenticación Compartida

```
┌──────────────────────────────────────────────────────────┐
│                  Browser (localStorage)                   │
│                  auth_token: "jwt_xxx"                    │
└────────────────────┬─────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
┌───────────────┐         ┌──────────────────┐
│  Main System  │         │ Staff Timetable  │
│      (/)      │         │ (/staff-timetable/)│
└───────┬───────┘         └────────┬─────────┘
        │                          │
        │                          │
        ▼                          ▼
┌───────────────────────────────────────────────┐
│              Backend (PostgreSQL)             │
│  Tablas compartidas: user, token, agent       │
│  Tabla específica: staff                      │
└───────────────────────────────────────────────┘
```

---

## Próximos Pasos

1. ✅ Sistema completamente funcional
2. Opcional: Agregar validación de horarios solapados
3. Opcional: Exportar horarios a PDF
4. Opcional: Notificaciones de cambios de horario
5. Opcional: Historial de cambios de horarios

---

## Contacto y Soporte

- **Documentación principal**: `CLAUDE.md`
- **Deployment general**: `AGENT_DEPLOY_KUBERNETES.md`
- **POS deployment**: `AGENT_DEPLOY_POS_KUBERNETES.md`
