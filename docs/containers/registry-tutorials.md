---
tags:
  - Containers
  - Registries
---

# Registry Tutorials

This page continues from the [Container Tool Tutorials](container-tool-tutorials.md). You will take the `my-web-app:v1.1` image you built there, tag it for a registry, and push it. Select a registry below to get started.

## Before you begin

Make sure the `my-web-app:v1.1` image from the previous tutorial exists locally:

=== "Podman"

    ```bash
    podman images my-web-app
    ```

=== "Docker"

    ```bash
    docker images my-web-app
    ```

You should see a `v1.1` tag in the output.

!!! warning "Image not found?"
    If the image is missing, go back to the [Container Tool Tutorials](container-tool-tutorials.md) and complete steps 6 through 9 to build it. Do not remove `v1.1` with `rmi` until you have finished this tutorial.

=== "Red Hat Quay"

    ## Red Hat Quay

    Red Hat Quay is an enterprise container registry that provides security scanning, geo-replication, and detailed access controls. It's commonly used in OpenShift environments.

    ### Prerequisites

    - Docker or Podman installed and running
    - The `my-web-app:v1.1` image from the [Container Tool Tutorials](container-tool-tutorials.md)
    - A free Quay.io account ([Sign up here](https://quay.io/signin/){target="_blank"})

    ### Tutorial

    === "Podman"

        **1. Login to Quay.io**

        ```bash
        podman login quay.io
        ```

        Enter your Quay.io username and password when prompted:

        ```text
        Username: your_username
        Password: your_password
        Login Succeeded!
        ```

        **2. Tag for Quay.io**

        Tag your image with the registry, your Quay.io username, and a repository name:

        ```bash
        podman tag my-web-app:v1.1 quay.io/your_username/my-web-app:v1.1
        ```

        !!! note "Replace `your_username`"
            Replace `your_username` with your actual Quay.io username.

        Confirm both names point at the same image ID:

        ```bash
        podman images my-web-app
        ```

        **3. Push to Quay.io**

        ```bash
        podman push quay.io/your_username/my-web-app:v1.1
        ```

        **4. Verify your push**

        Visit [Quay.io Repositories](https://quay.io/repository/){target="_blank"} to see your pushed image.

        !!! tip "Repository Visibility"
            By default, new repositories on Quay.io are private. You can change visibility in the repository settings.

        **5. Run the image from the registry**

        Remove the local registry-tagged image, then run it again to prove it is pulled from Quay.io:

        ```bash
        podman rmi quay.io/your_username/my-web-app:v1.1
        podman run -d --name my-web-app -p 8080:80 quay.io/your_username/my-web-app:v1.1
        curl http://localhost:8080
        ```

        !!! note "Private repositories"
            If the repository is private, you must stay logged in (step 1) for the pull to succeed.

        **6. Clean up**

        ```bash
        podman rm -f my-web-app
        podman rmi quay.io/your_username/my-web-app:v1.1 my-web-app:v1.1
        ```

    === "Docker"

        **1. Login to Quay.io**

        ```bash
        docker login quay.io
        ```

        Enter your Quay.io username and password when prompted:

        ```text
        Username: your_username
        Password: your_password
        Login Succeeded!
        ```

        **2. Tag for Quay.io**

        Tag your image with the registry, your Quay.io username, and a repository name:

        ```bash
        docker tag my-web-app:v1.1 quay.io/your_username/my-web-app:v1.1
        ```

        !!! note "Replace `your_username`"
            Replace `your_username` with your actual Quay.io username.

        Confirm both names point at the same image ID:

        ```bash
        docker images my-web-app
        ```

        **3. Push to Quay.io**

        ```bash
        docker push quay.io/your_username/my-web-app:v1.1
        ```

        **4. Verify your push**

        Visit [Quay.io Repositories](https://quay.io/repository/){target="_blank"} to see your pushed image.

        !!! tip "Repository Visibility"
            By default, new repositories on Quay.io are private. You can change visibility in the repository settings.

        **5. Run the image from the registry**

        Remove the local registry-tagged image, then run it again to prove it is pulled from Quay.io:

        ```bash
        docker rmi quay.io/your_username/my-web-app:v1.1
        docker run -d --name my-web-app -p 8080:80 quay.io/your_username/my-web-app:v1.1
        curl http://localhost:8080
        ```

        !!! note "Private repositories"
            If the repository is private, you must stay logged in (step 1) for the pull to succeed.

        **6. Clean up**

        ```bash
        docker rm -f my-web-app
        docker rmi quay.io/your_username/my-web-app:v1.1 my-web-app:v1.1
        ```

    ### Quay.io Features

    Once your image is pushed, Quay.io automatically provides:

    | Feature | Description |
    | ------- | ----------- |
    | **Security Scanning** | Automatic vulnerability scanning of your images |
    | **Build Triggers** | Automatically build images from Git repositories |
    | **Robot Accounts** | Service accounts for CI/CD automation |
    | **Teams & Permissions** | Fine-grained access control |
    | **Image Expiration** | Automatic cleanup of old tags |

    ### Resources

    [Quay.io Documentation :fontawesome-brands-redhat:](https://docs.redhat.com/en/documentation/red_hat_quay/3/){ .md-button target="_blank"}

=== "IBM Cloud Registry"

    ## IBM Cloud Container Registry

    IBM Cloud Container Registry provides a multi-tenant, highly available, and scalable private image registry with integrated vulnerability scanning.

    ### Prerequisites

    - Docker or Podman installed and running
    - The `my-web-app:v1.1` image from the [Container Tool Tutorials](container-tool-tutorials.md)
    - IBM Cloud account ([Sign up here](https://cloud.ibm.com/registration){target="_blank"})
    - IBM Cloud CLI installed

    ### Tutorial

    === "Podman"

        **1. Install the IBM Cloud CLI**

        ```bash
        curl -fsSL https://clis.cloud.ibm.com/install/osx | sh
        ```

        **2. Install the Container Registry plugin**

        ```bash
        ibmcloud plugin install container-registry
        ```

        **3. Login to IBM Cloud**

        ```bash
        ibmcloud login
        ```

        For federated accounts, use:

        ```bash
        ibmcloud login --sso
        ```

        **4. Set up a namespace**

        Create a namespace to organize your images:

        ```bash
        ibmcloud cr namespace-add my_namespace
        ```

        !!! note "Namespace Names"
            Namespace names must be unique across all IBM Cloud accounts in the same region. Choose a descriptive name like `mycompany-dev`.

        Verify the namespace was created:

        ```bash
        ibmcloud cr namespace-list
        ```

        **5. Login to the registry**

        ```bash
        ibmcloud cr login --client podman
        ```

        **6. Tag for IBM Cloud Registry**

        Tag your image with your region and namespace:

        ```bash
        podman tag my-web-app:v1.1 us.icr.io/my_namespace/my-web-app:v1.1
        ```

        !!! info "Regional Endpoints"
            | Region | Endpoint |
            | ------ | -------- |
            | US South | `us.icr.io` |
            | EU Central | `de.icr.io` |
            | UK South | `uk.icr.io` |
            | AP North | `jp.icr.io` |
            | AP South | `au.icr.io` |

        **7. Push to IBM Cloud Registry**

        ```bash
        podman push us.icr.io/my_namespace/my-web-app:v1.1
        ```

        **8. Verify your push**

        ```bash
        ibmcloud cr image-list
        ```

        **9. Scan for vulnerabilities**

        IBM Cloud Registry includes Vulnerability Advisor:

        ```bash
        ibmcloud cr va us.icr.io/my_namespace/my-web-app:v1.1
        ```

        You can also view your images in the [IBM Cloud Console](https://cloud.ibm.com/containers/registry/images){target="_blank"}.

        **10. Clean up**

        ```bash
        podman rmi us.icr.io/my_namespace/my-web-app:v1.1 my-web-app:v1.1
        ```

    === "Docker"

        **1. Install the IBM Cloud CLI**

        ```bash
        curl -fsSL https://clis.cloud.ibm.com/install/osx | sh
        ```

        **2. Install the Container Registry plugin**

        ```bash
        ibmcloud plugin install container-registry
        ```

        **3. Login to IBM Cloud**

        ```bash
        ibmcloud login
        ```

        For federated accounts, use:

        ```bash
        ibmcloud login --sso
        ```

        **4. Set up a namespace**

        Create a namespace to organize your images:

        ```bash
        ibmcloud cr namespace-add my_namespace
        ```

        !!! note "Namespace Names"
            Namespace names must be unique across all IBM Cloud accounts in the same region. Choose a descriptive name like `mycompany-dev`.

        Verify the namespace was created:

        ```bash
        ibmcloud cr namespace-list
        ```

        **5. Login to the registry**

        ```bash
        ibmcloud cr login --client docker
        ```

        **6. Tag for IBM Cloud Registry**

        Tag your image with your region and namespace:

        ```bash
        docker tag my-web-app:v1.1 us.icr.io/my_namespace/my-web-app:v1.1
        ```

        !!! info "Regional Endpoints"
            | Region | Endpoint |
            | ------ | -------- |
            | US South | `us.icr.io` |
            | EU Central | `de.icr.io` |
            | UK South | `uk.icr.io` |
            | AP North | `jp.icr.io` |
            | AP South | `au.icr.io` |

        **7. Push to IBM Cloud Registry**

        ```bash
        docker push us.icr.io/my_namespace/my-web-app:v1.1
        ```

        **8. Verify your push**

        ```bash
        ibmcloud cr image-list
        ```

        **9. Scan for vulnerabilities**

        IBM Cloud Registry includes Vulnerability Advisor:

        ```bash
        ibmcloud cr va us.icr.io/my_namespace/my-web-app:v1.1
        ```

        You can also view your images in the [IBM Cloud Console](https://cloud.ibm.com/containers/registry/images){target="_blank"}.

        **10. Clean up**

        ```bash
        docker rmi us.icr.io/my_namespace/my-web-app:v1.1 my-web-app:v1.1
        ```

    ### IBM Cloud Registry Features

    | Feature | Description |
    | ------- | ----------- |
    | **Vulnerability Advisor** | Automatic security scanning with detailed reports |
    | **IAM Integration** | Fine-grained access control with IBM Cloud IAM |
    | **Multi-Region** | Deploy images across multiple regions |
    | **Retention Policies** | Automatic cleanup of old images |
    | **Private Network** | Access via IBM Cloud private network |

    ### Resources

    [IBM Cloud Registry Documentation :fontawesome-solid-cloud:](https://cloud.ibm.com/docs/Registry?topic=Registry-getting-started){ .md-button target="_blank"}

=== "Docker Hub"

    ## Docker Hub

    Docker Hub is the default registry for Docker and contains millions of public images. It's the easiest way to get started with container registries.

    ### Prerequisites

    - Docker or Podman installed and running
    - The `my-web-app:v1.1` image from the [Container Tool Tutorials](container-tool-tutorials.md)
    - A free Docker Hub account ([Sign up here](https://hub.docker.com/signup){target="_blank"})

    ### Tutorial

    === "Podman"

        **1. Login to Docker Hub**

        ```bash
        podman login docker.io
        ```

        Enter your Docker Hub username and password when prompted.

        **2. Tag your image**

        ```bash
        podman tag my-web-app:v1.1 docker.io/your_username/my-web-app:v1.1
        ```

        !!! note "Replace `your_username`"
            Replace `your_username` with your actual Docker Hub username.

        **3. Push to Docker Hub**

        ```bash
        podman push docker.io/your_username/my-web-app:v1.1
        ```

        **4. Verify your push**

        Visit [Docker Hub](https://hub.docker.com/){target="_blank"} and navigate to your repositories to see your pushed image.

        **5. Run the image from the registry**

        Remove the local registry-tagged image, then run it again to prove it is pulled from Docker Hub:

        ```bash
        podman rmi docker.io/your_username/my-web-app:v1.1
        podman run -d --name my-web-app -p 8080:80 docker.io/your_username/my-web-app:v1.1
        curl http://localhost:8080
        ```

        **6. Clean up**

        ```bash
        podman rm -f my-web-app
        podman rmi docker.io/your_username/my-web-app:v1.1 my-web-app:v1.1
        ```

    === "Docker"

        **1. Login to Docker Hub**

        ```bash
        docker login
        ```

        Enter your Docker Hub username and password when prompted.

        **2. Tag your image**

        ```bash
        docker tag my-web-app:v1.1 your_username/my-web-app:v1.1
        ```

        !!! note "Replace `your_username`"
            Replace `your_username` with your actual Docker Hub username.

        **3. Push to Docker Hub**

        ```bash
        docker push your_username/my-web-app:v1.1
        ```

        **4. Verify your push**

        Visit [Docker Hub](https://hub.docker.com/){target="_blank"} and navigate to your repositories to see your pushed image.

        **5. Run the image from the registry**

        Remove the local registry-tagged image, then run it again to prove it is pulled from Docker Hub:

        ```bash
        docker rmi your_username/my-web-app:v1.1
        docker run -d --name my-web-app -p 8080:80 your_username/my-web-app:v1.1
        curl http://localhost:8080
        ```

        **6. Clean up**

        ```bash
        docker rm -f my-web-app
        docker rmi your_username/my-web-app:v1.1 my-web-app:v1.1
        ```

    ### Resources

    [Docker Hub Documentation :fontawesome-brands-docker:](https://docs.docker.com/docker-hub/){ .md-button target="_blank"}
