# Activity 3: Automated Report Generation with Jobs, CronJobs, ConfigMaps, and Secrets

## Overview

In this hands-on activity, you will create a small report-generator application and run it as a Kubernetes batch workload. You will first execute the application once by using a Job. You will then schedule it to run repeatedly by using a CronJob.

The application will read ordinary configuration from a ConfigMap and a dummy credential from a Secret. Each successful run will generate a uniquely identifiable report in a directory mounted from a Kubernetes node.

This activity is designed for the free [Killercoda Kubernetes playground](https://killercoda.com/playgrounds/scenario/kubernetes). It provides requirements, checkpoints, and documentation links but intentionally provides **no application code, Dockerfile, Kubernetes manifest, command, or command syntax**. You must research and develop the solution independently.

## Learning objectives

By the end of this activity, you should be able to:

- explain the difference between a Deployment, Job, and CronJob;
- design an application that runs to completion instead of operating continuously;
- build and publish a container image for a batch application;
- run a one-time task with a Kubernetes Job;
- schedule a recurring task with a Kubernetes CronJob;
- separate application configuration from a container image by using a ConfigMap;
- provide sensitive-style data to a workload through a Secret;
- mount storage into a batch workload;
- inspect Job and CronJob execution results; and
- explain important limitations of Secrets and node-local storage.

## Prerequisites

Before starting, you should have:

- completed Activity 1 or possess equivalent Docker and Kubernetes knowledge;
- a [Docker Hub account](https://hub.docker.com/signup);
- a [Killercoda account](https://killercoda.com/);
- Docker on your computer or access to the [Killercoda Ubuntu playground](https://killercoda.com/playgrounds/scenario/ubuntu);
- a GitHub account if you intend to transfer your source through GitHub; and
- a basic understanding of files, environment-based application configuration, and scheduled tasks.

## Scenario

Your team needs a small application that produces periodic operational reports. The same application must support both an immediate one-time run and recurring scheduled execution.

For this activity, each report must include:

- a report title obtained from ordinary configuration;
- a student identifier obtained from ordinary configuration;
- the date and time at which the report was generated;
- a unique value that distinguishes one run from another; and
- confirmation that a required dummy credential was available to the application, without revealing the credential itself.

The application must save every report as a separate file and finish successfully after the file is written. It must return a failure when required configuration is missing or when the report cannot be written.

## Required Kubernetes resources

Your solution must include:

- one ConfigMap containing non-sensitive application settings;
- one Secret containing a fictional credential created only for this activity;
- one node directory used as the report storage location;
- one successfully completed Job;
- one CronJob that creates Jobs according to a schedule; and
- the Pods created by the Job and CronJob.

You do not need a Deployment or Service because the report generator is a batch process. It should complete its task and exit rather than wait for network requests.

## Functional requirements

Your solution must meet all of the following requirements:

1. The application is packaged in a container image stored in a public Docker Hub repository.
2. The image does not contain activity-specific configuration or credentials.
3. The report title and student identifier come from a ConfigMap.
4. A fictional access value comes from a Kubernetes Secret.
5. The application confirms that the Secret value exists but never writes or prints its value.
6. Reports are written to a volume mounted into the container.
7. The mounted volume corresponds to a directory on the node used by the workload.
8. Every run creates a distinct report rather than replacing an earlier report.
9. The one-time Job finishes successfully.
10. The CronJob produces at least two successful scheduled Jobs during verification.
11. The CronJob does not start overlapping report-generation runs.
12. The workload uses a suitable restart policy for a task that must run to completion.

## Activity workflow

### Part 1 — Research batch workloads

Before building anything, study the purpose and lifecycle of Jobs and CronJobs.

Determine:

- when a Job is more appropriate than a Deployment;
- how Kubernetes decides whether a Job succeeded or failed;
- how a CronJob creates and manages Jobs;
- how scheduling expressions represent recurring execution;
- what happens if a scheduled run is missed;
- how concurrent runs can be allowed, replaced, or prevented; and
- how completed Job history can be limited.

Use these references:

- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Running automated tasks with a CronJob](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/)

Review the important Job and CronJob settings before proceeding. You will select the settings appropriate for this scenario yourself.

### Part 2 — Design the report-generator application

1. Choose Python or another suitable programming language.
2. Design the application so that one execution produces one report and then exits.
3. Decide how the application will receive ordinary configuration.
4. Decide how it will receive and validate the fictional credential without revealing it.
5. Decide how unique report filenames will be generated.
6. Decide how success and failure will be communicated through application output and the process result.
7. Make the report output directory configurable rather than permanently embedding an environment-specific path in the application.
8. Test the application outside Kubernetes with both valid and missing configuration.

Do not use a real password, API key, token, or personal credential. The Secret must contain a fictional value created solely for this temporary exercise.

### Part 3 — Containerize and publish the application

1. Create a Dockerfile appropriate for your application.
2. Build the container image.
3. Test that the container produces one report and then stops.
4. Confirm that the image does not contain the ConfigMap data, Secret value, or generated reports.
5. Create a public Docker Hub repository.
6. Publish the image with a clear tag.
7. Confirm that the image and selected tag are publicly available.
8. Record the complete image reference for use in Kubernetes.

Use these references:

- [Writing a Dockerfile](https://docs.docker.com/get-started/docker-concepts/building-images/writing-a-dockerfile/)
- [Build and push an image](https://docs.docker.com/get-started/introduction/build-and-push-first-image/)
- [Docker Hub repositories](https://docs.docker.com/docker-hub/repos/)

If you need to move source files into Killercoda, consult GitHub's [Cloning a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository) documentation.

### Part 4 — Prepare the Kubernetes environment and storage

1. Start the [Killercoda Kubernetes playground](https://killercoda.com/playgrounds/scenario/kubernetes).
2. Wait for the cluster to become ready.
3. Inspect the cluster and identify the node on which the batch workload will run.
4. Select a dedicated node directory for generated reports.
5. Prepare the directory with permissions that allow the application container to write its reports.
6. Plan how the Job and CronJob Pods will be scheduled onto the node containing this directory.
7. Plan the relationship between the node directory and the mount path used inside the container.

Use these references:

- [Kubernetes volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [hostPath volume](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
- [Configure a Pod to use a volume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/)
- [Assign Pods to nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

> **Storage limitation:** A node-directory volume is tied to a particular node. The reports will not automatically follow a workload scheduled onto another node. This is acceptable for this temporary learning activity but is not a production-grade persistent-storage design.

### Part 5 — Create the ConfigMap

1. Identify which application values are ordinary, non-sensitive configuration.
2. Create a ConfigMap containing the required report settings.
3. Make the ConfigMap available to the batch container by using a method supported by Kubernetes.
4. Ensure the names expected by the application agree with the keys defined in the ConfigMap.
5. Verify that the Pod can receive the configured values.

Use these references:

- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Configure a Pod to use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Define environment variables for a container](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)

### Part 6 — Create the Secret

1. Create a fictional credential for this activity. It must not be used by any real account or service.
2. Store it in a Kubernetes Secret rather than the ConfigMap or container image.
3. Make only the required Secret value available to the report-generator container.
4. Confirm that the application can determine whether the value is present.
5. Ensure that neither application output nor generated reports reveal the Secret value.

Use these references:

- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Distribute credentials securely using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
- [Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)

> **Security note:** Kubernetes Secrets are intended for confidential data, but they are not automatically encrypted in every cluster configuration. Encoding is not the same as encryption. Never place real credentials in this Killercoda exercise.

### Part 7 — Run the application once with a Job

1. Define a Job that uses the published report-generator image.
2. Connect the Job to the ConfigMap, Secret, and report volume.
3. Apply an appropriate restart and retry strategy.
4. Ensure the Job runs on the node associated with the selected report directory.
5. Start the Job and observe its lifecycle.
6. Verify that the Job reaches the completed state.
7. Inspect the output produced by the application.
8. Inspect the node directory and confirm that exactly one report file exists with the expected contents.
9. Confirm that the report does not contain the fictional credential.

Use these references:

- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Debug running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Viewing container logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)

Do not continue until the one-time Job works correctly. The CronJob will use the same workload design repeatedly.

### Part 8 — Schedule the application with a CronJob

1. Define a CronJob that runs the report generator frequently enough to observe multiple executions during the Killercoda session.
2. Connect it to the same ConfigMap, Secret, volume, and node-placement plan used by the successful Job.
3. Configure it so that report-generation runs do not overlap.
4. Select reasonable handling for missed schedules and retained execution history.
5. Create the CronJob and observe the Jobs and Pods it produces.
6. Wait until at least two scheduled Jobs complete successfully.
7. Verify that each scheduled run creates a distinct report file.
8. Compare the report timestamps or identifiers with the observed scheduled runs.

Use these references:

- [Kubernetes CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Running automated tasks with a CronJob](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/)

### Part 9 — Test configuration separation

1. Change one non-sensitive setting in the ConfigMap without rebuilding the container image.
2. Allow a new scheduled run to occur.
3. Verify that the new report reflects the changed configuration.
4. Confirm that previously generated reports remain unchanged.
5. Explain why configuration changes may not appear in an already running container immediately, depending on how the ConfigMap was consumed.

This test demonstrates why application settings should be separated from the container image.

### Part 10 — Test failure behavior

1. Introduce a controlled configuration problem, such as making one required value unavailable to a new test execution.
2. Observe how the application and Kubernetes report the failure.
3. Inspect the failed Pod and its output to identify the cause.
4. Restore the correct configuration.
5. Confirm that a later execution succeeds.

Do not expose the Secret value while troubleshooting.

## Expected outcome

At completion:

- the report-generator image is available in a public Docker Hub repository;
- the application reads ordinary settings from a ConfigMap;
- the application receives a fictional credential from a Secret without revealing it;
- a one-time Job completes successfully and generates a report;
- a CronJob creates at least two successful scheduled Jobs;
- scheduled runs do not overlap;
- each successful execution creates a distinct report in the mounted node directory;
- changing the ConfigMap affects a later report without rebuilding the image; and
- an intentionally failed execution can be diagnosed and corrected.

## Completion checklist

- [ ] I created a batch application that produces one report and exits.
- [ ] I built and tested its container image.
- [ ] I published the image to a public Docker Hub repository.
- [ ] I prepared a dedicated report directory on the selected Kubernetes node.
- [ ] I created a ConfigMap for non-sensitive report settings.
- [ ] I created a Secret containing only a fictional activity credential.
- [ ] The application uses the ConfigMap and Secret without embedding them in the image.
- [ ] The application does not reveal the Secret value.
- [ ] I mounted the node report directory into the workload container.
- [ ] I successfully completed a one-time Job.
- [ ] The one-time Job created the expected report file.
- [ ] I created a CronJob with a suitable schedule and concurrency policy.
- [ ] At least two scheduled Jobs completed successfully.
- [ ] Each successful run created a distinct report.
- [ ] I changed a ConfigMap value and observed it in a later report.
- [ ] I tested and diagnosed a controlled failure.
- [ ] I understand why node-local storage is unsuitable for many production workloads.

---

**Completion standard:** The activity is complete when the one-time Job and at least two scheduled Jobs finish successfully, each successful execution produces a distinct report in the mounted node directory, configuration comes from a ConfigMap, and the fictional credential comes from a Secret without being exposed.
