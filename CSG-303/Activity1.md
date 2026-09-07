# Activity 1: From Application to Docker and Kubernetes

## Overview

In this hands-on activity, you will create a small application, package it as a Docker image, publish the image to a public Docker Hub repository, and deploy it to Kubernetes. Your Kubernetes Deployment must run **exactly 3 replicas**, and the application must be exposed through a **NodePort Service**.

This activity gives you the required outcomes, checkpoints, and reference material. It intentionally does not provide commands, command syntax, application code, a Dockerfile, or Kubernetes configuration. You are expected to study the linked documentation, choose the appropriate procedure, and solve implementation problems independently.

## Learning objectives

By the end of this activity, you should be able to:

- Create and test a simple application in Python or another programming language.
- describe how a Dockerfile packages an application and its dependencies;
- build and test a container image;
- publish an image to a public Docker Hub repository;
- create and inspect a Kubernetes Deployment;
- configure a Deployment to maintain exactly three replicas;
- expose an application outside the cluster with a NodePort Service; and
- verify a working containerized deployment.

## Prerequisites and accounts

You need:

- A [Docker Hub account](https://hub.docker.com/signup) for publishing your image.
- A free [Killercoda account](https://killercoda.com/) if you will use its browser-based environments.
- A text editor or development environment.
- Docker on your computer **or** access to the [Killercoda Ubuntu playground](https://killercoda.com/playgrounds/scenario/ubuntu).
- A [GitHub account](https://github.com/signup) only if you choose to transfer your source through a GitHub repository.
- A web browser and internet connection.

> **Security note:** Killercoda environments are temporary. Do not place passwords, access tokens, private keys, or other sensitive information in application files, screenshots, or a public source repository. Enter credentials only when the relevant sign-in process requests them. Review [Killercoda's FAQ](https://killercoda.com/faq) for environment lifetime information.

## Activity workflow

### Part 1 — Create and test a simple application

1. Choose Python or another language you know.
2. Create a small application with an observable result that can later be reached through a Kubernetes Service. Its response should make your work identifiable, for example by displaying the activity name and your name or student ID.
3. Record the network port used by the application. You will need the same container port when configuring Kubernetes.
4. Run the application outside Docker and confirm that it works.
5. Keep the project small, and include all files needed to install its dependencies and start it.

### Part 2 — Prepare a Docker environment

Choose one of these routes:

**Route A: Work locally**

Use Docker already installed on your computer, or follow Docker's [Get Docker](https://docs.docker.com/get-started/get-docker/) guidance for your operating system. Confirm that Docker is running before continuing.

**Route B: Use Killercoda**

1. Sign in to Killercoda and open the [Ubuntu Linux playground](https://killercoda.com/playgrounds/scenario/ubuntu).
2. Check whether Docker is already available in the environment. If it is unavailable, follow Docker's official [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/) instructions.
3. Confirm from the installation guide's verification section that Docker is working.
4. Remember that the playground is temporary. Save important source files and capture evidence before the session ends.

### Part 3 — Move the source to the Ubuntu VM (optional)

If your application was created elsewhere and you are building it in Killercoda, you may store the source in a GitHub repository and clone it into the Ubuntu VM.

1. Create a repository and add only the application source and safe configuration files. See [Create a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository).
2. Make sure all required project files are present in the repository.
3. In the Ubuntu environment, follow GitHub's [Cloning a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository) guide.
4. Confirm that the cloned folder contains the expected source and dependency files.

Do not commit passwords, tokens, or other secrets. A public GitHub repository is optional unless your instructor requires its URL as part of the submission.

### Part 4 — Containerize the application

1. Study the Dockerfile and image-building documentation below.
2. Create a Dockerfile suitable for your chosen application and language.
3. Build an appropriately named image for your Docker Hub repository.
4. Test the image and confirm that the containerized application works before publishing it.

Use these references:

- [Writing a Dockerfile](https://docs.docker.com/get-started/docker-concepts/building-images/writing-a-dockerfile/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Building images](https://docs.docker.com/build/concepts/overview/)
- [Build and push your first image](https://docs.docker.com/get-started/introduction/build-and-push-first-image/)

### Part 5 — Publish the image to Docker Hub

1. Sign in to Docker Hub.
2. Create a new repository under your own Docker Hub account.
3. Set the repository visibility to **Public** so that the Killercoda Kubernetes cluster can pull the image without registry credentials.
4. Use the Docker documentation to determine the correct image naming and tagging requirements for this repository.
5. Authenticate from the environment where you built the image and publish it to Docker Hub.
6. Open the repository page in a signed-out/private browser window and confirm that the repository and pushed tag are publicly visible.
7. Record the complete image reference, including Docker Hub username, repository name, and tag. You will use this exact reference in Kubernetes.

Use these references:

- [Docker Hub repositories](https://docs.docker.com/docker-hub/repos/)
- [Docker Hub quickstart](https://docs.docker.com/docker-hub/quickstart/)
- [Push an image to Docker Hub](https://docs.docker.com/docker-hub/repos/manage/hub-images/push/)

### Part 6 — Create the Kubernetes environment

1. Start a separate [Killercoda Kubernetes playground](https://killercoda.com/playgrounds/scenario/kubernetes). Do not use the Ubuntu environment from the Docker stage as a substitute for the Kubernetes playground.
2. Wait for the cluster to finish initializing.
3. Confirm that the cluster is reachable and inspect its nodes before creating resources.
4. Use the Kubernetes tools already provided in the playground. The [kubectl overview](https://kubernetes.io/docs/concepts/overview/kubectl/) and [kubectl quick reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/) explain how to manage and inspect cluster resources.

### Part 7 — Deploy exactly three replicas

1. Create a Kubernetes Deployment that uses the complete public Docker Hub image reference from Part 5.
2. Configure the Deployment for **exactly 3 replicas**.
3. Use the documentation to determine the remaining Deployment settings needed by your application.
4. Wait for the Deployment to become ready.
5. Inspect the Deployment and its Pods. Confirm that Kubernetes is maintaining three healthy application instances.
6. If a Pod does not start, use Kubernetes inspection and troubleshooting documentation to investigate the cause.

Use these references:

- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Running multiple instances of an application](https://kubernetes.io/docs/tutorials/kubernetes-basics/scale/scale-intro/)
- [Viewing Pods and nodes](https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/)
- [Deployment rollout status](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)
- [Viewing container logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)

### Part 8 — Expose and verify the application

1. Create a Service for the Deployment.
2. Set the Service type to **NodePort**.
3. Use the documentation to determine how the Service must connect to the Deployment and application.
4. Inspect the Service and identify its assigned NodePort.
5. Use Killercoda's traffic/port access feature to open the application through that NodePort. Refer to Killercoda's [network traffic documentation](https://killercoda.com/creators#network-traffic-into-environments).
6. Confirm that the application returns its expected output through the Service.
7. Recheck the Deployment and Pods and confirm that **all 3 replicas are running and ready** while the application remains accessible.

Use these references:

- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Service type: NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
- [Use a Service to access an application](https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/)

## Expected outcome

At completion:

- your simple application works from a container image;
- the image and its tag are available in a public repository under your Docker Hub account;
- a Kubernetes Deployment uses that image and reports exactly three desired and three ready replicas;
- three application Pods are running and ready;
- a NodePort Service selects those Pods and forwards to the correct application port; and
- the application is reachable through the Killercoda environment and displays the expected response.

## Completion checklist

- [ ] I created and tested a simple application.
- [ ] I recorded the port on which the application listens.
- [ ] I prepared a working Docker environment locally or in Killercoda Ubuntu.
- [ ] I transferred the source to the build environment, if necessary.
- [ ] I created a Dockerfile without placing secrets in it.
- [ ] I built the image and tested the application in a container.
- [ ] I created a **public** Docker Hub repository under my account.
- [ ] I pushed the image and verified the required tag is publicly visible.
- [ ] I recorded the complete Docker Hub image reference.
- [ ] I started a Killercoda Kubernetes playground and confirmed the cluster was ready.
- [ ] I created a Deployment using my Docker Hub image.
- [ ] The Deployment is configured for **exactly 3 replicas**.
- [ ] All three Pods are running and ready.
- [ ] I created a Service of type **NodePort**.
- [ ] The Service selects the correct Pods and targets the correct application port.
- [ ] I accessed the application through the NodePort and verified its output.

---

**Completion standard:** The activity is complete only when the public image can be pulled by the Kubernetes environment, the Deployment shows three healthy replicas, and the application is reachable through the NodePort Service.
