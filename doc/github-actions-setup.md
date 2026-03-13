# Configuracion de GitHub Actions

Guia operativa para dejar correctamente configurados secrets, variables y environments en GitHub.

Ruta en GitHub:
- Settings -> Secrets and variables -> Actions
- Settings -> Environments

## 1. Repository secrets

Crear exactamente estos nombres en Repository secrets:

| Nombre | Requerido para | Formato |
|--------|----------------|---------|
| AZURE_CREDENTIALS | Azure (infra y deploy) | JSON |
| AWS_ACCESS_KEY_ID | AWS (infra y deploy) | texto |
| AWS_SECRET_ACCESS_KEY | AWS (infra y deploy) | texto |

### Formato de AZURE_CREDENTIALS

```json
{
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

Archivo de referencia:
- doc/azure-credentials-example.json

## 2. Repository variables

Crear exactamente estos nombres en Repository variables:

| Nombre | Ejemplo | Uso principal |
|--------|---------|---------------|
| CLOUD_PROVIDER | azure o aws | Selecciona el proveedor en los pipelines CD |
| RESOURCE_GROUP | kml_rg_main-e8ca19bff36b4228 | Azure infra y deploy |
| AKS_NAME | cve01az | Azure cluster |
| CONTAINER_NAME | ed-team-app | Base de nombre de imagen |
| REGISTRY_NAME | acredteamdevsecops | Azure Container Registry |
| AZURE_LOCATION | eastus | Region de Azure |
| AWS_REGION | us-east-1 | Region AWS |
| EKS_CLUSTER_NAME | cve01aws | AWS cluster |
| ECR_REPOSITORY_NAME | registryedteamdevsecops | AWS ECR |

Archivos de referencia:
- doc/azure-variables-example.json
- doc/aws-variables-example.json

## 3. Environments

Crear estos environments:
- desarrollo
- production

Estado esperado:
- production: usado por jobs deploy en .github/workflows/cd.yml y .github/workflows/cd-go.yml
- desarrollo: reservado para pruebas/expansion (no obligatorio para los workflows actuales)

### Configuracion recomendada para production

En Settings -> Environments -> production configurar:

1. Deployment branches
- Permitir solo main

2. Required reviewers
- Activar al menos 1 reviewer para aprobar deploys a production

3. Wait timer (opcional)
- 5 a 10 minutos para ventana de control antes de deploy

4. Environment secrets
- Actualmente no son obligatorios para este repo
- Mantener vacio salvo que quieras mover secrets desde Repository secrets a nivel Environment

## 4. Mapeo rapido variable -> workflow

| Workflow | Variables/secrets clave |
|----------|-------------------------|
| .github/workflows/ci.yml | Sin variables cloud obligatorias |
| .github/workflows/ci-go.yml | Sin variables cloud obligatorias |
| .github/workflows/cd.yml | CLOUD_PROVIDER, AZURE_* o AWS_*, AZURE_CREDENTIALS o AWS keys |
| .github/workflows/cd-go.yml | CLOUD_PROVIDER, AZURE_* o AWS_*, AZURE_CREDENTIALS o AWS keys |
| .github/workflows/cd-go-multicloud.yml | Requiere ambos providers configurados |
| .github/workflows/infraestructure.yml | Requiere provider elegido segun input |
| .github/workflows/infra-multicloud.yml | Requiere ambos providers configurados |

## 5. Checklist final de validacion

1. Secrets creados con nombres exactos
2. Variables creadas con nombres exactos
3. Environment production creado
4. Environment production con branch main permitido
5. Lanzar workflow manual .github/workflows/infraestructure.yml con action=create para validar credenciales
6. Hacer push a main para validar .github/workflows/cd.yml o .github/workflows/cd-go.yml segun app
