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
```bash
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
```bash
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

Arquivo: `applications-type/helm-project/.github/workflows/infrastructure.yml`

```yaml
name: Infrastructure CI/CD
on:
  push:
    branches: [main]
    paths:
      - 'helm/**'
      - 'argocd-apps/**'
      - 'overlays/**'
  workflow_dispatch:
    inputs:
      deploy_environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - 'staging'
          - 'production'
          - 'both'
        default: 'staging'
      force_recreate:
        description: 'Force recreate applications'
        required: false
        type: boolean
        default: false
jobs:
  helm-validation:
    uses: lolmeida/reusable-github-actions/.github/workflows/helm-ci.yml@main
    with:
      run_argocd_apps: 'true'
      run_helm: 'true'
      envs: 'staging,production'
      debug_mode: false
  deploy-argocd:
    needs: helm-validation
    uses: lolmeida/reusable-github-actions/.github/workflows/argocd-deploy.yml@main
    with:
      dry_run: false
      appset_enabled: 'true'
      appset_list_enabled: 'true'
      appset_git_enabled: 'false'
      force_recreate: ${{ github.event.inputs.force_recreate || 'false' }}
      validation_environment: 'production'
      wait_timeout: '15m'
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
  sync-staging:
    needs: deploy-argocd
    if: contains(fromJson('["staging", "both"]'), github.event.inputs.deploy_environment) || github.event_name == 'push'
    uses: lolmeida/reusable-github-actions/.github/workflows/argocd-manage.yml@main
    with:
      action: sync-staging
      dry_run: false
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
  sync-production:
    needs: [deploy-argocd, sync-staging]
    if: contains(fromJson('["production", "both"]'), github.event.inputs.deploy_environment)
    uses: lolmeida/reusable-github-actions/.github/workflows/argocd-manage.yml@main
    with:
      action: sync-prod
      dry_run: false
      force_delete: true
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```

### 4. Microservice

Arquivo: `applications-type/microservice/.github/workflows/ci-cd.yml`

```yaml
name: Microservice CI/CD Pipeline
on:
  push:
    branches: [main, 'feature/*']
  pull_request:
    branches: [main]
env:
  SERVICE_NAME: user-management-service
  IS_MAIN_BRANCH: ${{ github.ref_name == 'main' }}
  IS_DEVELOP_BRANCH: ${{ github.ref_name == 'develop' }}
jobs:
  spotless-check:
    if: contains(fromJson('["java", "kotlin"]'), github.event.repository.language)
    uses: lolmeida/reusable-github-actions/.github/workflows/spotless-check.yml@main
    with:
      java_version: '21'
      auto_fix: ${{ github.event_name == 'pull_request' }}
  build:
    needs: spotless-check
    if: always() && !failure()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: |
          echo "Running tests for ${{ env.SERVICE_NAME }}"
          # Your test commands here
  docker-build:
    needs: build
    if: github.event_name == 'push' && (env.IS_MAIN_BRANCH == 'true' || env.IS_DEVELOP_BRANCH == 'true')
    uses: lolmeida/reusable-github-actions/.github/workflows/docker-jib-push.yml@main
    with:
      java_version: '21'
      image_name: ${{ env.SERVICE_NAME }}
      maven_profiles: ${{ env.IS_MAIN_BRANCH == 'true' && 'prod' || 'staging' }}
    secrets:
      DOCKER_HUB_USER: ${{ secrets.DOCKER_HUB_USER }}
      DOCKER_HUB_TOKEN: ${{ secrets.DOCKER_HUB_TOKEN }}
  deploy-staging:
    needs: docker-build
    if: env.IS_DEVELOP_BRANCH == 'true'
    uses: lolmeida/reusable-github-actions/.github/workflows/argocd-manage.yml@main
    with:
      action: sync-staging
      dry_run: false
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
  deploy-production:
    needs: docker-build
    if: env.IS_MAIN_BRANCH == 'true'
    uses: lolmeida/reusable-github-actions/.github/workflows/argocd-manage.yml@main
    with:
      action: sync-prod
      dry_run: false
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```

### 5. Monorepo

Arquivo: `applications-type/monorepo/.github/workflows/monorepo.yml`

```yaml
name: Monorepo CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
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
              - 'argocd-apps/**'
  frontend-pipeline:
    needs: detect-changes
    if: needs.detect-changes.outputs.frontend == 'true'
    uses: lolmeida/reusable-github-actions/.github/workflows/docker-build-push.yml@main
    with:
      image_name: my-app-frontend
      dockerfile_path: ./apps/frontend/Dockerfile
      build_context: ./apps/frontend
      tag_strategy: ${{ github.ref_name }}
    secrets:
      DOCKER_HUB_USER: ${{ secrets.DOCKER_HUB_USER }}
      DOCKER_HUB_TOKEN: ${{ secrets.DOCKER_HUB_TOKEN }}
  backend-quality:
    needs: detect-changes
    if: needs.detect-changes.outputs.backend == 'true'
    uses: lolmeida/reusable-github-actions/.github/workflows/spotless-check.yml@main
    with:
      working_directory: ./apps/backend
      java_version: '21'
  backend-build:
    needs: [detect-changes, backend-quality]
    if: needs.detect-changes.outputs.backend == 'true' && github.event_name == 'push'
    uses: lolmeida/reusable-github-actions/.github/workflows/docker-jib-push.yml@main
    with:
      working_directory: ./apps/backend
      image_name: my-app-backend
      java_version: '21'
    secrets:
      DOCKER_HUB_USER: ${{ secrets.DOCKER_HUB_USER }}
      DOCKER_HUB_TOKEN: ${{ secrets.DOCKER_HUB_TOKEN }}
  helm-pipeline:
    needs: detect-changes
    if: needs.detect-changes.outputs.helm == 'true'
    uses: lolmeida/reusable-github-actions/.github/workflows/helm-ci.yml@main
    with:
      run_argocd_apps: 'true'
      run_helm: 'true'
      envs: 'staging,production'
  deploy:
    needs: [frontend-pipeline, backend-build, helm-pipeline]
    if: always() && !failure() && github.ref_name == 'main'
    uses: lolmeida/reusable-github-actions/.github/workflows/argocd-deploy.yml@main
    with:
      appset_enabled: 'true'
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```