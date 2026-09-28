# DSO202 Practical 05 Report

## Objective

This practical uses Kustomize to maintain one reusable webapp base and deploy environment-specific dev, staging, production, QA, and sandbox variants without copying the base Deployment or Service.

## Environment

Record the Kind, Kubernetes, kubectl, and node details in `evidence/stage0-environment.txt`. In this run, kubectl was installed but the Docker Desktop Kubernetes endpoint was unreachable from the execution environment, so cluster-side apply/rollout/port-forward evidence remains a clearly documented follow-up.

## Procedure and observations

Complete the stage descriptions using the captured evidence files. Include the base render, dev/prod diff, safe deployment workflow, port-forward response, ConfigMap hash rollout, production resource patch, QA JSON 6902 annotation, and sandbox namePrefix prediction. Local rendering produced `web-content-b2kmc2fdht` before the dev content change and `web-content-6tmbt2df5m` after it.

## Required analysis

The generated ConfigMap hash connects file content to rollout: changing `index.html` changes the generated ConfigMap name, the Deployment's ConfigMap reference, the Pod template hash, and therefore the Pods.

The production resource patch is a strategic merge patch. It merges the matching Deployment/container fields into the base. The QA owner annotation is a JSON 6902 patch, which applies an explicit operation at a precise path. Both keep environment policy in the overlay and avoid duplicating the base manifests.

## Reflection

Document one Kustomize mistake, the rendered output that exposed it, and the correction.
