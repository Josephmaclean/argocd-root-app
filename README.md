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
- wave `10`: Istiod, dashboard, KServe, MLflow, vLLM, Kubeflow platform (Central Dashboard, Pipelines, Trainer, Model Registry)
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

## Kubeflow

`system-apps/kubeflow-platform/` installs the **Kubeflow Community
Distribution**, pinned to tag `26.03.1`, as a single Argo CD Application. This
is the full platform (not standalone components) so that the unified **Kubeflow
Central Dashboard** UI is available.

The Application's source is the in-repo kustomization
`resources/kubeflow-platform/`, which references the upstream
`kubeflow/manifests` paths as remote kustomize bases pinned to `?ref=26.03.1`.
Argo CD renders the kustomization and fetches the remote bases on sync. Pinning
every component to one distribution release keeps versions coherent.

### What is installed

Platform (required by the dashboard):

- **Istio** service mesh + ingress gateway (wired for OAuth2-Proxy auth)
- **Dex** OIDC identity provider and **OAuth2-Proxy** (`m2m-dex-only`)
- Kubeflow namespace, aggregated roles, shared Istio resources, and a default
  user profile namespace (`kubeflow-user-example-com`)

Applications:

- **Central Dashboard** — the unified Kubeflow UI
- **Pipelines** — includes the Pipelines web UI (`ml-pipeline-ui`); uses
  SeaweedFS for S3-compatible artifact storage
- **Trainer** (Training Operator v2)
- **Model Registry** (per-tenant) and **Model Catalog** (cluster singleton),
  with the Model Registry UI surfaced inside the Central Dashboard

### Reused and excluded components

- **cert-manager is reused** from this repo's existing `cert-manager`
  Application. The overlay includes only
  `common/cert-manager/overlays/kubeflow` (the `kubeflow-self-signing-issuer`
  ClusterIssuer plus two NetworkPolicies); the cert-manager controller itself
  is **not** re-installed.
- **KServe is reused** from `system-apps/kserve/`; the distribution's own
  KServe and KServe-UI overlays are excluded.
- **Knative, Katib, Notebooks, Spark, and Workspaces are excluded** — not
  requested. KServe here runs in `Standard` (non-Knative) mode, so Knative is
  not required.

The repo's older standalone Istio Helm Applications
(`system-apps/istio/istio-base.yaml`, `istio-d.yaml`) remain commented out; the
distribution installs its own kustomize-based Istio instead.

### Sync behaviour

The Application uses `ServerSideApply=true` (several Kubeflow CRDs exceed the
client-side apply annotation limit), `SkipDryRunOnMissingResource=true` (custom
resources render before their CRDs are established on the first sync), and a
generous retry (`limit: 20`) because the platform's CRDs and webhooks become
ready asynchronously. Expect the first sync to take several minutes and to
retry a few times before converging.

### Access and credentials

The dashboard sits behind Dex with the default demo credentials
(`user@example.com` / `12341234`). **Change these before exposing the dashboard,
and do not expose it publicly with the defaults.** For a practice cluster,
reach the dashboard by port-forwarding the Istio ingress gateway:

```sh
kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80
# then open http://localhost:8080 and log in via Dex
```

Verify the platform after syncing:

```sh
kubectl get pods -n kubeflow
kubectl get pods -n istio-system
kubectl get pods -n auth            # Dex
kubectl get pods -n oauth2-proxy
kubectl get pods -n kubeflow-system # Trainer / JobSet controllers
kubectl rollout status deployment/centraldashboard -n kubeflow
kubectl rollout status deployment/ml-pipeline-ui -n kubeflow
```
