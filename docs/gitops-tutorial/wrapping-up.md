+++
title = 'GitOps Tutorial Series: Wrapping Up'
+++

# What you have learned

In Lesson 1, you learned the two foundational principles of GitOps. 
First, your infrastructure is defined as configuration stored in a Git repository, making it the single source of truth for the state of your cluster. 
Second, a reconciliation loop continuously monitors that repository and automatically deploys any detected changes. 
You pushed a commit, and the system reconciled to match without requiring manual intervention.

In Lesson 2, you scaled that workflow to manage multiple environments. 
By using the Kustomize base and overlay pattern, you managed staging and production configurations from a single repository. 
Independent Argo CD Application resources governed each environment by watching its specific directory path. 
Promoting a change from staging to production was not a separate deployment step. 
It was a configuration update and a Git commit: the same workflow that you had already learned.

In Lesson 3, you observed that GitOps manages deployment failures through Git. 
A rollback is simply another commit that returns the configuration to a known-good state. 
Because the reconciliation loop applies the configuration declared in the repository, recovery is automatic. 
Furthermore, because every modification is recorded as a Git commit, the complete history of the incident, from the initial breaking change to the resolution, is preserved as an auditable record.

# The tools behind the workflow

Two primary tools enabled this workflow: 

Kustomize provides a declarative method for managing Kubernetes manifests. 
In Lesson 1, Kustomize defined the specific resources that Argo CD deployed. 
In Lesson 2, shared base configurations combined with environment-specific overlays managed multi-environment configurations without duplicating YAML files. 

Argo CD serves as the continuous delivery engine that transforms your Git repository into a live, self-healing system. 
It monitors for configuration changes, applies differences to the cluster, and continuously reports both sync and health status. 
Argo CD automates deployment reconciliation so that when you roll back an invalid change in Git, cluster recovery is automatic. 

# Final thoughts on GitOps

Throughout this series, every modification to your cluster followed an identical pattern: edit the configuration, commit the change, and push to the repository. 
Deployments, promotions, and rollbacks were all executed as Git commits; this operational consistency is the core strength of GitOps. 
You gain reproducibility, auditability, and safe rollbacks automatically by establishing your Git repository as the single source of truth. 
GitOps offers a structured, reliable method to manage infrastructure using standard version control workflows.

# Next steps

Now that you understand the fundamentals, explore the `examples/` directory in this repository to see GitOps applied to production-grade streaming infrastructure. 
All examples support both Red Hat OpenShift and standard Kubernetes.

* **Basic Kafka** (`examples/scenarios/basic-kafka/`): A production-ready Kafka cluster running in KRaft mode with Cruise Control for automatic rebalancing and Prometheus metrics for monitoring.
* **Kafka with OAuth** (`examples/scenarios/kafka-oauth/`): Extends the basic cluster with OAuth and OIDC authentication powered by Keycloak, including a full realm configuration with roles, users, and clients.
* **Kafka Mirroring** (`examples/scenarios/kafka-mirror/`): Cross-cluster data replication using MirrorMaker 2, with the External Secrets Operator handling cross-namespace TLS credential management.
* **App of Apps** (`examples/app-of-apps/`): An Argo CD ApplicationSet that deploys all operators and scenarios with a single command, demonstrating a real-world pattern for managing multiple applications at scale.

Each scenario includes its own README file, verification steps, and customization guidance. 
They are designed as starting points that you can adapt for your infrastructure.
