---
tags:
  - Solution
  - Containers
---

# Lab 1 Solution - Containers Lab


## Part 1: Verify your setup

```bash

podman version
podman run --rm quay.io/podman/hello
podman images

```

```text title="Expected output"
user@server ~ % podman version
Client:        Podman Engine
Version:       5.8.2
API Version:   5.8.2
Go Version:    go1.26.2
Built:         Mon Apr 13 17:06:58 2026
Build Origin:  brew
OS/Arch:       darwin/arm64
...
user@server ~ % podman run --rm quay.io/podman/hello
Trying to pull quay.io/podman/hello:latest...
...
!... Hello Podman World ...!

user@server ~ % podman images
REPOSITORY                            TAG         IMAGE ID      CREATED            SIZE
quay.io/podman/hello                  latest      83fc7ce1224f  2 years ago        580 kB
```

!!! tip
    `--rm` deletes the container as soon as it exits, it is not required but helpful when testing images in succession.

## Part 2: Build an image

Run the following from inside the `greeting` directory.

```bash

podman build -t greeting:v1.0.0 .
podman images

```

```text title="Expected output"
user@server greeting % podman build -t greeting:v1.0.0 .
[1/2] STEP 1/3: FROM registry.access.redhat.com/ubi9/go-toolset AS builder
...
[2/2] STEP 1/5: FROM registry.access.redhat.com/ubi9/ubi-micro
...
Successfully tagged localhost/greeting:v1.0.0

user@server greeting % podman images
REPOSITORY                                  TAG         IMAGE ID      CREATED         SIZE
localhost/greeting                          v1.0.0      b598ef58cead  10 seconds ago  32.9 MB
registry.access.redhat.com/ubi9/go-toolset  latest      3e4571255899  15 hours ago    1.16 GB
registry.access.redhat.com/ubi9/ubi-micro   latest      954ff113830b  21 hours ago    24.7 MB
quay.io/podman/hello                        latest      83fc7ce1224f  2 years ago     580 kB
```

!!! tip
    `-t` tags the image `name:version`, and the `.` is the build context, the directory containing the `Containerfile`. The final `greeting` image is about 33 MB, compared to 1.16 GB for `go-toolset`, because the multi-stage build leaves the Go toolchain behind and keeps only the compiled binary.

## Part 3: Run and inspect the container

```bash

podman run -d --name greeting -p 8080:8080 greeting:v1.0.0
curl "localhost:8080/greeting?name=John"
podman run -d --name greeting -p 8081:8080 greeting:v1.0.0
podman ps
podman inspect greeting
podman inspect greeting --format '{{.State.Status}} {{.NetworkSettings.IPAddress}}'
podman logs greeting
podman exec greeting id
podman stop greeting
podman rm greeting
podman run -d --name greeting -p 8080:8080 -e GREETING="Hello from my container" greeting:v1.0.0
curl "localhost:8080/greeting?name=John"

```

```text title="Expected output"
user@server greeting % podman run -d --name greeting -p 8080:8080 greeting:v1.0.0
3c1f0a7d9b52...

user@server greeting % curl "localhost:8080/greeting?name=John"
{"message":"Welcome to the Cloud Native Bootcamp!","name":"John"}

user@server greeting % podman run -d --name greeting -p 8081:8080 greeting:v1.0.0
Error: creating container storage: the container name "greeting" is already in use by 3c1f.... You have to remove that container to be able to reuse that name: that name is already in use

user@server greeting % podman ps
CONTAINER ID  IMAGE                      COMMAND     CREATED        STATUS        PORTS                   NAMES
3c1f0a7d9b52  localhost/greeting:v1.0.0  greeting    8 seconds ago  Up 8 seconds  0.0.0.0:8080->8080/tcp  greeting

user@server greeting % podman inspect greeting --format '{{.State.Status}} {{.NetworkSettings.IPAddress}}'
running 10.88.0.2

user@server greeting % podman logs greeting
2026/09/24 23:29:00 greeting service listening on :8080

user@server greeting % podman exec greeting id
uid=1001(1001) gid=0(root) groups=0(root)

user@server greeting % curl "localhost:8080/greeting?name=John"
{"message":"Hello from my container","name":"John"}
```

!!! tip
    Container names must be unique, which makes containers easy to find and manage. Add `-f` to `podman logs` to follow the logs as new lines are written.

    Setting `GREETING` with `-e` shows how containers are configured: the same image, with different settings supplied at runtime. Kubernetes does the same with ConfigMaps and Secrets.

## Part 4: Push the image to a registry

```bash

podman login quay.io
export QUAY_USER=<your-username>
podman tag greeting:v1.0.0 quay.io/${QUAY_USER}/greeting:v1.0.0
podman push quay.io/${QUAY_USER}/greeting:v1.0.0

```

Make the repository public on [quay.io](https://quay.io/): open the new `greeting` repository, go to **Settings** and set the visibility to **public**. Then remove the local images and pull the image back:

```bash

podman rm -f greeting
podman rmi quay.io/${QUAY_USER}/greeting:v1.0.0 greeting:v1.0.0
podman pull quay.io/${QUAY_USER}/greeting:v1.0.0

```

```text title="Expected output"
user@server greeting % podman login quay.io
Username: <your-username>
Password:
Login Succeeded!

user@server greeting % podman push quay.io/${QUAY_USER}/greeting:v1.0.0
Getting image source signatures
Copying blob 5f70bf18a086 done
...
Writing manifest to image destination

user@server greeting % podman rmi quay.io/${QUAY_USER}/greeting:v1.0.0 greeting:v1.0.0
Untagged: quay.io/<your-username>/greeting:v1.0.0
Untagged: localhost/greeting:v1.0.0
Deleted: b598ef58cead...

user@server greeting % podman pull quay.io/${QUAY_USER}/greeting:v1.0.0
Trying to pull quay.io/<your-username>/greeting:v1.0.0...
Getting image source signatures
...
Writing manifest to image destination
```

!!! tip
    The image name includes the registry hostname (`quay.io`), which tells the engine where to push. New Quay repositories are private by default, so the image must be made public before OpenShift can pull it without credentials.
