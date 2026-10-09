+++
title = 'GitOps Lesson 2: Promoting Changes Across Environments'
+++

# Background

In Lesson 1, you edited your cluster configuration and pushed the updates to Git. 
Argo CD then automatically reconciled and deployed the changes. 
This workflow forms the foundation of GitOps: you define infrastructure as code, and automation keeps your live cluster aligned with that target state.

As your project grows, pushing changes directly to production creates deployment risk. 
To ensure system stability, you need a way to validate configuration changes in a staging environment before promoting them to production. 

The core challenge is keeping staging and production environments synchronized. 
While these environments require intentional parameter differences, such as replica counts or resource quotas, their underlying configurations must remain identical.
If the environments diverge, a successful test in staging no longer guarantees a predictable outcome in production. 
Relying on manual console changes makes configuration divergence inevitable. 
Without an automated GitOps pipeline, minor discrepancies accumulate over time into untracked configuration drift.

In a GitOps workflow, you define your infrastructure as configuration files managed within a Git repository. 
This repository serves as the single source of truth for your environments, eliminating manual, error-prone processes that cause configuration drift. 

In this lesson, you use Kustomize overlays to structure multi-environment configurations and configure Argo CD to manage them independently from a single repository. 
You practice environment promotion without running manual deployment commands. Instead, promoting a change involves modifying configuration files and committing them to Git to update the desired state for your target environment.

## Core concepts

### Multi-environment configuration

Previously, you explored why organizations maintain multiple environments and how GitOps helps you avoid configuration drift by defining your environments in configuration files. 
However, simply creating separate configuration files for each environment is still problematic. 
Maintaining multiple files is error-prone because updating a shared cluster definition requires you to modify every copy independently.
If you miss a single update, you introduce the configuration drift you intend to avoid. 
Instead, you need to define your shared configuration once and layer environment-specific differences on top.

## The Kustomize base and overlay pattern

To eliminate configuration duplication, you can organize your application manifests by using Kustomize. 
Kustomize addresses this challenge through its base and overlay pattern. 
Consider the base as your primary configuration definition for your environments. 
It contains all the shared configuration that every environment needs. 
You define your core cluster configuration here, exactly once.
Next, you construct your overlays. 
These are separate directories for each environment, such as staging or production. 
An overlay references the base configuration and then layers on only the differences. 
If you need a different namespace or extra scaling parameters for production, you define those specific overrides in that environment's overlay.

Each overlay directory includes a `kustomization.yaml` file that instructs Kustomize how to combine your resources. 
When it runs, Kustomize takes the base manifests, injects the environment-specific settings, and generates the final configuration.

Key benefits of the base and overlay pattern:

* **Eliminates configuration duplication:** Because shared resources live in the base, you update them once and the change flows to every environment automatically.  
* **Provides clear environment customization:** Each overlay contains only the specific differences for that environment. You can see at a glance exactly how staging differs from production.  
* **Simplifies environment scaling:** To add a new environment, create a new overlay directory and a matching Argo CD `Application` resource. 
  You do not need to copy entire sets of files or build a complex new pipeline.

## Promotion as a Git commit

As you practiced in Lesson 1, you change cluster state by updating configuration files and pushing them to your Git repository. Environment promotion follows the same workflow: you update the production overlay and commit the changes to Git. This approach ensures that every promotion is auditable and reviewable. In Lesson 3, you explore why auditability is important.

## Multiple Argo CD applications

Argo CD supports multi-environment deployments through separate `Application` resources, each configured to watch a different directory path in the same Git repository and deploy to a distinct namespace or cluster. 
In this exercise, you configure a separate `Application` resource for staging and production. 
Argo CD evaluates each `Application` resource independently on every poll cycle.
This independence provides **environment isolation**. 
When you push a commit that adds a resource to the production overlay, only the production `Application` resource detects a change and triggers a rollout. 
Changes to one environment cannot affect another, because the scope of each `Application` resource is limited to its own overlay directory and target namespace.

## What to observe in the lesson

Now that you have explored the core concepts and technologies behind environment promotion, look for these key observations during the lesson:

* When you explore the `manifests/` directory and view `base/`, `overlays/staging/`, and `overlays/production/`, you see the Kustomize base and overlay pattern in practice: shared configuration in `base/`, with environment-specific layers on top.
* When you copy `topic.yaml` into the `production` overlay, add it to `kustomization.yaml`, and run `git push`, you perform a GitOps promotion. The commit that updates the target environment's desired state is the only required deployment action.
* When Argo CD syncs the `kafka-production` `Application` resource while `kafka-staging` remains unchanged, you observe environment isolation. Because each `Application` resource independently watches its own overlay path, changes to one environment do not affect another.

You are now ready to work through the hands-on tutorial that follows.

# Tutorial: Lesson 2

## Learning objectives

After completing this lesson, you understand:

* How **Kustomize overlays** share a `base` configuration and layer environment-specific differences on top
* How Argo CD manages multiple `Application` resources from a single Git repository, each watching a different path
* How promoting a change from staging to production is simply a configuration update pushed to your Git repository

You accomplish this by observing a staging environment with a deployed Kafka topic, then promoting that topic to production by copying it into the production overlay and pushing to Git.

## Prerequisites

* You completed the steps in the [Preparing for the tutorials](setup.md) guide. You only need to complete this setup once.


## Why environments matter

In Lesson 1, you made a single change in the configuration hosted in the Git repository and watched Argo CD apply it to the cluster automatically. In practice, organizations do not push changes directly to production. Instead, they promote changes through a chain of environments: developers push to staging first, validate the change, then promote to production.

The key insight: In a GitOps workflow, promoting a change between environments is not a deploy command. It is a configuration change in a Git repository. You define the desired state for each environment in the repository, and Argo CD continuously reconciles the cluster to match. Promoting a change simply means updating the target environment's desired state and pushing the commit.

**Kustomize overlays** are the Kubernetes-native mechanism for managing configurations. You keep a shared base configuration and then have one overlay per environment that references the base configuration and adds or patches environment-specific resources. Argo CD points a separate `Application` resource at each overlay.

## Setup

Run the preparation script from this directory:

```bash
./prep.sh
```

This process takes approximately 5 minutes and performs the following actions:

1. Removes the Lesson 1 state from the cluster
2. Seeds the Gitea repository with the multi-environment overlay structure
3. Creates two Argo CD `Application` resources, one for staging and one for production
4. Waits for both Kafka clusters to become ready

When the script finishes, the output displays the credentials for Gitea and Argo CD.

**Note:** You can rerun ./prep.sh at any time to reset the environment to the starting state for the lesson. This is useful if you need to restart the exercise.

## Part 1: Explore the environment

Before you make any changes, explore the initial environment deployed by Argo CD from the Git repository.

### Clone the repository

The Gitea server runs inside the cluster. The output from the `./prep.sh` script includes the specific `git clone` command for your environment. 

1. Clone the Git repository:
   
   ```bash
   git clone <gitea_server_address> /tmp/gitops-lesson-2
   ```

2. Change to the newly cloned repository directory:

   ```bash
   cd /tmp/gitops-lesson-2
   ```

### Check for active Kafka clusters and topics across environments

1. Verify that a Kafka cluster is running in the `kafka-staging` namespace:
   
   ```bash
   kubectl get kafka -n kafka-staging
   ```

2. Verify that a Kafka cluster is running in the `kafka-production` namespace:

   ```bash
   kubectl get kafka -n kafka-production
   ```

   Both commands return `READY: True`, confirming that an independent Kafka cluster is running in each namespace.

3. Check for existing Kafka topics in the `kafka-staging` namespace:

   ```bash
   kubectl get kafkatopic -n kafka-staging
   ```

4. Check for existing Kafka topics in the `kafka-production` namespace:

   ```bash
   kubectl get kafkatopic -n kafka-production
   ```

   The output displays `my-first-topic` in the `kafka-staging` namespace and no resources in the `kafka-production` namespace, establishing the starting state where staging is ahead of production.
   
     
### Check the Argo CD applications

1. List the applications in the `argocd` namespace:
   
   ```bash
   kubectl get application -n argocd
   ```
   
   The output displays two `Application` resources: `kafka-staging` and `kafka-production`. Each resource watches a different path in the same Git repository, and each deploys to a different namespace.

## Part 2: Explore the overlay structure

Display the directory tree of the `manifests/` folder:

```bash
tree manifests/
```

The command output displays the following directory layout:

```
manifests/
├── base/
│   ├── kustomization.yaml
│   ├── kafka.yaml
│   └── combined-pool.yaml
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   └── topic.yaml
    └── production/
        ├── kustomization.yaml
        └── namespace.yaml
```

### The base configuration

Display the contents of `manifests/base/kustomization.yaml`:

```bash
cat manifests/base/kustomization.yaml
```

Example YAML output:
   
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - combined-pool.yaml
  - kafka.yaml
```

The `base` directory contains the shared Kafka cluster definition without any namespace or environment-specific configuration, which comes from the `overlays` directory.

### The staging overlay

Display the contents of `manifests/overlays/staging/kustomization.yaml`:

```bash
cat manifests/overlays/staging/kustomization.yaml
```
Example YAML output:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: kafka-staging
resources:
  - ../../base
  - namespace.yaml
  - topic.yaml
```

The staging overlay performs the following configurations:
* Sets `namespace: kafka-staging` across all base resources.
* Includes the shared base Kafka configuration.
* Adds its own `namespace.yaml` to create the `kafka-staging` namespace.
* Adds `topic.yaml` to define the Kafka topic.

The Argo CD `kafka-staging` `Application` resource points to this directory. When Kustomize renders it, Argo CD receives the full set of resources: `namespace`, `Kafka` cluster, `KafkaNodePool`, and topic, all in the `kafka-staging` namespace.

To render the full overlay and verify what Argo CD applies, run the following command:

```bash
kubectl kustomize manifests/overlays/staging
```

Every resource in the output has `namespace: kafka-staging` injected, demonstrating how Kustomize combines the base configuration with the overlay.

### The production overlay

Display the contents of `manifests/overlays/production/kustomization.yaml`:

```bash
cat manifests/overlays/production/kustomization.yaml
```
Example YAML output:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: kafka-production
resources:
  - ../../base
  - namespace.yaml
```

The `topic.yaml` file is omitted from both the `resources` list and the overlay directory. As a result, the topic exists only in the staging overlay. Although the production Kafka cluster is running, no topic has been promoted to production yet.

## Part 3: Promote the topic to production

Your staging team has validated `my-first-topic` and it is ready for production. Promoting it means copying the topic definition from the staging overlay into the production overlay and adding it to the production `kustomization.yaml` file in a single atomic commit.

1. Copy the topic definition from staging into production:

   ```bash
   cp manifests/overlays/staging/topic.yaml manifests/overlays/production/topic.yaml
   ```

2. Open `manifests/overlays/production/kustomization.yaml` in your editor and add `- topic.yaml` to the `resources` list:

   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   namespace: kafka-production
   resources:
     - ../../base
     - namespace.yaml
     - topic.yaml
   ```

3. Stage the changes:

   ```bash
   git add manifests/overlays/production/
   ```

4. Commit the changes:

   ```bash
   git commit -m "Promote my-first-topic to production"
   ```

5. Push the commit to the repository:

   ```bash
   git push
   ```

This commit completes the promotion. The topic definition and the Kustomize entry arrive together, just as they would in a real pull request. By updating the configuration in the Git repository, the system automatically reconciles to match.

## Part 4: Watch both Argo CD applications

Argo CD polls the Git repository every 30 seconds to reconcile cluster state. 

1. Watch the production application detect and apply your change:

   ```bash
   kubectl get application kafka-production -n argocd -w
   ```

2. Observe the `SYNC STATUS` column. Within approximately 30 seconds, the status changes from `Synced` to `OutOfSync` when Argo CD detects the Git push, and then returns to `Synced` once the changes are applied.

3. Press Ctrl+C once you see the status return to `Synced`.

Because you updated only the production overlay, only the production application reconciles.

### Verify that the topic is in production

1. Once `kafka-production` shows `Synced`, verify that the Kafka topic exists in the `kafka-production` namespace:

   ```bash
   kubectl get kafkatopic -n kafka-production
   ```

   Expected output:

   ```
   NAME             CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
   my-first-topic   my-cluster   3            1                    True
   ```

2. Confirm that the staging environment remains unchanged:

   ```bash
   kubectl get kafkatopic -n kafka-staging
   ```
   
   The topic configuration in the staging environment remains identical. The promotion copied the resource and updated the `kustomization.yaml` file in the production overlay only, combining both changes into a single atomic Git commit.

## GitOps reconciliation process

The following sequence describes the process that occurs when you execute `git push`: 

1. Gitea (the Git server inside the cluster) receives the push.
2. Argo CD polls Gitea every 30 seconds to check for updates.
3. Argo CD does not detect configuration changes for the `kafka-staging` `Application` (`manifests/overlays/staging`). The status remains `Synced`.
4. Argo CD detects that the `kustomization.yaml` file for the `kafka-production` `Application` (`manifests/overlays/production`) includes `topic.yaml`.
5. Argo CD applies the manifest differences to the `kafka-production` namespace, creating the `KafkaTopic` resource.
6. The Strimzi Operator detects the new `KafkaTopic` resource and creates the topic inside the `kafka-production` Kafka broker.
   
Each application is independent. Changes to one overlay do not affect the other. The configuration in the Git repository is the source of truth for both environments, and the overlay structure makes it clear exactly what each environment contains.

## View the Argo CD dashboard (optional)

1. Port-forward the Argo CD server service in a separate terminal window:

   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
2. Open `https://localhost:8080` in your browser (accept the self-signed certificate warning).

3. Retrieve the administrator password:

   ```bash
   kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d; echo
   ```

4. Log in using `admin` as the username and the retrieved password.

   Both the `kafka-staging` and `kafka-production` applications are displayed. 

5. Select each application to view its resource tree.

   Both applications share the same base resources (`Namespace`, `KafkaNodePool`, and `Kafka`), but only `kafka-production` includes the `KafkaTopic` resource.

## Configure environment-specific resources (optional)

In the previous procedure, you promoted `my-first-topic` to production with the same configuration as staging (3 partitions). In production environments, topics often require different configurations, such as higher partition counts for throughput, longer retention periods, or increased replication.

You can customize resources in an overlay by editing the overlay copy of the file.

1. Open `manifests/overlays/production/topic.yaml` and increase the partition count to match production-scale requirements:

   ```yaml
   apiVersion: kafka.strimzi.io/v1
   kind: KafkaTopic
   metadata:
     name: my-first-topic
     labels:
       strimzi.io/cluster: my-cluster
   spec:
     partitions: 10
     replicas: 1
     config:
       retention.ms: "86400000"
       segment.bytes: "1073741824"
   ```

2. Stage the changes:

   ```bash
   git add manifests/overlays/production/topic.yaml
   ```

3. Commit the changes:

   ```bash
   git commit -m "Set production topic to 10 partitions"
   ```

4. Push the commit to the repository:

   ```bash
   git push
   ```

5. After Argo CD completes reconciliation, verify the partition count in each environment:

   ```bash
   kubectl get kafkatopic my-first-topic -n kafka-production -o jsonpath='{.spec.partitions}'; echo
   kubectl get kafkatopic my-first-topic -n kafka-staging -o jsonpath='{.spec.partitions}'; echo
   ```
The output displays `10` for production and `3` for staging. Environments are independently configurable, so changes to one overlay do not affect other overlays.

## Key takeaways

* Kustomize overlays allow you to share a base configuration and apply environment-specific changes without duplicating files.
* Argo CD can manage multiple `Application` resources from a single Git repository, with each resource watching a distinct directory path.
* Promotion is a configuration change: copying a resource into the target overlay directory, adding it to the `kustomization.yaml` file, and executing a Git commit and push.
* Environments are isolated from each other, so a change to one overlay does not affect other overlays.
* In production, you typically use separate clusters or Argo CD instances for each environment. The underlying promotion principle remains identical: a configuration change pushed to your Git repository triggers the synchronization process.

## Troubleshooting

If `./prep.sh` reports that the cluster or Strimzi is not found, run the setup script: 

```bash
../00-setup/setup.sh
```

### Kafka clusters fail to reach a ready state

Both clusters start in parallel. 

1. Check the pod status in the `kafka-staging` namespace:

   ```bash
   kubectl get pods -n kafka-staging
   ```

2. Check the pod status in the `kafka-production` namespace:

   ```bash
   kubectl get pods -n kafka-production
   ```
   
If the pods remain in the `Pending` state, Docker memory might be insufficient. This lesson requires approximately 8 GB of memory. Check Docker Desktop memory settings.

### Argo CD application is not syncing

1. Check the application for error messages:

```bash
kubectl get application kafka-production -n argocd -o yaml
```

2. If Gitea is unreachable from inside the cluster, verify that the Gitea pod is running:

```bash
kubectl get pods -n gitea
```

### Topic is not appearing after sync  

1. Confirm that both files were committed to the repository:

   ```bash
   git log --oneline -3
   ```

2. Inspect the `kustomization.yaml` file in the latest commit:

   ```bash
   git show HEAD:manifests/overlays/production/kustomization.yaml
   ```

3. Confirm that the `- topic.yaml` entry appears in the `resources` list and that the `topic.yaml` file exists in the repository:

   ```bash
   git show HEAD:manifests/overlays/production/topic.yaml
   ```

### Strimzi Cluster Operator fails to manage Kafka clusters  

1. Check that the Operator deployment is running:

   ```bash
   kubectl get deployment strimzi-cluster-operator -n strimzi-operator
   ```
   
2. Verify that the Operator is configured to watch all namespaces:
   
   ```bash
   kubectl logs deployment/strimzi-cluster-operator -n strimzi-operator | grep STRIMZI_NAMESPACE
   ```

   The command output displays `STRIMZI_NAMESPACE` set to `*`. If the operator is not running, re-run the `../00-setup/setup.sh` script.

## Next steps

In [Lesson 3: Reverting changes](lesson-3.md), you use `git revert` to undo a broken configuration that reached production, and observe how the automated reconciliation process restores the cluster to the last known good state.
