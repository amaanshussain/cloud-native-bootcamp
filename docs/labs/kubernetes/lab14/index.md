---
tags:
  - Lab
  - Intermediate
  - Virtualization
  - KubeVirt
  - OpenShift
  - Kubernetes
---

# Virtual Machines Lab

<span class="lab-badge">45 min</span> <span class="lab-badge">Intermediate</span> <span class="lab-badge">OpenShift Virtualization</span>

A legacy "hello" web service cannot be containerized yet: it must run on a full Fedora operating system. In this lab, you run it as a **virtual machine** on OpenShift alongside containers, access its serial console, manage its lifecycle with `virtctl`, and expose it to other containers and external users using standard Kubernetes Services and OpenShift Routes.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [OpenShift Virtualization](../../../openshift/virtualization/index.md), [Services](../../../openshift/services-networking/services.md), [Routes](../../../openshift/services-networking/routes.md), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or manifest syntax you need. **You build the manifests and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command or manifest and explains what it does.

!!! tip "Keep the documentation open"
    Open [OpenShift Virtualization](../../../openshift/virtualization/index.md) in a second tab before you start. It explains KubeVirt architecture (`VirtualMachine`, `VirtualMachineInstance`, `virt-launcher`), `virtctl` commands, storage types, and networking.

## Prerequisites

- An OpenShift cluster with the **OpenShift Virtualization** operator installed, and worker nodes that support virtualization (bare metal, or cloud instances with virtualization enabled). See [Lab Environments](../../../lab-environments.md#virtualization).
- The `virtctl` CLI installed. In the web console, open the **?** menu > **Command Line Tools** and download **virtctl**, or on macOS run `brew install virtctl`.
- Create a project named `vm-lab`:

```bash
oc new-project vm-lab
```

## Part 1: Define and Deploy a VirtualMachine

1. Create a `VirtualMachine` manifest named `fedora-vm.yaml`.
    - **Reference:** [OpenShift Virtualization, Key concepts](../../../openshift/virtualization/index.md#key-concepts) describes instance types (`u1.small`), OS preferences (`fedora`), `containerDisk`, and cloud-init configuration.
    - Sizing: `u1.small` instance type (1 vCPU, 2 GiB RAM) with `fedora` preference.
    - Template labels: `app: fedora-vm` (used by Kubernetes Services).
    - Disk: `containerDisk` with image `quay.io/containerdisks/fedora:43`.
    - Cloud-init: create user `fedora` with password `bootcamp`, write `/srv/www/index.html`, and start a systemd web service on port `8080`.

    ```yaml title="fedora-vm.yaml"
    apiVersion: kubevirt.io/v1
    kind: VirtualMachine
    metadata:
      name: fedora-vm
    spec:
      runStrategy: Always
      instancetype:
        kind: VirtualMachineClusterInstancetype
        name: u1.small
      preference:
        kind: VirtualMachineClusterPreference
        name: fedora
      template:
        metadata:
          labels:
            app: fedora-vm
        spec:
          domain:
            devices: {}
          volumes:
            - name: rootdisk
              containerDisk:
                image: quay.io/containerdisks/fedora:43
            - name: cloudinitdisk
              cloudInitNoCloud:
                userData: |
                  #cloud-config
                  user: fedora
                  password: bootcamp
                  chpasswd: { expire: false }
                  write_files:
                    - path: /srv/www/index.html
                      content: |
                        Hello from a virtual machine running on Kubernetes!
                    - path: /etc/systemd/system/hello-web.service
                      content: |
                        [Unit]
                        Description=Hello web server
                        After=network-online.target
                        [Service]
                        ExecStart=/usr/bin/python3 -m http.server 8080 --directory /srv/www
                        Restart=always
                        [Install]
                        WantedBy=multi-user.target
                  runcmd:
                    - systemctl daemon-reload
                    - systemctl enable --now hello-web.service
    ```

    !!! note "ARM clusters"
        On a cluster with ARM (aarch64) worker nodes, use the `fedora.arm64` preference.

2. Apply the VM manifest and inspect the resources created.
    - **Reference:** [OpenShift Virtualization, Working with VMs](../../../openshift/virtualization/index.md#working-with-vms) shows `oc get vm,vmi`.
    - Apply and watch:

    ```bash
    oc apply -f fedora-vm.yaml
    oc get vm,vmi
    oc get pods -l app=fedora-vm -w
    ```

    | Object | Purpose |
    | --- | --- |
    | `VirtualMachine` (VM) | Desired configuration and lifecycle state (`runStrategy: Always`). Persists when stopped. |
    | `VirtualMachineInstance` (VMI) | Ephemeral runtime instance created when the VM runs. Contains assigned IP and node name. |
    | `virt-launcher` pod | Kubernetes pod hosting QEMU/KVM. Inherits template labels (`app: fedora-vm`) to route Service traffic. |

    - **Check:** `oc get vm,vmi` shows `fedora-vm` with Status/Phase `Running`, and the `virt-launcher` pod reaches status `Running`.

## Part 2: Access the VM Console

3. Open the VM serial console and verify the web service.
    - **Reference:** [OpenShift Virtualization, Working with VMs](../../../openshift/virtualization/index.md#working-with-vms) shows `virtctl console`.

    ```bash
    virtctl console fedora-vm
    ```

    - Log in with username `fedora` and password `bootcamp`.
    - Inside the guest OS, check the web server:

    ```bash
    systemctl status hello-web --no-pager
    curl localhost:8080
    ```

    - Exit the serial console with ++ctrl+bracket-right++.
    - **Check:** `curl localhost:8080` returns `Hello from a virtual machine running on Kubernetes!`.

## Part 3: Manage VM Lifecycle and Ephemeral Disks

4. Verify that `containerDisk` volumes are ephemeral across restarts.
    - **Reference:** [OpenShift Virtualization, Key concepts](../../../openshift/virtualization/index.md#key-concepts) explains that `containerDisk` resets on restart. [Working with VMs](../../../openshift/virtualization/index.md#working-with-vms) shows `virtctl restart`.
    - Connect to the VM console, create a test file: `echo "I was here" > ~/note.txt`, and exit the console.
    - Restart the VM:

    ```bash
    virtctl restart fedora-vm
    oc get vmi -w
    ```

    - Reconnect with `virtctl console fedora-vm`, log in, and check for the file:

    ```bash
    ls ~/note.txt
    ```

    - **Check:** `ls ~/note.txt` returns `No such file or directory` because the root filesystem resets to the original container image.

5. Stop and restart the VM using `virtctl`.
    - **Reference:** [OpenShift Virtualization, Working with VMs](../../../openshift/virtualization/index.md#working-with-vms) shows `virtctl stop` and `virtctl start`.

    ```bash
    virtctl stop fedora-vm
    oc get vm,vmi,pods
    virtctl start fedora-vm
    oc get vm,vmi
    ```

    - **Check:** when stopped, `oc get vm` shows status `Stopped` (`runStrategy: Halted`), while the VMI and `virt-launcher` pod are removed. When started, a new VMI and pod are provisioned.

## Part 4: Expose the VM via Service and Route

6. Create a Kubernetes Service to route traffic from container pods to the VM web server.
    - **Reference:** [OpenShift Virtualization, Networking](../../../openshift/virtualization/index.md#networking) explains masquerade binding and Services. [Services, References](../../../openshift/services-networking/services.md#references) shows Service syntax.
    - Target: selector `app: fedora-vm`, port `80`, targetPort `8080`.

    ```yaml title="fedora-vm-service.yaml"
    apiVersion: v1
    kind: Service
    metadata:
      name: fedora-vm
    spec:
      selector:
        app: fedora-vm
      ports:
        - port: 80
          targetPort: 8080
    ```

    - Apply the Service and test access from a client pod:

    ```bash
    oc apply -f fedora-vm-service.yaml
    oc run client --image=registry.access.redhat.com/ubi9/ubi-minimal -- sleep infinity
    oc wait --for=condition=Ready pod/client
    oc exec client -- curl -s fedora-vm
    ```

    - **Check:** `oc exec client -- curl -s fedora-vm` returns `Hello from a virtual machine running on Kubernetes!`.

7. Expose the Service externally with an OpenShift Route.
    - **Reference:** [Routes, Commands](../../../openshift/services-networking/routes.md#commands) shows `oc expose service`.

    ```bash
    oc expose service fedora-vm
    curl "http://$(oc get route fedora-vm -o jsonpath='{.spec.host}')"
    ```

    - **Check:** the Route URL returns `Hello from a virtual machine running on Kubernetes!`.

## Part 5 (optional): Create a VM with Persistent Golden Image Storage

8. Provision a VM with persistent disk storage cloned from OpenShift's built-in RHEL 9 golden image.
    - **Reference:** [OpenShift Virtualization, Working with VMs](../../../openshift/virtualization/index.md#working-with-vms) shows `virtctl create vm` with `--volume-import`. [Storage](../../../openshift/virtualization/index.md#storage) explains DataVolumes and DataSource golden images.

    ```bash
    virtctl create vm --name rhel9-vm \
      --instancetype u1.small --preference rhel.9 \
      --volume-import type:ds,src:openshift-virtualization-os-images/rhel9,size:30Gi \
      --user cloud-user --ssh-key "$(cat ~/.ssh/id_ed25519.pub)" | oc apply -f -

    oc get datavolume -w
    virtctl ssh cloud-user@vm/rhel9-vm
    ```

    - **Check:** `oc get datavolume` transitions to `Succeeded`, and `virtctl ssh cloud-user@vm/rhel9-vm` connects to the persistent guest.

## Cleanup

Delete the project to remove the VM, Service, Route, and client pod.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references).

```bash
oc delete project vm-lab
```

| Part | What it does |
| --- | --- |
| `oc delete project vm-lab` | Deletes the namespace and all associated VirtualMachines, VMIs, pods, Services, and Routes |

## What you learned

- [x] Defining KubeVirt `VirtualMachine` resources using instance types, preferences, containerDisks, and cloud-init
- [x] Understanding the relationship between `VirtualMachine`, `VirtualMachineInstance`, and `virt-launcher` pods
- [x] Accessing VM serial consoles and managing VM lifecycle with `virtctl console`, `restart`, `stop`, and `start`
- [x] Differentiating between ephemeral `containerDisk` images and persistent `DataVolume` storage
- [x] Exposing VM workloads to containerized microservices and external clients using Services and Routes
