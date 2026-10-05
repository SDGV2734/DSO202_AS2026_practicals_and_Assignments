# DSO202 Practical 06 — Helm Charts and Release Lifecycle

This practical creates and operates a Helm 4 chart for an nginx webapp. It demonstrates chart metadata, values, Go templates, schema validation, environment values, checksum-based rollouts, release inspection, upgrade, rollback, and uninstall.

## Commands

```bash
helm lint webapp -f environments/dev.yaml
helm template webapp-dev ./webapp -f environments/dev.yaml --namespace dso202-dev
helm install webapp-dev ./webapp -f environments/dev.yaml -n dso202-dev --create-namespace --wait
helm history webapp-dev -n dso202-dev
helm uninstall webapp-dev -n dso202-dev
```
