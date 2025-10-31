# Agent Hub - Kubernetes Deployment Guide

Esta guía está diseñada para ser seguida paso a paso por agentes CLI como Claude Code o Gemini CLI.

## 📁 Descripción de archivos en `kubernetes/`

### **Database (PostgreSQL)**

- `namespace.yaml` - Namespace configurable para todo el proyecto
- `postgres-secret.yaml` - Credenciales de PostgreSQL (user, password, db)
- `postgres-pvc.yaml` - Almacenamiento persistente para PostgreSQL (10Gi)
- `postgres-statefulset.yaml` - StatefulSet de PostgreSQL 15
- `postgres-service.yaml` - Service para acceso a PostgreSQL

### **Cache (Redis)**

- `redis-deployment.yaml` - Deployment de Redis 7 para Celery broker
- `redis-service.yaml` - Service para acceso a Redis

### **Backend (FastAPI + Celery)**

- `backend-configmap.yaml` - Variables de entorno no sensibles
- `backend-deployment.yaml` - Deployment de FastAPI
- `backend-service.yaml` - Service para la API
- `celery-deployment.yaml` - Deployment del Celery Worker
- `db-init-job.yaml` - Job para ejecutar `manage.py init_db`

### **Frontend (Vue + Nginx)**

- `frontend-deployment.yaml` - Deployment de Vue app con Nginx
- `frontend-service.yaml` - Service para el frontend

### **Ingress & TLS**

- `tls-secret.yaml` - Certificados TLS (generados automáticamente)
- `ingress.yaml` - Ingress para acceso externo

---

## 🚀 Pasos de Deployment

**INSTRUCCIONES PARA AGENTES CLI**:

1. **Ejecutar UN SOLO PASO a la vez**
2. **Después de cada paso**: Resumir lo que se hizo
3. **SIEMPRE preguntar**: "¿Continúo con el siguiente paso?" antes de proceder
4. **NO ejecutar múltiples pasos** de forma automática
5. **ESPERAR confirmación del usuario** antes de cada paso

**MANEJO DE VARIABLES DE ENTORNO**:

- **Crear archivo temporal**: `deployment_vars.env` para persistir variables entre comandos
- **Formato del archivo**: `KEY=value` (una variable por línea)
- **Cargar variables**: `source deployment_vars.env && [comando]` cuando necesites las variables
- **El archivo se crea** en el paso 1.3 y se va completando en pasos posteriores
- **VERIFICAR SIEMPRE**: Cada paso debe verificar si `deployment_vars.env` ya existe y tiene las variables necesarias

**CONFIGURACIÓN DE KUBECONFIG**:

- **Por defecto**: Los comandos kubectl asumen que tu kubeconfig está en `~/.kube/config`
- **Si tienes kubeconfig en otra ubicación**: Pregunta al usuario si su kubeconfig está configurado estándar
- **Si NO está en `~/.kube/config`**: Usar formato `KUBECONFIG="ruta/al/archivo" && kubectl [comando]` en TODOS los comandos kubectl
- **Archivo local**: Si el usuario tiene `.kubeconfig` en la raíz del proyecto, usar `KUBECONFIG="$(pwd)/.kubeconfig" && kubectl [comando]`

---

## 1. CONFIGURACIÓN INICIAL

### **Paso 1.1: Verificar dependencias**

**Verificar herramientas necesarias**:

- `which openssl` - Para generar certificados TLS
- `which envsubst` - Para sustituir variables en manifiestos
- `which kubectl` - Para interactuar con Kubernetes

**Si faltan dependencias**: **DETENER AQUÍ** - Consultar instalación según tu SO:

- **openssl**: macOS: `brew install openssl` | Ubuntu: `sudo apt-get install openssl`
- **envsubst**: macOS: `brew install gettext` | Ubuntu: `sudo apt-get install gettext-base`
- **kubectl**: https://kubernetes.io/docs/tasks/tools/

**PAUSA**: Resumir qué dependencias se verificaron y preguntar al usuario si continuar con el paso 1.2.

### **Paso 1.2: Verificar kubeconfig**

**Acción requerida**: Verificar que tienes acceso a tu cluster Kubernetes.

**PREGUNTAR AL USUARIO**: "¿Tu kubeconfig está configurado en la ubicación estándar ~/.kube/config?"

**Si responde SÍ** - Usar comandos estándar:
- `kubectl cluster-info`
- `kubectl get nodes`

**Si responde NO** - Usar el formato con KUBECONFIG en todos los comandos kubectl:
- `KUBECONFIG="ruta/al/archivo" && kubectl cluster-info`
- `KUBECONFIG="ruta/al/archivo" && kubectl get nodes`

**Si no tienes kubeconfig o falla la conexión**: **DETENER AQUÍ** - No se puede continuar sin acceso al cluster.

**PAUSA**: Resumir el estado de la conexión a Kubernetes y preguntar al usuario si continuar con el paso 1.3.

### **Paso 1.3: Configurar PROJECT_DOMAIN**

**VERIFICAR PRIMERO**: `cat deployment_vars.env 2>/dev/null | grep PROJECT_DOMAIN`

**Si ya existe PROJECT_DOMAIN**:
- Mostrar valor actual: `source deployment_vars.env && echo "PROJECT_DOMAIN actual: $PROJECT_DOMAIN"`
- Preguntar al usuario si quiere mantenerlo o cambiarlo

**Si NO existe deployment_vars.env o no tiene PROJECT_DOMAIN**:
- **PREGUNTAR AL USUARIO**: "¿Cuál es el dominio donde quieres desplegar la aplicación? (ejemplo: https://api.tu-dominio.com)"
- Crear/actualizar archivo: `echo "PROJECT_DOMAIN=[DOMINIO_PROPORCIONADO_POR_USUARIO]" > deployment_vars.env`

**Verificar archivo**:
- `cat deployment_vars.env` - Mostrar contenido actual

**NO PROCEDER** sin obtener esta información del usuario.

**PAUSA**: Confirmar que el PROJECT_DOMAIN fue guardado en deployment_vars.env y preguntar al usuario si continuar con el paso 1.4.

### **Paso 1.4: Configurar NAMESPACE**

**VERIFICAR PRIMERO**: `cat deployment_vars.env 2>/dev/null | grep NAMESPACE`

**Si ya existe NAMESPACE**:
- Mostrar valor actual: `source deployment_vars.env && echo "NAMESPACE actual: $NAMESPACE"`
- Preguntar al usuario si quiere mantenerlo o cambiarlo

**Si NO existe NAMESPACE en deployment_vars.env**:
- **PREGUNTAR AL USUARIO**: "¿Qué namespace quieres usar para el deployment? (por defecto: agent-hub)"
- Si el usuario no especifica, usar "agent-hub"
- Agregar al archivo: `echo "NAMESPACE=[NAMESPACE_ELEGIDO]" >> deployment_vars.env`

**Verificar archivo**:
- `cat deployment_vars.env` - Mostrar todas las variables configuradas

**PAUSA**: Confirmar que el NAMESPACE fue guardado en deployment_vars.env y preguntar al usuario si continuar con el paso 1.5.

### **Paso 1.5: Configurar variables de Ingress**

**VERIFICAR PRIMERO**: `cat deployment_vars.env 2>/dev/null | grep INGRESS_HOST`

**Si ya existe INGRESS_HOST**:
- Mostrar valor actual y preguntar si mantener o cambiarlo

**Si NO existe**:
- **Derivar variables**: `source deployment_vars.env && echo "INGRESS_HOST=$(echo $PROJECT_DOMAIN | sed 's|https\\?://||')" >> deployment_vars.env`
- **Agregar TLS secret name**: `echo "TLS_SECRET_NAME=agent-hub-tls" >> deployment_vars.env`

**Verificar variables**:
- `cat deployment_vars.env` - Mostrar todas las variables configuradas

**PAUSA**: Confirmar variables de Ingress y preguntar si continuar con el paso 1.6.

### **Paso 1.6: Configurar email para Let's Encrypt**

**VERIFICAR PRIMERO**: `cat deployment_vars.env 2>/dev/null | grep LETSENCRYPT_EMAIL`

**Si ya existe LETSENCRYPT_EMAIL**:
- Mostrar valor actual: `source deployment_vars.env && echo "LETSENCRYPT_EMAIL actual: $LETSENCRYPT_EMAIL"`
- Preguntar al usuario si quiere mantenerlo o cambiarlo

**Si NO existe LETSENCRYPT_EMAIL en deployment_vars.env**:
- **PREGUNTAR AL USUARIO**: "¿Cuál es el email para Let's Encrypt? (necesario para recibir notificaciones de renovación)"
- Agregar al archivo: `echo "LETSENCRYPT_EMAIL=[EMAIL_PROPORCIONADO_POR_USUARIO]" >> deployment_vars.env`

**Verificar archivo**:
- `cat deployment_vars.env` - Mostrar todas las variables configuradas

**PAUSA**: Confirmar que el LETSENCRYPT_EMAIL fue guardado en deployment_vars.env y preguntar al usuario si continuar con el paso 1.7.

### **Paso 1.7: Configurar credenciales PostgreSQL**

**VERIFICAR PRIMERO**: `cat deployment_vars.env 2>/dev/null | grep -E "(POSTGRES_USER|POSTGRES_PASSWORD)"`

**Si ya existen credenciales PostgreSQL**:
- Mostrar que existen (NO mostrar la contraseña por seguridad)
- Preguntar al usuario si quiere mantenerlas o regenerarlas

**Si NO existen**:
- **Agregar credenciales**:
  - `echo "POSTGRES_USER=agent_hub" >> deployment_vars.env`
  - `echo "POSTGRES_PASSWORD=$(openssl rand -base64 32 | tr -d '=+/' | cut -c1-16)" >> deployment_vars.env`

**Verificar credenciales**:
- `cat deployment_vars.env` - Mostrar todas las variables (contraseña incluida)

**PAUSA**: Confirmar que las credenciales PostgreSQL fueron agregadas y preguntar si continuar con el paso 2.1.

---

## 2. DEPLOYMENT DE SERVICIOS

### **Paso 2.1: Crear namespace y secrets**

**Aplicar namespace**:
- `source deployment_vars.env && envsubst < kubernetes/namespace.yaml | kubectl apply -f -`

**Crear PostgreSQL secret**:
- `source deployment_vars.env && kubectl create secret generic postgres-secret -n "$NAMESPACE" --from-literal=POSTGRES_USER="$POSTGRES_USER" --from-literal=POSTGRES_PASSWORD="$POSTGRES_PASSWORD" --from-literal=POSTGRES_DB="agent_hub"`

**Aplicar ConfigMap**:
- `source deployment_vars.env && envsubst < kubernetes/backend-configmap.yaml | kubectl apply -f -`

**Verificar**:
- `source deployment_vars.env && kubectl get secrets -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl get configmap -n "$NAMESPACE"`

**PAUSA**: Confirmar que namespace y secrets se crearon correctamente y preguntar si continuar con el paso 2.2.

### **Paso 2.2: Desplegar infraestructura (PostgreSQL y Redis)**

**Aplicar PostgreSQL**:
- `source deployment_vars.env && envsubst < kubernetes/postgres-pvc.yaml | kubectl apply -f -`
- `source deployment_vars.env && envsubst < kubernetes/postgres-statefulset.yaml | kubectl apply -f -`
- `source deployment_vars.env && envsubst < kubernetes/postgres-service.yaml | kubectl apply -f -`

**Aplicar Redis**:
- `source deployment_vars.env && envsubst < kubernetes/redis-deployment.yaml | kubectl apply -f -`
- `source deployment_vars.env && envsubst < kubernetes/redis-service.yaml | kubectl apply -f -`

**Esperar y verificar**:
- `source deployment_vars.env && kubectl wait --for=condition=ready --timeout=300s pod -l app=postgres -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl wait --for=condition=ready --timeout=300s pod -l app=redis -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl get pods -n "$NAMESPACE"`

**PAUSA**: Confirmar que PostgreSQL y Redis están funcionando y preguntar si continuar con el paso 2.3.

### **Paso 2.3: Inicializar database**

**Ejecutar job de inicialización**:
- `source deployment_vars.env && envsubst < kubernetes/db-init-job.yaml | kubectl apply -f -`
- `source deployment_vars.env && kubectl wait --for=condition=complete --timeout=300s job/db-init -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl logs job/db-init -n "$NAMESPACE"` - Verificar logs

**Resultado esperado**: Job completa exitosamente, tablas creadas en PostgreSQL.

**PAUSA**: Confirmar que la inicialización de la base de datos fue exitosa y preguntar si continuar con el paso 2.4.

### **Paso 2.4: Desplegar aplicaciones**

**Desplegar backend API**:
- `source deployment_vars.env && envsubst < kubernetes/backend-deployment.yaml | kubectl apply -f -`
- `source deployment_vars.env && envsubst < kubernetes/backend-service.yaml | kubectl apply -f -`

**Desplegar Celery worker**:
- `source deployment_vars.env && envsubst < kubernetes/celery-deployment.yaml | kubectl apply -f -`

**Desplegar frontend**:
- `source deployment_vars.env && envsubst < kubernetes/frontend-deployment.yaml | kubectl apply -f -`
- `source deployment_vars.env && envsubst < kubernetes/frontend-service.yaml | kubectl apply -f -`

**Esperar y verificar**:
- `source deployment_vars.env && kubectl wait --for=condition=available --timeout=300s deployment/backend -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl wait --for=condition=available --timeout=300s deployment/celery-worker -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl wait --for=condition=available --timeout=300s deployment/frontend -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl get pods -n "$NAMESPACE"`

**PAUSA**: Confirmar que todas las aplicaciones están desplegadas correctamente y preguntar si continuar con el paso 3.1.

---

## 3. INGRESS Y FINALIZACIÓN

### **Paso 3.1: Instalar cert-manager**

**Instalar cert-manager en el cluster**:
- `kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.1/cert-manager.yaml`

**Esperar a que cert-manager esté listo**:
- `kubectl wait --for=condition=available --timeout=300s deployment/cert-manager -n cert-manager`
- `kubectl wait --for=condition=available --timeout=300s deployment/cert-manager-webhook -n cert-manager`
- `kubectl wait --for=condition=available --timeout=300s deployment/cert-manager-cainjector -n cert-manager`

**Verificar instalación**:
- `kubectl get pods -n cert-manager`

**PAUSA**: Confirmar que cert-manager está funcionando correctamente y preguntar si continuar con el paso 3.2.

### **Paso 3.2: Configurar Let's Encrypt ClusterIssuer**

**Eliminar webhook de validación que interfiere con certificados**:
- `kubectl delete validatingwebhookconfiguration ingress-nginx-admission --ignore-not-found=true`

**Aplicar ClusterIssuer para Let's Encrypt**:
- `export $(cat deployment_vars.env | xargs) && envsubst < kubernetes/cluster-issuer-production.yaml | kubectl apply -f -`

**Verificar ClusterIssuer**:
- `kubectl get clusterissuer`
- `kubectl describe clusterissuer letsencrypt-prod`

**PAUSA**: Confirmar que el ClusterIssuer está listo y preguntar si continuar con el paso 3.3.

### **Paso 3.3: Desplegar Ingress con certificados automáticos**

**Eliminar secret TLS manual si existe**:
- `source deployment_vars.env && kubectl delete secret "$TLS_SECRET_NAME" -n "$NAMESPACE" --ignore-not-found=true`

**Aplicar Ingress con cert-manager**:
- `export $(cat deployment_vars.env | xargs) && envsubst < kubernetes/ingress.yaml | kubectl apply -f -`

**Verificar Ingress y certificado**:
- `source deployment_vars.env && kubectl get ingress -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl get certificate -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl describe certificate -n "$NAMESPACE"`

**Verificar que el certificado correcto está siendo servido**:
- `source deployment_vars.env && echo | openssl s_client -servername $INGRESS_HOST -connect $INGRESS_HOST:443 2>/dev/null | openssl x509 -text -noout | grep -A2 "Issuer:"`

**Resultado esperado**: Debe mostrar `Issuer: C=US, O=Let's Encrypt` (no "Kubernetes Ingress Controller Fake Certificate")

**PAUSA**: Confirmar que el Ingress está configurado correctamente y el certificado de Let's Encrypt está funcionando, luego preguntar si continuar con el paso 3.4.

### **Paso 3.4: Verificación final**

**Mostrar estado del deployment**:
- `source deployment_vars.env && kubectl get pods -n "$NAMESPACE"` - Estado de todos los pods
- `source deployment_vars.env && kubectl get services -n "$NAMESPACE"` - Servicios disponibles
- `source deployment_vars.env && kubectl get ingress -n "$NAMESPACE"` - Configuración de Ingress

**Verificar logs si hay problemas**:
- `source deployment_vars.env && kubectl logs -l app=backend -n "$NAMESPACE" --tail=50` - Logs del backend
- `source deployment_vars.env && kubectl logs -l app=frontend -n "$NAMESPACE" --tail=50` - Logs del frontend

**Servicios disponibles**:
- **Acceso externo**: `https://$INGRESS_HOST` (Frontend y API en `/api`)
- Backend API interno: `backend.$NAMESPACE.svc.cluster.local:8000`
- Frontend interno: `frontend.$NAMESPACE.svc.cluster.local:80`

**PAUSA**: Confirmar que todo está funcionando correctamente y preguntar si continuar con el paso 3.5.

### **Paso 3.5: Crear resumen de deployment**

**Crear archivo de resumen**:

```bash
source deployment_vars.env && cat > RESUME_KUBERNETES_DEPLOY.md << EOF
# Agent Hub - Resumen de Deployment

**Fecha de deployment**: $(date)

## Variables de configuración
- **PROJECT_DOMAIN**: $PROJECT_DOMAIN
- **NAMESPACE**: $NAMESPACE
- **INGRESS_HOST**: $INGRESS_HOST
- **TLS_SECRET_NAME**: $TLS_SECRET_NAME

## Credenciales PostgreSQL
- **Usuario**: $POSTGRES_USER
- **Password**: $POSTGRES_PASSWORD
- **Base de datos**: agent_hub

## URLs de acceso
- **Frontend**: https://$INGRESS_HOST
- **Backend API**: https://$INGRESS_HOST/api

## Imágenes utilizadas
- **Backend**: ghcr.io/darkkaze/chat-agent-hub-backend:latest
- **Frontend**: ghcr.io/darkkaze/chat-agent-hub-frontend:latest

## Estado del deployment
\$(kubectl get pods -n "$NAMESPACE")
\$(kubectl get services -n "$NAMESPACE")
\$(kubectl get ingress -n "$NAMESPACE")
EOF
```

**Verificar archivo creado**:
- `cat RESUME_KUBERNETES_DEPLOY.md`

---

## 🔧 Troubleshooting

### **Ingress**

```bash
# Si el Ingress no está funcionando
source deployment_vars.env && kubectl describe ingress agent-hub-ingress -n "$NAMESPACE"
source deployment_vars.env && kubectl get ingress -n "$NAMESPACE"

# Verificar que el Ingress Controller esté instalado
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Verificar certificados TLS
source deployment_vars.env && kubectl get secret agent-hub-tls -n "$NAMESPACE" -o yaml
```

### **PostgreSQL**

```bash
# Si PostgreSQL no inicia
source deployment_vars.env && kubectl describe pod -l app=postgres -n "$NAMESPACE"
source deployment_vars.env && kubectl logs -l app=postgres -n "$NAMESPACE"

# Conectar a PostgreSQL para debug
source deployment_vars.env && kubectl exec -it postgres-0 -n "$NAMESPACE" -- psql -U agent_hub -d agent_hub
```

### **Backend**

```bash
# Si el backend falla
source deployment_vars.env && kubectl describe pod -l app=backend -n "$NAMESPACE"
source deployment_vars.env && kubectl logs -l app=backend -n "$NAMESPACE"

# Verificar variables de entorno
source deployment_vars.env && kubectl exec -it deployment/backend -n "$NAMESPACE" -- env | grep -E "(POSTGRES|REDIS)"
```

### **Frontend**

```bash
# Si el frontend no carga
source deployment_vars.env && kubectl describe pod -l app=frontend -n "$NAMESPACE"
source deployment_vars.env && kubectl logs -l app=frontend -n "$NAMESPACE"

# Verificar nginx config
source deployment_vars.env && kubectl exec -it deployment/frontend -n "$NAMESPACE" -- nginx -t
```

### **Job de init_db**

```bash
# Si el job de inicialización falla
source deployment_vars.env && kubectl describe job db-init -n "$NAMESPACE"
source deployment_vars.env && kubectl logs job/db-init -n "$NAMESPACE"

# Re-ejecutar el job
source deployment_vars.env && kubectl delete job db-init -n "$NAMESPACE"
source deployment_vars.env && envsubst < kubernetes/db-init-job.yaml | kubectl apply -f -
```

### **Comandos útiles generales**

```bash
# Ver eventos del namespace
source deployment_vars.env && kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp'

# Reiniciar un deployment
source deployment_vars.env && kubectl rollout restart deployment/backend -n "$NAMESPACE"

# Escalar replicas
source deployment_vars.env && kubectl scale deployment/backend --replicas=3 -n "$NAMESPACE"

# Port-forward para debug local
source deployment_vars.env && kubectl port-forward service/backend 8000:8000 -n "$NAMESPACE"
source deployment_vars.env && kubectl port-forward service/frontend 3000:80 -n "$NAMESPACE"
```

---

## ✅ Deployment Completado

Si todos los pasos se ejecutaron exitosamente, tu aplicación Agent Hub debería estar ejecutándose en el cluster Kubernetes.

**IMPORTANTE**: Revisa el archivo `RESUME_KUBERNETES_DEPLOY.md` generado con todas las credenciales y detalles del deployment.

La aplicación estará disponible en `https://INGRESS_HOST` con la API en `/api`.