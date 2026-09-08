# AWS EKS Apps

GitOps source of truth for application manifests deployed to [`aws-eks-cluster`](https://github.com/jalcalaroot/aws-eks-cluster) — separate from the infrastructure repo on purpose. Argo CD (installed in the cluster, not managed here) watches this repo and keeps the cluster in sync automatically.

## Structure

```
apps/<name>/
  base/                  # plain Deployment + Service, no environment-specific values
    kustomization.yaml
    deployment.yaml
    service.yaml
  overlays/dev/           # patches over base (replica count, namespace, etc.)
    kustomization.yaml
bootstrap/
  applicationset.yaml     # tells Argo CD to watch apps/*/overlays/dev — apply once, manually
```

Adding a new app = adding a new `apps/<name>/` directory with this same `base` + `overlays/dev` shape. The `ApplicationSet` in `bootstrap/` auto-discovers it — no need to touch `bootstrap/` for each new app.

## Why Kustomize, not Helm

No templating language to learn, native `kubectl kustomize` support, and the `base` + `overlays` pattern maps directly onto "one app, multiple environments" without inventing a values schema per app. Helm makes more sense if these apps were ever *distributed* to others — not the case for internal demos.

## Why a Deployment, not a bare Pod

Every app here uses `Deployment` (the standard controller for stateless HTTP services — replica management, rolling updates, self-healing). Reach for `StatefulSet` only if an app needs stable identity/ordering, `Job`/`CronJob` for anything that runs to completion instead of serving traffic. Never manage bare `Pod` objects directly.

## Example app: `podinfo`

[`stefanprodan/podinfo`](https://github.com/stefanprodan/podinfo) — the de facto reference app for testing GitOps pipelines (built by Flux's own maintainer, used by Flux/Flagger's own end-to-end tests). Used here instead of a custom image so this repo has a real, working example without maintaining a Dockerfile just for the demo.

## Prerequisites

- Argo CD installed in the target cluster (see `aws-eks-cluster`)
- The target namespace (`default`, currently — see below) already has a matching **Fargate Profile** in `aws-eks-cluster` — Fargate scheduling is decided by namespace selector in Terraform, not by anything in this repo. Deploying to a *new* namespace requires adding a Fargate Profile there first, or the pods stay `Pending` forever.

## Usage

```bash
# One-time bootstrap, once Argo CD is installed:
kubectl apply -f bootstrap/applicationset.yaml

# Test an overlay locally without Argo:
kubectl kustomize apps/podinfo/overlays/dev | kubectl apply -f -
# or, without touching the cluster:
kubectl kustomize apps/podinfo/overlays/dev
```

## Exposing an app publicly

Not wired up by default — `podinfo` only gets a `ClusterIP` Service (test with `kubectl port-forward svc/podinfo 9898:80`). Two real options when an app needs a public URL:

1. **Its own `Ingress` + own ALB** — simplest, but a new Application Load Balancer per app is a real recurring cost (hourly + LCU).
2. **Share the existing ALB** via `alb.ingress.kubernetes.io/group.name` (same group as `aws-eks-cluster`'s `hello-world` and Argo CD Ingresses) — no extra ALB cost. Path-based routing to a path other than `/` needs the ALB Controller's URL rewrite feature (added in v2.13/v2.14, our controller is v3.5.0) to strip the prefix before it reaches the app — the exact annotation syntax for that still isn't verified, so **use host-based routing instead** (each app gets its own subdomain, not a path) until that's confirmed. Host-based routing is what the planned apps below use.

**Planned upgrade (not deployed yet, see `aws-eks-cluster` CLAUDE.md for the full plan)**: replace the current one-ACM-cert-per-subdomain pattern with a single wildcard cert (`*.aws.jalcalaroot.com`) plus **External DNS**, so a new app's Ingress just declares its `host:` and gets a working cert + Route 53 record automatically — no Terraform change in `aws-eks-cluster` per app. Until that lands, a new public subdomain still means a manual cert + DNS step there.

## Roadmap: 3 apps to exercise each pattern

Decided, not built yet — each proves a different path through this repo + Argo:

| App | Pattern | Namespace | Planned host | Notes |
|---|---|---|---|---|
| [`2048`](https://github.com/aws-samples/eks-workshop-samples) (`public.ecr.aws/l6m2t8p7/docker-2048`) | Kustomize (this repo), like `podinfo` | `default` | `2048.aws.jalcalaroot.com` | AWS's own EKS workshop sample for testing ALB Ingress — playable, not just a health-check demo |
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) (`louislam/uptime-kuma`) | Kustomize (this repo) | `default` | `status.aws.jalcalaroot.com` | Status/monitoring dashboard — can genuinely monitor `hello-world`/Argo CD/`2048` once it's up, not purely decorative. No official Helm chart worth trusting, hence Kustomize. |
| [Kubernetes Dashboard](https://github.com/kubernetes/dashboard) | **Helm** (official chart, `source.helm` on the Argo `Application` — not Kustomize) | new namespace, TBD (needs its own Fargate Profile first, see below) | `k8s.aws.jalcalaroot.com` | Real UI for the cluster this whole stack runs on. Stateless — no PVC needed, avoids the Fargate/EBS gotcha other charts (Grafana, etc.) would hit. |

Prerequisites before building these for real:
- The wildcard cert + External DNS upgrade above (or fall back to a manual cert+DNS step per app, same as `eks.*`/`argocd.*`)
- A Fargate Profile for Kubernetes Dashboard's namespace, added in `aws-eks-cluster/eks.tf` (same pattern as `argocd`'s)
- Kubernetes Dashboard needs its own auth/RBAC decision (token-based login by default) — not designed yet, do it when actually building this app, not before

## CI

| Workflow | Checks |
|---|---|
| `validate.yml` | `kubectl kustomize` builds every overlay cleanly, [`kubeconform`](https://github.com/yannh/kubeconform) validates the rendered manifests against the Kubernetes API schema, Checkov (`framework: kubernetes`, blocking) scans for misconfigurations — SARIF uploaded to the Security tab |
| `gitleaks.yml` | Secret scanning (PR + push to `main`) |

No CD workflow here on purpose — Argo CD (pull-based, running in-cluster) is the deployment mechanism, not GitHub Actions.

## Not automated yet

Bumping `podinfo`'s image tag (or any future app's) isn't automated — Dependabot doesn't scan plain Kubernetes YAML for image references the way it does Dockerfiles. If that becomes worth automating, look at Flux's `image-automation-controller` or Renovate before hand-rolling something.
