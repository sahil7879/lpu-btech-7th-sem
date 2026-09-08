# Activity 2: Frontend-to-Backend Communication and Node-Mounted Storage

## Overview

In this hands-on activity, you will extend the Docker and Kubernetes skills developed in Activity 1. You will create two separate applications:

- a **frontend application** that displays a button; and
- a **backend application** that receives a request and creates a file.

Both applications will be containerized and deployed to the same Kubernetes environment. A user will access the frontend from outside the cluster. When the user selects the button, the frontend will communicate with the backend through a cluster-internal Service. The backend will create a file in a directory mounted from the Kubernetes node into the backend container.

This activity provides requirements, checkpoints, and documentation links. It intentionally does **not** provide application code, Dockerfiles, Kubernetes manifests, commands, or command syntax. You are expected to research and implement the solution independently.

## Learning objectives

By the end of this activity, you should be able to:

- design a simple frontend-and-backend application;
- build and publish separate container images for two applications;
- deploy multiple applications to the same Kubernetes cluster;
- distinguish between externally accessible and cluster-internal Services;
- use a ClusterIP Service for communication between workloads;
- use Kubernetes DNS-based Service discovery;
- define a volume and mount it into a container;
- connect a directory on a Kubernetes node to a backend Pod;
- verify application communication and file creation; and
- explain the limitations of node-local storage.

## Prerequisites

Before starting, you should have:

- completed Activity 1 or be comfortable with its Docker and Kubernetes topics;
- a [Docker Hub account](https://hub.docker.com/signup);
- a [Killercoda account](https://killercoda.com/);
- access to the [Killercoda Ubuntu playground](https://killercoda.com/playgrounds/scenario/ubuntu) if Docker is unavailable locally;
- access to the [Killercoda Kubernetes playground](https://killercoda.com/playgrounds/scenario/kubernetes);
- a GitHub account if you plan to transfer your source through GitHub; and
- a basic understanding of HTTP requests and responses.

The frontend and backend may use the same programming language or different languages.

## Required architecture

Your completed application must follow this communication path:

**User's browser → frontend Service → frontend Pod → backend ClusterIP Service → backend Pod → mounted node directory**

The required Kubernetes resources are:

- one Deployment for the frontend application;
- one Deployment for the backend application;
- one externally accessible Service for the frontend;
- one ClusterIP Service for the backend; and
- one node-directory volume mounted into the backend container.

Only the frontend should be directly accessible from outside the Kubernetes cluster. The backend must remain reachable only from within the cluster through its ClusterIP Service.

> **Important design constraint:** A ClusterIP is available only inside the Kubernetes cluster. Browser-based JavaScript running on your computer cannot directly contact it. Your frontend must therefore include a server-side component that receives the button action and forwards the request from the frontend Pod to the backend Service. Research this request flow before choosing your frontend technology.

## Functional requirements

Your solution must meet all of the following requirements:

1. The frontend presents a page containing a clearly labelled button.
2. Selecting the button starts a request from the frontend application to the backend application.
3. The frontend contacts the backend through the backend Service, not through a Pod IP.
4. The backend accepts the request and creates a new file in its mounted directory.
5. The created file contains meaningful request information, such as a timestamp or generated request identifier.
6. The backend returns a clear success or failure response.
7. The frontend displays the backend result to the user.
8. The file is visible in the chosen directory on the node that runs the backend Pod.
9. Repeated button selections produce a result that can be verified without silently overwriting the previous result.
10. Neither image contains passwords, access tokens, or other secrets.

## Activity workflow

### Part 1 — Plan the two-application design

1. Decide which languages or frameworks you will use for the frontend and backend.
2. Decide how the frontend server will receive the button action and forward it to the backend.
3. Define the backend request endpoint and the success and failure responses.
4. Decide what each created file will be called and what information it will contain.
5. Record the listening port used by each application.
6. Decide which path the backend will use inside its container and which directory on the node will be mounted there.
7. Draw or describe the complete request path before writing the applications.

### Part 2 — Develop and test the applications

1. Create the frontend application with its page, button, server-side request handling, and result display.
2. Create the backend application with an endpoint that creates a file in a configurable directory.
3. Ensure that user-controlled values cannot be used to write files outside the intended directory.
4. Test the backend independently and confirm that it creates the expected file.
5. Test the frontend and backend together outside Kubernetes.
6. Confirm that the frontend shows a useful error when the backend is unavailable or returns a failure.

You must design and write both applications yourself. Consult the official documentation for your selected language or framework when needed.

### Part 3 — Containerize both applications

1. Create a separate Dockerfile for each application.
2. Build a separate container image for the frontend and backend.
3. Test both images before publishing them.
4. Confirm that each container listens on its intended port.
5. Confirm that the backend can write to the directory that will become its volume mount location.

Use these references:

- [Writing a Dockerfile](https://docs.docker.com/get-started/docker-concepts/building-images/writing-a-dockerfile/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Building images](https://docs.docker.com/build/concepts/overview/)
- [Build and push your first image](https://docs.docker.com/get-started/introduction/build-and-push-first-image/)

### Part 4 — Publish both images

1. Create public Docker Hub repositories for the frontend and backend images.
2. Name and tag each image so it targets the correct repository.
3. Publish both images.
4. Confirm that both repositories and the required tags are publicly available.
5. Record the complete frontend and backend image references for use in Kubernetes.

Use these references:

- [Docker Hub repositories](https://docs.docker.com/docker-hub/repos/)
- [Docker Hub quickstart](https://docs.docker.com/docker-hub/quickstart/)
- [Push an image to Docker Hub](https://docs.docker.com/docker-hub/repos/manage/hub-images/push/)

If you need to move your source into the Killercoda Ubuntu environment, consult GitHub's [Cloning a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository) guide.

### Part 5 — Prepare the Kubernetes environment

1. Open the [Killercoda Kubernetes playground](https://killercoda.com/playgrounds/scenario/kubernetes).
2. Wait until the cluster is ready.
3. Inspect the cluster topology and identify its schedulable node or nodes.
4. Choose a directory on a node for this activity's backend files.
5. Prepare the directory with permissions appropriate for the user running inside your backend container.

Do not store sensitive or important data in this directory. Killercoda environments are temporary and are deleted when their session ends.

### Part 6 — Deploy the backend with a mounted volume

1. Create a Deployment for the backend using its public Docker Hub image.
2. Configure the backend Deployment with a volume that represents the selected directory on the node.
3. Mount that volume at the directory where the backend application creates files.
4. Configure the backend application with any non-secret settings it needs, including its write location.
5. Consider the relationship between Pod scheduling and the node that owns the selected directory.
6. Start with one backend replica so that file placement is unambiguous for this node-local storage exercise.
7. Wait for the backend Pod to become healthy and inspect its status before proceeding.

Use these references:

- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [hostPath volume](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
- [Configure a Pod to use a volume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/)
- [Assign Pods to nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Security context for Pods and containers](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

> **Storage warning:** A node-directory volume is tied to one node. If the Pod moves to another node, the same files may not be available there. Multiple backend replicas scheduled on different nodes would write to different node directories. This approach is useful for learning about mounts, but it is not a replacement for proper persistent storage in a production system.

### Part 7 — Create the backend ClusterIP Service

1. Create a Service that selects the backend Pods.
2. Use the ClusterIP Service type.
3. Map the Service to the port used by the backend application.
4. Confirm that the Service has a ready backend endpoint.
5. Determine the DNS name that other Pods in the same namespace can use to reach this Service.
6. Test backend access from inside the cluster before connecting the frontend.

Use these references:

- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Service type: ClusterIP](https://kubernetes.io/docs/concepts/services-networking/service/#type-clusterip)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Connect a frontend to a backend using Services](https://kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/)

### Part 8 — Deploy and expose the frontend

1. Create a Deployment for the frontend using its public Docker Hub image.
2. Configure the frontend with the backend Service location. Use the Service's stable DNS identity rather than a backend Pod IP.
3. Create an externally accessible Service for the frontend using the same access approach as Activity 1.
4. Confirm that the frontend Pod is healthy and the Service selects it.
5. Open the frontend through Killercoda's traffic/port interface.

Use these references:

- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Service type: NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
- [Use a Service to access an application](https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/)
- [Killercoda network traffic](https://killercoda.com/creators#network-traffic-into-environments)

### Part 9 — Verify the complete request flow

1. Open the frontend application from outside the cluster.
2. Select the button and confirm that the frontend displays a successful backend response.
3. Verify that the backend received the request.
4. Identify the node running the backend Pod.
5. Inspect the selected directory on that node and confirm that a new file exists.
6. Confirm that the file contents match the request result shown by the frontend.
7. Select the button again and confirm that another verifiable result is produced.
8. Confirm that the backend cannot be opened directly from outside the cluster through its ClusterIP.
9. Inspect the frontend and backend resources and confirm that all Pods remain healthy.

If the flow fails, investigate it in stages: external access to the frontend, frontend processing, Service discovery, backend Service endpoints, backend processing, volume mounting, and node-directory permissions.

Use these references:

- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Viewing container logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)

## Expected outcome

At completion:

- the frontend and backend exist as separate container images in public Docker Hub repositories;
- both applications run as separate Deployments in the same Kubernetes cluster;
- users can reach the frontend from outside the cluster;
- the backend is exposed only through a ClusterIP Service;
- the frontend Pod reaches the backend using Kubernetes Service discovery;
- selecting the frontend button causes the backend to create a file;
- the file appears in the node directory mounted into the backend container; and
- the frontend clearly reports whether the operation succeeded or failed.

## Completion checklist

- [ ] I designed separate frontend and backend applications.
- [ ] My frontend includes a server-side path for contacting the cluster-internal backend.
- [ ] Selecting the frontend button initiates the required backend operation.
- [ ] My backend creates a verifiable file in its configured write directory.
- [ ] I created and tested separate container images.
- [ ] Both images are available in public Docker Hub repositories.
- [ ] I deployed the frontend and backend to the same Kubernetes cluster.
- [ ] The backend Deployment mounts a directory from its node.
- [ ] The backend runs with one replica for this node-local storage exercise.
- [ ] The backend Service uses ClusterIP and has a ready endpoint.
- [ ] The frontend uses the backend Service identity rather than a Pod IP.
- [ ] Only the frontend is directly accessible from outside the cluster.
- [ ] I successfully opened the frontend through Killercoda.
- [ ] Selecting the button produced a successful response.
- [ ] I confirmed that the expected file exists in the correct node directory.
- [ ] Repeating the operation produced another verifiable result.
- [ ] I understand why this node-local volume is not suitable as general production storage.

---

**Completion standard:** The activity is complete only when the frontend is externally accessible, the backend remains cluster-internal, the button action reaches the backend through its ClusterIP Service, and the resulting file is verified in the directory mounted from the backend Pod's node.
