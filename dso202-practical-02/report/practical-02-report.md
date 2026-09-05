# DSO202 Practical 2 Report

## 1. Objective

The objective was to implement persistent storage for Kubernetes workloads and observe how storage lifetime differs from Pod lifetime. The practical covers PersistentVolumes, PersistentVolumeClaims, StorageClasses, reclaim policies, deferred binding, StatefulSet identity, per-ordinal storage, stable DNS, and a PostgreSQL workload. It addresses LO4 directly and reinforces LO1–LO3.

## 2. Environment

The work used Docker Desktop 28.4.0, kind v0.32.0, kubectl v1.36.3, and Kubernetes v1.36.1 on an Apple Silicon Docker environment. The cluster was `dso202-p2` with nodes `control-plane`, `worker-node-1`, and `worker-node-2`. The PostgreSQL image was `postgres:18-alpine`; the webnote image was initially `nginx:1.30-alpine` and was updated to `nginx:1.31-alpine` during Stage 6. Version and node output is in [`stage0-environment.txt`](../evidence/stage0-environment.txt).

## 3. Procedure and Observations

### Stage 0 — Prerequisites

The tools reported their versions, all three nodes became Ready, and the host directory was made available to the first worker. The environment output is recorded in `stage0-environment.txt`.

### Stage 1 — Cluster and StorageClasses

The namespace, quota, LimitRange, and `dso202-retain` StorageClass were applied. The cluster also contained kind’s default `standard` class. Both use `rancher.io/local-path` and `WaitForFirstConsumer`; their important difference is `Retain` versus `Delete`. The quota limits claim count and requested storage.

### Stage 2 — Static provisioning

`pv-web-static` was created as an Available 1Gi PV with `manual` class, `ReadWriteOnce`, hostPath storage, node affinity for `worker-node-1`, and `Retain`. The claim bound immediately. The writer Pod was scheduled on the selected worker, and its ledger remained after Pod deletion, claim deletion, PV-object deletion, and recreation. The consolidated output is in [`stage2-static-storage.txt`](../evidence/stage2-static-storage.txt); the final three-line ledger extract is [`stage2-ledger.txt`](../evidence/stage2-ledger.txt).

### Stage 3 — Dynamic provisioning

The `dynamic-data` claim stayed Pending with a `WaitForFirstConsumer` event until `dynamic-writer` was created. The provisioner then created a PV automatically on the Pod’s node. `df -h /data` showed the node filesystem rather than a 1Gi device, demonstrating that local-path capacity is recorded but not enforced. Expanding the claim was rejected because the class does not allow expansion. Deleting the Pod and claim removed the standard volume after asynchronous reclaim. Output is in [`stage3-dynamic-storage.txt`](../evidence/stage3-dynamic-storage.txt).

### Stage 4 — Deployment anti-pattern

Three Deployment replicas mounted one claim. All three were scheduled on one node because the RWO volume was node-local. They appended to one shared `visitors.log`. Deleting the replicas caused new hash-suffixed names, so no replica identity survived. The experiment was then removed. Output is in [`stage4-deployment-shared-pvc.txt`](../evidence/stage4-deployment-shared-pvc.txt).

### Stage 5 — StatefulSet identity and storage

The headless `webnote` Service returned one DNS address per ready Pod. `webnote-0`, `webnote-1`, and `webnote-2` were created in ordinal order and received separate claims. A note written to `webnote-0` was absent from `webnote-1`. After deleting `webnote-1`, the replacement kept the same name and claim, received a new IP, and retained its original `created:` timestamp. Output is in [`stage5-statefulset.txt`](../evidence/stage5-statefulset.txt).

### Stage 6 — Scaling and updates

Scaling to four created `content-webnote-3`. Scaling down to two terminated ordinals 3 and 2 while all four claims remained because both retention fields were `Retain`. Scaling back to three reused ordinal 2’s claim and data. With `partition: 2` and nginx 1.31, only ordinal 2 updated; returning the partition to 0 completed the descending rollout. Deleting and recreating the StatefulSet left all claims and their data intact. Output is in [`stage6-scaling-updates.txt`](../evidence/stage6-scaling-updates.txt).

### Stage 7 — PostgreSQL

The Secret supplied the database, user, and password values. The ClusterIP Service is the application endpoint, while the headless Service provides the stable Pod DNS name. PostgreSQL mounted the retained claim at `/var/lib/postgresql`, with the PostgreSQL 18 data directory below it. Three rows were inserted into `tasks`; after deleting and replacing `postgres-0`, `SELECT count(*)` still returned 3. Both PostgreSQL DNS names resolved from the client Pod. Output is in [`stage7-postgres.txt`](../evidence/stage7-postgres.txt).

### Stage 8 — Cleanup

The final state and a logical SQL dump were captured before deleting workloads. Deleting claims removed the standard webnote volumes, while the static and PostgreSQL retained volumes entered `Released`. The static PV object was removed, leaving the retained PostgreSQL PV as the deliberate cleanup example. The dump is [`tasktracker-dump.sql`](../evidence/tasktracker-dump.sql), and cleanup output is [`stage8-cleanup.txt`](../evidence/stage8-cleanup.txt). The kind cluster itself was left in place because it also hosts Assignment 1 resources.

## 4. Analysis

1. The field responsible for the initial Pending state is `volumeBindingMode: WaitForFirstConsumer`. It delays provisioning until scheduling identifies the consuming node, preventing node-local storage from being created where the Pod cannot run.
2. `reclaimPolicy` on the StorageClass determines what happens after claim deletion. The cluster administrator or platform team chooses it; the claim author only selects the class.
3. The Deployment replicas were placed on one node because the RWO claim was backed by node-local storage and its PV node affinity restricted scheduling. With a zonal cloud disk, replicas placed on different nodes would usually remain Pending or report a multi-attach error.
4. The second replica is `webnote-2.webnote.dso202-practical-02.svc.cluster.local`. It requires the `webnote` headless Service, the StatefulSet’s `serviceName: webnote`, the `webnote-2` Pod, and the namespace DNS service.
5. Scaling from four to two terminated ordinals 3 then 2 but retained all four claims. Scaling back to three recreated ordinal 2 and reused its claim. `whenScaled` controls scale-down claims and `whenDeleted` controls controller deletion; both were explicitly set to `Retain`, which is also the safe default used here.
6. PostgreSQL 18 stores data below `/var/lib/postgresql/18/docker`, so the parent `/var/lib/postgresql` is mounted. Mounting directly over the data directory can hide image-created contents and can make `initdb` reject a non-empty volume.
7. A StatefulSet does not replicate database data or provide backup, leader election, or failover. Application replication or a database Operator provides replication and membership; logical dumps, snapshots, or backup tooling provide recovery.
8. A retained PV becomes `Released` because its old claim reference remains and Kubernetes will not hand possibly sensitive old data to an unrelated claim automatically. An administrator must inspect the data and reclaim, delete, or recreate the PV deliberately.

## 5. Reflection

The main troubleshooting issue was the static hostPath mount: the existing Docker bind mount had lost its host child directory, so kubelet reported `mkdir /mnt/dso202-static/pv-web-static: no such file or directory`. Inspecting Pod events identified the cause; recreating the Docker bind source and restarting only the worker restored the mount. PostgreSQL also took several minutes to pull its image, but its logs showed that the server was ready and the readiness probe then passed. If repeating this work, I would start from a dedicated cluster and a clean host directory so the first ledger contains only the three intended lines. The remaining operational limitation is that local-path volumes are node-local and their requested capacity is not enforced; a CSI-backed cloud cluster would behave differently.

## 6. References

- DSO202 Practical 2 Guide, supplied course document, accessed 4 October 2026.
- DSO202 Practical 2 Manifests, supplied companion listing, accessed 4 October 2026.
- Kubernetes documentation for PersistentVolumes, PersistentVolumeClaims, StorageClasses, Services, and StatefulSets, consulted 4 October 2026.
