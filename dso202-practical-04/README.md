# DSO202 GitOps and Argo CD Practical

This repository is the GitOps configuration repository for the DSO202 Argo CD lab. Git is the desired-state source of truth; Argo CD watches this repository and reconciles the Kubernetes cluster automatically.

## Introduction

This practical demonstrates GitOps using GitHub, Kubernetes, and Argo CD. Instead of applying application manifests manually with `kubectl`, the desired Kubernetes configuration is stored in Git and Argo CD continuously compares that configuration with the live cluster. Changes are introduced through commits, reviewed through Git history, and reconciled automatically by Argo CD. The exercises cover automated synchronization, configuration drift, self-healing, and pruning of resources removed from Git.

## Repository contents

```text
dso202-gitops-demo/
├── README.md
├── argocd-app.yaml
└── manifests/
    └── deployment.yaml
```

## Completed checklist

| Guide requirement | Completion | Where it is demonstrated |
|---|---|---|
| Declarative Kubernetes manifests | Complete | `manifests/deployment.yaml` |
| GitHub as the desired-state source | Complete | This repository and its `main` branch |
| Argo CD Application created declaratively | Complete | `argocd-app.yaml` |
| Argo CD watches the `manifests` path | Complete | `spec.source.path: manifests` in `argocd-app.yaml` |
| Deployment target is the `demo-app` namespace | Complete | `spec.destination.namespace` in `argocd-app.yaml` |
| Automated synchronization | Complete | `spec.syncPolicy.automated` |
| Automatic pruning | Complete | `prune: true` in `argocd-app.yaml`; demonstrated by commit `a6cda9a` |
| Automatic self-healing | Complete | `selfHeal: true` in `argocd-app.yaml`; demonstrated after manual scaling drift |
| Automatic namespace creation | Complete | `CreateNamespace=true` in `argocd-app.yaml` |
| Initial two-replica deployment | Complete | Commit `70b512f` and the initial Deployment configuration |
| Git-based scale from 2 to 4 replicas | Complete | Commit `525f433`; `replicas: 4` in `manifests/deployment.yaml` |
| Configuration-drift experiment | Complete | Manual `kubectl scale` was corrected by Argo CD self-healing |
| Pruning experiment | Complete | Commit `a6cda9a` removes `manifests/service.yaml` |
| Application health verification | Complete | `kubectl get applications -n argocd` and Argo CD UI |
| Git commit history/audit trail | Complete | `git log --oneline` |
| Screenshots | Intentionally omitted | Terminal/UI screenshots were not added, as requested |

## Argo CD Application

`argocd-app.yaml` defines:

- Application name: `my-webapp`
- Argo CD namespace: `argocd`
- Git repository: `https://github.com/SDGV2734/dso202-gitops-demo.git`
- Branch: `main`
- Manifest path: `manifests`
- Destination namespace: `demo-app`
- Automated sync, pruning, and self-healing

Apply the Application definition with:

```bash
kubectl apply -f argocd-app.yaml
```

## Verification commands

```bash
kubectl get applications -n argocd
kubectl describe application my-webapp -n argocd
kubectl get all -n demo-app
kubectl get pods -n demo-app -o wide
```

Expected Argo CD status after it can reach GitHub:

```text
SYNC STATUS:   Synced
HEALTH STATUS: Healthy
```

## GitOps experiments

### Scale through Git

Change `replicas` in `manifests/deployment.yaml`, commit, and push:

```bash
git add manifests/deployment.yaml
git commit -m "Scale webapp to 4 replicas"
git push origin main
```

Argo CD detects the new commit and reconciles the Deployment.

### Self-healing

The manual drift command used for the experiment was:

```bash
kubectl scale deployment webapp-deploy -n demo-app --replicas=1
```

Because Git still specifies four replicas and `selfHeal: true` is enabled, Argo CD restores the Deployment to four replicas.

### Pruning

The Service was removed from Git in commit `a6cda9a`:

```bash
git show --stat a6cda9a
```

With `prune: true`, Argo CD removes the corresponding `webapp-svc` Service from `demo-app`.

## Important status note

If Argo CD reports `ComparisonError` with `Could not resolve host: github.com`, the cluster's Argo CD repository server cannot resolve or reach GitHub. This is a cluster DNS/network issue, not a manifest-format issue. Check the repository server logs and CoreDNS before changing the Application URL.

## Conclusion

The practical demonstrates the main GitOps workflow successfully. Kubernetes resources are declared in Git, Argo CD uses the repository as the desired-state source, and automated synchronization keeps the cluster aligned with the committed configuration. Scaling through a Git commit showed Git-based deployment, manually changing the replica count demonstrated self-healing, and removing the Service manifest demonstrated automatic pruning. The repository history provides an auditable record of each infrastructure change. Screenshots were intentionally omitted, but the repository files, commit history, commands, and Argo CD configuration document the completed work.


