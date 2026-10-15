+++
title = 'GitOps Lesson 3: Rolling Back a Bad Change'
+++

# Background

In Lesson 1, you made your first GitOps change: you edited a configuration file, pushed it to your Git repository, and observed Argo CD reconcile the cluster.
In Lesson 2, you scaled that workflow across multiple environments using Kustomize overlays, promoting a change from staging to production with a single configuration update and Git commit.

In practice, a configuration can be syntactically valid yet operationally broken. 
Even when your YAML is well-formed, accepted by Kubernetes, and successfully synchronized by Argo CD, the application can remain broken underneath if it requests actions the underlying platform cannot execute. 
This scenario is a common failure mode in production infrastructure.
In this lesson, you learn how to verify runtime deployment health beyond synchronization status and roll back failed changes using a single Git commit.

## Core concepts

### Sync status versus health status

Argo CD tracks two independent statuses for every `Application` resource:

* **Sync status**: Indicates whether Argo CD successfully applied your Git configuration files to Kubernetes. Once the Kubernetes API accepts the YAML manifests, Argo CD reports the resource as `Synced`.

* **Health status**: Indicates whether the underlying resources are functioning as expected.

A resource can be `Synced` while remaining `Degraded` or persisting in a `Progressing` state. This situation occurs when Kubernetes stores the requested state without error, but the responsible controller fails to execute it. A deployment is complete only when it is both `Synced` and `Healthy`.

### Impact of synced but unhealthy resources

Reconciliation by Argo CD ends at the Kubernetes API server. Argo CD renders your manifests, applies them, and confirms that Kubernetes accepts the request. However, whether the underlying platform, such as a message broker, a database, or an operator-managed system, can execute the change is a separate concern.

Consider a configuration change that the Kubernetes API accepts but the underlying platform rejects. Because Kubernetes stores the custom resource without error, Argo CD reports a status of `Synced`. When the managing operator inspects the resource, it determines that the update violates a system constraint and marks the resource as not ready. Argo CD detects this condition and updates the status to `Progressing` or `Degraded`, but it cannot resolve the failure.

Although the GitOps engine completes synchronization, the application deployment remains broken. This scenario requires a rollback to restore operational health.

### Rollback as a Git Commit

In Lesson 2, you saw that promoting a change is not a special operation; it is a configuration update pushed to a Git repository. Likewise, a rollback is not a special operation. 
There is no dedicated rollback command, no emergency cluster access, and no pipeline to trigger. 
The Git repository remains your single source of truth, so fixing the cluster means updating the repository configuration. 
You create a commit that reverts the configuration to a known-good state, push it to the repository, and allow the reconciliation loop to synchronize the cluster.
This workflow is identical to the one you used for the original deployment. 
The only difference is that the commit undoes a previous change instead of introducing a new one. To Git and Argo CD, it is another commit.

### `git revert` versus `git reset --hard`

Although Git provides multiple methods to undo a commit, these commands have fundamentally different implications in a GitOps workflow:

* `git revert <commit_hash>`: Creates a new commit that applies the exact inverse of the target commit. The original commit remains in the repository history, and the rollback commit is appended to the branch history, preserving an intact audit trail. Because this approach is additive, it does not alter existing commits and is safe to use on shared branches. 
* `git reset --hard` followed by `git push --force`: Rewrites history by removing the target commit from the branch history. In a GitOps environment, this approach is an anti-pattern because it destroys the audit trail, removing all record of the incident and its resolution.

Use `git revert` to undo changes in a GitOps workflow. It preserves the complete event history and maintains a safe, consistent branch state that all collaborators can pull from.

### Self-healing reconciliation

After you push the revert commit, recovery is automatic. Argo CD detects the new commit during its next polling cycle and applies it to the cluster. 
When the resource returns to a valid state, Argo CD marks it as healthy. This process requires no manual intervention on the cluster.
The complete sequence of the breaking change, failure, and recovery remains in the Git commit history for inspection. 
This demonstrates the power of treating a Git repository as the single source of truth: even incidents become part of the auditable record.

## What to observe in the lesson

Now that you have explored the core concepts behind rollback and health monitoring, look for these key observations during the lesson:

* When Argo CD displays `Synced` but `Progressing` after you push the breaking change, you see the distinction between sync status and health status in action. Argo CD completed the synchronization, but the underlying system did not accept the change. 
* When you inspect the resource and see `Ready: False` with a reason explaining why the operation was rejected, you see the platform enforce its constraints independently of Kubernetes. 
* When you run `git revert` and push the change, you perform a GitOps rollback. Because this commit is additive, it does not rewrite history and preserves an intact audit trail. 
* When the Application resource returns to `Synced` and `Healthy` without executing `kubectl` commands, you observe self-healing reconciliation.

You are now ready to work through the hands-on tutorial that follows.

# Tutorial

## Learning objectives

After completing this lesson, you understand:

* Why an Argo CD status of `Synced` does not indicate that a change succeeded, because sync status and health status are separate concerns. 
* How Strimzi enforces Kafka constraints, such as requiring partition counts to increase rather than decrease. 
* How to roll back a breaking change by using `git revert` to create a new commit that undoes the previous commit and preserves the complete audit trail.
* Why `git revert` is the recommended GitOps strategy and why rewriting history with `git reset --hard` is an anti-pattern. 

To observe these concepts, you deploy a configuration change that Argo CD accepts but Strimzi rejects, inspect the resulting failure, and execute a GitOps rollback th

## Prerequisites

* You completed the steps in the [Preparing for the tutorials](setup.md) guide. You only need to complete this setup once.

## Essential concepts

### Sync status versus health status

Argo CD tracks two independent statuses for every `Application` resource:

* **Sync status (`Synced` or `OutOfSync`)**: Indicates whether Argo CD successfully applied the manifests from Git to Kubernetes. This status reflects whether the Kubernetes API accepted the YAML declarations. Executing `kubectl apply` successfully results in a `Synced` status.
  
* **Health status (`Healthy`, `Degraded`, or `Progressing`)**: Indicates whether the underlying resource is operating correctly. Argo CD assesses health status by reading resource conditions. For a `KafkaTopic` resource, Argo CD checks the `Ready` condition set by the Strimzi Topic Operator.
  
These two statuses can diverge. A manifest can be `Synced` when Kubernetes stores the desired state, even while the application remains `Degraded` because the underlying platform rejected the semantic change. This lesson demonstrates that divergence concretely.

### GitOps rollback

Because the configuration in the Git repository is the source of truth, rolling back a deployment is a standard Git commit rather than a specialized recovery operation. When you revert the breaking commit and push the change, the reconciliation loop heals the cluster automatically. This process does not require `kubectl rollout undo` commands, external deployment pipelines, or emergency cluster access.

## Setup

Run the preparation script from this directory:

```bash
./prep.sh
```

This script takes approximately 5 minutes to complete and performs the following actions:
1. Removes state from previous lessons.
2. Seeds the Gitea repository with a working Kafka cluster and topic.
3. Waits for both the Kafka cluster and topic to become ready.


When the script completes, the output displays the credentials for Gitea and Argo CD.

You can rerun `./prep.sh` at any time to reset the environment to the starting state for the lesson. If you already cloned the repository to `/tmp/gitops-lesson-3`, delete the directory before running the script again because the reset operation rewrites the Gitea commit history:

```bash
rm -rf /tmp/gitops-lesson-3
```

## Part 1: Explore the starting state

### Clone the repository

The Gitea server runs inside the cluster. The output of the `./prep.sh` script includes the specific `git clone` command for your environment.

1. Clone the Git repository:
   ```bash
   git clone <gitea_server_address> /tmp/gitops-lesson-3
   ```

2. Change to the newly cloned repository directory:
   ```bash
   cd /tmp/gitops-lesson-3
   ```

### Check the running resources

1. Verify that a Kafka cluster is running in the `kafka-tutorial` namespace:

   ```bash
   kubectl get kafka -n kafka-tutorial
   ```

   Expected output:

   ```
   NAME         READY   WARNINGS   KAFKA VERSION   METADATA VERSION
   my-cluster   True               4.2.0           4.2-IV0
   ```

2. Check for existing Kafka topics in the `kafka-tutorial` namespace:

   ```bash
   kubectl get kafkatopic -n kafka-tutorial
   ```

   Expected output:

   ```
   NAME             CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
   my-first-topic   my-cluster   3            1                    True
   ```

Both the Kafka cluster and topic show a `READY` status of `True`. This is the known-good baseline you return to at the end of the lesson.


### Check the Git history

View the Git commit history:

```bash
git log --oneline
```

Expected output:

```
abc1234 Reset to lesson-3 starting state
...
```

The Git commit history expands as you complete subsequent steps in the lesson.

### Review the manifest

1. Display the contents of the `manifests/topic.yaml` file:

   ```bash
   cat manifests/topic.yaml
   ```

2. Verify that the `partitions` parameter is set to `3`.

## Part 2: Deploy the breaking change

In this scenario, a team member attempts to reduce the partition count of `my-first-topic` from 3 to 1 to save resources. However, Apache Kafka does not support reducing topic partitions; you can increase the partition count of a topic, but you cannot decrease it.

1. Open `manifests/topic.yaml` in your text editor and change `partitions: 3` to `partitions: 1`: 

   ```yaml
   spec:
     partitions: 1
     replicas: 1
   ```

2. Stage the change:

   ```bash
   git add manifests/topic.yaml
   ```

3. Commit the change:

   ```bash
   git commit -m "Reduce my-first-topic to 1 partition"
   ```

4. Push the commit to the repository:

   ```bash
   git push
   ```

   Argo CD polls the repository every 30 seconds.

5. Monitor the Argo CD application status:

   ```bash
   kubectl get application kafka-tutorial -n argocd -w
   ```

6. Observe the `SYNC STATUS` column.
   
   Within approximately 30 seconds, the status changes from `Synced` to `OutOfSync` when Argo CD detects the Git push, and then returns to `Synced` once the changes are applied.
   Expected output:

   ```
   NAME             SYNC STATUS   HEALTH STATUS
   kafka-tutorial   Synced        Progressing
   ```
8. Press Ctrl+C once you see the status return to `Synced`.

Argo CD reports a `SYNC STATUS` of `Synced` because it successfully applied the manifest to Kubernetes and the API server accepted the changes. However, the `HEALTH STATUS` column displays `Progressing` instead of `Healthy`. Argo CD waits for the resource to become ready, but the underlying resource cannot reach a healthy state. 

The next section explains how to investigate this issue.

## Part 3: Investigate the failure

1. Check the Kafka topic status in the `kafka-tutorial` namespace:

   ```bash
   kubectl get kafkatopic -n kafka-tutorial
   ```
   Expected output:
   ```
   NAME             CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
   my-first-topic   my-cluster   1            1                    False
   ```
   The resource reports a `READY` status of `False`. Although the manifest was applied, the partition change failed. The `PARTITIONS` column shows `1`, which reflects the updated `KafkaTopic` specification. However, Strimzi could not reduce the partition count inside the Kafka cluster.
   
2. Inspect the `KafkaTopic` resource details to determine why the operation failed:

   ```bash
   kubectl describe kafkatopic my-first-topic -n kafka-tutorial
   ```
   Expected output:
   ```
   Conditions:
     Type:    Ready
     Status:  False
     Reason:  NotSupported
     Message: Decreasing partitions not supported
   ```
   Review the `Conditions` section in the command output. The Strimzi Topic Operator attempted to reconfigure the topic, but Kafka rejected the operation. The actual topic inside the broker retains its original 3 partitions. However, the `KafkaTopic` custom resource enters an error state because the specification in the Git repository conflicts with the actual broker state.
   
3. Check the Argo CD `Application` status:
   
   ```bash
   kubectl get application kafka-tutorial -n argocd
   ```
   Expected output:
   ```
   NAME             SYNC STATUS   HEALTH STATUS
   kafka-tutorial   Synced        Progressing
   ```
   The application status demonstrates the key distinction between synchronization and health:
   * `SYNC STATUS` of `Synced`: Argo CD successfully applied the manifest to Kubernetes.
   * `HEALTH STATUS` of `Progressing`: Argo CD is waiting for the resource to become ready. Because Strimzi enforces Kafka's partition constraint, the topic remains in a `READY: False` state and cannot reach a healthy state.
   Both the `SYNC STATUS` and `HEALTH STATUS` metrics are critical, and they can disagree. A deployment is complete only when the application status is both `Synced` and `Healthy`.
   An application that reports `Synced` but remains `Progressing` indefinitely indicates that a GitOps rollback is required.

## Part 4: Roll back changes with git revert

1. View the commit history to identify the commit to revert:

   ```bash
   git log --oneline
   ```
   Expected output:
   ```
   def5678 Reduce my-first-topic to 1 partition
   abc1234 Reset to lesson-3 starting state
   ```

   The breaking change is the most recent commit (`HEAD`). 

2. Revert the most recent commit:

   ```bash
   git revert HEAD --no-edit
   ```

   The `git revert HEAD` command creates a new commit that applies the exact inverse of `HEAD`. If `HEAD` changed `partitions: 3` to `partitions: 1`, the revert changes it back to `partitions: 3`. The `--no-edit` flag accepts the default commit message without opening an interactive editor.

3. Verify that the revert commit was added to the history:

   ```bash
   git log --oneline
   ```
   Expected output:
   ```
   789abcd Revert "Reduce my-first-topic to 1 partition"
   def5678 Reduce my-first-topic to 1 partition
   abc1234 Reset to lesson-3 starting state
   ```
   The commit history now contains three entries. The bad commit is preserved in the history to maintain an explicit audit trail, while the revert commit rests on top to record what was undone and when.

4. Push the revert commit to the remote repository:

   ```bash
   git push
   ```

   This completes the GitOps rollback. Because the Git repository serves as the single source of truth, you do not need to run `kubectl` commands or manually trigger deployment pipelines. The automated controller detects the configuration change in Git and reconciles the live cluster state to match.

### Risks of rewriting Git history with `git reset --hard`

You can also roll back changes by rewriting Git history:
```
git reset --hard HEAD~1
```
```
git push --force
```
> **Warning:** Do not force-push (`git push --force`) in a production environment.

This approach removes the commit from history as if it never existed. In a GitOps context, rewriting history is an anti-pattern for the following reasons:

* Anyone who pulled the bad commit has a divergent Git history and encounters errors when running `git pull`.
* Deleting the commit removes the audit trail: no record remains of what changed or when the issue was resolved.
* Force-pushing to a shared branch can overwrite commits made by other team members.
* In regulated environments, rewriting history can violate auditing and compliance requirements.

In contrast, the `git revert` command is additive and safe. The complete sequence of events remains in the commit history for future inspection.

## Part 5: Observe system recovery

1. Monitor the application synchronization status as Argo CD detects the revert commit:
   
   ```bash
   kubectl get application kafka-tutorial -n argocd -w
   ```

   Argo CD detects the new commit within approximately 30 seconds and transitions through `OutOfSync` to `Synced` status as it applies the restored manifest. Press Ctrl+C once you see the status return to `Synced`.

2. Verify that the Kafka topic returned to a healthy state:
   
   ```bash
   kubectl get kafkatopic my-first-topic -n kafka-tutorial
   ```
   Expected output:
   ```
   NAME             CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
   my-first-topic   my-cluster   3            1                    True
   ```  

   The topic reports a `READY` status of `True` and a `PARTITIONS` count of `3`.

3. Confirm the Argo CD application sync and health status:

   ```bash
   kubectl get application kafka-tutorial -n argocd
   ```
   Expected output:
   ```
   NAME             SYNC STATUS   HEALTH STATUS
   kafka-tutorial   Synced        Healthy
   ```
   
   The application displays both `SYNC STATUS` of `Synced` and `HEALTH STATUS` of `Healthy`. This status combination indicates that the live cluster state matches the Git repository specification and the resource is fully operational.

4. Review the complete audit trail in the Git commit history:
   
   ```bash
   git log --oneline
   ```

   Expected output:
   ```
   789abcd Revert "Reduce my-first-topic to 1 partition" 
   def5678 Reduce my-first-topic to 1 partition 
   abc1234 Reset to lesson-3 starting state
   ```
   
   The commit log preserves the complete sequence of events and documents what changed, when the change occurred, and when you reverted the change.

### GitOps reconciliation process

The following sequence describes the process that occurs when you execute `git push`:

1. Gitea (the Git server inside the cluster) receives the push.
2. Argo CD polls Gitea every 30 seconds and detects that `HEAD` has moved to the new revert commit. Argo CD renders the Kustomize manifests (`topic.yaml` now specifies `partitions: 3`).
3. Argo CD compares the rendered state against the live cluster state and sets `partitions: 3` in the `KafkaTopic` custom resource (CR).
4. The Strimzi Topic Operator detects the updated `partitions` value in the CR and checks the actual Kafka topic in the broker. Observing that the topic already has 3 partitions, the operator performs no Kafka operation and sets the `KafkaTopic` status to `Ready: True`.
5. Argo CD verifies that the `KafkaTopic` `Ready` condition is `True` and updates the application health status to `Health: Healthy`.

> **Note:** The partition count of the Kafka topic inside the broker remained unchanged. Strimzi protected the broker from the invalid operation throughout the process. Executing the `git revert` command brought the configuration in the Git repository and the `KafkaTopic` CR back into synchronization with the state of the broker.

### View the Argo CD dashboard (optional)

1. Port-forward the Argo CD server service in a separate terminal window:
   
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
2. Open https://localhost:8080 in your browser and accept the self-signed certificate warning.
   
3. Retrieve the administrator password:

   ```bash
   kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d; echo
   ```
4. Log in using `admin` as the username and the retrieved password.
   
   The **kafka-tutorial** tile displays two distinct indicators: a sync badge and a health badge. After deploying the breaking change, the tile displays **Synced** alongside **Degraded**. After you revert the change, both badges transition to **Synced** and **Healthy**.

5. Select the **kafka-tutorial** application to open the resource tree. Click **my-first-topic** to view the `KafkaTopic` custom resource details, including the error condition reported by Strimzi.

## Key takeaways

* The Argo CD **Synced** status indicates that the Kubernetes API accepted the manifest; it does not mean the change succeeded at the platform level.
* A resource in the **Progressing** status that never reaches **Healthy** signals an issue that requires investigation, even when synchronization succeeds.
* Strimzi enforces Kafka-specific constraints. Some invalid changes deploy silently through Kubernetes but are rejected by the operator.
* Executing `git revert` is the standard GitOps rollback method because it creates a new commit, preserves commit history, and is safe for shared branches.
* Executing `git reset --hard` with a force push destroys the audit trail and creates conflicts for collaborators who pulled the broken commit.
* Rolling back in a GitOps workflow requires no specialized tooling: updating the configuration in the Git repository allows the reconciler to heal the cluster automatically.

## Troubleshooting

### Infrastructure is not running 

If `./prep.sh` reports that the cluster or Kafka is not found, run the setup script: 

```bash
../00-setup/setup.sh
```

### The `prep.sh` script fails while waiting for `KafkaTopic` to be ready

The Topic Operator starts only after the Kafka cluster is ready. If the Kafka wait succeeded but the `KafkaTopic` wait timed out, check the Entity Operator:

```bash
kubectl get pods -n kafka-tutorial
```
```bash
kubectl logs -n kafka-tutorial -l strimzi.io/component-type=entity-operator
```
Rerunning `./prep.sh` resets the environment and retries.

### Argo CD displays Synced but health status is Progressing after the revert

This is transient. The Argo CD health check polls on a short interval. Wait 10 to 15 seconds and check again. If it persists, check the topic status directly:

```bash
kubectl describe kafkatopic my-first-topic -n kafka-tutorial
```

### Exiting the text editor during git revert 

If you run `git revert HEAD` without the `--no-edit` flag, your default text editor opens. To exit `vi`, press `Esc`, type `:wq`, and press `Enter`. To exit `nano`, press `Ctrl+X`, press `Y`, and press `Enter`. Alternatively, press `Ctrl+C` to abort the command and rerun it with the `--no-edit` flag:

```bash
git revert HEAD --no-edit
```

### The topic still displays PARTITIONS: 1 after the revert and Argo CD sync

Confirm that the revert commit was pushed and that Argo CD has synchronized the changes:

```bash
git log --oneline          # confirm the revert commit is the latest
git status                 # confirm nothing is unpushed ("Your branch is up to date")
kubectl get application kafka-tutorial -n argocd   # confirm SYNC STATUS is Synced
```

If the revert commit was not pushed, run `git push`. To trigger an immediate Argo CD sync without waiting for the next 30-second poll cycle:

```bash
kubectl annotate application kafka-tutorial -n argocd \
  argocd.argoproj.io/refresh=normal --overwrite
```

### Strimzi is not managing the kafka-tutorial namespace 

Verify that the Strimzi operator is running and watching all namespaces:

```bash
kubectl get deployment strimzi-cluster-operator -n strimzi-operator \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="STRIMZI_NAMESPACE")].value}'; echo
```

The command output displays `*`. If it does not, rerun `../00-setup/setup.sh`.

## Next steps

Congratulations on completing the third and final lesson of this series. See [Wrapping up](wrapping-up.md) to consolidate what you have learned and explore ideas for your next steps.
