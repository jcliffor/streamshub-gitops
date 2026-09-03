+++
title = 'GitOps Tutorial Series: Lesson 1'
+++

# Introduction

In our introductory article, "[Introduction to GitOps](../introduction/_index.md)" we explained how GitOps applies the proven principles of version control and continuous delivery (CD) to infrastructure, allowing you to treat operations the same way you treat code. 
With GitOps, you skip clicking through menus and checking boxes. 
Instead, you describe your system in a set of configuration files stored in a git repository. 
Our repository becomes your *single source of truth* for how everything should look. 
You then rely on automation to turn your description into reality.

This tutorial aims to give you a practical demonstration of two key GitOps concepts, Infrastructure as Code (IaC) and the reconciliation loop, alongside two tools that bring them to life: Kustomize and Argo CD.

# Core Concepts

## Infrastructure as Code

IaC is the practice of managing and provisioning your infrastructure using configuration files rather than manual processes, e.g. ClickOps. 
By treating infrastructure as code, you define your system's desired state in configuration files and check them into a version control system like a git repository. 
This approach transforms operations, allowing you to track changes, collaborate, and automate deployments.

Key benefits of adopting IaC include:

* **Consistency and eliminating configuration drift**: By using versioned configuration files, you ensure environments remain consistent, effectively eliminating the manual changes that cause configuration drift.  
* **Disaster recovery**: Because your infrastructure state is codified in version control, rebuilding your environment in the event of a failure is as straightforward as applying your existing configurations.  
* **Reproducible builds**: IaC enables reliable, repeatable infrastructure deployments, ensuring that the same configuration results in the same environment every time.

## Kustomize

Kustomize is a configuration management tool built directly into the Kubernetes command line tool (`kubectl`), which simplifies managing Kubernetes objects.
It enables IaC by allowing you to define a common "base" set of configuration files and then apply "overlays" to patch them for different environments, such as staging or production, without relying on messy templating. 
This ensures that your configurations remain clean, consistent, and reproducible. 

Throughout these lessons you will create, edit and deploy Kustomize manifests to effect changes to the deployed cluster. 
You can learn more in the [official Kustomize documentation](https://kustomize.io/).

## The Reconciliation Loop

The reconciliation loop is the mechanism that transforms IaC into actual, running infrastructure by continuously monitoring the configuration repository and automatically applying changes to the running infrastructure to reach the desired state.
By choosing off-the-shelf tooling, you gain significant speed and operational efficiency from this automation. 
Popular tools that implement this reconciliation pattern include [Argo CD](https://argoproj.github.io/cd/) and [Flux](https://fluxcd.io).

## Argo CD

Argo CD is an open-source, CD tool that runs inside your Kubernetes cluster and implements the reconciliation loop described above. 
It is the CD technology you will be working with in all the lessons in this series. 

You tell Argo CD what to watch by creating an *Application* resource in Kubernetes, a small piece of configuration that says: "monitor this Git repository, look at this directory path, and deploy whatever you find there into this namespace." 
Argo CD then polls the repository on a regular interval (every three minutes by default, however in our tutorial series we’ve reduced that to thirty seconds for convenience), renders the manifests it finds, and syncs the cluster to match.

Argo CD exposes the state of this process through two key concepts: *sync status* and *health status*. 
*Sync status* tells you whether the cluster matches the configuration in your Git repository; *Synced* means they match, whereas *OutOfSync* means Argo CD has detected a difference and will act on it. 
In the lesson, you will see this status transition when you push a change: it moves from *Synced* to *OutOfSync* (Argo CD noticed the new commit) and back to *Synced* (Argo CD applied the change). 
Health status is a separate concern that tells you whether the resources themselves are functioning correctly, we’ll explore health status in more detail in Lesson 3. 

# What to watch for in the lesson

Now that you’ve looked at the core concepts and technologies you’ll be working with in this lesson, it’s almost time to dive in, but as you do look out for these moments where the concepts above become concrete:

* When you edit `kustomization.yaml`, you are declaratively changing the desired state of the cluster.  
* When you run `git push`, you are updating the single source of truth. From this moment, the configuration in the repository says a topic should exist.  
* When Argo CD's status transitions from `Synced` to `OutOfSync` and back to `Synced`, you are watching the reconciliation loop complete a full cycle: observe the change, calculate the required changes and then roll them out.  

Now you’re ready to try the first lesson. To get started you will need to clone the tutorial repository from GitHub:

```bash
git clone https://github.com/streamshub/streamshub-gitops.git
cd streamshub-gitops
```

Next, follow the setup guide at `lessons/00-setup` to deploy a local Kubernetes cluster with all the required components. This takes about eight minutes and only needs to be done once.

When setup is complete, open the lesson 1 README at `lessons/01-lesson-1/README.md` and work through the tutorial.
