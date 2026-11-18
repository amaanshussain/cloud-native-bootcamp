---
tags:
  - Lab
  - Intermediate
  - Packaging
  - Helm
  - Kustomize
  - OpenShift
  - Kubernetes
---

# Helm & Kustomize Lab

<span class="lab-badge">45 min</span> <span class="lab-badge">Intermediate</span>

In this lab you'll package and deploy the `greeting` microservice across multiple environments using **Kustomize** overlays and manage releases, upgrades, and rollbacks using **Helm**.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Helm & Kustomize](../../../openshift/packaging/index.md) or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the manifests and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Helm & Kustomize](../../../openshift/packaging/index.md) in a second tab before you start.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with `kubectl`). See [Lab Environments](../../../lab-environments.md).
- The `helm` CLI (v3 or v4).
- A local clone of the bootcamp repository with your terminal located in `sample-app`:

```bash
cd sample-app
```

## Part 1: Kustomize Overlays

1. Inspect the Kustomize directory structure and render the `dev` and `prod` overlays locally without applying them.
    - **Reference:** [Helm & Kustomize, Kustomize](../../../openshift/packaging/index.md#kustomize) shows `oc kustomize <directory>`.
    - Compare differences between rendered manifests:

    ```bash
    oc kustomize kustomize/overlays/dev
    diff <(oc kustomize kustomize/overlays/dev) <(oc kustomize kustomize/overlays/prod)
    ```

    - **Check:** `diff` reveals replica counts (1 in dev vs 3 in prod), labels (`environment: dev` vs `prod`), and ConfigMap greeting messages.

2. Create `greeting-dev` and `greeting-prod` projects and apply the respective Kustomize overlays.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows `oc new-project`. [Helm & Kustomize, Kustomize](../../../openshift/packaging/index.md#kustomize) shows `oc apply -k <directory>`.

    ```bash
    oc new-project greeting-dev
    oc apply -k kustomize/overlays/dev -n greeting-dev
    oc new-project greeting-prod
    oc apply -k kustomize/overlays/prod -n greeting-prod
    oc rollout status deployment/greeting -n greeting-dev
    oc rollout status deployment/greeting -n greeting-prod
    ```

    - **Check:** both deployments roll out successfully, and `oc get pods -n greeting-prod` shows 3 running pods.

3. Verify both environments by calling their respective Route endpoints.
    - **Reference:** [Routes, Commands](../../../openshift/services-networking/routes.md#commands) shows retrieving Route hostnames.

    ```bash
    for env in dev prod; do
      curl -sk "https://$(oc get route greeting -n greeting-$env -o jsonpath='{.spec.host}')/greeting"; echo
    done
    ```

    - **Check:** `greeting-dev` outputs `"message":"Hello from dev"` and `greeting-prod` outputs `"message":"Welcome to production"`.

4. Modify `kustomize/overlays/dev/kustomization.yaml` to change the greeting text, preview with `oc diff -k`, and apply.
    - **Reference:** [Helm & Kustomize, Kustomize](../../../openshift/packaging/index.md#kustomize) shows `oc diff -k` and `oc apply -k`.
    - **Check:** `oc diff -k kustomize/overlays/dev -n greeting-dev` highlights the generated ConfigMap hash name and deployment pod template update, causing an automatic rolling update when applied.

5. Create a new overlay directory `kustomize/overlays/test` with 2 replicas, label `environment: test`, and greeting `Hello from test`, then deploy it to `greeting-test`.
    - **Reference:** [Helm & Kustomize, Kustomize](../../../openshift/packaging/index.md#kustomize) shows overlay structure with `resources`, `components`, `labels`, `replicas`, and `configMapGenerator`.
    - **Check:** `curl -sk "https://$(oc get route greeting -n greeting-test -o jsonpath='{.spec.host}')/greeting"` returns `{"message":"Hello from test","name":"World"}`.

## Part 2: Helm Releases

6. Lint and render the `greeting` Helm chart locally.
    - **Reference:** [Helm & Kustomize, Helm](../../../openshift/packaging/index.md#helm) shows `helm lint` and `helm template`.
    - **Check:** `helm lint helm/greeting` reports `0 chart(s) failed`, and `helm template` outputs valid Kubernetes YAML.

7. Create a project named `greeting-helm` and install the chart as a release named `greeting` with a custom greeting.
    - **Reference:** [Helm & Kustomize, Helm](../../../openshift/packaging/index.md#helm) shows `helm install <release> <chart> --set ...`.

    ```bash
    oc new-project greeting-helm
    helm install greeting ./helm/greeting --set greeting="Hello from Helm" --wait
    ```

    - **Check:** `helm list` shows release `greeting` with status `deployed` (revision 1).

8. Upgrade the release to `replicaCount=2` while preserving values, view history, and run chart tests.
    - **Reference:** [Helm & Kustomize, Helm](../../../openshift/packaging/index.md#helm) shows `helm upgrade`, `helm history`, and `helm test`.

    ```bash
    helm upgrade greeting ./helm/greeting --reuse-values --set replicaCount=2 --wait
    helm history greeting
    helm test greeting
    ```

    - **Check:** `helm history greeting` shows revision 2, and `helm test greeting` passes (`Phase: Succeeded`).

9. Roll back the release to revision 1 and verify the replica count scales back to 1.
    - **Reference:** [Helm & Kustomize, Helm](../../../openshift/packaging/index.md#helm) shows `helm rollback <release> <revision>`.
    - **Check:** `helm rollback greeting 1 --wait` creates revision 3, and `oc get deployment greeting -n greeting-helm` shows 1 replica.

10. Upgrade the release using `values-prod.yaml` to apply production resource requirements and replicas.
    - **Reference:** [Helm & Kustomize, Helm](../../../openshift/packaging/index.md#helm) shows `helm upgrade --install -f <values-file>`.

    ```bash
    helm upgrade --install greeting ./helm/greeting -f helm/greeting/values-prod.yaml --wait
    ```

    - **Check:** `helm get values greeting` displays production values and `oc get pods -n greeting-helm` lists 3 pods.

## Cleanup

Uninstall the Helm release and delete all created projects.

- **Reference:** [Helm & Kustomize, Helm](../../../openshift/packaging/index.md#helm) and [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md#references).

```bash
helm uninstall greeting -n greeting-helm
oc delete project greeting-dev greeting-prod greeting-test greeting-helm
```

| Part | What it does |
| --- | --- |
| `helm uninstall greeting ...` | Removes all resources managed by the Helm release |
| `oc delete project ...` | Deletes all lab projects |

## What you learned

- [x] Structuring multi-environment deployments using Kustomize bases and overlays
- [x] Leveraging `configMapGenerator` content hashes for automated deployment rollouts
- [x] Packaging applications into Helm charts with configurable `values.yaml`
- [x] Managing Helm release lifecycles: linting, templating, installing, upgrading, and rolling back
