# Savia Up - Manifiestos de Infraestructura y Kubernetes

Este directorio contiene la arquitectura de despliegue en Kubernetes para la plataforma **Savia Up**, organizada por ambientes (`dev`, `qa`, `prod`) y desacoplada en microservicios independientes.

---

## 1. Proyectos e Imágenes en GHCR

Las imágenes se construyen automáticamente con GitHub Actions y se publican en **GitHub Container Registry (GHCR)**:

| Componente | Repositorio | Imagen GHCR |
|---|---|---|
| **Backend API** | `saviaup.backend` | `ghcr.io/clepineuxp/saviaup-backend:<tag>` |
| **Frontend Web** | `saviaup.frontend` | `ghcr.io/clepineuxp/saviaup-frontend:<tag>` |
| **Admin Backend API** | `saviaup.backend.admin` | `ghcr.io/clepineuxp/saviaup-backend-admin:<tag>` |
| **Admin Frontend Web** | `saviaup.frontend.admin` | `ghcr.io/clepineuxp/saviaup-frontend-admin:<tag>` |

### Versionamiento por ambiente:
- **Dev**: tag `dev` o `dev-<commit-sha>`
- **QA**: tag `qa` o `qa-<commit-sha>`
- **Prod**: tag `latest` o semver `v1.x.x`

---

## 2. Mapeo de Dominios e Ingress

| Ambiente | Namespace | Componente | Dominio Ingress | Servicio Interno |
|---|---|---|---|---|
| **Dev** | `saviaup-dev` | Frontend Web | `dev.saviaup.com` | `frontend:80` |
| **Dev** | `saviaup-dev` | Backend API | `dev.api.saviaup.com` | `backend:8080` |
| **Dev** | `saviaup-dev` | Admin Frontend | `dev.admin.saviaup.com` | `admin-frontend:80` |
| **Dev** | `saviaup-dev` | Admin Backend | `dev.api.admin.saviaup.com` | `admin-backend:8080` |
|---|---|---|---|---|
| **QA** | `saviaup-qa` | Frontend Web | `qa.saviaup.com` | `frontend:80` |
| **QA** | `saviaup-qa` | Backend API | `qa.api.saviaup.com` | `backend:8080` |
| **QA** | `saviaup-qa` | Admin Frontend | `qa.admin.saviaup.com` | `admin-frontend:80` |
| **QA** | `saviaup-qa` | Admin Backend | `qa.api.admin.saviaup.com` | `admin-backend:8080` |
|---|---|---|---|---|
| **Prod** | `saviaup-prod` | Frontend Web | `saviaup.com` / `www.saviaup.com` | `frontend:80` |
| **Prod** | `saviaup-prod` | Backend API | `api.saviaup.com` | `backend:8080` |
| **Prod** | `saviaup-prod` | Admin Frontend | `admin.saviaup.com` | `admin-frontend:80` |
| **Prod** | `saviaup-prod` | Admin Backend | `api.admin.saviaup.com` | `admin-backend:8080` |

---

## 3. Estructura del Directorio

```text
saviaup.environments/
├── bootstrap/
│   └── saviaup-infra-bootstrap.yaml     # Namespaces e ingress-nginx controller
├── dev/
│   ├── backend/                         # ConfigMap, Secret, Deployment, Service
│   ├── frontend/                        # ConfigMap, Secret, Deployment, Service
│   ├── admin-backend/                   # ConfigMap, Secret, Deployment, Service
│   ├── admin-frontend/                  # ConfigMap, Secret, Deployment, Service
│   ├── media/                           # PVC, Nginx de archivos y Service
│   ├── ingress.yaml                     # Reglas de enrutamiento y TLS para dev.*
│   ├── ghcr-secret.yaml                 # Plantilla del secret de pull de imágenes
│   └── kustomization.yaml               # Manifiesto Kustomize
├── qa/
│   ├── backend/, frontend/, ...         # Componentes configurados para QA
│   ├── ingress.yaml                     # Reglas de enrutamiento y TLS para qa.*
│   └── kustomization.yaml
└── prod/
    ├── backend/, frontend/, ...         # Componentes configurados para Prod (alta disponibilidad)
    ├── ingress.yaml                     # Reglas de enrutamiento y TLS para producción
    └── kustomization.yaml
```

---

## 4. Guía de Despliegue

### Paso 1: Bootstrap de Infraestructura (Namespaces e Ingress Nginx)

```bash
kubectl apply -f bootstrap/saviaup-infra-bootstrap.yaml
```

### Paso 2: Autenticación con GitHub Container Registry (GHCR)

En cada namespace (`saviaup-dev`, `saviaup-qa`, `saviaup-prod`), crear el secreto de autenticación de Docker para descargar las imágenes privadas de GHCR:

```bash
# Para DEV
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<GITHUB_USER> \
  --docker-password=<GITHUB_PAT_O_TOKEN> \
  --namespace=saviaup-dev

# Para QA
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<GITHUB_USER> \
  --docker-password=<GITHUB_PAT_O_TOKEN> \
  --namespace=saviaup-qa

# Para PROD
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<GITHUB_USER> \
  --docker-password=<GITHUB_PAT_O_TOKEN> \
  --namespace=saviaup-prod
```

> **Nota**: Asegúrate de que el token personal (PAT) de GitHub tenga el permiso `read:packages`.

### Paso 3: Configurar Secretos de Base de Datos y JWT

Editar o crear los secrets de base de datos (`backend-secret`, `admin-backend-secret`) reemplazando los valores por defecto:

```bash
# Ejemplo: editar secret interactivo en DEV
kubectl edit secret backend-secret -n saviaup-dev
```

### Paso 4: Desplegar el Ambiente con Kustomize

Para desplegar un ambiente completo de forma atómica:

```bash
# Desplegar ambiente DEV
kubectl apply -k ./dev

# Desplegar ambiente QA
kubectl apply -k ./qa

# Desplegar ambiente PROD
kubectl apply -k ./prod
```

### Paso 5: Verificar el Despliegue

```bash
# Ver pods en dev
kubectl get pods -n saviaup-dev

# Ver servicios en dev
kubectl get svc -n saviaup-dev

# Ver ingress
kubectl get ingress -n saviaup-dev
```

## Afinidad y descubrimiento de agentes de impresión

El backend de producción conserva dos réplicas. El descubrimiento pendiente de agentes se comparte mediante PostgreSQL y no depende de memoria local. El ingress de API usa afinidad por cookie para mantener estables las conexiones SignalR operativas, mientras que el registro y polling de vinculación pueden cambiar de réplica sin perder estado.

Los manifiestos declaran las redes privadas del clúster en `ReverseProxy__KnownNetworks__*`. ASP.NET Core solo procesa `X-Forwarded-For` cuando la conexión proviene de esas redes confiables; no se deben volver a limpiar las listas de proxies conocidos ni aceptar cabeceras de origen directamente desde Internet. Si cambia el CIDR de pods o del ingress, actualiza estos valores antes del despliegue.

## Almacenamiento de imágenes en PVC

Cada ambiente crea su propio claim `media-storage` con la clase `local-path`: 5 GiB en dev, 10 GiB en QA y 50 GiB en producción. El backend lo monta con escritura en `/var/lib/saviaup/files`; el deployment `media-server` monta el mismo volumen en modo lectura y sirve las referencias WebP bajo `/pvc/{tenant}/...`.

El ingress publica `/pvc` tanto en el host web principal como en el host administrativo. Nginx admite únicamente `GET` y `HEAD` y responde con caché pública inmutable de un año; las referencias cambian cuando se reemplaza una imagen, evitando invalidaciones manuales.

`local-path` y `ReadWriteOnce` son apropiados para un k3s de un solo nodo. Antes de distribuir réplicas entre varios nodos se debe cambiar el claim a una clase con `ReadWriteMany` (por ejemplo Longhorn/NFS) o implementar otro adaptador de `IFileStorage`, como S3. El PVC no sustituye una política de backup: debe incluirse el volumen de cada ambiente en las copias de seguridad.

Comprobaciones útiles:

```bash
kubectl get pvc media-storage -n saviaup-dev
kubectl get pods -n saviaup-dev -l app.kubernetes.io/name=media-server
curl -I https://dev.saviaup.com/pvc/<tenant>/<ruta>.webp
```
