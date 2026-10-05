# DSO202 Practical 06 Report — Helm

## Objective

This practical creates and operates a Helm chart using Helm 4. It demonstrates templates, values, JSON Schema validation, releases, upgrades, rollback, history, and uninstall.

## Evidence

Command output is stored in `evidence/`. The live release reached Helm revision 5 after install, upgrades, rollback, and value restoration. `helm test` completed with `Phase: Succeeded`. The requested `nginx:1.30-alpine` image could not be pulled because Docker Desktop's registry mirror returned HTTP 500, so the live release used `--set image.tag=1.27-alpine`; the chart default remains `1.30-alpine`.

## Analysis

Explain the chart-versus-release distinction, values precedence, checksum-based rollouts, why `helm lint` is insufficient by itself, and why `--reuse-values` or complete version-controlled values files are safer during upgrades. The reset-values demonstration showed only `image.tag` and `page.environment` as user-supplied values, so the original page message was replaced by the chart default; the following `--reuse-values` upgrade restored it.

## Review Questions and Answers

### 3.2.1.1 — The Problem Helm Solves

1. `kubectl apply -f` does not track an application as one unit. If an object is removed from a manifest, it can remain as an orphan. Helm records release ownership and removes the release objects during `helm uninstall`.
2. A database password can be read from the values file or repository and from the Helm release record stored in the cluster. It may also appear in shell history or CI logs when supplied with `--set`.
3. Kustomize is suitable for small environment-specific changes to team-owned manifests. Helm is suitable for packaging, versioning, configuring, installing, upgrading, rolling back, and distributing an application as one unit.

### 3.2.1.2 — Helm Architecture and Core Objects

1. Revision 7 of release `payments` in namespace `finance` is stored in `sh.helm.release.v1.payments.v7`.
2. Removing Tiller improved security because Helm no longer needs a privileged server component. Helm uses the caller's Kubernetes credentials and RBAC permissions.
3. Release names are unique per namespace, so `web` may exist in both `team-a` and `team-b`. Use `helm list -A` to list both.

### 3.2.1.4 — Consuming a Published Chart

1. After `helm upgrade r c --set a=1`, a previous value `b=2` is discarded unless supplied again. `--reuse-values` preserves previous user values.
2. `helm rollback r 1` creates a new revision containing revision 1's configuration; it does not rewrite history. Thus it becomes revision 4 when revisions 1–3 already exist.
3. OCI registries provide shared authentication and access control for images and charts, and immutable digests make exact artifact verification and promotion easier.

### 3.2.1.5 — Chart Structure and `Chart.yaml`

1. If the chart is `1.4.2` and only its default replica count changes, a valid new chart version is `1.4.3`.
2. Files beginning with `_`, such as `_helpers.tpl`, contain named templates and never render as Kubernetes objects.
3. The `-0` in `>=1.30.0-0` allows compatible pre-release Kubernetes versions to satisfy the constraint.

### 3.2.2.1 — Template Language

1. `{{ quote (upper .Values.name) }}` becomes `{{ .Values.name | upper | quote }}`.
2. Inside `range`, dot refers to the current item, so `.Release.Name` fails. Use `$.Release.Name` for the root context.
3. `include` returns text and can be piped into `nindent`; `template` writes directly and cannot be piped in the same way.

### 3.2.2.3 — Values and Precedence

1. In `helm install r ./c --set a=1 -f x.yaml`, where `x.yaml` sets `a: 2`, the final value is `1` because `--set` has higher precedence.
2. `-f dev.yaml -f prod.yaml` can leave a dev NodePort in the production render because maps merge deeply and prod may not define `service`. Each environment file should define its complete intended policy.
3. Unquoted `1.30` in YAML can become numeric `1.3`, while `--set image.tag=1.30` remains a string. Version-like YAML values should be quoted.

### 3.2.2.4 — Validation

1. `helm lint` can pass when rendering fails because missing required values may be warnings. CI should run both lint and a render for each values file.
2. A schema restricting the port to 80 or 8080 is:

   ```json
   "port": { "type": "integer", "enum": [80, 8080] }
   ```

3. `helm template` only produces text and does not send objects to the API server, so it cannot detect an unknown field such as `spec.replica`. Server-side Kubernetes validation detects it.

### 3.2.3 — Release Lifecycle

1. Install creates revision 1, upgrade creates a new revision, rollback creates another revision based on an earlier one, and uninstall removes the release objects and records by default.
2. Helm 4 calls the failure-safe behavior `--rollback-on-failure`; it was called `--atomic` in Helm 3.
3. Rollback restores rendered Kubernetes objects, not database writes, persistent-volume data, namespace changes, or CRD changes.
4. Helm 4 uses server-side apply for new releases and records the apply method. A field owned by another manager can cause a conflict; `--force-conflicts` explicitly takes ownership back.

## Reflection

Document one templating or validation mistake and how the rendered output or error message exposed it.
