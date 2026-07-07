## 🎯 Lab Goal

Use `sync-wave` annotations to enforce a specific deployment order, ensuring dependent resources are created and healthy before the main application starts.

## 📝 Overview & Concepts

In this lab, you will orchestrate a multi-step deployment using Sync Waves. To clearly visualize the delay between waves, we will create a chain of dependencies:

1.  **Wave 1**: A `ConfigMap` representing a database configuration.
2.  **Wave 2**: A `Job` that simulates a database connectivity check. It will read the configuration and sleep for a few seconds.
3.  **Wave 3**: The main application `Deployment`, which should only start after the check passes.

When you sync the application, you will observe Argo CD pausing between each wave, waiting for the resources in the current wave to become "Healthy" before proceeding to the next.

## 📋 Lab Tasks

1.  In your repository fork, under the `helm-guestbook` folder, create a new manifest file named `templates/database-config.yaml`.
2.  Define a `ConfigMap` named `db-config` with some dummy data (for example, `host: "postgres"`).
3.  Add the annotation `argocd.argoproj.io/sync-wave: "1"` to the `ConfigMap`.
4.  Create another file named `templates/db-check-job.yaml`.
5.  Define a Kubernetes `Job` that reads the `ConfigMap` via an environment variable and runs a command to print the data and sleep for 5 seconds.
6.  Add the annotation `argocd.argoproj.io/sync-wave: "2"` to the `Job`.
7.  Open the existing `templates/deployment.yaml` file.
8.  Add the annotation `argocd.argoproj.io/sync-wave: "3"` to the `Deployment`.
9.  Commit and push all changes to your Git repository.
10. In the Argo CD UI, trigger a `SYNC` of your `guestbook` application.
11. Carefully observe the sync process. Note how the UI updates in stages: ConfigMap -> Job (Running -> Completed) -> Deployment.

## 📚 Helpful Resources

- [Argo CD - Sync Waves Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [Kubernetes Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

## 💭 Reflection Questions

1.  How do Sync Waves differ from Resource Hooks (like PreSync)?
2.  What happens if the Job in Wave 2 fails? Does Wave 3 start?
3.  Why might you use Sync Waves instead of `initContainers` for dependency management?
