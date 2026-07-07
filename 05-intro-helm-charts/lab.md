## 🎯 Lab Goal

Refactor an existing Argo CD `Application` to deploy the same application, but this time from a Helm chart located within our Git repository.

## 📝 Overview & Concepts

Deploying applications from plain YAML is great, but many real-world applications are packaged as Helm charts to manage complexity and templating. In this lab, you'll learn how to adapt an Argo CD `Application` manifest to deploy from a Helm chart instead of a directory of raw manifests.

We will be using a pre-made Helm chart for our `guestbook` application, which is already located in our examples repository. You will modify your existing `guestbook-app.yaml` manifest, changing the `spec.source` to point to this Helm chart, and observe as Argo CD seamlessly transitions the live application to be managed by the chart.

## 📋 Lab Tasks

1.  Explore the `helm-guestbook` directory in your `argocd-example-apps` repository to familiarize yourself with the simple Helm chart structure.
2.  Open your `guestbook-app.yaml` manifest for editing.
3.  Modify the `spec.source` section of the manifest:
    - Change the `repoURL` to point to your own fork of `lm-academy/argocd-example-apps` repository.
    - Set the `path` to `helm-guestbook`.
    - Add a new `helm` block.
    - Inside the `helm` block, add a `valueFiles` entry pointing to the chart's default `values.yaml` file.
4.  Apply the updated manifest to the cluster.
5.  Open the Argo CD UI and wait for the application to be considered `OutOfSync`.
6.  Trigger a `Sync` operation and observe in the Argo CD UI as the application syncs. Notice that although the source has fundamentally changed, the deployed resources remain the same, demonstrating Argo CD's powerful diffing capabilities.
7.  Verify with `kubectl` that the `guestbook` pods are still running correctly.

## 📚 Helpful Resources

- [Argo CD - Helm Chart Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [Helm `valueFiles` Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#values-files)

## 💭 Reflection Questions

1. Why do most real-world applications use Helm charts instead of plain Kubernetes YAML manifests, and what complexity does Helm help manage?
2. How does Argo CD's ability to detect that the same resources are being deployed (despite switching from raw manifests to a Helm chart) demonstrate its sophisticated diffing capabilities?
3. What are the advantages of storing Helm charts in Git repositories versus distributing them through chart repositories?
