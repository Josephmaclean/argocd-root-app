# Kubernetes Bootstrap Root App

This repository is structured as an Argo CD app-of-apps bootstrap for a Kubernetes cluster.

Apply `root-app.yaml` to an existing Argo CD installation:

```sh
kubectl apply -f root-app.yaml
```

The root application syncs every child `Application` under `system-apps/`. Child apps either install Helm charts directly or deploy local manifests from `resources/`.

## Layout

- `root-app.yaml` - root Argo CD application for cluster bootstrap.
- `system-apps/` - child Argo CD applications managed by the root app.
- `resources/` - plain Kubernetes manifests consumed by child apps.
- `argo-values/` - Helm values referenced by chart-based child apps.

## Sync Order

Child applications use Argo CD sync waves for basic dependency ordering:

- wave `0`: Argo CD exposure, EBS gp3 StorageClass, Istio base, Jaeger, Kyverno, Metrics Server, NVIDIA Device Plugin, Prometheus stack
- wave `5`: PostgreSQL in `postgres`
- wave `10`: Istiod, dashboard, MLflow, vLLM
- wave `20`: Istio gateway
- wave `30`: Kiali
