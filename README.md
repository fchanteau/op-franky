# franky — GitOps platform

GitOps repository for managing Kubernetes workloads with Helm and ArgoCD.

- A **versioned common chart** (`dotnet-api`) published to an OCI registry.
- One **umbrella chart per application/environment** that pins the common chart version.
- One **ArgoCD ApplicationSet per application** (git directory generator).
- An **App-of-Apps** root that manages the project and every ApplicationSet.
- One **`appsettings.json` per environment**, mounted read-only into the pod.

## Repository layout

```
charts/dotnet-api/                 # versioned common chart (source of the OCI package)
apps/
  project.yaml                     # AppProject "franky" (allowed sources + destinations)
  root-app.yaml                    # App-of-Apps root (syncs apps/)
  api-orders-appset.yaml           # ApplicationSet: one Application per env
  api-customers-appset.yaml
api-orders/{devci,uatrc,prod}/     # Chart.yaml (pins dotnet-api version) · values.yaml · appsettings.json
api-customers/{devci,uatrc,prod}/  # same
.github/workflows/                 # publishes the common chart to OCI on tag
```

## Environment topology

| Environment | Cluster | Namespace |
| ----------- | ------- | --------- |
| devci       | nonprod | devci     |
| uatrc       | nonprod | uatrc     |
| prod        | prod    | prod      |

ApplicationSets route `devci`/`uatrc` to the nonprod cluster and `prod` to the prod
cluster via the `destination.server` expression in each ApplicationSet.

## How `appsettings.json` reaches the pod

Helm subcharts cannot read parent-chart files, so the per-environment
`appsettings.json` is injected into the common chart through ArgoCD
`helm.fileParameters` (equivalent to `--set-file dotnet-api.appsettings=appsettings.json`).
The common chart renders it into a ConfigMap mounted read-only at
`/app/config/appsettings.json`. A `checksum/appsettings` pod annotation rolls the
Deployment whenever the content changes.

## Evolving the common chart safely

Each `<app>/<env>/Chart.yaml` pins its own `dotnet-api` dependency version. Bump a
single environment (e.g. `devci`) to validate a new common-chart release before
promoting it to `uatrc` and then `prod` — no other app or environment is affected.

## Placeholders to replace before use

- `registry.example.com/franky/helm` — OCI registry for the common chart.
- `registry.example.com/franky/*` — container image repositories.
- `https://github.com/your-org/franky.git` — this repo's URL (in `apps/*.yaml`).
- `https://prod.k8s.example.com:6443` — prod cluster API server.
- Ingress hosts and the `__REPLACE__` database passwords (move secrets to Sealed
  Secrets / External Secrets before production).

## Bootstrap (run once)

1. **Publish the common chart** to the OCI registry:

   ```sh
   # Via CI: push a tag matching Chart.yaml version.
   git tag chart-dotnet-api-v1.0.0 && git push origin chart-dotnet-api-v1.0.0

   # Or manually:
   helm registry login registry.example.com
   helm package charts/dotnet-api --destination dist
   helm push dist/dotnet-api-1.0.0.tgz oci://registry.example.com/franky/helm
   ```

2. **Register the prod cluster** in ArgoCD (the nonprod cluster is the in-cluster
   `https://kubernetes.default.svc`):

   ```sh
   argocd cluster add <prod-context> --name prod
   ```

3. **Allow the OCI registry** as a Helm repository so ArgoCD can resolve the
   dependency during manifest generation:

   ```sh
   argocd repo add registry.example.com/franky/helm \
     --type helm --enable-oci \
     --username <user> --password <token>
   ```

4. **Apply the project, then the App-of-Apps root:**

   ```sh
   kubectl apply -f apps/project.yaml
   kubectl apply -f apps/root-app.yaml
   ```

   The root app syncs `apps/`, creating both ApplicationSets, which each generate one
   Application per environment.

## Local validation

```sh
helm lint charts/dotnet-api
helm template api-orders-devci charts/dotnet-api \
  --set-file appsettings=api-orders/devci/appsettings.json \
  --set ingress.enabled=true
```

The render should show a ConfigMap containing `appsettings.json` and a Deployment
mounting it at `/app/config`.
