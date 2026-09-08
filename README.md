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

Not wired up by default — `podinfo` only gets a `ClusterIP` Service (test with `kubectl port-forward svc/podinfo 9898:80`). Two real options when an app needs a public URL, neither implemented here yet:

1. **Its own `Ingress` + own ALB** — simplest, but a new Application Load Balancer per app is a real recurring cost (hourly + LCU).
2. **Share the existing ALB** via `alb.ingress.kubernetes.io/group.name` (same group as `aws-eks-cluster`'s `hello-world` Ingress) — no extra ALB cost, but path-based routing to a different path than `/` needs the ALB Controller's URL rewrite feature (added in v2.13/v2.14, our controller is v3.5.0) to strip the path prefix before it reaches the app — the exact annotation syntax for that wasn't verified before writing this repo, so it's not shipped here as a working example. Confirm the syntax against the [AWS Load Balancer Controller docs](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html) before relying on it.

## CI

| Workflow | Checks |
|---|---|
| `validate.yml` | `kubectl kustomize` builds every overlay cleanly, [`kubeconform`](https://github.com/yannh/kubeconform) validates the rendered manifests against the Kubernetes API schema, Checkov (`framework: kubernetes`, blocking) scans for misconfigurations — SARIF uploaded to the Security tab |
| `gitleaks.yml` | Secret scanning (PR + push to `main`) |

No CD workflow here on purpose — Argo CD (pull-based, running in-cluster) is the deployment mechanism, not GitHub Actions.

## Not automated yet

Bumping `podinfo`'s image tag (or any future app's) isn't automated — Dependabot doesn't scan plain Kubernetes YAML for image references the way it does Dockerfiles. If that becomes worth automating, look at Flux's `image-automation-controller` or Renovate before hand-rolling something.
