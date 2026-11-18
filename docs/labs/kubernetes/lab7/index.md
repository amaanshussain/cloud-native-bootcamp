---
tags:
  - Lab
  - Beginner
  - Workloads
  - Deployments
  - OpenShift
  - Kubernetes
---

# Rolling Updates Lab

<span class="lab-badge">25 min</span> <span class="lab-badge">Beginner</span>

In this lab you'll perform a zero-downtime rolling update on an OpenShift Deployment, record change causes in rollout history, simulate a failed image rollout, and execute an immediate rollback.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Rolling Updates and Rollbacks](../../../openshift/deployments/updates/index.md), [Deployments](../../../openshift/deployments/index.md), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Rolling Updates and Rollbacks](../../../openshift/deployments/updates/index.md) in a second tab before you start. This lab uses `jedi-deployment` and container `jedi-ws` where the documentation uses `my-deployment` and `nginx`.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with `kubectl`). See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Deploy the initial application

1. Create a project named `lab7` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create `lab7` instead of `dev`.
    - **Check:** `oc project` shows `Using project "lab7"`.

2. Save the initial manifest to `jedi-deployment.yaml` and deploy it with 3 replicas:

    ```yaml title="jedi-deployment.yaml"
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: jedi-deployment
      labels:
        app: jedi
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: jedi
      template:
        metadata:
          labels:
            app: jedi
        spec:
          containers:
            - name: jedi-ws
              image: quay.io/nginx/nginx-unprivileged:1.28
              ports:
                - containerPort: 8080
    ```

    - **Reference:** [Deployments, References](../../../openshift/deployments/index.md#references) shows applying manifests with `oc apply -f` and checking rollout status.
    - **Check:** `oc rollout status deployment/jedi-deployment` outputs `deployment "jedi-deployment" successfully rolled out`.

## Part 2: Perform a rolling update

3. Update the container image of `jedi-ws` in `jedi-deployment` to `quay.io/nginx/nginx-unprivileged:1.29` and record the change cause.
    - **Reference:** [Rolling Updates and Rollbacks, References](../../../openshift/deployments/updates/index.md#references) shows `oc set image` and annotating `kubernetes.io/change-cause`. Set the image for `jedi-ws=quay.io/nginx/nginx-unprivileged:1.29` and annotate with `"Upgrade to nginx 1.29"`.
    - **Check:** `oc get deployment jedi-deployment -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'` shows `quay.io/nginx/nginx-unprivileged:1.29`.

4. Watch the rollout progress until all 3 replicas are updated and ready.
    - **Reference:** [Rolling Updates and Rollbacks, References](../../../openshift/deployments/updates/index.md#references) shows `oc rollout status`.
    - **Check:** `oc rollout status deployment/jedi-deployment` finishes with `deployment "jedi-deployment" successfully rolled out`.

5. View the rollout history and verify that revision 2 records your change cause.
    - **Reference:** [Rolling Updates and Rollbacks, References](../../../openshift/deployments/updates/index.md#references) shows `oc rollout history`.
    - **Check:** `oc rollout history deployment/jedi-deployment` lists revisions 1 and 2 with change cause `Upgrade to nginx 1.29`.

## Part 3: Deploy a broken release and roll back

6. Update the image to `quay.io/nginx/nginx-unprivileged:9.99` (a non-existent tag) and observe how rolling updates protect existing traffic.
    - **Reference:** [Rolling Updates and Rollbacks, References](../../../openshift/deployments/updates/index.md#references). Set the image to tag `9.99`, annotate with `"Upgrade to nginx 9.99"`, and check pod status.
    - **Check:** `oc get pods -l app=jedi` shows 1 new pod in `ErrImagePull`/`ImagePullBackOff`, while the 3 working pods on version 1.29 remain running and serving traffic.

7. Roll back the deployment to the previous working revision and verify all 3 pods are healthy.
    - **Reference:** [Rolling Updates and Rollbacks, References](../../../openshift/deployments/updates/index.md#references) shows `oc rollout undo`.
    - **Check:** `oc rollout undo deployment/jedi-deployment` rolls back the deployment, `oc rollout status` succeeds, and `oc get deployment jedi-deployment -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'` returns `quay.io/nginx/nginx-unprivileged:1.29`.

## Cleanup

Delete the `lab7` project and all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. To delete the project:

```bash
oc delete project lab7
```

| Part | What it does |
| --- | --- |
| `oc delete project lab7` | Deletes the `lab7` project and all resources inside it |

## What you learned

- [x] Executing zero-downtime rolling updates with `oc set image`
- [x] Tracking deployment revisions with the `kubernetes.io/change-cause` annotation
- [x] Understanding how `maxUnavailable` and `maxSurge` prevent broken releases from dropping all running pods
- [x] Reverting failed rollouts with `oc rollout undo`
