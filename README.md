# k8s Apps

GitOps source of truth for application manifests, shared across [`aws-eks-cluster`](https://github.com/jalcalaroot/aws-eks-cluster) (EKS) and [`azure-aks-cluster`](https://github.com/jalcalaroot/azure-aks-cluster) (AKS) — separate from the infrastructure repos on purpose. Each cluster runs its own Argo CD (installed in the cluster, not managed here), which watches this repo and keeps that cluster in sync automatically.

## Apps in this repo

| App | What it is | Autoscaling | Why it's here |
|---|---|---|---|
| [`podinfo`](apps/podinfo/) | [`stefanprodan/podinfo`](https://github.com/stefanprodan/podinfo) | KEDA, cpu trigger, 2-5 replicas | The de facto reference app for testing GitOps pipelines — built by Flux's own maintainer. |
| [`game-2048`](apps/game-2048/) | 2048 game ([AWS EKS Workshop's](https://github.com/aws-samples/eks-workshop-samples) reference image) | KEDA, cpu trigger, 2-5 replicas | A real (playable), non-trivial legacy image — nginx on Alpine, ~4 years old, no non-root user, no health endpoints of its own. Exercises real container-hardening tradeoffs, not a toy. |
| [`uptime-kuma`](apps/uptime-kuma/) | [`louislam/uptime-kuma`](https://github.com/louislam/uptime-kuma) | **None, on purpose** — fixed `replicas: 1` | Monitors the other apps for real. Single SQLite instance — see below for why it does *not* get KEDA. |
| [`headlamp`](apps/headlamp/) | [`kubernetes-sigs/headlamp`](https://github.com/kubernetes-sigs/headlamp) (Helm chart, official) | KEDA, cpu trigger, 1-3 replicas | Cluster web UI. Replaces the Kubernetes Dashboard, which is **archived upstream** ("This project is now archived and no longer maintained" — its own README recommends Headlamp) — found while going to install it, not assumed from an old plan. |

## Structure

```
apps/<name>/
  base/                    # plain Deployment + Service + ScaledObject (if any)
    kustomization.yaml
    deployment.yaml
    service.yaml
    scaledobject.yaml
  overlays/
    eks/                    # EKS Fargate - no extra scheduling config needed
      kustomization.yaml
    aks/                    # AKS - patches to force scheduling onto Virtual Nodes (ACI)
      kustomization.yaml
      patch-virtual-node.yaml
apps/headlamp/extra/         # NOT Kustomize - a plain manifest set (viewer ServiceAccount, ScaledObject)
                              # applied as a second Argo `source` alongside the Helm chart
bootstrap/
  applicationset-eks.yaml    # apply once on the EKS cluster's Argo CD
  applicationset-aks.yaml    # apply once on the AKS cluster's Argo CD
  headlamp-application.yaml  # apply once on EACH cluster's Argo CD (multi-source Application, not part of the ApplicationSet)
```

Adding a new Kustomize-based app = a new `apps/<name>/` with `base/` + `overlays/eks/` + `overlays/aks/`. Each cluster's `ApplicationSet` auto-discovers its own overlay directory — no need to touch `bootstrap/` for each new app.

### Why `overlays/eks` and `overlays/aks`, not one shared `overlays/dev`

EKS Fargate and AKS Virtual Nodes claim pods differently by default, and this difference isn't cosmetic:

- **EKS Fargate** claims a pod automatically based on its namespace, via a **Fargate Profile** defined in `aws-eks-cluster/eks.tf` (not in this repo). No annotation or `nodeSelector` needed here.
- **AKS** schedules onto the *real* node pool by default. Going to **Virtual Nodes (ACI)** instead needs an explicit `nodeSelector`/`tolerations` patch (`overlays/aks/patch-virtual-node.yaml`) — found the hard way when all 3 demo apps landed on the real node pool and starved it of CPU that Argo CD/KEDA's own control-plane pods needed there.

One shared overlay can't express both defaults at once, so each app gets a real per-cloud split instead of a workaround.

## Why Kustomize, not Helm (for the 3 demo apps)

No templating language to learn, native `kubectl kustomize` support, and the `base` + `overlays` pattern maps directly onto "one app, multiple clouds" without inventing a values schema per app. Headlamp is the exception — it ships an official Helm chart, so it's consumed as Helm via Argo's native multi-source support instead of being re-wrapped in Kustomize for no reason.

## Prerequisites

- Argo CD installed in the target cluster (see `aws-eks-cluster` or `azure-aks-cluster`).
- On EKS: a matching **Fargate Profile** in `aws-eks-cluster` for the target namespace (`default`, currently) — without it, pods stay `Pending` forever with no obvious error.
- On AKS: the **Virtual Node (ACI)** add-on enabled in `azure-aks-cluster` — the `overlays/aks` patch assumes it exists.
- `metrics-server` installed in the cluster — required for KEDA's `cpu` trigger to report anything other than `<unknown>`. Not installed by default on EKS (see gotcha below); ships by default on AKS.

## KEDA autoscaling — what actually works, and what doesn't

All 4 apps except `uptime-kuma` (see below) have a KEDA `ScaledObject` with a `cpu` trigger targeting 50% utilization. **Confirmed with a real load test** (a busybox pod hammering `podinfo`'s Service in a loop): CPU usage rose from ~1% to ~22% per pod and the Deployment scaled from 2 to 4 replicas within about 2 minutes, then scaled back down after the load stopped — on **EKS**.

**On AKS, the same `cpu` trigger does not currently scale anything for pods on Virtual Nodes.** `kubectl describe hpa` there shows `FailedGetResourceMetric` permanently — `metrics-server` never returns pod metrics for anything scheduled on `virtual-node-aci-linux`, confirmed by directly querying `metrics.k8s.io` repeatedly. Microsoft's own docs list **init containers as unsupported on Virtual Nodes** at all, which is the likely root cause (a terminated-but-unsupported init container leaves the pod's stats in a state ACI's virtual-kubelet can't report cleanly) — but pods *without* an init container (`uptime-kuma`) show the exact same failure, so this is a genuine, reproducible platform gap, not one app's bug. `headlamp`, which runs on AKS's real node pool (not Virtual Nodes), gets metrics fine. See `CLAUDE.md` for the full writeup. If AKS autoscaling for Virtual Node pods becomes a real requirement, use a KEDA *external* trigger (cron, queue depth, custom metric) instead of `cpu`/`memory` — those don't depend on `metrics-server`.

### Why `uptime-kuma` has no KEDA `ScaledObject`

It stores its data (monitors, history, users) in a local SQLite file, with no documented clustering/external-DB support. Scaling it to N replicas wouldn't be "autoscaling" — it would mean N replicas each showing a different, inconsistent set of monitors depending on which one the Service happened to route to. `replicas: 1` is fixed on purpose, not a missed feature.

## Headlamp — cluster web UI

Installed as a two-source Argo `Application` (`bootstrap/headlamp-application.yaml`), not part of the `ApplicationSet`:

1. The official Helm chart (`kubernetes-sigs.github.io/headlamp`), with `clusterRoleBinding.create=false` — the chart's default binds the pod's own ServiceAccount to `cluster-admin`, which is unnecessary attack surface here: `config.unsafeUseServiceAccountToken=false` (also the chart default) means every human logs in with their *own* pasted token, not the pod's identity.
2. `apps/headlamp/extra/` from this same repo — a plain (non-Kustomize) manifest set: a `headlamp-viewer` ServiceAccount + `ClusterRoleBinding` (bound to the built-in `view` ClusterRole, so a human can log in and actually see cluster state) and the KEDA `ScaledObject`.

The chart's `resources: {}` default (no CPU/memory request) is overridden via `helm.parameters` in the same `Application`, because KEDA's admission webhook rejects a `cpu`-trigger `ScaledObject` for a container with no CPU request.

## Usage

```bash
# One-time bootstrap per cluster, once Argo CD is installed:
kubectl apply -f bootstrap/applicationset-eks.yaml    # on the EKS cluster
kubectl apply -f bootstrap/applicationset-aks.yaml    # on the AKS cluster
kubectl apply -f bootstrap/headlamp-application.yaml  # on EACH cluster

# Test an overlay locally without Argo:
kubectl kustomize apps/podinfo/overlays/eks | kubectl apply -f -
```

## Exposing an app publicly

Not wired up by default — apps only get a `ClusterIP` Service (test with `kubectl port-forward`). Two real options when an app needs a public URL:

1. **Its own `Ingress` + own ALB** — simplest, but a new Application Load Balancer per app is a real recurring cost (hourly + LCU).
2. **Share the existing ALB** via `alb.ingress.kubernetes.io/group.name` (same group as `aws-eks-cluster`'s `hello-world` and Argo CD Ingresses) — no extra ALB cost. Path-based routing to a path other than `/` needs the ALB Controller's URL rewrite feature (added in v2.13/v2.14, our controller is v3.5.0) to strip the prefix before it reaches the app — the exact annotation syntax for that still isn't verified, so **use host-based routing instead** (each app gets its own subdomain, not a path) until that's confirmed.

## CI

| Workflow | Checks |
|---|---|
| `validate.yml` | `kubectl kustomize` builds every overlay cleanly, [`kubeconform`](https://github.com/yannh/kubeconform) validates the rendered manifests against the Kubernetes API schema, Checkov (`framework: kubernetes`, blocking) scans the **rendered** output (never raw `apps/*/base|overlays` — see `CLAUDE.md`) for misconfigurations — SARIF uploaded to the Security tab |
| `gitleaks.yml` | Secret scanning (PR + push to `main`) |

No CD workflow here on purpose — Argo CD (pull-based, running in-cluster) is the deployment mechanism, not GitHub Actions.

## Real gotchas found building this (full detail in `CLAUDE.md`)

- `metrics-server` isn't installed by default on EKS, and Fargate reserves port `10250` for its own use — the deployment needs `--secure-port=10251` (or the Amazon EKS add-on, pre-configured for this) or metrics-server can't even serve its own endpoint, let alone scrape kubelet stats.
- Legacy root-requiring images (this repo's `game-2048`, `uptime-kuma`) under `capabilities.drop: ["ALL"]` lose `CAP_DAC_OVERRIDE` even as root — needs the actual capability(ies) the image's own startup path uses (`chown`, `setuid`/`setgid`), confirmed against the real error each time, not guessed.
- Azure Container Instances (Virtual Nodes) rejects any **initContainer that declares `resources`** outright, and truncates `Mi`→`GB` conversions to 1 decimal — `64Mi` becomes `0.0 GB` and gets rejected as non-positive. Both confirmed via the literal ACI error text.
- Checkov's `#checkov:skip=` comment syntax silently does nothing on Kubernetes manifests (works fine on Terraform) — needs a `checkov.io/skipN` **annotation** instead. Its parser also crashes the entire run (not just one check) if a skip justification contains a literal `=` anywhere in the text.
- Checkov scans raw YAML files independently, without running `kustomize build` first — a Kustomize *patch* fragment (e.g. `patch-virtual-node.yaml`, which only sets `nodeSelector`) gets scanned as if it were a complete Deployment, producing dozens of false positives that no skip annotation fixes, because the finding is attached to the patch file, not the real merged resource. Real fix: `validate.yml` scans `kubectl kustomize`-rendered output, not the source.

## Not automated yet

Bumping any app's image tag isn't automated — Dependabot doesn't scan plain Kubernetes YAML for image references the way it does Dockerfiles. If that becomes worth automating, look at Flux's `image-automation-controller` or Renovate before hand-rolling something.
