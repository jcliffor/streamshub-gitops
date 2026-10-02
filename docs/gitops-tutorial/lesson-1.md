+++
title = 'GitOps Lesson 1: Your first GitOps change'
+++

# Background

The [Introduction to GitOps](../introduction/_index.md) article explains how GitOps applies version control and continuous delivery (CD) principles to infrastructure management, treating operations in the same way as code. 
Instead of manually configuring resources through graphical interfaces, you define your system state in configuration files stored in a Git repository. 
This repository acts as the single source of truth for the system state, while automated processes synchronize the live environment with the repository definitions.

This tutorial demonstrates two core GitOps concepts, Infrastructure as Code (IaC) and the reconciliation loop, by using Kustomize and Argo CD.

## Core concepts

### Infrastructure as Code

Infrastructure as Code (IaC) is the practice of managing and provisioning infrastructure by using configuration files rather than manual web console procedures (often referred to as ClickOps). 
By treating infrastructure as code, you define the desired system state in configuration files and store them in a version control system, such as a Git repository. This approach transforms operations by enabling change tracking, team collaboration, and automated deployments.

Key benefits of adopting IaC include:

* **Configuration consistency**: Prevents configuration drift by ensuring environments rely on version-controlled files rather than manual changes.
* **Disaster recovery**: Simplifies environment recovery following a failure by reapplying version-controlled configurations.
* **Reproducible deployments**: Enables reliable, repeatable infrastructure deployments that produce consistent results.

### Kustomize

Kustomize is a configuration management tool integrated directly into the Kubernetes command-line client (`kubectl`) to simplify managing Kubernetes objects. 
It enables Infrastructure as Code (IaC) by defining a common base set of configuration files. You can then apply overlays to patch resources for specific environments, such as staging or production, without complex templating. 
This structure keeps configurations clean, consistent, and reproducible.

Throughout these lessons, you create, edit, and deploy Kustomize manifests to apply changes to the deployed cluster. 
You can learn more in the [official Kustomize documentation](https://kustomize.io/).

### The reconciliation loop

The reconciliation loop transforms IaC into running infrastructure. It continuously monitors the Git repository and automatically applies changes to keep live infrastructure synchronized with the desired state. 

Automating this process with tools like [Argo CD](https://argoproj.github.io/cd/) and [Flux](https://fluxcd.io/) eliminates manual intervention and prevents configuration drift.

### Argo CD

Argo CD is a Kubernetes-native continuous delivery (CD) tool that implements the reconciliation loop inside a cluster. 
It is the CD technology that you use throughout the lessons in this series.

To configure Argo CD, you create an `Application` custom resource that specifies the target Git repository, directory path, and destination namespace. 
Argo CD then polls the repository at regular intervals (every 3 minutes by default, though reduced to 30 seconds in this tutorial series for convenience), renders the manifests it finds, and synchronizes the cluster to match.

Argo CD exposes the state of this process through two indicators:

**Sync status**: Indicates whether the cluster matches the configuration in your Git repository. `Synced` means they match, whereas `OutOfSync` means Argo CD has detected a difference and prepares to apply it. 

In this lesson, you see this status transition when you push a change: it moves from `Synced` to `OutOfSync` (Argo CD noticed the new commit) and back to `Synced` (Argo CD applied the change). 

**Health status**: Indicates whether the resources themselves are functioning correctly. Lesson 3 explores health status in more detail.

## What to observe in the lesson

Now that you have reviewed the core concepts and technologies for this lesson, observe how these principles apply in practice during the exercise:

* When you edit `kustomization.yaml`, you declaratively change the desired state of the cluster.  
* When you run `git push`, you update the single source of truth. From this moment, the repository configuration specifies that a topic exists.  
* When the status in Argo CD transitions from `Synced` to `OutOfSync` and back to `Synced`, you observe the reconciliation loop complete a full cycle: detect the change, calculate the required changes, and roll them out.
  
You are now ready to work through the hands-on tutorial that follows.

# Tutorial: Lesson 1

## Learning objectives

After completing this lesson, you understand:

* How the GitOps workflow operates in practice
* How Argo CD monitors a Git repository and automatically applies changes to a Kubernetes cluster
* How Strimzi manages Kafka resources declaratively

You accomplish this by adding a Kafka topic as a real configuration change and observing it flow automatically from Git to a running cluster without manually running `kubectl apply`.

## Prerequisites

* You completed the steps in the [Preparing for the tutorials](setup.md) guide. You only need to complete this setup once.

## Understanding GitOps

In traditional operations, you make changes to a running system by executing commands directly against it, such as by using `kubectl apply`, a configuration panel, or an API call. GitOps reverses this approach: a Git repository serves as the single source of truth for the desired state of the system. A tool such as Argo CD monitors the repository and continuously reconciles the live system to match. 
If the configuration in the Git repository specifies that a topic exists, Argo CD creates the topic. If you remove the configuration from the repository, the topic is removed from the cluster. You never modify the live system directly; you only update the configuration in the repository. Through Git, you have a record of all changes made, when they were made, and by whom. You can also set up automated validation checks to run against changes before they are applied.


## Setup

Run the preparation script from this directory:

```bash
./prep.sh
```
   
This process takes less than 1 minute. The script resets the Gitea repository to the initial state for Lesson 1 and verifies that Argo CD is synchronized. When it finishes, the output displays the credentials for Gitea and Argo CD.

**Note:** You can re-run ./prep.sh at any time to reset the environment to the starting state for the lesson. This is useful if you need to restart the exercise.

## Part 1: Explore the initial environment

Before you make any changes, explore the initial environment deployed by Argo CD from the Git repository. 

### Clone the Git repository

The Gitea server runs inside the cluster. The output from the `./prep.sh` script includes the specific git clone command for your environment. 

1. Clone the Git repository:

   ```bash
   git clone <external_address_of_gitea_server> /tmp/gitops-lesson-1
   ```
2. Change to the newly cloned repository directory:
   
   ```bash
   cd /tmp/gitops-lesson-1
   ```
   This is the Git repository that Argo CD monitors. Any changes that you push to this repository are automatically synchronized to the cluster.

### Check the running Kafka cluster

1. Verify that the Kafka cluster is running in the `kafka-tutorial` namespace:
   
   ```bash
   kubectl get kafka -n kafka-tutorial
   ```

   **Expected output**:

   ```
   NAME         READY    WARNINGS    KAFKA VERSION    METADATA VERSION
   my-cluster   True                 4.2.0            4.2-IV0
   ```
   A value of `True` in the `READY` column indicates that the Kafka cluster is operational.

2. Inspect the manifest that defines the Kafka cluster:

   ```bash
   cat manifests/kafka.yaml
   ```

   This YAML file, committed to the Git repository, contains the declarative configuration for the Kafka cluster. Argo CD reads the file from the repository and applies it to the Kubernetes cluster, where the Strimzi operator uses the file to create the Kafka cluster. You do not need to run `kubectl apply` commands; the setup script pushes the configuration file to the Git repository, and Argo CD manages the deployment.


### Check for existing Kafka topics

Verify whether any Kafka topics exist in the `kafka-tutorial` namespace:
   
```bash
kubectl get kafkatopic -n kafka-tutorial
```
   
The output indicates that no Kafka topics are currently deployed in the cluster.

### Understand the kustomization file

Argo CD uses [Kustomize](https://kustomize.io/) to determine which YAML files to deploy. The entry point is `manifests/kustomization.yaml`.

1. Display the contents of `manifests/kustomization.yaml`:

   ```bash
   cat manifests/kustomization.yaml
   ```
   **Example YAML output**:

   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - namespace.yaml
     - combined-pool.yaml
     - kafka.yaml
   ```
   This configuration instructs Argo CD to deploy the configuration in the three files specified in the resource list to Kubernetes. Notice that `topic.yaml` is not listed, even though the file exists in the repository.

2. List all files in the `manifests/` directory:

   ```bash
   ls manifests/
   ```
   Although `topic.yaml` exists in the `manifests/` directory, Argo CD ignores it because it is not included in `kustomization.yaml`. The cluster state is determined exclusively by the files declared in the Kustomize configuration, rather than all files present in the directory.

## Part 2: Make your first GitOps change

Your application team needs a Kafka topic to send and receive messages. Your task is to add it to the cluster.

### View the topic definition

Display the contents of `manifests/topic.yaml`:

```bash
cat manifests/topic.yaml
```

**Example YAML output**:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaTopic
metadata:
  name: my-first-topic
  namespace: kafka-tutorial
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 3
  replicas: 1
  config:
    retention.ms: "86400000"
    segment.bytes: "1073741824"
```

This manifest defines a Kafka topic named `my-first-topic` with `3` partitions. The `strimzi.io/cluster: my-cluster` label directs the Strimzi Operator to deploy the topic to `my-cluster`. The `retention.ms` setting retains messages for 24 hours (`86400000` ms).

The manifest is complete. To deploy the Kafka topic, include `topic.yaml` in the `kustomization.yaml` manifest.

### Edit the `kustomization.yaml` file

1. Open `manifests/kustomization.yaml` in your text editor and add `- topic.yaml` as the last entry in the resources list:

   **Example updated file**

   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - namespace.yaml
     - combined-pool.yaml
     - kafka.yaml
     - topic.yaml
   ```
   
2. Save the file.

### Commit and push changes to the repository

1. Stage the updated `kustomization.yaml` file:
   
   ```bash
   git add manifests/kustomization.yaml
   ```
2. Commit the changes to the Git repository:

   ```bash
   git commit -m "Add my-first-topic Kafka topic"
   ```
3. Push the commit to the Git repository:
   
   ```bash
   git push
   ```

Your first GitOps change is complete. The commit is in the Git repository monitored by Argo CD.


## Part 3: Observe the GitOps loop

Argo CD polls the Git repository every 30 seconds to reconcile cluster state. 

1. Monitor the Argo CD application status:

   ```bash
   kubectl get application kafka-tutorial -n argocd -w
   ```
   
2. Observe the `SYNC STATUS` column. Within approximately 30 seconds, the status changes from `Synced` to `OutOfSync` when Argo CD detects the Git push, and then returns to `Synced` once the changes are applied.

3. Press Ctrl+C once you see the status return to `Synced`.

### Verify the topic was created on the cluster

Once the Argo CD status shows `Synced`, check that the topic now exists:

```bash
kubectl get kafkatopic my-first-topic -n kafka-tutorial
```

**Expected output:**

```
NAME             CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
my-first-topic   my-cluster   3            1                    True
```

A `READY` value of `True` confirms that Strimzi's Topic Operator received the `KafkaTopic` resource from Argo CD and created the topic inside the Kafka broker.

## GitOps reconciliation process

The following sequence describes the process that occurs when you execute `git push`: 

1. Gitea (the Git server inside the cluster) receives the push.
2. Argo CD polls Gitea every 30 seconds to check for updates.
3. Argo CD detects that `kustomization.yaml` includes `topic.yaml`.
4. Argo CD renders the Kustomize manifests (updating the resource count from 3 to 4).
5. Argo CD compares the rendered state against the live cluster state and applies the difference, creating the `KafkaTopic` resource.
6. The Strimzi Operator detects the new `KafkaTopic` resource and creates the topic inside the Kafka broker.
   
Without running `kubectl apply`, updating the Git repository causes the system to automatically reconcile its live state to match the target configuration.

## Optional: View the Argo CD dashboard

You can use the Argo CD web console to view the application resource tree, synchronization history, and cluster state. 

1. Port-forward the Argo CD server service in a separate terminal window:

   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
   
2. Open [https://localhost:8080](https://localhost:8080) in your browser (accept the self-signed certificate warning).

3. Retrieve the administrator password:

   ```bash
   kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d; echo
   ```
   
4. Log in with username `admin` and the retrieved password.
   
5. Select the `kafka-tutorial` application to view the resource tree, which displays the `Namespace`, `KafkaNodePool`, `Kafka`, and `KafkaTopic` resources managed by Argo CD from a single Git repository.


## Troubleshooting

### Infrastructure is not running
If `./prep.sh` reports that the cluster or Kafka is not found, run the setup script: 

```bash
../00-setup/setup.sh
```

See [Getting Started guide](../00-setup/README.md) for setup troubleshooting.

### Kafka cluster is not becoming ready
Kafka requires several minutes to start, particularly in environments with limited resources.

1. Check the pod status in the `kafka-tutorial` namespace:

   ```bash
   kubectl get pods -n kafka-tutorial
   ```
   
2. Inspect the events for the `my-cluster` Kafka cluster:

   ```bash
   kubectl describe kafka my-cluster -n kafka-tutorial
   ```

### Argo CD application is not syncing

1. Check the application for error messages:

   ```bash
   kubectl get application kafka-tutorial -n argocd -o yaml
   ```

2. If Gitea is unreachable from inside the cluster, verify that the Gitea pod is running:

   ```bash
   kubectl get pods -n gitea
   ```

### Topic is not appearing after sync

1. Check the recent Git commit log:

   ```bash
   git log --oneline -3
   ```
   
2. Inspect the `kustomization.yaml` manifest in the latest commit:

   ```bash
   git show HEAD:manifests/kustomization.yaml
   ```
   
3. Confirm that `- topic.yaml` appears in the `resources` list. If it does not appear, edit, commit, and push the file again.


### Next steps

In [Lesson 2](lesson-2.md), you build on this setup by creating separate staging and production configurations. You then learn how to promote changes between environments while applying Git source-of-truth principles to multi-environment workflows.
