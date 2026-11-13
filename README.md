# Reto DevSecOps - ED Team

Dos APIs (Python Flask y Go) con pipeline DevSecOps completo sobre GitHub Actions: análisis de seguridad estático y de contenedor, despliegue automatizado en Kubernetes y soporte multi-cloud (Azure AKS / AWS EKS).

> Para diagramas de arquitectura e infraestructura ver [ARCHITECTURE.md](ARCHITECTURE.md).

## APIs

Ambas APIs exponen los mismos endpoints:

| Endpoint | Respuesta |
|----------|-----------|
| `GET /` | `{"message": "Hola ED Team"}` |
| `GET /health` | `{"status": "healthy"}` |

| App | Lenguaje | Puerto | Directorio |
|-----|----------|--------|------------|
| Python (Flask + Gunicorn) | Python 3.13 | 5000 | `src/` |
| Go (net/http) | Go 1.23 | 8080 | `src-go/` |

## Estructura del proyecto

```
.github/workflows/
  ci.yml                  # CI Python — SAST, SCA, escaneo de imagen
  ci-go.yml               # CI Go — tests, SAST, SCA, escaneo de imagen
  cd.yml                  # CD Python — build, release, deploy (un cloud)
  cd-go.yml               # CD Go — build, release, deploy (un cloud)
  cd-go-multicloud.yml    # CD Go Multi-Cloud — build único → ACR + ECR en paralelo (manual)
  infraestructure.yml     # Infraestructura — provisión por cloud (manual)
  infra-multicloud.yml    # Infraestructura Multi-Cloud — dispara Azure + AWS a la vez (manual)
doc/
  azure-credentials-example.json   # Ejemplo del secret AZURE_CREDENTIALS
  azure-variables-example.json     # Ejemplo de variables para Azure
  aws-variables-example.json       # Ejemplo de variables para AWS
src/
  app.py                  # Aplicación Flask
  Dockerfile              # Imagen multi-stage (Python 3.13 + Gunicorn)
  requirements.txt        # Flask 2.3.3, Gunicorn 23.0.0
src-go/
  main.go                 # Aplicación Go
  main_test.go            # Tests unitarios
  go.mod                  # Módulo Go
  Dockerfile              # Imagen multi-stage (Go 1.23)
k8s/                      # Manifiestos Kubernetes — Python app
  deployment.yml          # 2 réplicas, rolling update, security context
  service.yml             # ClusterIP (80 → 5000)
  ingress.yml             # Ingress Traefik
  hpa.yml                 # HPA 2-10 pods (CPU 70% / RAM 80%)
k8s-go/                   # Manifiestos Kubernetes — Go app
  deployment.yml          # 2 réplicas, rolling update, security context
  service.yml             # ClusterIP (80 → 8080)
  ingress.yml             # Ingress Traefik (prefijo /go)
  middleware.yml          # Traefik Middleware strip-prefix /go
  hpa.yml                 # HPA 2-10 pods (CPU 70% / RAM 80%)
```

## Pipelines

### CI Python — `ci.yml`
Se ejecuta en push y PR a `main` cuando hay cambios en `src/`.

| Job | Herramienta | Cuándo |
|-----|-------------|--------|
| SAST | CodeQL (Python, queries `security-extended`) | siempre |
| SCA | `dependency-review-action` | solo PRs |
| Container Security | Anchore/Grype → SARIF a GitHub Security | siempre |

### CI Go — `ci-go.yml`
Se ejecuta en push y PR a `main` cuando hay cambios en `src-go/`.

| Job | Herramienta | Cuándo |
|-----|-------------|--------|
| Test | `go test -v -race ./...` | siempre |
| SAST | CodeQL (Go, queries `security-extended`) | siempre |
| SCA | `dependency-review-action` | solo PRs |
| Container Security | Anchore/Grype → SARIF a GitHub Security | siempre |

### CD Python — `cd.yml`
Se ejecuta en push a `main` (ignora cambios en `.github/`). El cloud destino se determina por `CLOUD_PROVIDER`.

```
buildImage → release → deploy (environment: production)
```

1. **buildImage** — autentifica con ACR o ECR, construye la imagen `src/Dockerfile` y la publica con tag `<sha-short>`.
2. **release** — crea un GitHub Release con tag `v<YYYYMMDD>-<sha-short>`.
3. **deploy** — obtiene credenciales del cluster (AKS o EKS), aplica los manifiestos de `k8s/` y actualiza la imagen.

### CD Go — `cd-go.yml`
Se ejecuta en push a `main` cuando hay cambios en `src-go/` o `k8s-go/`. Flujo idéntico al CD Python pero usa `src-go/Dockerfile`, `k8s-go/` y tag `go-v<YYYYMMDD>-<sha-short>`.

```
buildImage (incluye go test -race) → release → deploy
```

### CD Go Multi-Cloud — `cd-go-multicloud.yml`
Ejecución **manual** (`workflow_dispatch`). Construye la imagen **una sola vez** en GHCR y la promueve en paralelo a ACR y ECR, luego despliega en AKS y EKS simultáneamente.

```
build_once (GHCR) ──┬── promote_azure (ACR) ── deploy_azure (AKS)
                    ├── promote_aws   (ECR) ── deploy_aws   (EKS)
                    └── release (tag go-mc-v<YYYYMMDD>-<sha-short>)
```

Ventaja: una sola compilación de Docker garantiza que la imagen que llega a ambos clouds es binariamente idéntica.

### Infraestructura — `infraestructure.yml`
Ejecución **manual**. Seleccionar proveedor (`azure` / `aws`) y acción (`create` / `destroy`).

- **Azure + kubernetes** — crea ACR + AKS (1 nodo `Standard_D2s_v3`) + instala Traefik vía Helm + configura el secret `registry-secret` en el cluster.
- **Azure + serverless** — crea `Function App` (Python 3.11, Functions v4, Linux Consumption) y su `Storage Account`.
- **AWS** — crea ECR + EKS (1 nodo `t3.medium`) + infraestructura base de red y acceso al cluster.
- **destroy** — elimina todos los recursos creados.

### Infraestructura Multi-Cloud — `infra-multicloud.yml`
Ejecución **manual**. Seleccionar acción (`create` / `destroy`) y dispara `infraestructure.yml` simultáneamente para Azure y AWS.

## Configuración

Configura todo en **GitHub → Settings → Secrets and variables → Actions**.

Guia completa paso a paso (secrets, variables y environments): [doc/github-actions-setup.md](doc/github-actions-setup.md)

### Secrets

| Secret | Proveedor | Descripción |
|--------|-----------|-------------|
| `AZURE_CREDENTIALS` | Azure | JSON del Service Principal (ver `doc/azure-credentials-example.json`) |
| `AWS_ACCESS_KEY_ID` | AWS | Access Key ID de IAM |
| `AWS_SECRET_ACCESS_KEY` | AWS | Secret Access Key de IAM |

> `GITHUB_TOKEN` se genera automáticamente.

Formato de `AZURE_CREDENTIALS`:
```json
{
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### Variables de repositorio

| Variable | Descripción | Ejemplo Azure | Ejemplo AWS |
|----------|-------------|---------------|-------------|
| `CLOUD_PROVIDER` | Proveedor cloud activo | `azure` | `aws` |
| `RESOURCE_GROUP` | Resource Group existente | `rg-edteam-devsecops` | — |
| `AKS_NAME` | Nombre del cluster AKS | `aks-edteam-cluster` | — |
| `CONTAINER_NAME` | Nombre base de la imagen | `ed-team-app` | — |
| `REGISTRY_NAME` | Nombre del ACR (sin `.azurecr.io`) | `acredteamdevsecops` | — |
| `AZURE_LOCATION` | Región de Azure | `eastus` | — |
| `FUNCTION_APP_NAME` | Nombre del Azure Function App (serverless) | `func-edteam-devsecops` | — |
| `FUNCTION_STORAGE_ACCOUNT` | Storage account usado por Azure Functions | `edteamsa001` | — |
| `AWS_REGION` | Región de AWS | — | `us-east-1` |
| `EKS_CLUSTER_NAME` | Nombre del cluster EKS | — | `eks-ed-team-cluster` |
| `ECR_REPOSITORY_NAME` | Nombre del repositorio ECR | — | `ed-team-app` |

> Los archivos de ejemplo están en `doc/`.

### Environments de GitHub

Crear estos environments en **GitHub → Settings → Environments**:

- `production` (obligatorio para deploy, lo usan `cd.yml` y `cd-go.yml`)
- `desarrollo` (opcional para pruebas)

Configuración recomendada para `production`:

1. Permitir despliegues solo desde la rama `main`.
2. Configurar al menos 1 revisor obligatorio antes de desplegar.
3. (Opcional) Agregar `wait timer` de 5 a 10 minutos.
4. No requiere environment secrets actualmente (se usan repository secrets).

## Ejecución local

### Requisitos

| Herramienta | macOS | Linux/Windows |
|-------------|-------|---------------|
| Python 3.x | `brew install python` | [python.org](https://www.python.org/downloads/) |
| Go 1.23+ | `brew install go` | [go.dev/dl](https://go.dev/dl/) |
| Docker | [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/) | [Docker Desktop](https://docs.docker.com/desktop/) |

### Python

```bash
cd src
pip3 install -r requirements.txt
python3 app.py
# http://localhost:5000
```

### Go

```bash
cd src-go
go run main.go
# http://localhost:8080
```

Tests unitarios:
```bash
cd src-go
go test -v -race ./...
```

### Docker

```bash
# Python
docker build -t ed-team-app src/
docker run -p 5000:5000 ed-team-app

# Go
docker build -t ed-team-app-go src-go/
docker run -p 8080:8080 ed-team-app-go
```
