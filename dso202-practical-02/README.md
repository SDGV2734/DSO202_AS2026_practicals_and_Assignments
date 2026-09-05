# DSO202 Practical 2 — Persistent Storage and StatefulSets

This repository implements persistent storage for a stateful Kubernetes workload. It demonstrates static and dynamic provisioning, StorageClass policies, the failure mode of sharing one claim from a Deployment, StatefulSet identity and storage, and PostgreSQL persistence.

## Environment

The practical uses Docker Desktop, kind v0.32.0, Kubernetes v1.36.1, kubectl v1.36.3, and the official `postgres:18-alpine` image. The cluster is named `dso202-p2` and contains one control-plane node and two workers. The first worker receives `/tmp/dso202-p2-storage` at `/mnt/dso202-static` for the static PV experiment.

## Repository contents

- `cluster/` contains the primary and fallback kind configurations.
- `manifests/` contains the official Listings 2–16, with explicit namespaces on every namespaced object.
- `evidence/` contains command output for Stages 0–8 and `tasktracker-dump.sql`.
- `report/practical-02-report.md` contains the assessed report.

## Rebuild sequence

```sh
mkdir -p /tmp/dso202-p2-storage
kind create cluster --config cluster/kind-cluster.yaml
kubectl apply -f manifests/00-namespace.yaml
kubectl config set-context --current --namespace=dso202-practical-02
kubectl apply -f manifests/01-quota-and-limits.yaml
kubectl apply -f manifests/02-storageclass-retain.yaml
kubectl apply -f manifests/03-pv-static.yaml
kubectl apply -f manifests/04-pvc-static.yaml
kubectl apply -f manifests/05-pod-static-writer.yaml
kubectl apply -f manifests/06-pvc-dynamic.yaml
kubectl apply -f manifests/07-pod-dynamic-writer.yaml
kubectl apply -f manifests/08-deployment-shared-pvc.yaml
kubectl delete -f manifests/08-deployment-shared-pvc.yaml
kubectl apply -f manifests/09-service-webnote.yaml
kubectl apply -f manifests/10-statefulset-webnote.yaml
kubectl apply -f manifests/11-pod-client.yaml
kubectl apply -f manifests/12-secret-postgres.yaml
kubectl apply -f manifests/13-service-postgres.yaml
kubectl apply -f manifests/14-statefulset-postgres.yaml
```

The practical stages include deliberate deletion and recreation commands. The exact observations and outputs are recorded in `evidence/`.

## Cleanup

Delete the workloads first, then delete claims explicitly so the difference between `Delete` and `Retain` is visible:

```sh
kubectl delete -f manifests/14-statefulset-postgres.yaml
kubectl delete -f manifests/10-statefulset-webnote.yaml
kubectl delete -f manifests/11-pod-client.yaml
kubectl delete -f manifests/05-pod-static-writer.yaml
kubectl delete pvc --all
kubectl delete pv pv-web-static
```

