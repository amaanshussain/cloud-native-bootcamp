---
tags:
  - Lab
  - Beginner
  - Containers
---

# Containers Lab: Tutorial Reference

<span class="lab-badge">40 min</span> <span class="lab-badge">Beginner</span>

In this lab you'll build a container image for a small web service, run it, inspect it, and push it to a public registry.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Container Tool Tutorials](../../../containers/container-tool-tutorials.md) or [Registry Tutorials](../../../containers/registry-tutorials.md) that shows the command you need. **You build the command yourself** by adapting the tutorial's example to this lab. When the tutorials don't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the tutorials open"
    Open the [Container Tool Tutorials](../../../containers/container-tool-tutorials.md) in a second tab before you start. Its Podman and Docker tabs show the same commands for both engines. This lab uses `quay.io/podman/hello` and `greeting` where the tutorial uses `nginx` and `my-web-app`, and it listens on port `8080` inside the container where the tutorial's NGINX uses port `80`.

## Prerequisites

- A container engine:
    - **[Podman](https://podman.io/docs/installation)** (recommended). On macOS and Windows, run `podman machine init` and `podman machine start` once after installing.
    - **[Docker](https://docs.docker.com/get-started/get-docker/)** also works. Every `podman` command in this lab works the same with `docker`.
- A terminal and a text editor.
- A free **[Quay.io](https://quay.io/)** account, used to push your image in Part 4.

## Part 1: Verify your setup

1. Find out which version of your container engine is installed.
    - **Reference:** [Container Tool Tutorials, step 1](../../../containers/container-tool-tutorials.md#1-verify-your-installation)
    - **Check:** the output shows a client version.

2. Run the `quay.io/podman/hello` container. The container should be deleted automatically when it exits.
    - **Reference:** [Container Tool Tutorials, step 3](../../../containers/container-tool-tutorials.md#3-run-a-container) shows how to run a container. It doesn't cover deleting a container on exit, so add this flag: `--rm`
    - **Check:** you see `Hello Podman World`, and `podman ps -a` doesn't list the container.

3. List the images stored on your machine.
    - **Reference:** [Container Tool Tutorials, step 2](../../../containers/container-tool-tutorials.md#2-pull-an-image)
    - **Check:** `quay.io/podman/hello` is in the list, and it's less than 1 MB.

## Part 2: Build an image

### Create the application

Create a directory named `greeting` and move into it:

```bash
mkdir greeting && cd greeting
```

Save the following two files in it. The service returns a JSON greeting and reads its message from the `GREETING` environment variable.

```go title="main.go"
--8<-- "docs/labs/containers/lab1/greeting/main.go"
```

```text title="go.mod"
--8<-- "docs/labs/containers/lab1/greeting/go.mod"
```

You don't need Go installed. The build happens inside a container.

### Write the Containerfile

A `Containerfile` describes how to build the image. Save this file as `Containerfile` in the same directory:

```dockerfile title="Containerfile"
--8<-- "docs/labs/containers/lab1/greeting/Containerfile"
```

[Container Tool Tutorials, step 6](../../../containers/container-tool-tutorials.md#6-write-a-dockerfile) explains `FROM`, `COPY` and `EXPOSE`. It doesn't cover multi-stage builds, `RUN`, `USER` or `CMD`:

- `AS builder` names the first stage so a later stage can copy files from it.
- `COPY --from=builder` copies from an earlier stage instead of from your directory.
- `RUN` runs a command at build time. Here it compiles the Go program into a single binary.
- `USER 1001` runs the app as a non-root user. OpenShift ignores this and assigns a random UID, which is why the app doesn't depend on a particular user.
- `CMD` is the command that runs when the container starts.

This is a **multi-stage build**. The first stage contains the whole Go toolchain, which is over 1 GB. The final image contains only the compiled binary on top of `ubi-micro`, so it's around 30 MB. Smaller images pull faster, start faster, and have fewer packages that could contain vulnerabilities.

### Build it

4. Build the image from the current directory and tag it `greeting:v1.0.0`.
    - **Reference:** [Container Tool Tutorials, step 7](../../../containers/container-tool-tutorials.md#7-build-the-image)
    - **Check:** the build ends with `Successfully tagged localhost/greeting:v1.0.0`.

5. List your images and compare the size of `greeting` with the size of `go-toolset`.
    - **Reference:** [Container Tool Tutorials, step 2](../../../containers/container-tool-tutorials.md#2-pull-an-image)
    - **Check:** `greeting` is around 30 MB, and `go-toolset` is over 1 GB. Why is there such a difference?

## Part 3: Run and inspect the container

!!! tip "Port already in use?"
    If something else on your machine uses port 8080, map a different host port, for example `-p 9080:8080`, and use that port in the steps below.

6. Run the image in the background, name the container `greeting`, and make the service reachable on port `8080` of your machine.
    - **Reference:** [Container Tool Tutorials, step 3](../../../containers/container-tool-tutorials.md#3-run-a-container). The flag table explains `-d`, `--name` and `-p`. Use the image name `localhost/greeting:v1.0.0` with Podman, and the container listens on port `8080`.
    - **Check:** the command prints a container ID and returns your prompt.

7. Call the `/greeting` endpoint with the name `John` using `curl`.
    - **Reference:** [Container Tool Tutorials, step 3](../../../containers/container-tool-tutorials.md#3-run-a-container) shows `curl`. The path is `/greeting` and the query parameter is `name`.
    - **Check:** the response is JSON containing `"name":"John"`.

8. Try to start a second container with the same name `greeting`, on host port `8081`.
    - **Reference:** [Container Tool Tutorials, step 3](../../../containers/container-tool-tutorials.md#3-run-a-container)
    - **Check:** it fails. Read the error and explain why.

9. List the running containers.
    - **Reference:** [Container Tool Tutorials, step 4](../../../containers/container-tool-tutorials.md#4-inspect-and-debug-a-running-container)
    - **Check:** `greeting` is listed with its port mapping.

10. Look at the container's full configuration. Then print only its **status** and **IP address**.
    - **Reference:** [Container Tool Tutorials, step 4](../../../containers/container-tool-tutorials.md#4-inspect-and-debug-a-running-container) shows the full configuration. It doesn't cover printing selected fields, so use this command:

    ```bash
    podman inspect greeting --format '{{.State.Status}} {{.NetworkSettings.IPAddress}}'
    ```

    | Part | What it does |
    | --- | --- |
    | `podman inspect greeting` | Prints the container's configuration as JSON |
    | `--format '...'` | Takes a Go template. Each `{{ }}` selects one field from the JSON |

    ```text title="Expected output"
    running 10.88.0.2
    ```

    - **Check:** the output is a single line such as `running 10.88.0.2`.

11. Show the container's logs.
    - **Reference:** [Container Tool Tutorials, step 4](../../../containers/container-tool-tutorials.md#4-inspect-and-debug-a-running-container)
    - **Check:** you see `greeting service listening on :8080`.

12. Run a command **inside** the running container to find out which user the app runs as. The command that prints the current user is `id`.
    - **Reference:** [Container Tool Tutorials, step 4](../../../containers/container-tool-tutorials.md#4-inspect-and-debug-a-running-container). The tutorial opens a shell. Run `id` directly instead of `/bin/sh`, and leave out `-it`.
    - **Check:** the output shows UID `1001`.

13. Stop and remove the container. Start a new one with the same name and port, but change the greeting message by setting the `GREETING` environment variable. Call it again.
    - **Reference:** [Container Tool Tutorials, step 5](../../../containers/container-tool-tutorials.md#5-stop-and-remove-the-container) covers stopping and removing. It doesn't cover environment variables, so add this flag to your `run` command: `-e GREETING="Hello from my container"`
    - **Check:** the `message` in the response is your custom text.

## Part 4: Push the image to a registry

A **container registry** stores and distributes images. Docker Hub, Quay.io and GitHub Container Registry are public registries. OpenShift also has a built-in internal registry.

14. Log in to Quay.io.
    - **Reference:** [Registry Tutorials](../../../containers/registry-tutorials.md), Red Hat Quay tab, step 1
    - **Check:** you see `Login Succeeded!`

    !!! tip "Single sign-on?"
        If your Quay account uses single sign-on (for example, a Red Hat login), create an **encrypted password** under **Account Settings > Docker CLI Password** and use it to log in instead of your account password.

15. Give your image a second name that points to your Quay repository: `quay.io/<your-username>/greeting:v1.0.0`.
    - **Reference:** [Registry Tutorials](../../../containers/registry-tutorials.md), Red Hat Quay tab, step 2
    - **Check:** `greeting:v1.0.0` and your Quay name appear in the image list with the **same image ID**.

16. Push the image to Quay.
    - **Reference:** [Registry Tutorials](../../../containers/registry-tutorials.md), Red Hat Quay tab, step 3
    - **Check:** the push finishes with `Writing manifest to image destination`.

17. On [quay.io](https://quay.io/), open the new `greeting` repository and, under **Settings**, make it **public**. New repositories are private by default.
    - **Check:** the repository shows a **Public** badge.

18. Remove all local copies of the `greeting` image (and the container), then pull the image back from Quay.
    - **Reference:** [Container Tool Tutorials, step 5](../../../containers/container-tool-tutorials.md#5-stop-and-remove-the-container) shows how to remove a container. [Step 10](../../../containers/container-tool-tutorials.md#10-clean-up) shows how to remove images, and [step 2](../../../containers/container-tool-tutorials.md#2-pull-an-image) shows how to pull.
    - **Check:** the pull succeeds without asking you to log in.

!!! warning "Building on Apple Silicon (M-series Macs)?"
    Images you build on an ARM Mac are `arm64` by default, but most OpenShift clusters run on `amd64` (x86_64) nodes. Pods fail there with `exec format error`. The tutorials don't cover building for another platform, so use one of these:

    ```bash
    # Single platform
    podman build --platform linux/amd64 -t quay.io/<your-username>/greeting:v1.0.0 .

    # Multi-arch manifest list for amd64 and arm64
    podman build --platform linux/amd64,linux/arm64 --manifest quay.io/<your-username>/greeting:v1.0.0 .
    podman manifest push quay.io/<your-username>/greeting:v1.0.0
    ```

    | Part | What it does |
    | --- | --- |
    | `--platform linux/amd64` | Builds the image for x86_64 nodes instead of your machine's CPU architecture |
    | `--platform linux/amd64,linux/arm64` | Builds one image per platform |
    | `--manifest <name>` | Groups those images into a **manifest list**, so a pull picks the image that matches the node's architecture |
    | `podman manifest push` | Pushes the manifest list and every image in it |

## Cleanup

Remove the `greeting` container and the local `greeting:v1.0.0` image.

- **Reference:** [Container Tool Tutorials, step 10](../../../containers/container-tool-tutorials.md#10-clean-up)

Keep the `greeting` image on Quay. The [Image Registry Lab](../lab2/index.md) deploys it to OpenShift.

## What you learned

- [x] Building images with a Containerfile and multi-stage builds
- [x] Running, inspecting and configuring containers
- [x] Tagging and pushing images to a registry
- [x] Building images for the right CPU architecture
