# Reusable GitHub Actions

This repository contains a collection of reusable GitHub Actions workflows designed to streamline CI/CD processes for various types of projects. The workflows are organized into different categories based on their functionality, including Docker, Helm, and Argo CD.

## Project Structure

- **.github/workflows/**: Contains the GitHub Actions workflows for different tasks.
  - **docker-build-push.yml**: Workflow for building and pushing Docker images.
  - **docker-jib-push.yml**: Workflow for building and pushing Docker images using Jib.
  - **spotless-check.yml**: Workflow for checking code formatting using Spotless.
  - **helm-ci.yml**: Workflow for continuous integration with Helm.
  - **argocd-deploy.yml**: Workflow for deploying applications using Argo CD.
  - **argocd-manage.yml**: Workflow for managing Argo CD applications.
  
- **aplicatoins-type/**: Contains example projects demonstrating the use of the workflows.
  - **frontend-project/.github/workflows/ci.yml**: CI workflow for a frontend project.
  - **backend-project/.github/workflows/ci.yml**: CI workflow for a backend project.
  - **helm-project/.github/workflows/ci.yml**: CI workflow for a Helm project.

- **docs/**: Documentation for the workflows.
  - **docker-workflows.md**: Documentation for Docker workflows.
  - **helm-workflows.md**: Documentation for Helm workflows.
  - **argocd-workflows.md**: Documentation for Argo CD workflows.

## Getting Started

To use the reusable GitHub Actions in your own projects, you can reference the workflows in your repository's `.github/workflows/` directory. Make sure to customize the workflows according to your project's requirements.

## Contributing

Contributions are welcome! Please feel free to submit a pull request or open an issue if you have suggestions or improvements.

## License

This project is licensed under the MIT License. See the LICENSE file for more details.

## Exemplos de Uso por Tipo de Aplicação

### 1. Backend Project
1- create file `.github/workflows/ci.yml`
```code
mkdir -p .github/workflows/
touch .github/workflows/ci.yml
```

2- add the following content to the file `.github/workflows/ci.yml`

```yaml
name: Backend Java CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  code-quality:
    uses: lolmeida/github-actions/.github/workflows/spotless-check.yml@main
    with:
      java_version: '21'
      auto_fix: ${{ github.event_name == 'pull_request' }}
      fail_on_error: true

  build-and-push:
    needs: code-quality
    if: github.event_name == 'push'
    uses: lolmeida/github-actions/.github/workflows/docker-jib-push.yml@main
    with:
      java_version: '21'
      image_name: auth-be
      maven_profiles: ${{ github.ref_name == 'main' && 'prod' || 'dev' }}
      skip_tests: false
    secrets:
      DOCKER_HUB_USER: ${{ secrets.DOCKER_HUB_USER }}
      DOCKER_HUB_TOKEN: ${{ secrets.DOCKER_HUB_TOKEN }}
```
---

### 2. Frontend Project
1- create file `.github/workflows/ci.yml`
```code
mkdir -p .github/workflows/
touch .github/workflows/ci.yml
```

2- add the following content to the file `.github/workflows/ci.yml`
```yaml
name: Frontend CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-deploy:
    uses: lolmeida/github-actions/.github/workflows/docker-build-push.yml@main
    with:
      image_name: auth-fe
      dockerfile_path: ./Dockerfile
      build_context: .
      platforms: linux/amd64,linux/arm64
      api_base_url: ${{ github.ref_name == 'main' && 'https://api.production.com' || 'https://api.staging.com' }}
      tag_strategy: ${{ github.ref_name == 'main' && 'latest' || github.ref_name }}
    secrets:
      DOCKER_HUB_USER: ${{ secrets.DOCKER_HUB_USER }}
      DOCKER_HUB_TOKEN: ${{ secrets.DOCKER_HUB_TOKEN }}
```
---

### 3. Helm Project
1- create file `.github/workflows/ci.yml`
```code
mkdir -p .github/workflows/
touch .github/workflows/ci.yml
```

2- add the following content to the file `.github/workflows/ci.yml`

```yaml
name: Infrastructure Pipeline

on:
  push:
    branches: [main]
    paths:
      - 'helm/**'
      - 'argocd-apps/**'
      - 'overlays/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy environment'
        type: choice
        options: ['staging', 'production', 'both']
        default: 'staging'

jobs:
  # 1. Validar charts Helm
  validate-helm:
    uses: lolmeida/github-actions/.github/workflows/helm-ci.yml@main
    with:
      run_argocd_apps: 'true'
      run_helm: 'true'
      envs: 'staging,production'

  # 2. Deploy ArgoCD applications
  deploy-argocd:
    needs: validate-helm
    uses: lolmeida/github-actions/.github/workflows/argocd-deploy.yml@main
    with:
      appset_enabled: 'true'
      appset_list_enabled: 'true'
      force_recreate: 'false'
      wait_timeout: '10m'
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}

  # 3. Sync applications baseado no input
  sync-apps:
    needs: deploy-argocd
    if: github.event_name == 'workflow_dispatch'
    uses: lolmeida/github-actions/.github/workflows/argocd-manage.yml@main
    with:
      action: sync-${{ github.event.inputs.environment }}
      dry_run: false
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```

### 4. Microservice
```code
mkdir -p .github/workflows/
touch .github/workflows/ci.yml
```

2- add the following content to the file `.github/workflows/ci.yml`

```yaml
name: Microservice Pipeline

on:
  push:
    branches: [main, develop, 'feature/*']
  pull_request:
    branches: [main, develop]

env:
  SERVICE_NAME: user-service
  IMAGE_NAME: my-org/user-service

jobs:
  # Stage 1: Qualidade do código
  quality-check:
    uses: lolmeida/github-actions/.github/workflows/spotless-check.yml@main
    with:
      java_version: '21'
      auto_fix: ${{ github.event_name == 'pull_request' }}

  # Stage 2: Testes (custom)
  tests:
    needs: quality-check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Run tests
        run: ./mvnw test

  # Stage 3: Build (apenas para main/develop)
  build-image:
    needs: tests
    if: github.event_name == 'push' && contains(fromJson('["main", "develop"]'), github.ref_name)
    uses: lolmeida/github-actions/.github/workflows/docker-jib-push.yml@main
    with:
      java_version: '21'
      image_name: ${{ env.SERVICE_NAME }}
      maven_profiles: ${{ github.ref_name == 'main' && 'prod' || 'dev' }}
    secrets:
      DOCKER_HUB_USER: ${{ secrets.DOCKER_HUB_USER }}
      DOCKER_HUB_TOKEN: ${{ secrets.DOCKER_HUB_TOKEN }}

  # Stage 4: Deploy to staging (develop)
  deploy-staging:
    needs: build-image
    if: github.ref_name == 'develop'
    uses: lolmeida/github-actions/.github/workflows/argocd-manage.yml@main
    with:
      action: sync-staging
      dry_run: false
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}

  # Stage 5: Deploy to production (main)
  deploy-production:
    needs: build-image
    if: github.ref_name == 'main'
    uses: lolmeida/github-actions/.github/workflows/argocd-manage.yml@main
    with:
      action: sync-prod
      dry_run: false
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```

### 5. Monorepo
```code
mkdir -p .github/workflows/
touch .github/workflows/ci.yml
```

2- add the following content to the file `.github/workflows/ci.yml`

```yaml
name: Monorepo CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # Detectar mudanças
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.changes.outputs.frontend }}
      backend: ${{ steps.changes.outputs.backend }}
      helm: ${{ steps.changes.outputs.helm }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: changes
        with:
          filters: |
            frontend:
              - 'apps/frontend/**'
            backend:
              - 'apps/backend/**'
            helm:
              - 'helm/**'

  # Pipeline frontend
  frontend-build:
    needs: detect-changes
    if: needs.detect-changes.outputs.frontend == 'true'
    uses: lolmeida/github-actions/.github/workflows/docker-build-push.yml@main
    with:
      image_name: monorepo-frontend
      dockerfile_path: ./apps/frontend/Dockerfile
      build_context: ./apps/frontend
    secrets:
      DOCKER_HUB_USERNAME: ${{ secrets.DOCKER_HUB_USERNAME }}
      DOCKER_HUB_ACCESS_TOKEN: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}

  # Pipeline backend
  backend-quality:
    needs: detect-changes
    if: needs.detect-changes.outputs.backend == 'true'
    uses: lolmeida/github-actions/.github/workflows/spotless-check.yml@main
    with:
      working_directory: ./apps/backend
      java_version: '21'

  backend-build:
    needs: [detect-changes, backend-quality]
    if: needs.detect-changes.outputs.backend == 'true' && github.event_name == 'push'
    uses: lolmeida/github-actions/.github/workflows/docker-jib-push.yml@main
    with:
      working_directory: ./apps/backend
      image_name: monorepo-backend
      java_version: '21'
    secrets:
      DOCKER_HUB_USER: ${{ secrets.DOCKER_HUB_USER }}
      DOCKER_HUB_TOKEN: ${{ secrets.DOCKER_HUB_TOKEN }}

  # Pipeline Helm
  helm-validation:
    needs: detect-changes
    if: needs.detect-changes.outputs.helm == 'true'
    uses: lolmeida/github-actions/.github/workflows/helm-ci.yml@main
    with:
      run_helm: 'true'
      envs: 'staging,production'
```
---

### 5. Manage ArgoCD
```code
mkdir -p .github/workflows/
touch .github/workflows/ci.yml
```

2- add the following content to the file `.github/workflows/ci.yml`

```yaml
name: Manage ArgoCD

on:
  workflow_dispatch:
    inputs:
      action:
        description: 'Action to perform'
        type: choice
        options:
          - 'sync-all'
          - 'sync-staging'
          - 'sync-prod'
          - 'remove-staging'
          - 'remove-prod'
        required: true
      dry_run:
        description: 'Dry run mode'
        type: boolean
        default: true

jobs:
  manage-argocd:
    uses: lolmeida/github-actions/.github/workflows/argocd-manage.yml@main
    with:
      action: ${{ github.event.inputs.action }}
      dry_run: ${{ github.event.inputs.dry_run }}
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```
---

### 6. Release Pipeline
```code
mkdir -p .github/workflows/
touch .github/workflows/ci.yml
```

2- add the following content to the file `.github/workflows/ci.yml`

```yaml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  build-and-release:
    uses: lolmeida/github-actions/.github/workflows/docker-build-push.yml@main
    with:
      image_name: ${{ github.event.repository.name }}
      tag_strategy: ${{ github.ref_name }}
      platforms: linux/amd64,linux/arm64
    secrets:
      DOCKER_HUB_USERNAME: ${{ secrets.DOCKER_HUB_USERNAME }}
      DOCKER_HUB_ACCESS_TOKEN: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}

  deploy-production:
    needs: build-and-release
    uses: lolmeida/github-actions/.github/workflows/argocd-manage.yml@main
    with:
      action: sync-prod
      dry_run: false
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```
