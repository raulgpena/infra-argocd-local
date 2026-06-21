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

## Conventions

- **Project = middle folder** (`snow-white`, `cars`) — also the namespace.
- **Service = leaf folder** (`api`, `consumer`, `migration`).
- Manifests **do not** set `metadata.namespace`; Argo CD injects the destination
  namespace (and creates it via `CreateNamespace=true`).
- Service roles:
  - `api`       — `Deployment` + `Service` (+ `Ingress`), HTTP-facing.
  - `consumer`  — `Deployment` only, background worker (no `Service`).
  - `migration` — `Job` run as an Argo CD **Sync hook** before the rollout.

## Add a new service

Create `apps/<project>/<service>/` with its manifests, commit, push. Argo CD
picks it up automatically — no Terraform change needed.

## Add a new project

Create `apps/<new-project>/...` **and** add the project to `argocd_projects`
in the `infra-k8s-local` Terraform (AppProjects are governed centrally).
