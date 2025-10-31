# Agent Hub POS - Kubernetes Deployment Guide

Esta guía está diseñada para ser seguida paso a paso por agentes CLI como Claude Code o Gemini CLI.

**PREREQUISITO**: El deployment base de Agent Hub debe estar funcionando correctamente antes de proceder con esta guía. Seguir primero `AGENT_DEPLOY_KUBERNETES.md`.

## 📁 Descripción de archivos POS en `kubernetes/`

### **Backend POS (FastAPI)**

- `backend-pos-configmap.yaml` - Variables de entorno para backend POS (PostgreSQL + Redis compartidos)
- `backend-pos-deployment.yaml` - Deployment del backend POS en puerto 8001
- `backend-pos-service.yaml` - Service para acceso al backend POS
- `pos-init-job.yaml` - Job para inicializar tablas POS específicas

### **Frontend POS (Vue + Nginx)**

- `frontend-pos-deployment.yaml` - Deployment del frontend POS con Nginx
- `frontend-pos-service.yaml` - Service para el frontend POS

### **Ingress actualizado**

- `ingress.yaml` - Ingress actualizado con rutas `/pos` y `/pos/api`

---

## 🚀 Pasos de Deployment POS

**INSTRUCCIONES PARA AGENTES CLI**:

1. **Ejecutar UN SOLO PASO a la vez**
2. **Después de cada paso**: Resumir lo que se hizo
3. **SIEMPRE preguntar**: "¿Continúo con el siguiente paso?" antes de proceder
4. **NO ejecutar múltiples pasos** de forma automática
5. **ESPERAR confirmación del usuario** antes de cada paso

**MANEJO DE VARIABLES DE ENTORNO**:

- **Reutilizar archivo**: `deployment_vars.env` del deployment base
- **Cargar variables**: `source deployment_vars.env && [comando]` cuando necesites las variables
- **VERIFICAR SIEMPRE**: Las variables necesarias ya deben existir del deployment base

**CONFIGURACIÓN DE KUBECONFIG**:

- **Usar la misma configuración** del deployment base
- Si usas `KUBECONFIG` personalizado, mantener el mismo formato en todos los comandos

---

## 1. VERIFICACIÓN PREVIA

### **Paso 1.1: Verificar deployment base**

**Verificar que el sistema base esté funcionando**:

```bash
# Verificar que las variables existen
cat deployment_vars.env

# Verificar que el namespace está funcionando
source deployment_vars.env && kubectl get pods -n "$NAMESPACE"

# Verificar que PostgreSQL y Redis están activos
source deployment_vars.env && kubectl get pods -n "$NAMESPACE" -l app=postgres
source deployment_vars.env && kubectl get pods -n "$NAMESPACE" -l app=redis
```

**Resultado esperado**: Pods de PostgreSQL y Redis deben estar en estado `Running`.

**Si algún pod no está funcionando**: **DETENER AQUÍ** - Corregir el deployment base primero.

**PAUSA**: Confirmar que el deployment base está funcionando correctamente y preguntar si continuar con el paso 2.1.

---

## 2. DEPLOYMENT DE SERVICIOS POS

### **Paso 2.1: Crear ConfigMap y servicios POS**

**Aplicar ConfigMap POS**:
- `source deployment_vars.env && envsubst < kubernetes/backend-pos-configmap.yaml | kubectl apply -f -`

**Verificar**:
- `source deployment_vars.env && kubectl get configmap -n "$NAMESPACE" | grep pos`

**Resultado esperado**: ConfigMap `backend-pos-config` creado exitosamente.

**PAUSA**: Confirmar que el ConfigMap POS se creó correctamente y preguntar si continuar con el paso 2.2.

### **Paso 2.2: Inicializar tablas POS**

**Ejecutar job de inicialización POS**:
- `source deployment_vars.env && envsubst < kubernetes/pos-init-job.yaml | kubectl apply -f -`
- `source deployment_vars.env && kubectl wait --for=condition=complete --timeout=300s job/pos-init -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl logs job/pos-init -n "$NAMESPACE"` - Verificar logs

**Resultado esperado**: Job completa exitosamente, solo tablas POS creadas (Customer, Product, Sale). Las tablas de auth ya existen.

**PAUSA**: Confirmar que la inicialización de tablas POS fue exitosa y preguntar si continuar con el paso 2.3.

### **Paso 2.3: Desplegar aplicaciones POS**

**Desplegar backend POS**:
- `source deployment_vars.env && envsubst < kubernetes/backend-pos-deployment.yaml | kubectl apply -f -`
- `source deployment_vars.env && envsubst < kubernetes/backend-pos-service.yaml | kubectl apply -f -`

**Desplegar frontend POS**:
- `source deployment_vars.env && envsubst < kubernetes/frontend-pos-deployment.yaml | kubectl apply -f -`
- `source deployment_vars.env && envsubst < kubernetes/frontend-pos-service.yaml | kubectl apply -f -`

**Esperar y verificar**:
- `source deployment_vars.env && kubectl wait --for=condition=available --timeout=300s deployment/backend-pos -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl wait --for=condition=available --timeout=300s deployment/frontend-pos -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl get pods -n "$NAMESPACE" | grep pos`

**PAUSA**: Confirmar que las aplicaciones POS están desplegadas correctamente y preguntar si continuar con el paso 3.1.

---

## 3. ACTUALIZACIÓN DE INGRESS

### **Paso 3.1: Actualizar Ingress con rutas POS**

**Aplicar Ingress actualizado**:
- `source deployment_vars.env && envsubst < kubernetes/ingress.yaml | kubectl apply -f -`

**Verificar Ingress**:
- `source deployment_vars.env && kubectl get ingress -n "$NAMESPACE"`
- `source deployment_vars.env && kubectl describe ingress -n "$NAMESPACE"`

**Verificar rutas**:
- Confirmar que aparecen las rutas `/pos/api` → `backend-pos:8001` y `/pos` → `frontend-pos:80`

**PAUSA**: Confirmar que el Ingress se actualizó correctamente con las nuevas rutas y preguntar si continuar con el paso 3.2.

### **Paso 3.2: Verificación final POS**

**Mostrar estado completo del deployment**:
- `source deployment_vars.env && kubectl get pods -n "$NAMESPACE"` - Estado de todos los pods (base + POS)
- `source deployment_vars.env && kubectl get services -n "$NAMESPACE"` - Servicios disponibles
- `source deployment_vars.env && kubectl get ingress -n "$NAMESPACE"` - Configuración de Ingress

**Verificar logs POS si hay problemas**:
- `source deployment_vars.env && kubectl logs -l app=backend-pos -n "$NAMESPACE" --tail=50` - Logs del backend POS
- `source deployment_vars.env && kubectl logs -l app=frontend-pos -n "$NAMESPACE" --tail=50` - Logs del frontend POS

**Servicios POS disponibles**:
- **Frontend POS**: `https://$INGRESS_HOST/pos` (Interfaz POS)
- **Backend POS API**: `https://$INGRESS_HOST/pos/api` (API POS)
- Backend POS interno: `backend-pos.$NAMESPACE.svc.cluster.local:8001`
- Frontend POS interno: `frontend-pos.$NAMESPACE.svc.cluster.local:80`

**PAUSA**: Confirmar que todo está funcionando correctamente y preguntar si continuar con el paso 3.3.

### **Paso 3.3: Crear resumen de deployment POS**

**Crear archivo de resumen POS**:

```bash
source deployment_vars.env && cat > RESUME_POS_KUBERNETES_DEPLOY.md << EOF
# Agent Hub POS - Resumen de Deployment

**Fecha de deployment**: $(date)

## Variables de configuración (reutilizadas del deployment base)
- **PROJECT_DOMAIN**: $PROJECT_DOMAIN
- **NAMESPACE**: $NAMESPACE
- **INGRESS_HOST**: $INGRESS_HOST
- **TLS_SECRET_NAME**: $TLS_SECRET_NAME

## Base de datos compartida
- **PostgreSQL**: Compartida con el sistema base
- **Redis**: Compartida con el sistema base (DB 1 para POS)
- **Tablas POS**: customer, product, sale (auth reutilizada)

## URLs de acceso POS
- **Frontend POS**: https://$INGRESS_HOST/pos
- **Backend POS API**: https://$INGRESS_HOST/pos/api

## Imágenes utilizadas POS
- **Backend POS**: ghcr.io/darkkaze/agent-hub-pos-backend:latest
- **Frontend POS**: ghcr.io/darkkaze/agent-hub-pos-frontend:latest

## Configuración técnica
- **Backend POS Puerto**: 8001 (diferente del principal 8000)
- **Configuración Redis**: DB 1 (principal usa DB 0)
- **Autenticación**: Compartida con sistema base

## Estado del deployment POS
\\$(kubectl get pods -n "$NAMESPACE" | grep pos)
\\$(kubectl get services -n "$NAMESPACE" | grep pos)
EOF
```

**Verificar archivo creado**:
- `cat RESUME_POS_KUBERNETES_DEPLOY.md`

---

## 🔧 Troubleshooting POS

### **Backend POS**

```bash
# Si el backend POS falla
source deployment_vars.env && kubectl describe pod -l app=backend-pos -n "$NAMESPACE"
source deployment_vars.env && kubectl logs -l app=backend-pos -n "$NAMESPACE"

# Verificar variables de entorno
source deployment_vars.env && kubectl exec -it deployment/backend-pos -n "$NAMESPACE" -- env | grep -E "(POSTGRES|REDIS)"

# Verificar conexión a PostgreSQL
source deployment_vars.env && kubectl exec -it deployment/backend-pos -n "$NAMESPACE" -- python manage.py check_db
```

### **Frontend POS**

```bash
# Si el frontend POS no carga
source deployment_vars.env && kubectl describe pod -l app=frontend-pos -n "$NAMESPACE"
source deployment_vars.env && kubectl logs -l app=frontend-pos -n "$NAMESPACE"

# Verificar nginx config
source deployment_vars.env && kubectl exec -it deployment/frontend-pos -n "$NAMESPACE" -- nginx -t
```

### **Job de init POS**

```bash
# Si el job de inicialización POS falla
source deployment_vars.env && kubectl describe job pos-init -n "$NAMESPACE"
source deployment_vars.env && kubectl logs job/pos-init -n "$NAMESPACE"

# Re-ejecutar el job
source deployment_vars.env && kubectl delete job pos-init -n "$NAMESPACE"
source deployment_vars.env && envsubst < kubernetes/pos-init-job.yaml | kubectl apply -f -
```

### **Ingress POS**

```bash
# Verificar rutas POS en Ingress
source deployment_vars.env && kubectl describe ingress -n "$NAMESPACE" | grep -A 10 -B 5 pos

# Test de conectividad POS
source deployment_vars.env && curl -k https://$INGRESS_HOST/pos
source deployment_vars.env && curl -k https://$INGRESS_HOST/pos/api
```

### **Comandos útiles POS**

```bash
# Ver solo pods POS
source deployment_vars.env && kubectl get pods -n "$NAMESPACE" | grep pos

# Reiniciar deployments POS
source deployment_vars.env && kubectl rollout restart deployment/backend-pos -n "$NAMESPACE"
source deployment_vars.env && kubectl rollout restart deployment/frontend-pos -n "$NAMESPACE"

# Port-forward para debug local POS
source deployment_vars.env && kubectl port-forward service/backend-pos 8001:8001 -n "$NAMESPACE"
source deployment_vars.env && kubectl port-forward service/frontend-pos 8080:80 -n "$NAMESPACE"
```

---

## ✅ Deployment POS Completado

Si todos los pasos se ejecutaron exitosamente, el sistema POS debería estar ejecutándose junto al sistema base.

**Acceso al sistema**:
- **Sistema base**: `https://INGRESS_HOST` (con API en `/api`)
- **Sistema POS**: `https://INGRESS_HOST/pos` (con API en `/pos/api`)

**IMPORTANTE**: Revisa el archivo `RESUME_POS_KUBERNETES_DEPLOY.md` generado con todos los detalles del deployment POS.

**Configuración compartida**:
- PostgreSQL y Redis compartidos entre sistemas base y POS
- Autenticación unificada
- Mismo dominio con rutas separadas