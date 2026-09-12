# Security Policy

This is a personal learning/reference GitOps repository (Kubernetes manifests), not intended for production use. There are no supported version branches — only `main` is maintained.

## Reporting a Vulnerability

If you find a security issue — an exposed secret, a leaked credential in git history, a misconfigured manifest, or a vulnerable dependency — please report it privately using [GitHub's private vulnerability reporting](../../security/advisories/new) instead of opening a public issue.

## Scope

- The Kubernetes manifests in this repository (`apps/`, `bootstrap/`)
- Accidentally committed secrets, kubeconfig files, or credentials

Out of scope: vulnerabilities in the demo application images themselves (e.g. `podinfo`), Argo CD, Kustomize, or the underlying EKS/AKS cluster — please report those to their respective maintainers, or see [`aws-eks-cluster`](https://github.com/jalcalaroot/aws-eks-cluster) / [`azure-aks-cluster`](https://github.com/jalcalaroot/azure-aks-cluster) for the cluster infrastructure.
