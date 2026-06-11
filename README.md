# devops-core-pipelines

Repositorio centralizado de **pipelines base reutilizables** para GitHub Actions. Actúa como la fuente única de verdad para todos los workflows de CI/CD de la organización, siguiendo el patrón de **Reusable Workflows** de GitHub.

Cualquier repositorio de proyecto puede invocar estos pipelines con una sola línea usando `uses:`, sin copiar ni mantener lógica de CI/CD propia.

---

## Índice

- [Arquitectura](#arquitectura)
- [Catálogo de pipelines](#catálogo-de-pipelines)
  - [Reutilizables (workflow_call)](#reutilizables-workflow_call)
  - [Programados (schedule)](#programados-schedule)
- [Patrones de encadenamiento](#patrones-de-encadenamiento)
- [Configuración de secretos](#configuración-de-secretos)
- [Mantenimiento automático (Dependabot)](#mantenimiento-automático-dependabot)
- [Gobernanza del repositorio](#gobernanza-del-repositorio)

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────┐
│              devops-core-pipelines (este repo)          │
│                                                         │
│  ┌─────────────────────┐   ┌─────────────────────────┐  │
│  │  REUTILIZABLES       │   │  PROGRAMADOS            │  │
│  │  (workflow_call)     │   │  (schedule)             │  │
│  │                     │   │                         │  │
│  │  pr-validation      │   │  repo-deep-audit        │  │
│  │  script-validation  │   │  cloud-janitor          │  │
│  │  iac-simulation     │   │  drift-detection        │  │
│  │  iac-deployment     │   │  stale-resources        │  │
│  │  tf-dev-helper      │   └─────────────────────────┘  │
│  │  cost-estimation    │                                 │
│  │  container-build    │                                 │
│  │  release            │                                 │
│  │  rollback           │                                 │
│  │  env-promotion      │                                 │
│  │  smoke-test         │                                 │
│  └──────────┬──────────┘                                 │
└─────────────┼───────────────────────────────────────────┘
              │ uses: santiagodaros/devops-core-pipelines/...@main
              │
   ┌──────────▼──────────┐    ┌────────────────────────┐
   │  repo-proyecto-a    │    │  repo-proyecto-b        │
   │  .github/workflows/ │    │  .github/workflows/     │
   │  └── ci.yml         │    │  └── ci.yml             │
   └─────────────────────┘    └────────────────────────┘
```

Los pipelines programados (`schedule`) solo se ejecutan en **este repositorio**.
Los pipelines reutilizables (`workflow_call`) se invocan desde **cualquier repositorio de proyecto**.

---

## Catálogo de pipelines

### Reutilizables (`workflow_call`)

---

#### `pr-validation-base.yml`
**Propósito:** Validación integral de calidad en Pull Requests con reporte automático como comentario en el PR.

**Qué hace, paso a paso:**
1. Instala PSScriptAnalyzer en PowerShell Core
2. Escanea todos los `.ps1` del directorio configurado buscando aliases prohibidos, malas prácticas y violaciones de estilo
3. Corre Gitleaks sobre el historial completo del PR buscando secretos, tokens y credenciales expuestas
4. Publica un **comentario consolidado en el PR** con una tabla de resultados (✅/❌ por check). Si el comentario ya existe de una ejecución anterior, lo **actualiza** en lugar de duplicarlo

**Cuándo usarlo:** En cualquier workflow disparado por `pull_request`. Es el primer gate de calidad antes de que el código llegue a revisión humana.

**Inputs principales:**
| Input | Tipo | Default | Descripción |
|---|---|---|---|
| `scripts-directory` | string | `.` | Directorio con los `.ps1` a analizar |
| `fail-on-warnings` | boolean | `false` | Si `true`, los warnings también bloquean el PR |

**Permisos requeridos en el job caller:**
```yaml
permissions:
  pull-requests: write
```

---

#### `script-validation-base.yml`
**Propósito:** Análisis de calidad y seguridad de scripts PowerShell, pensado para runs en cualquier rama (no solo PRs).

**Qué hace, paso a paso:**
1. **Job `psscriptanalyzer`** (windows-latest): Instala PSScriptAnalyzer, descubre todos los `.ps1` recursivamente y los analiza buscando errores y warnings. Falla el pipeline si encuentra problemas
2. **Job `secret-scan`** (ubuntu-latest): Hace checkout con historial completo y corre Gitleaks. Guarda los resultados como artefacto SARIF

**Diferencia con `pr-validation-base.yml`:** Este pipeline no publica comentarios en PRs; solo pasa o falla el job. Ideal para branches de trabajo donde no hay PR abierto todavía.

**Inputs principales:**
| Input | Tipo | Default | Descripción |
|---|---|---|---|
| `working-directory` | string | `.` | Directorio raíz de los scripts |
| `fail-on-secrets` | boolean | `true` | Si `false`, Gitleaks reporta pero no bloquea |

---

#### `iac-simulation-base.yml`
**Propósito:** Validar el código de infraestructura (Bicep o Terraform) y detectar problemas de seguridad **antes** de cualquier despliegue real.

**Qué hace, paso a paso:**

*Para `iac-type: bicep`:*
1. Login a Azure con OIDC (sin credenciales estáticas)
2. Instala Bicep CLI y compila todos los `.bicep` del directorio para validar sintaxis
3. Si se provee `azure-resource-group`, ejecuta `az deployment group what-if` mostrando exactamente qué cambiaría en Azure sin aplicar nada

*Para `iac-type: terraform`:*
1. Corre `terraform init -backend=false` para resolver providers sin conectarse al backend remoto
2. Ejecuta `terraform validate` para verificar sintaxis y coherencia interna
3. Ejecuta `terraform fmt -check` para verificar que el formato sea correcto

*Para ambos tipos (job paralelo):*
4. **Job `checkov-security`**: Escanea el código IaC con Checkov buscando configuraciones inseguras (recursos sin cifrado, puertos abiertos, storage público, etc.). Publica los resultados en la pestaña **Security → Code scanning** del repositorio vía SARIF

**Cuándo usarlo:** En PRs que modifiquen código de infraestructura. Debe correr **antes** de `iac-deployment-base.yml`.

**Inputs principales:**
| Input | Tipo | Requerido | Descripción |
|---|---|---|---|
| `iac-type` | string | ✅ | `bicep` o `terraform` |
| `working-directory` | string | | Directorio del código IaC |
| `azure-resource-group` | string | | Para ejecutar What-If (solo Bicep) |
| `checkov-skip-checks` | string | | IDs a ignorar, ej: `CKV_AZURE_1,CKV_AZURE_2` |
| `fail-on-checkov-errors` | boolean | | Default `true` |

---

#### `iac-deployment-base.yml`
**Propósito:** Desplegar infraestructura real en Azure o AWS usando **identidades federadas OIDC** — sin ningún secreto de larga duración en el repositorio.

**Qué hace, paso a paso:**
1. Checkout del código
2. Autentica en Azure (OIDC) o AWS (OIDC) según `cloud-provider`
3. *Para Bicep:* Instala Bicep CLI y ejecuta `az deployment group create` con timestamp en el nombre para trazabilidad
4. *Para Terraform:* Ejecuta `terraform init` + `terraform apply -auto-approve` pasando el ambiente como variable

**Cuándo usarlo:** Solo en la rama `main` o en workflows con aprobación manual configurada como **GitHub Environment** (con reviewers requeridos).

**Inputs principales:**
| Input | Tipo | Requerido | Descripción |
|---|---|---|---|
| `cloud-provider` | string | ✅ | `azure` o `aws` |
| `iac-type` | string | ✅ | `bicep` o `terraform` |
| `environment` | string | ✅ | `dev`, `staging` o `prod` |
| `azure-resource-group` | string | | Grupo de recursos destino |

> **Prerequisito:** Debe haber corrido `iac-simulation-base.yml` y estar aprobado antes de invocar este pipeline.

---

#### `tf-development-helper.yml`
**Propósito:** Asistente automático de calidad para desarrolladores de Terraform. Corrige el código y genera documentación sin intervención manual.

**Qué hace, paso a paso:**
1. **Job `tf-format`**: Restaura caché de providers → corre `terraform fmt -recursive` → si hay cambios, sube los archivos formateados como artefacto
2. **Job `tf-lint`** (necesita `tf-format`): Restaura caché → instala TFLint → inicializa plugins → corre `tflint --recursive`. Reporta hallazgos pero puede configurarse para no bloquear
3. **Job `tf-docs`** (necesita `tf-lint`): Corre `terraform-docs` e inyecta la documentación generada (variables, outputs, recursos) entre los marcadores `<!-- BEGIN_TF_DOCS -->` y `<!-- END_TF_DOCS -->` del `README.md`. Hace **auto-commit** con todos los cambios directamente a la rama del proyecto

**Resultado visible:** Un commit nuevo firmado por `github-actions[bot]` con el código formateado y el README actualizado.

**Prerequisito en el README del módulo Terraform:**
```markdown
<!-- BEGIN_TF_DOCS -->
<!-- END_TF_DOCS -->
```

**Permisos requeridos en el job caller:**
```yaml
permissions:
  contents: write
```

---

#### `cost-estimation-base.yml`
**Propósito:** Calcular el impacto económico de un PR de Terraform antes de aprobarlo.

**Qué hace, paso a paso:**
1. Hace checkout de la rama **base** del PR (el estado actual)
2. Hace checkout de la rama **head** del PR (los cambios propuestos)
3. Corre `infracost breakdown` en ambas versiones para obtener el costo mensual de cada una
4. Corre `infracost diff` para calcular el **delta**: cuánto más (o menos) va a costar la infraestructura si se mergea el PR
5. Publica el reporte de costos como **comentario en el PR**, actualizándolo en cada nuevo push

**Prerequisito:** Configurar el secret `INFRACOST_API_KEY` (cuenta gratuita en [infracost.io](https://www.infracost.io)).

**Inputs principales:**
| Input | Tipo | Default | Descripción |
|---|---|---|---|
| `working-directory` | string | `.` | Directorio del código Terraform |
| `currency` | string | `USD` | Moneda del reporte |

---

#### `container-build-base.yml`
**Propósito:** Construir, escanear y publicar imágenes Docker de forma segura en el registry configurado.

**Qué hace, paso a paso:**
1. Autentica en el registry seleccionado (GHCR con `GITHUB_TOKEN`, ACR con OIDC, o ECR con OIDC)
2. Configura QEMU y Docker Buildx para builds multi-plataforma
3. Genera tags automáticos: por branch, por PR, semánticos (`v1.2.3`, `v1.2`) y por SHA corto
4. Construye la imagen con caché de GitHub Actions (`type=gha`) para acelerar builds sucesivos
5. **Escanea la imagen con Trivy** buscando CVEs. Publica resultados en Security → Code scanning
6. **Solo si Trivy pasa:** hace push de la imagen al registry. Si hay vulnerabilidades críticas, el push está bloqueado

**Inputs principales:**
| Input | Tipo | Requerido | Descripción |
|---|---|---|---|
| `image-name` | string | ✅ | Nombre de la imagen sin prefijo de registry |
| `registry` | string | | `ghcr` (default), `acr` o `ecr` |
| `severity-threshold` | string | | `CRITICAL` (default), `HIGH`, `MEDIUM` |
| `platforms` | string | | `linux/amd64` (default), o `linux/amd64,linux/arm64` |

---

#### `release-base.yml`
**Propósito:** Automatizar completamente el ciclo de versioning: análisis de commits → bump de versión → CHANGELOG → tag → GitHub Release.

**Qué hace, paso a paso:**
1. Analiza los mensajes de commit desde el último tag usando **Conventional Commits**:
   - `feat:` → bump **MINOR** (v1.1.0)
   - `fix:` / `perf:` → bump **PATCH** (v1.0.1)
   - `feat!:` o `BREAKING CHANGE` → bump **MAJOR** (v2.0.0)
   - `chore:` / `docs:` / `refactor:` → sin release
2. Si hay commits relevantes: actualiza `CHANGELOG.md`, crea el tag git (`v1.2.3`), hace commit del CHANGELOG y publica el **GitHub Release** con las release notes generadas automáticamente
3. Si no hay commits relevantes: termina silenciosamente sin crear release

**Outputs disponibles para encadenar:**
| Output | Descripción |
|---|---|
| `version` | Versión publicada, ej: `1.2.3` |
| `released` | `true` si se creó un nuevo release |

**Cuándo usarlo:** Solo en push a `main`, después de mergear un PR.

**Permisos requeridos en el job caller:**
```yaml
permissions:
  contents: write
```

---

---

#### `rollback-base.yml`
**Propósito:** Revertir un despliegue roto a la última versión estable en segundos. Reduce el MTTR de minutos a un click.

**Qué hace, paso a paso:**
1. Si no se especifica un tag, resuelve automáticamente el **penúltimo tag semántico** del repositorio (la versión anterior a la actual)
2. Hace checkout del código en ese tag específico
3. Requiere aprobación del **GitHub Environment** configurado (gate de seguridad antes de tocar producción)
4. Redesplega con Terraform o Bicep usando las credenciales OIDC
5. Genera un Job Summary con quién hizo el rollback, a qué versión y el timestamp

**Disparadores:** `workflow_call` desde otro pipeline (ej: al fallar un smoke test) **o** `workflow_dispatch` manual desde la UI de GitHub en emergencias.

**Inputs principales:**
| Input | Tipo | Requerido | Descripción |
|---|---|---|---|
| `environment` | string | ✅ | `dev`, `staging` o `prod` |
| `target-tag` | string | | Tag específico. Vacío = penúltimo tag automático |
| `iac-type` | string | | `terraform` (default) o `bicep` |

---

#### `env-promotion-base.yml`
**Propósito:** Formalizar el flujo `dev → staging → prod` con gates de aprobación humana entre cada ambiente.

**Qué hace, paso a paso:**
1. **Job `approval-gate`**: Se bloquea esperando que un reviewer apruebe la promoción desde la UI de GitHub (configurado como GitHub Environment con reviewers requeridos)
2. **Job `promote-iac`** o **`promote-container`**: Una vez aprobado, despliega el código IaC en el ambiente destino (Terraform/Bicep) **o** re-tagea y pushea la imagen Docker del ambiente origen al destino
3. **Job `smoke-test-post-promotion`** (opcional): Si se provee `smoke-test-url`, verifica que el ambiente destino responde correctamente después de la promoción

**Cuándo usarlo:** Como último job del pipeline de un PR mergeado a `main`, encadenado después de `iac-deployment-base.yml`.

**Inputs principales:**
| Input | Tipo | Requerido | Descripción |
|---|---|---|---|
| `promotion-type` | string | ✅ | `iac` o `container` |
| `source-environment` | string | ✅ | Ambiente de origen ya validado |
| `target-environment` | string | ✅ | Ambiente destino de la promoción |
| `smoke-test-url` | string | | URL para verificar post-promoción |

---

#### `smoke-test-base.yml`
**Propósito:** Verificar automáticamente que el sistema responde correctamente después de un despliegue o promoción.

**Qué hace, paso a paso:**
1. Espera N segundos configurables para que el deploy estabilice
2. Itera sobre un array JSON de endpoints configurados y verifica: HTTP status correcto + tiempo de respuesta dentro del límite
3. Publica una tabla de resultados en el Job Summary con status real, esperado y tiempo en ms por endpoint
4. **Opcionalmente:** lanza un test de carga con **k6** con usuarios virtuales y duración configurables. Umbral automático: < 5% errores, p95 < 3000ms

**Output disponible para encadenar:**
| Output | Descripción |
|---|---|
| `smoke-test-passed` | `true` si todos los endpoints pasaron |

**Inputs principales:**
| Input | Tipo | Requerido | Descripción |
|---|---|---|---|
| `base-url` | string | ✅ | URL base del sistema (ej: `https://api.miapp.com`) |
| `endpoints` | string | | JSON array de endpoints. Default: `[{"/health", GET, 200}]` |
| `max-response-time-ms` | number | | Default: `3000` ms |
| `run-load-test` | boolean | | Si `true`, lanza k6 |
| `k6-vus` | number | | Usuarios virtuales simultáneos. Default: `10` |

---

### Programados (`schedule`)

---

#### `repo-deep-audit-schedule.yml`
**Cuándo corre:** Cada **viernes a las 18:00 UTC** (automático) + manual vía `workflow_dispatch`.

**Qué hace, paso a paso:**
1. **Job `super-linter`**: Corre Super-Linter sobre todo el repositorio validando YAML, Bash, PowerShell, Terraform, JSON y Markdown simultáneamente. Reporta todos los problemas de estilo y calidad en el log del job
2. **Job `trivy-scan`**: Escanea el filesystem completo (o una imagen Docker si se especifica manualmente) con Trivy buscando dependencias con CVEs conocidos. Publica resultados en Security → Code scanning vía SARIF

**Parámetros del dispatch manual:**
| Input | Opciones | Descripción |
|---|---|---|
| `scan-target` | `fs` / `image` | Qué escanear con Trivy |
| `image-ref` | string | Referencia de imagen si `scan-target=image` |

---

#### `drift-detection-schedule.yml`
**Cuándo corre:** **Lunes a viernes a las 07:00 UTC** (antes del horario laboral) + manual vía `workflow_dispatch`.

**Qué hace, paso a paso:**
1. Login a Azure con OIDC
2. Corre `terraform plan -detailed-exitcode` contra la infraestructura real del ambiente configurado
3. **Exit code 0** = sin drift → termina exitosamente
4. **Exit code 2** = drift detectado → captura el output completo del plan
5. Busca si ya existe un **GitHub Issue** abierto de drift para ese ambiente:
   - Si existe: agrega un nuevo comentario con el diff actualizado
   - Si no existe: abre un nuevo Issue con el reporte completo, los recursos afectados y las acciones recomendadas
6. El Issue tiene labels `drift-detected`, `<ambiente>` e `infrastructure` para filtrado fácil

**Parámetros del dispatch manual:**
| Input | Opciones | Descripción |
|---|---|---|
| `working-directory` | string | Directorio del código Terraform |
| `environment` | `dev` / `staging` / `prod` | Ambiente a verificar |

---

#### `stale-resources-schedule.yml`
**Cuándo corre:** Cada **miércoles a las 09:00 UTC** + manual vía `workflow_dispatch`.

**Qué hace, paso a paso:**
1. Login a Azure con OIDC
2. Ejecuta 5 escaneos en paralelo:
   - **Discos**: sin VM asignada (`diskState == 'Unattached'`)
   - **IPs públicas**: sin `ipConfiguration` (sin usar)
   - **Snapshots**: creados hace más de N días (configurable, default 30)
   - **Resource Groups**: sin ningún recurso adentro
   - **Storage Accounts**: creados hace más de N días como proxy de inactividad
3. Calcula el total de recursos encontrados
4. Si hay recursos: cierra el Issue anterior de stale-resources (para no acumular) y abre uno nuevo con la lista detallada por tipo, labels `stale-resources`, `cost-optimization` e `infrastructure`

**Parámetros del dispatch manual:**
| Input | Opciones | Descripción |
|---|---|---|
| `dry-run` | `true` / `false` | Si `true`, solo reporta sin marcar para eliminación |
| `days-threshold` | number | Días de umbral para considerar recurso huérfano |

---

#### `cloud-janitor-schedule.yml`
**Cuándo corre:** De **lunes a viernes a las 20:00 UTC** (automático) + manual vía `workflow_dispatch`.

**Qué hace, paso a paso:**
1. Login a Azure con OIDC
2. Lista todas las VMs que tengan el tag `environment=dev` (o el valor configurado manualmente)
3. En modo real: envía señal de `deallocate` a todas las VMs encontradas (apagado completo, sin costo de cómputo)
4. Escribe un **Job Summary** con tabla: tag objetivo, cantidad de VMs apagadas, modo y timestamp

**Parámetros del dispatch manual:**
| Input | Opciones | Descripción |
|---|---|---|
| `dry-run` | `false` / `true` | Si `true`, lista las VMs pero no las apaga |
| `target-tag-value` | string | Valor del tag `environment` a buscar (default: `dev`) |

---

## Patrones de encadenamiento

### Patrón 1: CI completo para un repositorio de infraestructura Terraform

```yaml
# .github/workflows/ci.yml (en tu repo de infraestructura)
name: "CI"
on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # 1. En PRs: reporte de calidad como comentario
  validar-pr:
    if: github.event_name == 'pull_request'
    permissions:
      pull-requests: write
    uses: santiagodaros/devops-core-pipelines/.github/workflows/pr-validation-base.yml@main
    with:
      scripts-directory: "./scripts"
    secrets: inherit

  # 2. En PRs: validar y simular IaC (incluye Checkov)
  simular-iac:
    if: github.event_name == 'pull_request'
    uses: santiagodaros/devops-core-pipelines/.github/workflows/iac-simulation-base.yml@main
    with:
      iac-type: "terraform"
      working-directory: "./terraform"
    secrets: inherit

  # 3. En PRs: estimación de costos
  estimar-costos:
    if: github.event_name == 'pull_request'
    needs: simular-iac        # Solo si la simulación pasó
    permissions:
      pull-requests: write
    uses: santiagodaros/devops-core-pipelines/.github/workflows/cost-estimation-base.yml@main
    with:
      working-directory: "./terraform"
    secrets: inherit

  # 4. En PRs: autoformat + docs con auto-commit
  helper-terraform:
    if: github.event_name == 'pull_request'
    permissions:
      contents: write
    uses: santiagodaros/devops-core-pipelines/.github/workflows/tf-development-helper.yml@main
    with:
      working-directory: "./terraform"
    secrets: inherit

  # 5. En main: despliegue real (solo si viene de merge)
  desplegar:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: []                 # No depende de los jobs de PR
    uses: santiagodaros/devops-core-pipelines/.github/workflows/iac-deployment-base.yml@main
    with:
      cloud-provider: "azure"
      iac-type: "terraform"
      environment: "prod"
      azure-resource-group: "rg-mi-proyecto-prod"
    secrets: inherit

  # 6. En main: release semántico (después del despliegue)
  release:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: desplegar
    permissions:
      contents: write
    uses: santiagodaros/devops-core-pipelines/.github/workflows/release-base.yml@main
    secrets: inherit
```

**Flujo visual:**
```
PR abierto  →  pr-validation  ──────────────────────────────┐
            →  iac-simulation  →  cost-estimation            │  comentarios en PR
            →  tf-helper (auto-commit fmt+docs)              │
                                                             ▼
merge a main  →  iac-deployment  →  release (tag + CHANGELOG)
```

---

### Patrón 2: CI para un repositorio de aplicación con Docker

```yaml
# .github/workflows/ci.yml
jobs:
  # 1. Validar scripts y secretos en PRs
  validar-pr:
    if: github.event_name == 'pull_request'
    permissions:
      pull-requests: write
    uses: santiagodaros/devops-core-pipelines/.github/workflows/pr-validation-base.yml@main
    secrets: inherit

  # 2. Build, scan y push de imagen (en PRs y en main)
  build-imagen:
    uses: santiagodaros/devops-core-pipelines/.github/workflows/container-build-base.yml@main
    with:
      image-name: "mi-aplicacion"
      registry: "ghcr"
      severity-threshold: "CRITICAL"
    secrets: inherit

  # 3. Release solo en main
  release:
    if: github.ref == 'refs/heads/main'
    needs: build-imagen
    permissions:
      contents: write
    uses: santiagodaros/devops-core-pipelines/.github/workflows/release-base.yml@main
    secrets: inherit
```

**Flujo visual:**
```
PR abierto  →  pr-validation (comentario en PR)
            →  container-build → Trivy scan → push imagen con tag sha-xxxxx

merge a main  →  container-build → push imagen con tag v1.2.3
              →  release → CHANGELOG + GitHub Release v1.2.3
```

---

### Patrón 3: Pipeline mínimo (validación básica)

Para proyectos pequeños o primeras iteraciones, con solo 6 líneas:

```yaml
jobs:
  validar:
    uses: santiagodaros/devops-core-pipelines/.github/workflows/script-validation-base.yml@main
    with:
      working-directory: "./scripts"
    secrets: inherit
```

---

## Configuración de secretos

### Secretos de Azure (OIDC — sin credenciales estáticas)

| Secreto | Usado en | Descripción |
|---|---|---|
| `AZURE_CLIENT_ID` | iac-simulation, iac-deployment, container-build (ACR), cloud-janitor | Client ID de la App Registration |
| `AZURE_TENANT_ID` | Todos los pipelines de Azure | Tenant ID del directorio Azure AD |
| `AZURE_SUBSCRIPTION_ID` | Todos los pipelines de Azure | ID de la suscripción destino |

### Secretos de AWS (OIDC)

| Secreto | Usado en | Descripción |
|---|---|---|
| `AWS_ROLE_ARN` | iac-deployment (AWS), container-build (ECR) | ARN del rol IAM a asumir |

### Otros secretos

| Secreto | Usado en | Descripción |
|---|---|---|
| `INFRACOST_API_KEY` | cost-estimation | API key gratuita de infracost.io |
| `GITLEAKS_LICENSE` | script-validation, pr-validation | Opcional, para features avanzados de Gitleaks |

### Configurar Federated Credentials en Azure (una sola vez por repo)

```bash
az ad app federated-credential create \
  --id <APP_OBJECT_ID> \
  --parameters '{
    "name": "github-<nombre-repo>",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:santiagodaros/<nombre-repo>:environment:prod",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

---

## Mantenimiento automático (Dependabot)

Este repositorio tiene Dependabot configurado para actualizar automáticamente las versiones de todas las GitHub Actions cada **lunes a las 9:00 AM (Argentina)**. Los PRs de actualización se crean con el label `dependencies` y se asignan para revisión.

Esto garantiza que todos los proyectos que consumen estos pipelines siempre estén usando versiones seguras y actualizadas de las actions de la comunidad.

---

## Gobernanza del repositorio

### CODEOWNERS
El archivo `CODEOWNERS` en la raíz requiere que **@santiagodaros** apruebe cualquier cambio a workflows, Dependabot o documentación. Esto previene modificaciones no revisadas que afectarían a todos los repositorios consumidores.

### Branch protection rules (configurar manualmente una sola vez)

Ir a **Settings → Branches → Add rule** para la rama `main` y activar:

| Regla | Valor | Por qué |
|---|---|---|
| Require a pull request before merging | ✅ | Nadie pushea directo a main |
| Required approvals | `1` | Al menos 1 revisor humano |
| Require review from Code Owners | ✅ | Fuerza revisión del CODEOWNERS |
| Require status checks to pass | ✅ | Checks de CI deben estar verdes |
| Do not allow bypassing the above settings | ✅ | Ni los admins saltean las reglas |
| Restrict force pushes | ✅ | Previene reescritura del historial |

> Sin estas reglas, los pipelines son buenos pero se pueden saltear con un push directo a `main`.

---

## Contribuciones

1. Crea una rama: `git checkout -b feature/nuevo-pipeline`
2. Agrega o modifica el pipeline en `.github/workflows/`
3. Abre un Pull Request con descripción del cambio y casos de uso
4. Requiere aprobación de al menos 1 revisor antes de mergear a `main`

---

*Mantenido por el equipo de DevOps & Platform Engineering.*
