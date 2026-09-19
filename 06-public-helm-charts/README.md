## Lab Goal

Deploy an application to your cluster from a public Helm chart repository, using Headlamp — a web UI for Kubernetes.

## Overview & Concepts

Many applications are distributed through public Helm chart repositories rather than Git. Argo CD deploys from those directly: instead of pointing at a Git repository with a `path` to a chart directory, you name the chart with the `chart` field and point `repoURL` at the chart repository.

Headlamp is published to `https://kubernetes-sigs.github.io/headlamp/`, so nothing about the chart lives in your own Git. What *does* live in Git is the `Application` manifest — the record of which chart, which version, and which values you deployed.

That manifest also has to tighten the chart's defaults. Headlamp's chart binds its ServiceAccount to `cluster-admin` out of the box; the values in this lab bring it down to the read-only `view` role, which is enough to browse a cluster.

## Lab Tasks

1.  Create the `headlamp` namespace by applying `manifests/headlamp-ns.yaml`. The `Application` has no sync policy, so nothing creates it for you — the alternative is `CreateNamespace=true` in `spec.syncPolicy.syncOptions`.
2.  Read through `manifests/headlamp-app.yaml` and note the three fields that differ from a Git-based source: `repoURL`, `chart` and `targetRevision`.
3.  Apply the `Application` manifest to register it with Argo CD.
4.  Open the Argo CD UI and watch the application appear as `OutOfSync`, then sync it.
5.  Verify the rendered objects with `kubectl -n headlamp get all`. Note that they are named `headlamp` rather than `headlamp-headlamp` — the chart's fullname template collapses the two when the release name already contains the chart name.
6.  Confirm the RBAC override took effect:

    ```bash
    kubectl get clusterrolebinding headlamp-admin -o jsonpath='{.roleRef.name}'
    ```

    This should print `view`, not `cluster-admin`.
7.  (Optional) Port-forward to the service and log in, as described below.

## Key Differences from Git-based Charts

When deploying from a Helm repository instead of a Git repository:

- **`repoURL`** points at the chart repository, not a Git remote
- **`chart`** names the chart, replacing the `path` field used for Git sources
- **`targetRevision`** is a chart version, not a branch, tag or commit

Pinning that version matters more here than with Git. `HEAD` at least refers to something you control; a floating chart version means an upstream maintainer can change what deploys to your cluster without you touching anything.

## Accessing Headlamp

The chart's Service is `ClusterIP` on port 80:

```bash
kubectl -n headlamp port-forward svc/headlamp 9090:80
```

Then open `http://localhost:9090`.

You are then met with an **Authentication** screen asking you to paste an ID token. That is the expected behaviour, not a misconfiguration: Headlamp authenticates each user individually rather than running every visitor as the pod's ServiceAccount.

Mint a token for the ServiceAccount the chart created and paste it into the box:

```bash
kubectl -n headlamp create token headlamp
```

Two things to know about that token:

- **It expires after an hour by default.** When the UI logs you out, mint another — `--duration=8h` issues a longer-lived one if the re-prompting gets tedious.
- **The session is read-only**, because that ServiceAccount is bound to `view`. Headlamp still renders delete and edit controls; the API server rejects the calls behind them. A permission error in the UI is the RBAC working, not a bug.

### Why not skip the login screen

The chart offers `config.unsafeUseServiceAccountToken: true`, which authenticates every visitor as the pod's ServiceAccount and removes the prompt. Its own values file calls it UNSAFE and notes it is *"only safe behind an auth proxy (e.g. OIDC proxy)"*.

On a port-forward there is no such proxy, so it would mean anyone reaching the port is logged in. It is not worth trading that away to save one `kubectl` command.

## Helpful Resources

- [Argo CD - Helm Chart Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [Argo CD - Helm values files and valuesObject](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#values)
- [Headlamp Helm chart](https://artifacthub.io/packages/helm/headlamp/headlamp)

## Reflection Questions

1. What are the key differences between using the `chart` field versus the `path` field in an Argo CD Application, and when would you use each?
2. Why is it important to pin a specific chart version when deploying from a public Helm repository instead of tracking the latest?
3. The chart ships with `clusterRoleBinding.clusterRoleName: cluster-admin`. What would it have taken to notice that if the `Application` had simply been applied with the chart's defaults?
4. How does deploying from a public Helm repository affect your GitOps principles, since the chart itself isn't stored in your Git repository? What *is* recorded in Git, and is that enough to rebuild this deployment exactly?
