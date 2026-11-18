---
tags:
  - Lab
  - Intermediate
  - Pods
  - OpenShift
  - Kubernetes
---

# Multi-Container Pods Lab

<span class="lab-badge">30 min</span> <span class="lab-badge">Intermediate</span>

In this lab you'll implement the **ambassador pattern** using a multi-container Pod in OpenShift. An ambassador container running NGINX proxies incoming traffic on standard port `8080` to a legacy application listening on port `8989` over `127.0.0.1`.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Multi-Containers Pod](../../../openshift/pods/multi-container/index.md), [Config Maps](../../../openshift/configuration/config-map/index.md), [Pod Basics](../../../openshift/pods/index.md), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the manifest and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Multi-Containers Pod](../../../openshift/pods/multi-container/index.md) and [Config Maps](../../../openshift/configuration/config-map/index.md) in a second tab before you start. Because containers in the same Pod share the network namespace, the ambassador container can reach the legacy container on `127.0.0.1:8989`.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with `kubectl`). See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Create a project

1. Create a new project named `lab4` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create `lab4` instead of `dev`.
    - **Check:** `oc project` shows `Using project "lab4"`.

## Part 2: Create the Ambassador ConfigMap

2. Create a ConfigMap named `vader-service-ambassador-config` containing NGINX reverse-proxy configuration under the key `default.conf`.
    - **Reference:** [Config Maps, References](../../../openshift/configuration/config-map/index.md#references) shows how to define ConfigMaps. Under `data.default.conf`, configure NGINX to listen on port `8080` and proxy requests to `http://127.0.0.1:8989`:

    ```nginx
    server {
        listen 8080;
        location / {
            proxy_pass http://127.0.0.1:8989;
        }
    }
    ```

    - **Check:** `oc get configmap vader-service-ambassador-config` shows the ConfigMap with 1 data entry.

## Part 3: Create the Multi-Container Pod

Create a manifest file named `vader-service.yaml` containing the ConfigMap and the Pod.

3. Define the multi-container Pod named `vader-service` with both the legacy container and the ambassador container.
    - **Reference:** [Multi-Containers Pod, Sharing files through a volume](../../../openshift/pods/multi-container/index.md#sharing-files-through-a-volume) shows how multiple containers are declared in `spec.containers`. [Config Maps, All keys as files in a volume](../../../openshift/configuration/config-map/index.md#references) shows mounting a ConfigMap as a volume.
    - Your Pod definition must include:
        - A container named `millennium-falcon` running `registry.k8s.io/e2e-test-images/agnhost:2.56` with `args: ["netexec", "--http-port=8989"]`.
        - A container named `nginx-ambassador` running `quay.io/nginx/nginx-unprivileged:1.29` with `containerPort: 8080`.
        - A volume referencing `vader-service-ambassador-config` mounted into `nginx-ambassador` at `/etc/nginx/conf.d`.
    - **Check:** your manifest specifies two containers in `spec.containers`, mounts the ConfigMap at `/etc/nginx/conf.d`, and exposes port `8080`.

4. Apply `vader-service.yaml` and verify that both containers in the Pod start successfully (`2/2` Ready).
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references) shows applying manifests with `oc apply -f`.
    - **Check:** `oc get pod vader-service` shows `READY 2/2` and `STATUS Running`.

## Part 4: Test the Ambassador proxy

5. Deploy a client Pod and send an HTTP request to `vader-service` on port `8080`.
    - **Reference:** [Multi-Containers Pod, Sharing files through a volume](../../../openshift/pods/multi-container/index.md#sharing-files-through-a-volume) shows executing commands inside pods. The documentation doesn't cover launching a test client pod, so run these commands:

    ```bash
    oc apply -f - <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: client
    spec:
      containers:
        - name: client
          image: registry.access.redhat.com/ubi9/ubi-minimal
          command: ["sleep", "infinity"]
    EOF
    oc wait --for=condition=Ready pod/client
    oc exec client -- curl -s "$(oc get pod vader-service -o jsonpath='{.status.podIP}'):8080/echo?msg=The+hyperdrive+needs+repair"
    ```

    | Part | What it does |
    | --- | --- |
    | `oc apply -f - <<'EOF' ...` | Deploys an interactive client Pod |
    | `oc wait --for=condition=Ready` | Waits until the client Pod is running |
    | `oc exec client -- curl ...` | Makes an HTTP call through the NGINX ambassador on port 8080 |

    ```text title="Expected output"
    The hyperdrive needs repair
    ```

    - **Check:** the output returns `The hyperdrive needs repair`, proving that NGINX successfully forwarded the request on port 8080 to the legacy application on port 8989.

## Cleanup

Delete the `lab4` project and all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. To delete the project:

```bash
oc delete project lab4
```

| Part | What it does |
| --- | --- |
| `oc delete project lab4` | Deletes the `lab4` project and all resources inside it |

## What you learned

- [x] Designing multi-container Pods that cooperate over `localhost` (`127.0.0.1`)
- [x] Implementing the Ambassador pattern to adapt legacy network ports without modifying source code
- [x] Mounting ConfigMap files as configuration directory volumes into containers
- [x] Verifying multi-container readiness (`2/2`) and inter-container proxying
