---
tags:
  - Lab
  - Intermediate
  - Networking
  - Security
  - OpenShift
  - Kubernetes
---

# Network Policies Lab

<span class="lab-badge">30 min</span> <span class="lab-badge">Intermediate</span>

In this lab you'll isolate pods using Kubernetes `NetworkPolicy` resources to enforce zero-trust network segmentation, allowing ingress traffic only from clients with specific labels.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Network Policies](../../../openshift/services-networking/network-policies.md), [Labels, Selectors, and Annotations](../../../openshift/pods/tagging/index.md), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the manifests and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Network Policies](../../../openshift/services-networking/network-policies.md) in a second tab before you start. Once a Pod is selected by any policy specifying `policyTypes: [Ingress]`, all ingress traffic not explicitly matched is denied.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with a CNI that supports NetworkPolicies). See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Deploy test pods and verify unisolated traffic

1. Create a project named `lab10` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create `lab10` instead of `dev`.
    - **Check:** `oc project` shows `Using project "lab10"`.

2. Deploy the target secure pod (`network-policy-secure-pod` with label `app: secure-app`) and an unlabeled client pod (`network-policy-client-pod`):

    ```bash
    oc apply -f - <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: network-policy-secure-pod
      labels:
        app: secure-app
    spec:
      containers:
        - name: nginx
          image: quay.io/nginx/nginx-unprivileged:1.29
          ports:
            - containerPort: 8080
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: network-policy-client-pod
    spec:
      containers:
        - name: client
          image: registry.access.redhat.com/ubi9/ubi-minimal
          command: ["sleep", "infinity"]
    EOF
    oc wait --for=condition=Ready pod/network-policy-secure-pod pod/network-policy-client-pod
    ```

    - **Check:** both pods reach `Running` and `1/1` Ready.

3. Verify that the client pod can reach the secure pod before any NetworkPolicy is applied.
    - **Reference:** [Debugging Applications, Commands to debug with](../../../openshift/pods/troubleshooting/index.md#commands-to-debug-with) shows retrieving pod IPs and testing connectivity. Query the secure pod IP:

    ```bash
    SECURE_POD_IP=$(oc get pod network-policy-secure-pod -o jsonpath='{.status.podIP}')
    oc exec network-policy-client-pod -- curl -s --max-time 5 "http://${SECURE_POD_IP}:8080" | grep "<title>"
    ```

    - **Check:** output returns `<title>Welcome to nginx!</title>`.

## Part 2: Enforce Ingress isolation with a NetworkPolicy

Create a manifest file named `secure-app-policy.yaml` meeting these requirements:

- Policy name: `secure-app-policy`
- Pod selector: `app: secure-app`
- Policy type: `Ingress`
- Ingress rule: allow from pods with label `allow-access: "true"`

4. Write and apply `secure-app-policy.yaml`.
    - **Reference:** [Network Policies, Allow Traffic from Specific Pods](../../../openshift/services-networking/network-policies.md#references) shows configuring ingress from labelled pods using `ingress.from.podSelector.matchLabels`.
    - **Check:** `oc apply -f secure-app-policy.yaml` creates the policy, and `oc describe networkpolicy secure-app-policy` lists `Allowing ingress traffic: ... PodSelector: allow-access=true`.

## Part 3: Validate traffic restrictions and dynamic labeling

5. Test connectivity from the unlabeled client pod and confirm traffic is blocked.
    - **Reference:** [Network Policies, References](../../../openshift/services-networking/network-policies.md#references). Attempt to curl the secure pod:

    ```bash
    oc exec network-policy-client-pod -- curl -s --max-time 5 "http://${SECURE_POD_IP}:8080" || echo "Connection blocked"
    ```

    - **Check:** the command times out and prints `Connection blocked`.

6. Dynamically add the label `allow-access=true` to `network-policy-client-pod` and re-test connectivity.
    - **Reference:** [Labels, Selectors, and Annotations, References](../../../openshift/pods/tagging/index.md#references) shows updating labels with `oc label pod <pod-name> <key>=<value>`. Label the client pod and send an HTTP request:

    ```bash
    oc label pod network-policy-client-pod allow-access=true
    oc exec network-policy-client-pod -- curl -s --max-time 5 "http://${SECURE_POD_IP}:8080" | grep "<title>"
    ```

    - **Check:** `curl` succeeds and returns `<title>Welcome to nginx!</title>`.

## Cleanup

Delete the `lab10` project and all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. To delete the project:

```bash
oc delete project lab10
```

| Part | What it does |
| --- | --- |
| `oc delete project lab10` | Deletes the `lab10` project and all NetworkPolicies and Pods inside it |

## What you learned

- [x] Understanding default non-isolated network behavior vs policy-enforced isolation
- [x] Defining `NetworkPolicy` resources targeting specific workloads via `podSelector`
- [x] Configuring ingress rules that permit traffic solely from matching client labels
- [x] Observing real-time policy enforcement and dynamic label admittance
