---
tags:
  - Lab
  - Beginner
  - Networking
  - Services
  - OpenShift
  - Kubernetes
---

# Services Lab

<span class="lab-badge">25 min</span> <span class="lab-badge">Beginner</span>

In this lab you'll create Kubernetes Services to expose Deployments: an internal `ClusterIP` Service for pod-to-pod communication via DNS and a `NodePort` Service for external node access.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Services](../../../openshift/services-networking/services.md), [Networking Overview](../../../openshift/services-networking/index.md), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the manifests and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Services](../../../openshift/services-networking/services.md) and [Networking Overview](../../../openshift/services-networking/index.md) in a second tab before you start.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with `kubectl`). See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Deploy sample workloads

1. Create a project named `lab9` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create `lab9` instead of `dev`.
    - **Check:** `oc project` shows `Using project "lab9"`.

2. Deploy the `jedi-deployment` and `yoda-deployment` applications:

    ```bash
    oc apply -f - <<'EOF'
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: jedi-deployment
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: jedi
      template:
        metadata:
          labels:
            app: jedi
        spec:
          containers:
            - name: jedi
              image: quay.io/nginx/nginx-unprivileged:1.29
              ports:
                - containerPort: 8080
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: yoda-deployment
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: yoda
      template:
        metadata:
          labels:
            app: yoda
        spec:
          containers:
            - name: yoda
              image: quay.io/nginx/nginx-unprivileged:1.29
              ports:
                - containerPort: 8080
    EOF
    ```

    - **Check:** `oc get deployments` shows both `jedi-deployment` and `yoda-deployment` with `2/2` ready replicas.

## Part 2: Create Services

Create a manifest file named `services.yaml` with the two Services described below.

3. Create a `NodePort` Service named `jedi-svc` that exposes `jedi-deployment` on port `80`, routing to `targetPort: 8080`.
    - **Reference:** [Services, References](../../../openshift/services-networking/services.md#references) shows Service syntax and selecting pods with `selector.app`. Set `type: NodePort`, `port: 80`, and `targetPort: 8080`.
    - **Check:** your manifest defines `kind: Service`, `name: jedi-svc`, `type: NodePort`, and `selector.app: jedi`.

4. Create a `ClusterIP` Service named `yoda-svc` that exposes `yoda-deployment` on port `80`, routing to `targetPort: 8080`.
    - **Reference:** [Services, References](../../../openshift/services-networking/services.md#references) shows `ClusterIP` Service configuration.
    - **Check:** your manifest defines `kind: Service`, `name: yoda-svc`, `type: ClusterIP`, and `selector.app: yoda`.

5. Apply `services.yaml` and verify that both Services receive endpoints.
    - **Reference:** [Services, References](../../../openshift/services-networking/services.md#references) shows `oc get svc` and endpoint inspection.
    - **Check:** `oc get svc` lists `jedi-svc` (type NodePort) and `yoda-svc` (type ClusterIP), and `oc get endpointslices` lists 2 endpoints for each.

## Part 3: Test internal and external service routing

6. Test pod-to-service DNS resolution inside the cluster by querying `yoda-svc` from a `jedi-deployment` pod.
    - **Reference:** [Networking Overview, Service discovery with DNS](../../../openshift/services-networking/index.md#service-discovery-with-dns) explains cluster-internal DNS names. Execute `curl` from `deploy/jedi-deployment`:

    ```bash
    oc exec deploy/jedi-deployment -- curl -s -o /dev/null -w '%{http_code}\n' http://yoda-svc
    ```

    - **Check:** the command returns HTTP status `200`.

7. Test NodePort connectivity on the cluster node's IP and allocated NodePort.
    - **Reference:** [Networking Overview, Ways to expose an application](../../../openshift/services-networking/index.md#ways-to-expose-an-application) explains NodePort routing. Run this test:

    ```bash
    NODE_PORT=$(oc get service jedi-svc -o jsonpath='{.spec.ports[0].nodePort}')
    NODE_IP=$(oc get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
    oc exec deploy/yoda-deployment -- curl -s -o /dev/null -w '%{http_code}\n' "http://${NODE_IP}:${NODE_PORT}"
    ```

    - **Check:** the command returns HTTP status `200`.

## Cleanup

Delete the `lab9` project and all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. To delete the project:

```bash
oc delete project lab9
```

| Part | What it does |
| --- | --- |
| `oc delete project lab9` | Deletes the `lab9` project, Deployments, and Services |

## What you learned

- [x] Defining `ClusterIP` Services for internal microservice communication and DNS resolution
- [x] Defining `NodePort` Services for port-based cluster access across nodes
- [x] Matching Service selectors to Deployment Pod labels and mapping `port` to container `targetPort`
- [x] Inspecting EndpointSlices and validating Service load balancing
