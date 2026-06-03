# base-workflow
This is a reusable base workflow that the microservice can call it directly


# reusable-base-workflow Usage Guide

## Overview

A GitHub Actions **reusable workflow** that provides a standardised CI/CD pipeline for containerised microservices. Other repositories (e.g. `golang-app`) call this workflow via `workflow_call` to get a consistent build, test, scan, and deploy process without duplicating pipeline code.

| Item | Value |
|------|-------|
| Type | GitHub Actions Reusable Workflow |
| Supported language | Go (extensible) |
| Container registry | Docker Hub / ECR |
| Deployment target | Kubernetes via Helm |
| Security scanners | Twistlock + Snyk |

---

## Architecture Diagram

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │                        CALLER REPOSITORY                             │
 │  (e.g. golang-app)                                                   │
 │                                                                      │
 │  .github/workflows/build-pkg-deploy.yaml                             │
 │    uses: reusable-base-workflow/…/reusable-microservice-wf.yaml@main │
 │    with:                                                             │
 │      language, go-version, environment, namespace,                   │
 │      run-snyk-scan, run-e2e-test, run-code-coverage …               │
 │    secrets:                                                          │
 │      dockerhub-password, twistlock-secret-key, snyk-token …          │
 └───────────────────────────────┬──────────────────────────────────────┘
                                 │  workflow_call
                                 ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │            reusable-microservice-wf.yaml  (this repo)                │
 │                                                                      │
 │  ┌────────────────────────────────────────────────────────────────┐  │
 │  │ 1. PREPARE                                                     │  │
 │  │    - Fetch OIDC token                                          │  │
 │  │    - Checkout base-workflow repo + caller repo                 │  │
 │  │    - Read Chart.yaml → repo-name, chart-version, app-version   │  │
 │  │    - Compute image-name, image-tag, chart URL                  │  │
 │  │    OUTPUTS ──> metadata for all downstream jobs                │  │
 │  └──────────┬──────────────────┬──────────────────┬───────────────┘  │
 │             │                  │                  │                  │
 │             ▼                  ▼                  ▼                  │
 │  ┌──────────────┐  ┌──────────────────┐  ┌────────────────────┐     │
 │  │ 2. LINT      │  │ 3. E2E-TEST      │  │ 4. CODE-COVERAGE   │     │
 │  │  static      │  │  (conditional)   │  │  (conditional)     │     │
 │  │  analysis    │  │  run-e2e-test    │  │  run-code-coverage │     │
 │  └──────┬───────┘  └───────┬──────────┘  └─────────┬──────────┘     │
 │         └──────────────────┼────────────────────────┘               │
 │                            ▼                                        │
 │  ┌────────────────────────────────────────────────────────────────┐  │
 │  │ 5. BUILD                                                       │  │
 │  │    - Setup Go (specified version)                              │  │
 │  │    - docker build -t {image-name}:{image-tag}                  │  │
 │  │    - docker push → container registry                          │  │
 │  └──────────┬─────────────────────────────┬───────────────────────┘  │
 │             │                             │                         │
 │             ▼                             ▼                         │
 │  ┌──────────────────┐          ┌──────────────────┐                 │
 │  │ 6. TWISTLOCK     │          │ 7. SNYK SCAN     │                 │
 │  │    SCAN          │          │    (optional)    │                 │
 │  │  container       │          │  dependency      │                 │
 │  │  security        │          │  vulnerabilities │                 │
 │  └────────┬─────────┘          └────────┬─────────┘                 │
 │           └──────────────┬──────────────┘                           │
 │                          ▼                                          │
 │  ┌────────────────────────────────────────────────────────────────┐  │
 │  │ 8. BUILD-HELM-CHART                                            │  │
 │  │    - helm package (set chart-version + app-version)            │  │
 │  │    - helm push → OCI registry                                  │  │
 │  └──────────────────────────┬─────────────────────────────────────┘  │
 │                             │                                       │
 │                             ▼                                       │
 │  ┌────────────────────────────────────────────────────────────────┐  │
 │  │ 9. DEPLOY                                                      │  │
 │  │    - helm upgrade --install                                    │  │
 │  │    - target namespace + environment                            │  │
 │  │    - timeout 3600s                                             │  │
 │  └────────────────────────────────────────────────────────────────┘  │
 └──────────────────────────────────────────────────────────────────────┘
                                 │
                ┌────────────────┼────────────────┐
                ▼                ▼                ▼
        ┌─────────────┐  ┌────────────┐  ┌──────────────┐
        │ Docker Hub / │  │ Twistlock  │  │ Kubernetes   │
        │ ECR Registry │  │ + Snyk     │  │ Cluster (EKS)│
        └─────────────┘  └────────────┘  └──────────────┘
```

---

## Directory Structure

```
reusable-base-workflow/
├── .github/
│   └── workflows/
│       └── reusable-microservice-wf.yaml   # The reusable workflow (293 lines)
└── README.md
```

Intentionally minimal — the sole purpose of this repo is to host the shared workflow template.

---

## Workflow Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `environment` | string | yes | `dev` | Target environment (dev / staging / prod) |
| `namespace` | string | yes | `developer` | Kubernetes namespace |
| `ecr-registry` | string | no | `kubeclever` | Container registry URL |
| `subdir` | string | no | `.` | Subdirectory containing the service (monorepo support) |
| `language` | string | yes | `golang` | Programming language |
| `go-version` | string | no | `1.21` | Go version |
| `run-snyk-scan` | bool | no | `false` | Enable Snyk scanning |
| `dockerhub-username` | string | yes | — | Docker Hub username |
| `run-e2e-test` | bool | no | `false` | Enable E2E tests |
| `run-code-coverage` | bool | no | `false` | Enable code coverage |

## Required Secrets

| Secret | Required | Purpose |
|--------|----------|---------|
| `dockerhub-password` | yes | Docker Hub authentication |
| `twistlock-secret-key` | yes | Container security scanning |
| `snyk-token` | no | Vulnerability scanning |
| `base-workflows-gh-token` | no | Access to this workflow repo |

---

## Job Dependency Graph

```
prepare ────┬──── lint ────────────────┐
            ├──── e2e-test (optional) ─┤
            └──── coverage (optional) ─┘
                         │
                       build
                      /     \
           twistlock-scan   snyk-scan
                      \     /
                  build-helm-chart
                         │
                       deploy
```

## Prepare Job Outputs

| Output | Example | Description |
|--------|---------|-------------|
| `repo-name` | `golang-app` | Repository name |
| `chart-name` | `golang-app-chart` | Helm chart name |
| `chart-version` | `0.1.0` | From `Chart.yaml` |
| `app-version` | `1.0.0` | From `Chart.yaml` |
| `image-name` | `kubeclever/golang-app` | Full image name |
| `image-tag` | `1.0.0` | Image tag |
| `image` | `kubeclever/golang-app:1.0.0` | Full image reference |

---

## Caller Repository Requirements

The repository calling this workflow must provide:

```
<repo>/
├── {subdir}/
│   ├── Dockerfile
│   └── helm/
│       └── Chart.yaml          # must have `version` and `appVersion`
└── .github/workflows/*.yaml    # workflow_call to this repo
```

### Example Caller Workflow

```yaml
jobs:
  build-and-deploy:
    uses: zhangqiongjie/reusable-base-workflow/.github/workflows/reusable-microservice-wf.yaml@main
    with:
      environment: dev
      namespace: wanhao
      language: golang
      go-version: '1.26.2'
      dockerhub-username: ${{ vars.DOCKERHUB_USERNAME }}
      run-snyk-scan: true
      run-e2e-test: true
      run-code-coverage: true
    secrets:
      dockerhub-password: ${{ secrets.DOCKERHUB_PASSWORD }}
      twistlock-secret-key: ${{ secrets.TWISTLOCK_SECRET_KEY }}
      snyk-token: ${{ secrets.SNYK_TOKEN }}
```

---

## Relationship to Other Repos

| Repo | Role |
|------|------|
| `golang-app` | Caller — invokes this workflow for its CI/CD |
| `terraform-infrastructure` | Provides the EKS cluster that the deploy job targets |
