+++
title = 'GitOps Tutorial Series'
+++
This series explores the principles of GitOps and shows you how to apply them in practice with Strimzi and Argo CD. You learn how to manage your infrastructure in the same way you manage code: define the desired state of your Kafka clusters in Git and use automation to keep the running system synchronized. As you progress through the series, you observe how this approach makes changes reproducible, auditable, and straightforward to roll back.

The best way to understand GitOps is to practice it, so each lesson includes a hands-on, interactive tutorial. Rather than only reading about these concepts, you make real configuration changes and observe them take effect, building practical experience and confidence to apply this workflow to your infrastructure. Although the lessons are designed to be followed in order, each lesson is self-contained so you can jump ahead if you prefer.

The full series in order:
 * [Introduction to GitOps](../introduction/_index.md) - Covers the problems that GitOps solves, contrasts GitOps with the manual approach most teams start with, and introduces the key concepts that you observe in practice throughout the lessons.
 * [Preparing for the tutorials](setup.md) - Guides you through the one-time setup required to configure your Kubernetes cluster for the lessons.
 * [Lesson 1: Your first GitOps change](lesson-1.md) - Introduces Infrastructure as Code (IaC) and automated reconciliation by guiding you through your first GitOps change.
 * [Lesson 2: Promoting changes across environments](lesson-2.md) - Builds on Lesson 1 with multi-environment management, demonstrating how the Kustomize base and overlay pattern enables your project to scale.
 * [Lesson 3: Rolling back a bad change](lesson-3.md) - Shows you how to detect and resolve deployment issues, covering health monitoring and rollback by using a single `git revert` commit.
 * [Wrapping up](wrapping-up.md) - Summarizes the principles you have learned and suggests next steps for your project.
