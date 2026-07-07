## 🎯 Lab Goal

Configure an Argo CD application to sync automatically when a change is detected in Git and to prune resources that are no longer defined in the source of truth.

## 📝 Overview & Concepts

A truly automated GitOps workflow doesn't require manual intervention. In this lab, you will enable two crucial sync policies on your `guestbook` application: `automated` and `prune`. The `automated` policy will tell Argo CD to automatically apply changes from Git without you needing to click "Sync". The `prune` policy will ensure that if you delete a manifest from your Git repository, Argo CD will automatically delete the corresponding resource from the cluster.

You will test this by first adding a new `ConfigMap` to your Helm chart and watching it get deployed automatically. Then, you will delete the `ConfigMap` manifest from the chart and observe as Argo CD prunes it from the cluster, keeping your environment perfectly in sync with Git.

## 📋 Lab Tasks

1.  Add a `syncPolicy` block to your `guestbook-app.yaml` manifest.
2.  Inside the `syncPolicy`, set `automated` to an empty object `{}`.
3.  Apply the updated manifest to the cluster to update the application's settings.
4.  In your Helm chart, add a new manifest for a simple `ConfigMap`.
5.  Commit and push this change to your Git repository.
6.  Observe in the Argo CD UI as the change is automatically detected and synced without any manual clicks.
7.  Verify with `kubectl` that the new `ConfigMap` exists.
8.  Now, delete the `ConfigMap` manifest file from your Helm chart.
9.  Commit and push this deletion.
10. Observe in the Argo CD UI as the resource gets marked as non-existent, but is not immediately deleted.
11. Enable pruning by changing the `automated` option from an empty object to `prune: true`.
12. Apply the updated manifest to the cluster to update the application's settings.
13. Observe in the Argo CD UI as the `prune` policy takes effect, automatically deleting the `ConfigMap` from the cluster.
14. Verify with `kubectl` that the `ConfigMap` is gone.

## 📚 Helpful Resources

- [Argo CD - Automated Sync Policy](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [Kubernetes ConfigMaps Documentation](https://kubernetes.io/docs/concepts/configuration/configmap/)

## 💭 Reflection Questions

1. Why does automated sync not automatically remove resources when their manifests are deleted from Git, and what specific sync policy is required to enable this behavior?
2. What could happen if you enable pruning on an application that shares a namespace with resources managed by other tools or teams, and how would you prevent accidental deletions?
3. In what scenario might you choose to enable automated sync but intentionally leave pruning disabled, and what are the trade-offs of this configuration?
