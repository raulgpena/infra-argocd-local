# infra-argocd-local

GitOps source of truth for the local k3d cluster. Argo CD watches this repo and
auto-deploys every microservice it finds.

## How it works

An Argo CD `ApplicationSet` (defined in the `infra-k8s-local` Terraform) scans:

```
apps/<project>/<service>/
```

and turns each `<service>` folder into an Argo CD `Application`:

| Folder | Application | Project | Namespace |
|--------|-------------|---------|-----------|
| `apps/snow-white/api`       | `snow-white-api`       | `snow-white` | `snow-white` |
| `apps/snow-white/consumer`  | `snow-white-consumer`  | `snow-white` | `snow-white` |
| `apps/snow-white/migration` | `snow-white-migration` | `snow-white` | `snow-white` |
| `apps/cars/api`             | `cars-api`             | `cars`       | `cars`       |
| `apps/cars/consumer`        | `cars-consumer`        | `cars`       | `cars`       |
| `apps/cars/migration`       | `cars-migration`       | `cars`       | `cars`       |

## How apps are rendered

Apps are **not** raw manifests. Each service folder holds a single
`values.yaml` consumed by the reusable **`base-app`** Helm chart (published to
GitHub Packages). Argo CD builds each app as a **multi-source** Application:

1. the `base-app` chart (from `ghcr.io/<owner>/charts`)
2. this repo, supplying `apps/<project>/<service>/values.yaml`

The chart, its version, and the registry are configured in the `infra-k8s-local`
Terraform (`base_app_chart_*` variables).

## Conventions

- **Project = middle folder** (`snow-white`, `cars`) — also the namespace.
- **Service = leaf folder** (`api`, `consumer`, `migration`).
- Each leaf folder contains exactly one `values.yaml` for the `base-app` chart.
- `base-app` omits `metadata.namespace`; Argo CD injects the destination
  namespace (and creates it via `CreateNamespace=true`).
- Service roles (driven by toggles in `values.yaml`):
  - `api`       — `Deployment` + `Service` + `Ingress`, HTTP-facing.
  - `consumer`  — `Deployment` only, background worker (`service.enabled: false`).
  - `migration` — `Job` (`job.enabled: true`, `job.argocdHook: true`), runs as an
    Argo CD **Sync hook**.

## Add a new service

Create `apps/<project>/<service>/values.yaml`, commit, push. Argo CD picks it up
automatically — no Terraform change needed.

## Add a new project

Create `apps/<new-project>/...` **and** add the project to `argocd_projects`
in the `infra-k8s-local` Terraform (AppProjects are governed centrally).
