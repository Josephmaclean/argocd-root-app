# Kubernetes Bootstrap Root App

This repository is structured as an Argo CD app-of-apps bootstrap for a Kubernetes cluster.

Apply `root-app.yaml` to an existing Argo CD installation:

```sh
kubectl apply -f root-app.yaml
```

The root application syncs every child `Application` under `system-apps/`. Child apps either install Helm charts directly or deploy local manifests from `resources/`.

## Layout

- `root-app.yaml` - root Argo CD application for cluster bootstrap.
- `system-apps/` - child Argo CD applications managed by the root app: cluster lifecycle components, Kubernetes addons, and platform management services.
- `system-apps/app-catalog/` - bootstrap application for team-owned application definitions in the external app catalog.
- `resources/` - plain Kubernetes manifests consumed by child apps.
- `argo-values/` - Helm values referenced by chart-based child apps.

## Sync Order

Child applications use Argo CD sync waves for basic dependency ordering:

- wave `0`: Argo CD exposure, AWS Load Balancer Controller, cert-manager, EBS gp3 StorageClass, External Secrets, Istio base, Jaeger, Kyverno, Metrics Server, NVIDIA Device Plugin, Prometheus stack
- wave `5`: KServe CRDs, PostgreSQL in `postgres`
- wave `10`: Istiod, dashboard, KServe, MLflow, vLLM
- wave `15`: Bifrost, KServe LLMInferenceService controller, LiteLLM Operator
- wave `20`: Kubernetes Gateway API, Istio gateway
- wave `30`: Kiali

## KServe

`system-apps/kserve/` installs the official OCI Helm charts `kserve-crd`,
`kserve-llmisvc-crd`, `kserve-resources`, and `kserve-llmisvc-resources`, all pinned
to `v0.20.0`, into namespace `kserve`.
The root app discovers these applications automatically. cert-manager `v1.21.2`
is included to provision KServe webhook certificates.

The `kserve-crds` Application includes both `LLMInferenceService` and
`LLMInferenceServiceConfig` definitions through `kserve-llmisvc-crd`.
Both CRD charts render the same `ClusterStorageContainer` definition; keeping
them in one multi-source Application avoids competing owners. Argo CD reports
`RepeatedResourceWarning` for that shared definition and uses the last source.
The `kserve-llmisvc` Application installs the LLM controller, webhooks, its
`llmisvc-serving-cert` Certificate, and bundled inference-extension CRDs.
`argo-values/kserve-llmisvc-values.yaml` sets `createSharedResources: false` so
the existing `kserve` Application remains the sole owner of the shared
`inferenceservice-config`, `selfsigned-issuer`, and default storage container.
cert-manager must be healthy to issue the LLM webhook Secret
`llmisvc-webhook-server-cert`.

Gateway-based routing still requires Gateway API and a compatible configured
gateway; multi-node workloads require the LeaderWorkerSet operator. Neither
is enabled by this controller addition. Runtime templates from
`kserve-runtime-configs` are also separate from the controller. See the
[LLMInferenceService installation guide](https://kserve.github.io/website/docs/install/llmisvc-install).

KServe uses `Standard` deployment mode (no Knative dependency). Model services
remain internal by default: ingress creation is disabled in
`argo-values/kserve-values.yaml` because the Gateway and Istio applications in
this repository are currently commented out. Configure ingress before exposing
model endpoints. This addition installs the control plane; model deployments
and serving runtimes are separate configuration.

The [KServe installation guide](https://kserve.github.io/website/docs/admin-guide/kubernetes-deployment)
requires Kubernetes 1.32 or later. Confirm cluster compatibility before syncing.
Server-side apply handles the large CRDs. Sync waves order child Application
creation, but do not by themselves guarantee dependency readiness; the KServe
application retries while cert-manager and the CRDs become available.

After syncing, verify the controllers:

```sh
kubectl get pods -n cert-manager
kubectl get pods -n kserve
kubectl get crd inferenceservices.serving.kserve.io
kubectl get crd llminferenceservices.serving.kserve.io llminferenceserviceconfigs.serving.kserve.io
kubectl rollout status deployment/llmisvc-controller-manager -n kserve
```
