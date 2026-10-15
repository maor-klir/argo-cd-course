## Lab Goal

Refactor an existing Argo CD `Application` to deploy the same application, but this time from a Helm chart located within our Git repository.

## Overview & Concepts

Deploying applications from plain YAML is great, but many real-world applications are packaged as Helm charts to manage complexity and templating. In this lab, you'll learn how to adapt an Argo CD `Application` manifest to deploy from a Helm chart instead of a directory of raw manifests.

We will be using a pre-made Helm chart for our `guestbook` application, which is already located in our examples repository. You will modify your existing `guestbook-app.yaml` manifest, changing the `spec.source` to point to this Helm chart, and observe how Argo CD handles the switch. The chart names its objects differently from the plain manifests, so the switch is not an in-place update.

## Lab Tasks

1.  Explore the `helm-guestbook` directory in your `argocd-example-apps` repository to familiarize yourself with the simple Helm chart structure.
2.  Open your `guestbook-app.yaml` manifest for editing.
3.  Modify the `spec.source` section of the manifest:
    - Change the `repoURL` to point to your own fork of `lm-academy/argocd-example-apps` repository.
    - Set the `path` to `helm-guestbook`.
    - Add a new `helm` block.
    - Inside the `helm` block, add a `valueFiles` entry pointing to the chart's default `values.yaml` file.
4.  Apply the updated manifest to the cluster.
5.  Open the Argo CD UI and wait for the application to be considered `OutOfSync`.
6.  Before syncing, look at the resource tree. The chart renders its objects as `guestbook-helm-guestbook`, not `guestbook-ui`, and Argo CD tracks objects by name — so it now lists four resources: the chart's two, missing, and the old `guestbook-ui` pair, marked for pruning.
7.  Trigger a `Sync` with **Prune** checked. The chart's Deployment and Service are created and the old pair is deleted. Without **Prune**, the old pair keeps running, unmanaged.
8.  Verify with `kubectl -n default get deploy,svc` that only the `guestbook-helm-guestbook` objects remain.

To keep the old names instead, `guestbook-app-fullnameoverride.yaml` sets `fullnameOverride: guestbook-ui`. The Service then updates in place, but the Deployment does not: the chart's selector differs from the old one, a Deployment's `spec.selector` is immutable, and the sync fails until it is run with the **Replace** and **Force** options, which delete and re-create the Deployment.

## Helpful Resources

- [Argo CD - Helm Chart Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [Helm `valueFiles` Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#values-files)

## Reflection Questions

1. Why do most real-world applications use Helm charts instead of plain Kubernetes YAML manifests, and what complexity does Helm help manage?
2. Why does switching from raw manifests to a Helm chart produce new resources rather than updating the existing ones, and what decides whether Argo CD treats two objects as the same one?
3. What are the advantages of storing Helm charts in Git repositories versus distributing them through chart repositories?
