## Lab Goal

Configure and deploy a `PreSync` hook using a Kubernetes `Job` to run a task _before_ the main application deployment begins.

## Overview & Concepts

Many applications require setup tasks, like database migrations, to run before the application itself is deployed. In this lab, you'll implement this pattern using an Argo CD `PreSync` hook.  
You will create a standard Kubernetes `Job` manifest within your Helm chart's `templates` directory.  
This `Job` will simulate a database migration by printing a message and pausing for a few seconds. By adding specific Argo CD annotations to this `Job`, you will instruct Argo CD to run it to completion during the `PreSync` phase, before it attempts to sync the main `Deployment` and `Service`.

## Lab Tasks

1.  In your repository fork, create a new manifest file under the `helm-guestbook` folder named `templates/presync-job.yaml`.
2.  Define a standard Kubernetes `Job` resource in this new file.
3.  The `Job`'s container should use a simple `busybox` image. Its command should print a message like "Running a database migration...", sleep for 10 seconds, and then exit successfully.
4.  Add the `argocd.argoproj.io/hook: PreSync` annotation to the `Job`'s metadata to assign it to the PreSync phase.
5.  Add the `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded` annotation. `HookSucceeded` deletes the `Job` once the sync as a whole succeeds; `BeforeHookCreation` deletes any leftover `Job` with the same name before the next sync creates a new one, so the `Job` runs again on every sync.
6.  Commit and push the new `presync-job.yaml` manifest to your Git repository. Optionally, change `replicaCount` in `values.yaml` in the same commit: a hook on its own does not change the application's sync status, so without another change the application stays `Synced`.
7.  In the Argo CD UI, manually trigger a `Sync` of your `guestbook` application. If you did not change `replicaCount`, the application still shows `Synced`; sync it anyway, and the hook runs.
8.  Observe the application's sync status and resource tree. Notice how the `Job` resource appears and runs for about 10 seconds first, and only after it succeeds do the `Deployment` and other resources sync. Once the sync succeeds, `HookSucceeded` deletes the `Job` and its Pod, so they disappear from the resource tree; the run remains listed in the **Last Sync** details.
9.  *(Optional)* Remove `HookSucceeded` from the delete policy, leaving `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation`. Commit, push and sync again. This time the `Job` and its Pod remain after the sync, and you can read the run's output with `kubectl -n default logs job/<job-name>`.
10. *(Optional)* Delete the leftover `Job` with `kubectl -n default delete job <job-name>`. Its Pod is deleted with it, and the `Job` disappears from the Argo CD UI after a refresh. Leaving it is also fine: `BeforeHookCreation` deletes it before the next sync runs the hook again.

## Which Hooks Run

Argo CD runs only the hooks defined in the application's own source. A `Job` that already exists in the cluster is never run as a hook, even if it carries the same annotations and sits in the application's namespace. This directory includes `external-job.yaml` to show it: a `Job` with the same `PreSync` and delete-policy annotations as your hook, but not part of your Git repository.

1.  Create the external `Job` directly with `kubectl -n default apply -f external-job.yaml`.
2.  Watch it with `kubectl -n default get job external-job -w`. Kubernetes runs it straight away and it reaches `Complete`: the Argo CD annotations mean nothing to Kubernetes.
3.  In your fork, make sure `presync-job.yaml` has `BeforeHookCreation,HookSucceeded` again if you removed `HookSucceeded` in task 9, and change `replicaCount` so the application becomes `OutOfSync`. Commit, push and sync.
4.  Observe that only your `presync-job.yaml` `Job` runs. `external-job` stays `Complete`, and `kubectl -n default describe job external-job` shows events only from its own creation, not from the sync. Its `HookSucceeded` annotation did not delete it either: no sync of any application manages it.
5.  Delete it with `kubectl -n default delete job external-job`.

## Helpful Resources

- [Argo CD - Sync Phases and Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [Kubernetes `Job` Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)

## Reflection Questions

1. Why is it important to use `BeforeHookCreation` in the hook delete policy for jobs that run on every sync?
2. How does a `PreSync` hook differ from a standard `initContainer` in a Pod?
3. What happens to the main application deployment if the `PreSync` hook fails?
4. Once a sync has finished, which of drift detection, self-healing, pruning and health still apply to a hook?
