# Platform GitOps

This repository is the GitOps source that an already-running Argo CD instance reads. It does not create a kind cluster, install Argo CD, port-forward Argo CD, build images, or load local images. Those responsibilities belong to `../cluster-bootstrap`.

Demo app workloads use public OSS images directly, so this bundle does not include separate app source repositories.

## Repository structure

```text
platform-gitops/
├── bootstrap/
│   ├── root-app.yaml
│   └── argocd-projects.yaml
├── applicationsets/
│   ├── platform-addons.yaml
│   ├── app-data.yaml
│   ├── apps.yaml
│   └── observability-configs.yaml
├── clusters/local/
│   └── platform/
│       ├── ingress-nginx/values.yaml
│       ├── cnpg-operator/values.yaml
│       └── kube-prometheus-stack/values.yaml
└── charts/
    ├── demo-app/
    ├── app-data/
    └── observability-config/
```

## What Argo CD reads

- `bootstrap/root-app.yaml`: app-of-apps root Application that points Argo CD at `applicationsets/`.
- `bootstrap/argocd-projects.yaml`: `platform` and `apps` AppProjects plus the namespaces used by this example.
- `applicationsets/platform-addons.yaml`: creates platform addon Applications for ingress-nginx, CloudNativePG operator, and kube-prometheus-stack.
- `applicationsets/app-data.yaml`: creates `app-a-data`, `app-b-data`, and `app-c-data` Applications from `charts/app-data`.
- `applicationsets/apps.yaml`: creates `app-a`, `app-b`, and `app-c` workload Applications from `charts/demo-app` using public OSS images.
- `applicationsets/observability-configs.yaml`: creates app observability Applications from `charts/observability-config`.

## Repo URL placeholders

The manifests intentionally use only `GITOPS_REPO_URL`. Before an Argo CD instance reads this repository, replace that placeholder with a real Git remote URL in the manifests you apply, such as `bootstrap/root-app.yaml` and the ApplicationSet files. Argo CD does not run `envsubst` on Git contents.

## Demo app images

The app workload ApplicationSet deploys one reusable chart with public OSS images:

- `app-a`: `prom/prometheus:v2.55.1`
- `app-b`: `prom/alertmanager:v0.27.0`
- `app-c`: `prom/pushgateway:v1.9.0`

The per-app differences are inline in `applicationsets/apps.yaml`, `applicationsets/app-data.yaml`, and `applicationsets/observability-configs.yaml`, so there are no separate app source directories or per-app values directories to maintain for the demo.

## Why app workload and app-data are separated

The app workload Applications use `charts/demo-app` and deploy only Deployment, Service, Ingress, and optional ServiceMonitor resources. Database and Redis resources are deployed separately by the `app-data` Applications from `charts/app-data`. This prevents an app workload release and a data release from owning the same Kubernetes resources.

## Local validation

These commands were run locally while checking this README. They validate chart structure and render the documented `app-a` examples without needing a Kubernetes cluster.

```sh
helm lint charts/demo-app
helm template app-a charts/demo-app \
  --namespace app-a \
  --set fullnameOverride=app-a \
  --set nameOverride=app-a \
  --set image.repository=prom/prometheus \
  --set image.tag=v2.55.1 \
  --set containerPort=9090 \
  --set service.port=9090 \
  --set ingress.host=app-a.localtest.me \
  --set probes.path=/-/healthy

helm lint charts/app-data
helm template app-a-data charts/app-data \
  --namespace app-a \
  --set appName=app-a \
  --set database.clusterName=app-a-db \
  --set database.owner=app_a_user \
  --set database.secretName=app-a-db-credentials \
  --set database.localOnlyDummyPassword=local-only-dummy-password-change-me-app-a \
  --set redis.name=app-a-redis

helm lint charts/observability-config
helm template app-a-observability charts/observability-config \
  --namespace app-a \
  --set appName=app-a \
  --set service.name=app-a \
  --set service.port=9090
```

Repeat the same pattern for `app-b` and `app-c` with the values in the ApplicationSets.

## Platform addons

The platform addon ApplicationSet combines external Helm charts with values from this repo. Chart versions are pinned in `applicationsets/platform-addons.yaml`:

- ingress-nginx: `4.15.1`
- cloudnative-pg: `0.28.2`
- kube-prometheus-stack: `86.1.0`

These Applications use sync wave `-5`, before app data, workloads, and observability configs that depend on the installed controllers and CRDs.

## Observability config

`charts/observability-config` creates example `ScrapeConfig`, `PrometheusRule`, and Grafana dashboard ConfigMap resources. These are deployed at sync wave `10`, after platform monitoring and app workloads. The selected public OSS images expose `/metrics`, so the demo scrape examples can work without building custom apps.

## Sync waves

- Namespace/AppProject resources: `-10`
- Platform addons: `-5`
- App data resources: `0`
- App workloads: `5`
- Observability configs: `10`

## Production changes before reuse

Before turning this into a real environment, remove local dummy secrets, introduce a secret management tool such as external-secrets or sealed-secrets, mirror charts/images to an internal registry, strengthen pinning, add backup policy, set resource requests/limits, restrict AppProject permissions, finalize ingress/TLS, and pin application images by digest.
