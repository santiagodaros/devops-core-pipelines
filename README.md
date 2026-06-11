# devops-core-pipelines

Repositorio centralizado de **pipelines base reutilizables** para GitHub Actions. Actúa como la fuente única de verdad para los workflows de CI/CD compartidos por todos los proyectos de la organización, siguiendo el patrón de **Reusable Workflows** de GitHub.

---

## Estructura del repositorio

```
devops-core-pipelines/
├── .github/
│   └── workflows/
│       ├── script-validation-base.yml   # Calidad de scripts PowerShell + escaneo de secretos
│       ├── iac-simulation-base.yml      # Validación sintáctica y simulación What-If/Plan
│       ├── iac-deployment-base.yml      # Despliegue seguro con identidades federadas (OIDC)
│       ├── repo-deep-audit-schedule.yml # Auditoría semanal (Super-Linter + Trivy)
│       └── cloud-janitor-schedule.yml   # Apagado automático de VMs de desarrollo
├── README.md
└── .gitignore
```

---

## Pipelines disponibles

### 🔧 Pipelines Reutilizables (`workflow_call`)

Estos pipelines son **"Skills" interconectables**: se invocan desde los workflows de otros repositorios usando la propiedad `uses:`. No se ejecutan solos.

| Archivo | Propósito |
|---|---|
| `script-validation-base.yml` | Analiza scripts `.ps1` con PSScriptAnalyzer y escanea secretos con Gitleaks |
| `iac-simulation-base.yml` | Valida sintaxis Bicep/Terraform y ejecuta `What-If` o `terraform plan` |
| `iac-deployment-base.yml` | Despliega IaC en Azure o AWS usando OIDC (sin secretos estáticos) |

### ⏰ Pipelines Programados (`schedule`)

Se ejecutan automáticamente según el horario configurado. También pueden lanzarse manualmente con `workflow_dispatch`.

| Archivo | Horario | Propósito |
|---|---|---|
| `repo-deep-audit-schedule.yml` | Viernes 18:00 UTC | Super-Linter + Trivy sobre todo el repositorio |
| `cloud-janitor-schedule.yml` | Lun-Vie 20:00 UTC | Apaga VMs etiquetadas como `environment=dev` en Azure |

---

## Cómo usar los pipelines reutilizables desde tu proyecto

Desde cualquier repositorio de tu organización, crea un archivo `.github/workflows/ci.yml` e invoca el pipeline base usando `uses:`:

```yaml
# .github/workflows/ci.yml (en tu repositorio esclavo/proyecto)
jobs:
  validar-scripts:
    uses: TU_ORG/devops-core-pipelines/.github/workflows/script-validation-base.yml@main
    with:
      working-directory: "./scripts"
      fail-on-secrets: true
    secrets: inherit
```

> **Nota:** Reemplaza `TU_ORG` por tu nombre de usuario u organización de GitHub.

### Ejemplo: Invocar el pipeline de despliegue IaC

```yaml
# .github/workflows/deploy.yml (en tu repositorio de infraestructura)
jobs:
  desplegar-dev:
    uses: TU_ORG/devops-core-pipelines/.github/workflows/iac-deployment-base.yml@main
    with:
      cloud-provider: "azure"
      iac-type: "bicep"
      environment: "dev"
      azure-resource-group: "rg-mi-proyecto-dev"
    secrets: inherit
```

---

## Configuración de secretos necesarios

Para los pipelines de Azure, configura los siguientes **secretos** en tu repositorio (o en la organización):

| Secreto | Descripción |
|---|---|
| `AZURE_CLIENT_ID` | Client ID de la App Registration con Federated Credentials |
| `AZURE_TENANT_ID` | Tenant ID del directorio de Azure AD |
| `AZURE_SUBSCRIPTION_ID` | ID de la suscripción Azure de destino |

> Los pipelines usan **OIDC (OpenID Connect)** para autenticarse. Esto elimina la necesidad de secretos estáticos de larga duración.

### Configurar Federated Credentials en Azure

```bash
az ad app federated-credential create \
  --id <APP_OBJECT_ID> \
  --parameters '{
    "name": "github-actions",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:TU_ORG/tu-repositorio:environment:prod",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

---

## Versionado y mejores prácticas

- Siempre referencia los pipelines desde una **rama estable** (`@main`) o **tag** (`@v1.0.0`).
- Usa `secrets: inherit` para pasar los secretos del repositorio invocador automáticamente.
- Los pipelines programados solo se ejecutan desde el **repositorio donde están definidos** (este repositorio), no desde los repositorios esclavos.
- Para modificaciones, abre un Pull Request y solicita revisión antes de mergear a `main`.

---

## Contribuciones

1. Crea una rama: `git checkout -b feature/nuevo-pipeline`
2. Agrega o modifica el pipeline en `.github/workflows/`
3. Abre un Pull Request con descripción del cambio y casos de uso
4. Requiere aprobación de al menos 1 revisor antes de mergear

---

*Mantenido por el equipo de DevOps & Platform Engineering.*
