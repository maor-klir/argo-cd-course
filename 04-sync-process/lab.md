## 🎯 Lab Goal

Witness the core GitOps loop in action by making a declarative change in a Git repository and watching Argo CD automatically detect and apply it to the cluster.

## 📝 Overview & Concepts

This lab is the "Aha!" moment of GitOps. So far, we've told Argo CD to track a public repository that we don't control. To see the magic happen, we need to make a change to the desired state. To do this, you will first fork the example application repository into your own GitHub account. You'll then update your Argo CD `Application` manifest to point to your fork. Finally, you will make a simple change to a manifest in your forked repo—changing the replica count of a Deployment—and observe in the UI as Argo CD automatically detects the drift and syncs the cluster to match your change.

## 📋 Lab Tasks

1.  Navigate to the `lm-academy/argocd-example-apps` GitHub repository and create a personal fork of it.
2.  Edit your `guestbook-app.yaml` file and update the `spec.source.repoURL` to point to the URL of your newly forked repository.
3.  Apply this updated `Application` manifest to your cluster to inform Argo CD of the new source.
4.  In your **forked repository**, find and edit the `guestbook/guestbook-ui-deployment.yaml` file.
    1.  Change the value of `spec.replicas` from `1` to `3`.
    2.  Commit and push this change.
5.  Return to the Argo CD UI and observe. Watch as the application's status becomes `OutOfSync`. This can take up to 3 minutes, as this is the default interval that Argo CD uses to check for changes.
6.  Use the UI to trigger a Sync operation.
7.  Use `kubectl` to verify that there are now three `guestbook-ui` pods running in the `default` namespace.

## 📚 Helpful Resources

- [GitHub Docs - Fork a repo](https://docs.github.com/en/get-started/quickstart/fork-a-repo)
- [Argo CD - Sync Policies](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/) (Explains how auto-sync works)

## 💭 Reflection Questions

1. Why is forking the repository necessary to truly experience the GitOps workflow, rather than just observing changes in a repository you don't control?
2. What does it mean when Argo CD shows an application as `OutOfSync`, and why doesn't it automatically sync by default?
3. In a real production environment, what additional steps or safeguards would you want between pushing to Git and syncing changes to your cluster?
