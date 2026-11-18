---
tags:
  - Lab
  - Beginner
  - Pods
  - Observability
  - OpenShift
  - Kubernetes
---

# Probes Lab

<span class="lab-badge">20 min</span> <span class="lab-badge">Beginner</span>

In this lab you'll configure HTTP liveness and readiness probes on a container to automatically detect unhealthy states, restart failed processes, and hold off traffic until the application is ready.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Health and Monitoring](../../../openshift/pods/health-checks/index.md), [Health Probe Pattern](../../../openshift/orchestrationConcepts.md#health-probe-pattern), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the manifest and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Health and Monitoring](../../../openshift/pods/health-checks/index.md) in a second tab before you start. This lab checks path `/healthz` on port `8080` with 5-second initial delays and periods, where the documentation example checks path `/` on port `8080` with 10-second periods.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with `kubectl`). See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Create a project

1. Create a new project named `lab2` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create a project named `lab2` instead of `dev`.
    - **Check:** `oc project` shows `Using project "lab2"`.

## Part 2: Configure Liveness and Readiness Probes

The `energy-shield` service has two problems in production:
1. It enters an unhealthy state after running for about 10 seconds and returns HTTP 500 errors without crashing, so OpenShift never restarts it on its own.
2. It takes a few seconds after startup before it can serve requests.

Save the base manifest below as `energy-shield-service.yaml`:

```yaml title="energy-shield-service.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: energy-shield-service
spec:
  containers:
    - name: energy-shield
      image: registry.k8s.io/e2e-test-images/agnhost:2.56
      args: ["liveness"]
      ports:
        - containerPort: 8080
```

2. Add a **liveness probe** to the container using an HTTP GET check against path `/healthz` on port `8080`, with an initial delay of `5` seconds and a period of `5` seconds.
    - **Reference:** [Health and Monitoring, References](../../../openshift/pods/health-checks/index.md#references) shows configuring a `livenessProbe` with `httpGet`, `initialDelaySeconds`, and `periodSeconds`. Use path `/healthz`, port `8080`, and set both `initialDelaySeconds: 5` and `periodSeconds: 5`.
    - **Check:** your container spec has a `livenessProbe` block with `httpGet.path: /healthz` and `httpGet.port: 8080`.

3. Add a **readiness probe** to the container using an HTTP GET check against path `/healthz` on port `8080`, with an initial delay of `5` seconds and a period of `5` seconds.
    - **Reference:** [Health and Monitoring, References](../../../openshift/pods/health-checks/index.md#references) shows configuring a `readinessProbe` with `httpGet` and `periodSeconds`. Add `initialDelaySeconds: 5` and `periodSeconds: 5`.
    - **Check:** your container spec has a `readinessProbe` block with `httpGet.path: /healthz` and `httpGet.port: 8080`.

## Part 3: Deploy and observe probe behavior

4. Create the Pod in your cluster from `energy-shield-service.yaml`.
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references) shows how to apply manifests with `oc apply -f`.
    - **Check:** the command outputs `pod/energy-shield-service created`.

5. Watch the Pod status in real time for about 45–60 seconds.
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references) lists pod status. The documentation doesn't show streaming watch output, so use the `-w` flag:

    ```bash
    oc get pod energy-shield-service -w
    ```

    | Part | What it does |
    | --- | --- |
    | `oc get pod energy-shield-service` | Queries the status of `energy-shield-service` |
    | `-w` | Watches for changes and prints new status lines as they occur |

    ```text title="Expected output"
    NAME                    READY   STATUS    RESTARTS      AGE
    energy-shield-service   0/1     Running   0             5s
    energy-shield-service   1/1     Running   0             9s
    energy-shield-service   0/1     Running   0             25s
    energy-shield-service   0/1     Running   1 (3s ago)    29s
    energy-shield-service   1/1     Running   1 (7s ago)    33s
    ```

    - **Check:** `READY` transitions from `0/1` to `1/1` (readiness probe succeeds), then drops to `0/1` when `/healthz` fails, followed by an increase in `RESTARTS` (liveness probe triggers restart).

6. Inspect the Pod events to confirm the liveness probe failure and container restart.
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references) shows `oc describe pod`.
    - **Check:** `oc describe pod energy-shield-service` lists events containing `Liveness probe failed: HTTP probe failed with statuscode: 500` and `Container energy-shield failed liveness probe, will be restarted`.

## Cleanup

Delete the `lab2` project and all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. To delete the project:

```bash
oc delete project lab2
```

| Part | What it does |
| --- | --- |
| `oc delete project lab2` | Deletes the `lab2` project and all resources inside it |

## What you learned

- [x] The difference between liveness and readiness probes
- [x] Configuring HTTP GET probes with `initialDelaySeconds` and `periodSeconds`
- [x] Observing how readiness probes control endpoint traffic readiness (`READY 0/1` vs `1/1`)
- [x] Observing how liveness probes automatically self-heal and restart unhealthy containers
