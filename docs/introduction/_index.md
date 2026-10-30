+++
title = 'Introduction to GitOps'
+++

Most of us start managing infrastructure by clicking through web user interfaces and management consoles. 
It works well initially and is easy to learn. However, as your systems scale and teams grow, 
manual configuration management introduces risk, inconsistency, and operational friction. 

This document explains manual console management (often called ClickOps), why it creates problems in 
production environments, and how GitOps provides a reliable, automated alternative.

## What is ClickOps?

ClickOps refers to managing infrastructure manually by clicking through web consoles and administration interfaces. 
For example, if you need a new Apache Kafka topic, you log in to the web console, fill in a form, and select **Create**. 
If you need to grant user access to a namespace, you navigate to Identity and Access Management (IAM) settings, find the role,
assign it, and save the changes.

While manual console management works well for learning or running quick experiments, relying on manual steps to manage 
production infrastructure creates difficulties.

## The problems with ClickOps

### Drift and inconsistency

When engineers make infrastructure changes manually, they introduce subtle variations. 
Two engineers configuring identical components might select slightly different parameter values, miss required labels, or omit specific setup steps. 
Over time, staging and production environments diverge, creating configuration drift that is difficult to detect and troubleshoot.

### Audit logs without context

Most enterprise platforms provide audit logs. Kubernetes records API events, cloud providers offer tools such as AWS CloudTrail, and Red Hat OpenShift tracks event history.
However, these logs record low-level API calls, such as `user X updated ConfigMap Y at 14:32`. They do not record why an engineer made the change. 
To understand the context behind an update, you must search through ticket queues, chat histories, and audit logs while trying to contact team members.

### Reproducibility depends on discipline

If a cluster fails and you need to re-create it, or if you need to provision a new cluster that matches a known-good configuration, how do you do it? 
With ClickOps, success depends on whether your team accurately documented every manual step. 
In practice, the running system is often the only true record of configuration settings. 
If an environment is lost, re-creating it requires piecing together historical audit logs, backups, and what colleagues remember.
Even if the original environment is still running, how can you be sure you have found all the custom configurations that were applied?

### Rollbacks require manual effort

Platforms such as Kubernetes support automated rollbacks for specific resources using commands including kubectl rollout undo.
However, manual changes to Kafka topic configurations, config maps, or role-based access control (RBAC) policies cannot be reverted automatically.
Instead, you must determine the previous values and reapply them line by line.

## ClickOps in practice

While these issues might seem abstract, they cause real disruption in daily operations. Here is how they look in practice:

### Scenario: Deploying an identical Kafka cluster
A team maintains an Apache Kafka cluster in staging that required weeks of fine-tuning. The brokers use specific resource limits, topics use custom retention and replication parameters, and Kafka Connect connectors integrate external data sources.

A team lead asks a platform engineer to deploy a Kafka cluster in a new production namespace using the exact configuration from staging. 
The engineer opens the staging web console, steps through each resource, and manually notes down the settings. 
They then create the new cluster in production, entering each parameter value by hand.

In the process, the engineer introduces minor errors:
* Omits broker JVM settings by overlooking a configuration panel.
* Creates a topic with a replication factor of 1 instead of 3 due to a default setting.
* Points a Kafka Connect instance to the staging database endpoint because an external URL was copied without updating the target host.

These discrepancies cause problems in the following weeks, including data loss during broker restarts, incorrect database routing, and unpredictable cluster performance under heavy load.

### Scenario: Provisioning identical user access
A new developer requires the same access rights to Kafka and Kubernetes as an existing developer. The team lead asks an administrator to set it up.
The administrator manually inspects the existing user profile in the management console and creates matching role bindings. 
They check the `KafkaUser` resources and try to copy the access control lists (ACLs): read access to some topics, write access to others, and permissions on consumer groups.

A week later, the new developer can produce messages to a topic but cannot consume them because the administrator missed a consumer group ACL. 
Soon after, a security audit reveals that the administrator inadvertently granted the developer write permissions to a production topic due to a misclick in a dropdown menu.

### Scenario: Investigating recent changes

On Monday morning, the on-call engineer notices that an event processing pipeline is dropping messages. An engineer who modified the pipeline on Friday is currently on vacation.

The on-call engineer inspects the environment to locate the issue. Kubernetes audit logs show multiple `PATCH` operations applied to config maps on Friday, but provide no explanatory context. 
The Kafka console displays current topic retention settings, but does not indicate whether `retention.ms` was recently changed.

After three hours of comparing chat logs, audit entries, and old screenshots, the engineer discovers two independent modifications made by a colleague:
* Reduced `max.poll.records` in a consumer application config map to prevent connection timeouts.
* Decreased `retention.ms` on a topic as part of routine cleanup.

Together, these changes cause consumer processing to fall behind, resulting in messages expiring before processing completes. 

Now the engineer needs to roll back the previous changes, but to what values? The engineer is not sure, so they message the colleague on vacation.
The colleague responds hours later with values from memory. 

### Summary of manual management challenges

Each of these scenarios demonstrates the same fundamental problem: the system lacks a single, version-controlled source of truth where configurations are declared, reviewed, and audited. 
GitOps resolves these challenges. 

## What is GitOps?

GitOps is an operational framework for managing infrastructure where desired system states are defined in configuration files stored in a Git repository. A controller monitors the repository and synchronizes the live cluster to match the declared state in Git.

Core principles of GitOps:

* **Declarative configuration:** You describe infrastructure in version-controlled formats, such as YAML or JSON manifests, not as a list of manual steps.
* **Git as the single source of truth:** The Git repository serves as the definitive record of the desired system state. Every configuration change is submitted as a commit or pull request, providing a complete, auditable history of modifications and context.
* **Automated reconciliation:** A controller monitors the Git repository and automatically applies declared updates to the target environment, ensuring that the live infrastructure remains synchronized with Git. The controller can manage its local cluster or centrally manage remote clusters.

The following diagram illustrates how this process works in practice. Changes enter the Git repository through pull requests. A continuous deployment (CD) system monitors the repository, detects updates, and synchronizes resources to each environment.

![](./assets/gitops-flow.svg)

## How GitOps solves production challenges

### Deploying an identical Kafka cluster

With GitOps, cluster configurations are defined as declarative YAML manifests stored in a Git repository. Shared parameters, such as resource limits, JVM settings, and replication factors, are maintained in a common base configuration. Environment-specific variations, such as endpoints and namespaces, are applied by overlaying specific values using systems such as [Kustomize](https://kustomize.io/).

Kustomize is a configuration management utility built into the Kubernetes command-line client (`kubectl`).
It enables you to declare a shared base set of Kubernetes manifests and layer environment-specific overlays on top without duplicating configuration files. 
For example, a base Kafka manifest lives in `base/`, while production-specific settings are applied in `overlays/prod/`. 

All modifications are reviewed through Git pull requests before deployment, preventing configuration errors from reaching production.

### Provisioning identical user access

With GitOps, user permissions are defined in `KafkaUser` custom resources and Kubernetes `RoleBinding` manifests checked into the Git repository. 
Provisioning access for a new team member involves copying an existing manifest, updating the user identity field, and opening a pull request. 
Peers review the pull request before merging, ensuring that access policies remain accurate and auditable.
Because the new manifest reuses the structure of an existing file with only the user identity updated, permissions remain consistent by definition rather than through manual effort.

### Investigating recent changes

With GitOps, the on-call engineer can check the Git commit history when investigating a problem.
Every commit includes a descriptive message that explains what was modified and why. 
If a change causes an issue, reverting to the last known-good state requires running `git revert` and pushing the commit to the repository. 
The controller detects the push and automatically reconciles the cluster state.
The entire process requires significantly less effort for incident responders and takes far less time.

### Comparing ClickOps and GitOps

| Management challenge | ClickOps approach | GitOps approach |
| :--- | :--- | :--- |
| **Change context** | Audit logs show what happened, not why. | Git commits include detailed messages explaining why changes were made. |
| **Environment consistency** | Environments drift over time due to manual modifications. | Declarative base configurations and overlays keep environments aligned. |
| **System reproducibility** | Rebuilding environments depends on manual documentation. | Complete system states can be redeployed directly from Git. |
| **Rollback capabilities** | Rollbacks require manual, resource-by-resource updates. | Running `git revert` restores the previous state across all resources. |

## Common GitOps technologies

There are two main tools for GitOps in the Kubernetes ecosystem:

### Argo CD

[Argo CD](https://argo-cd.readthedocs.io/) is a Kubernetes-native continuous delivery tool that monitors Git repositories and reconciles live resources to match the target state defined in Git. Argo CD can manage the local cluster where it is installed, but it can also manage remote clusters, enabling a central instance to deploy across multiple target environments.

Argo CD provides both a command-line interface (CLI) for automation and a web-based console that displays the state of all your applications, their synchronization status, and resource health in real time.
The dashboard makes it easy to monitor what is deployed, identify problems, and see the difference between what is in Git and what is running in the cluster.

Key features of Argo CD include:
* Architecture built on Kubernetes custom resource definitions (CRDs).
* Web-based console providing real-time visibility into application sync and health status.
* CLI for automation and scripting.
* Native integration with Kustomize, Helm, and standard Kubernetes YAML manifests.
* Automated or manual reconciliation with automated retry and self-healing mechanisms.
* Single sign-on (SSO) integration and role-based access control (RBAC).

### Flux
[Flux](https://fluxcd.io/) is a Cloud Native Computing Foundation (CNCF) project for GitOps on Kubernetes. Flux uses in-cluster controllers to continuously reconcile cluster resources against Git repositories.

Key features of Flux include:
* Architecture built on Kubernetes custom resource definitions (CRDs).
* Support for Kustomize, Helm, and standard Kubernetes manifests.
* Automatic updates to Git manifests when new container images are published.
* Multitenancy support managed at the namespace level.
* Notification system for reconciliation events.

### Selecting a GitOps tool

Both Argo CD and Flux are mature, production-ready open source tools that support declarative GitOps workflows. 
Argo CD includes an integrated web dashboard that simplifies visual monitoring for teams adopting GitOps. Flux does not include a native dashboard, but external UIs such as [Capacitor](https://fluxcd.io/blog/2024/02/introducing-capacitor/) can provide similar visibility. 

This tutorial series uses Argo CD during hands-on exercises.

## Next steps

### Try the interactive GitOps tutorial

To see these principles in action, explore the interactive [GitOps Tutorial](../gitops-tutorial/_index.md) series. This hands-on series guides you through managing streaming infrastructure on a local Kubernetes cluster by using Argo CD and the Strimzi Operator:

* Lesson 1: Deploy your first GitOps change, observe automated reconciliation, and learn how Argo CD synchronizes cluster state.
* Lesson 2: Manage multi-environment configurations using Kustomize base and overlay patterns, and promote changes from staging to production.
* Lesson 3: Handle operational failures by using `git revert` to roll back invalid changes and restore the cluster to a known-good state.

Setup requires approximately 8 minutes, providing practical experience with standard GitOps workflows: editing configuration, committing changes, and pushing to Git.

### Documentation and examples
Beyond the tutorial, the rest of this documentation covers practical guides and examples for managing streaming infrastructure using GitOps. Examples are provided for common scenarios such as the following:

* Deploying Kafka clusters.
* Setting up OAuth-secured topics.
* Configuring cross-cluster replication by using MirrorMaker.
* Managing multi-application deployments at scale by using Argo CD ApplicationSets.
