# DSO202 Practical 05 — Environment-Specific Configuration with Kustomize

This practical demonstrates a reusable Kubernetes base with development, staging, production, QA, and sandbox overlays. The workflow is `render → diff → apply → verify`.

## Layout

- `cluster/` contains Kind configurations.
- `examples/webapp/` contains the Kustomize base and overlays.
- `evidence/` stores command output captured during the practical.
- `report/` contains the practical report.

## Rebuild and verification

```bash
kubectl kustomize examples/webapp/base
kubectl kustomize examples/webapp/overlays/dev
kubectl diff -k examples/webapp/overlays/dev || true
kubectl apply -k examples/webapp/overlays/dev
kubectl get all -n webapp-dev
```

Repeat the render/diff/apply/verify sequence for staging, prod, and qa. The report explains the ConfigMap hash rollout chain and the difference between strategic merge and JSON 6902 patches.
