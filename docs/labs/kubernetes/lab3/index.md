---
tags:
  - Lab
  - Intermediate
  - Pods
  - Troubleshooting
  - OpenShift
  - Kubernetes
---

# Debugging Lab

<span class="lab-badge">30 min</span> <span class="lab-badge">Intermediate</span>

In this lab you'll troubleshoot and repair a broken Deployment and Service in OpenShift by inspecting pod events, container logs, probe configurations, and service selectors.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Debugging Applications](../../../openshift/pods/troubleshooting/index.md), [Health and Monitoring](../../../openshift/pods/health-checks/index.md), [Services](../../../openshift/services-networking/services.md), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the commands and fixes yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Debugging Applications](../../../openshift/pods/troubleshooting/index.md) and [Services](../../../openshift/services-networking/services.md) in a second tab before you start. The app listens on port `8080` and exposes its health check at `/healthz`.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with `kubectl`). See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Deploy the broken application

1. Create a project named `vader` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create `vader` instead of `dev`.
    - **Check:** `oc project` shows `Using project "vader"`.

2. Apply the broken manifest containing the `hyper-drive` Deployment and Service:

    ```bash
    oc apply -f - <<'EOF'
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hyper-drive
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: hyper-drive
      template:
        metadata:
          labels:
            app: hyper-drive
        spec:
          containers:
            - name: hyper-drive
              image: registry.k8s.io/e2e-test-images/agnhost:2.56
              args: ["netexec", "--http-port=8080"]
              ports:
                - containerPort: 8080
              livenessProbe:
                tcpSocket:
                  port: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: hyper-drive
    spec:
      selector:
        run: hyper-drive
      ports:
        - protocol: TCP
          port: 80
    EOF
    ```

    - **Check:** `oc get deployment,svc` shows `hyper-drive` Deployment and Service created.

## Part 2: Investigate and diagnose the issues

3. Inspect the Pod status and events to discover why the Pods are restarting.
    - **Reference:** [Debugging Applications, Commands to debug with](../../../openshift/pods/troubleshooting/index.md#commands-to-debug-with) shows how to inspect pod status and events.
    - **Check:** `oc describe pod -l app=hyper-drive` shows event `Liveness probe failed: dial tcp ...:80: connect: connection refused`.

4. Save the logs of one of the previously crashed containers to a file named `broken-pod-logs.log`.
    - **Reference:** [Container Logging, References](../../../openshift/pods/health-checks/index.md#references) shows retrieving logs from crashed containers using `--previous`.
    - **Check:** `cat broken-pod-logs.log` contains startup output from the `agnhost netexec` process.

5. Compare the Service's selector with the Pods' labels and check the Service endpoints.
    - **Reference:** [Debugging Applications, Commands to debug with](../../../openshift/pods/troubleshooting/index.md#commands-to-debug-with) shows how to check pod labels and inspect `endpointslices`.
    - **Check:** `oc describe service hyper-drive` lists selector `run=hyper-drive`, while `oc get pods --show-labels` shows labels `app=hyper-drive`. `oc get endpointslices -l kubernetes.io/service-name=hyper-drive` shows `0` endpoints.

## Part 3: Fix the Deployment and Service

6. Fix the Deployment liveness probe so it checks HTTP path `/healthz` on container port `8080`.
    - **Reference:** [Debugging Applications, Commands to debug with](../../../openshift/pods/troubleshooting/index.md#commands-to-debug-with) shows `oc edit deployment`. [Health and Monitoring, References](../../../openshift/pods/health-checks/index.md#references) shows `livenessProbe.httpGet` syntax.
    - **Check:** all 3 `hyper-drive` pods reach status `Running` (`1/1` ready) and restart counts stop incrementing.

7. Fix the Service selector to match `app: hyper-drive` and configure `targetPort: 8080` (or `targetPort: 8080` on port `80`).
    - **Reference:** [Debugging Applications, Commands to debug with](../../../openshift/pods/troubleshooting/index.md#commands-to-debug-with) shows `oc edit service`. [Services, References](../../../openshift/services-networking/services.md#references) shows Service selector and `targetPort` definitions.
    - **Check:** `oc get endpointslices -l kubernetes.io/service-name=hyper-drive` lists all 3 pod IP addresses on port `8080`.

8. Run an ephemeral client pod and verify the Service successfully answers requests and load-balances across the pods.
    - **Reference:** [Debugging Applications, Commands to debug with](../../../openshift/pods/troubleshooting/index.md#commands-to-debug-with). The documentation doesn't show launching a test client pod, so run these commands:

    ```bash
    oc run client --image=registry.access.redhat.com/ubi9/ubi-minimal -- sleep infinity
    oc wait --for=condition=Ready pod/client
    oc exec client -- curl -s --max-time 5 hyper-drive/hostname
    ```

    | Part | What it does |
    | --- | --- |
    | `oc run client ...` | Starts a lightweight client Pod |
    | `oc wait --for=condition=Ready` | Pauses until the client Pod is running and ready |
    | `oc exec client -- curl ...` | Runs `curl` inside the cluster against `hyper-drive/hostname` |

    - **Check:** the command outputs the hostname of one of your `hyper-drive` pods.

## Cleanup

Delete the `vader` project and all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. To delete the project:

```bash
oc delete project vader
```

| Part | What it does |
| --- | --- |
| `oc delete project vader` | Deletes the `vader` project and all resources inside it |

## What you learned

- [x] Using `oc describe`, events, and `oc logs --previous` to identify probe failures
- [x] Diagnosing Service endpoint issues caused by mismatched label selectors
- [x] Configuring `targetPort` so incoming traffic routes to the correct container port
- [x] Verifying cluster internal service discovery and load balancing
