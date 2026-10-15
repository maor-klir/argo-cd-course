## Lab Goal

Use `sync-wave` annotations to enforce a specific deployment order, ensuring dependent resources are created and healthy before the main application starts.

## Overview & Concepts

In this lab, you will orchestrate a multi-step deployment using Sync Waves. To clearly visualize the delay between waves, we will create a chain of dependencies:

1.  **Wave 1**: A `ConfigMap` representing a database configuration.
2.  **Wave 2**: A `Job` that simulates a database connectivity check. It will read the configuration and sleep for a few seconds.
3.  **Wave 3**: The main application `Deployment`, which should only start after the check passes.

When you sync the application, you will observe Argo CD pausing between each wave, waiting for the resources in the current wave to become "Healthy" before proceeding to the next. A `ConfigMap` has no health check, so its wave adds only Argo CD's 2-second pause between waves; the visible wait is the `Job`, which stays `Running` until it completes.

## Lab Tasks

1.  In your repository fork, under the `helm-guestbook` folder, create a new manifest file named `templates/database-config.yaml`.
2.  Define a `ConfigMap` named `db-config` with some dummy data (for example, `host: "postgres"`).
3.  Add the annotation `argocd.argoproj.io/sync-wave: "1"` to the `ConfigMap`.
4.  Create another file named `templates/db-check-job.yaml`.
5.  Define a Kubernetes `Job` that reads the `ConfigMap` via an environment variable and runs a command to print the data and sleep for 5 seconds. Set the variable with `valueFrom.configMapKeyRef`, and print it inside double quotes, for example `echo "Connecting to $DB_HOST..."`: inside single quotes the shell prints `$DB_HOST` literally.
6.  Add the annotation `argocd.argoproj.io/sync-wave: "2"` to the `Job`.
7.  Open the existing `templates/deployment.yaml` file.
8.  Add the annotation `argocd.argoproj.io/sync-wave: "3"` to the `Deployment`.
9.  Commit and push all changes to your Git repository.
10. In the Argo CD UI, trigger a `SYNC` of your `guestbook` application.
11. Carefully observe the sync process. Note how the UI updates in stages: the `PreSync` `Job` from the previous lab runs first, if it is still in the chart; then the `Service`, which has no annotation and is therefore in wave 0; then ConfigMap -> Job (Running -> Completed) -> Deployment.
12. *(Optional)* Renumber the waves to `"10"`, `"20"` and `"30"`, so that a step can later be inserted between them without renumbering. Commit, push and refresh: the application becomes `OutOfSync`, because wave annotations are part of the desired state, unlike hooks. Sync it. The check `Job` does not run again: `kubectl -n default get job,pod` shows its `AGE` dating from the first sync, with a single Pod.
13. *(Optional)* Change the command of the check `Job`, then commit, push and sync. The sync fails with `spec.template: Invalid value: …: field is immutable`: a `Job`'s Pod template cannot be changed after the `Job` is created. Delete the `Job` with `kubectl -n default delete job <job-name>` and sync again; Argo CD recreates it, and it runs once more.

## Helpful Resources

- [Argo CD - Sync Waves Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [Kubernetes Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

## Reflection Questions

1.  How do Sync Waves differ from Resource Hooks (like PreSync)?
2.  What happens if the Job in Wave 2 fails? Does Wave 3 start?
3.  Why might you use Sync Waves instead of `initContainers` for dependency management?
