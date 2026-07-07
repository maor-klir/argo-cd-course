## 🎯 Lab Goal

Deploy an application to your cluster using Argo CD from a publicly available Helm chart repository, specifically the Kubernetes Dashboard.

## 📝 Overview & Concepts

While Git repositories containing Helm charts are common, many applications are distributed through public Helm chart repositories. Argo CD can seamlessly deploy applications from these public repositories, such as Bitnami charts, official Kubernetes charts, and many others.

In this lab, you'll learn how to configure an Argo CD `Application` to deploy from a public Helm repository. Instead of pointing to a Git repository with a `path` to a chart directory, you'll use the `chart` field to reference a chart by name and specify the chart repository URL.

We will deploy the Kubernetes Dashboard, a web-based UI for Kubernetes cluster management, which is hosted in the official Kubernetes Helm chart repository.

## 📋 Lab Tasks

1.  Create a new namespace named `k8s-dashboard`.
2.  Create a new Argo CD `Application` manifest file named `k8s-dashboard-app.yaml`.
3.  Configure the `spec.source` section to deploy from a public Helm repository:
    - Set the `repoURL` to `https://kubernetes.github.io/dashboard/` (the Kubernetes Dashboard Helm repository).
    - Instead of using `path`, use the `chart` field and set it to `kubernetes-dashboard`.
    - Set `targetRevision` to `7.13.0` to pin to a specific chart version.
4.  Configure the `spec.destination` section:
    - Set `server` to `https://kubernetes.default.svc` (the in-cluster API server).
    - Set `namespace` to `k8s-dashboard` (or your preferred namespace).
5.  Apply the manifest to create the Argo CD Application.
6.  Open the Argo CD UI and observe the application being synced.
7.  Once synced, verify the Kubernetes Dashboard pods are running using `kubectl`.
8.  (Optional) Access the Kubernetes Dashboard by port-forwarding to the service and exploring the UI.

## 🔍 Key Differences from Git-based Charts

When deploying from a public Helm repository instead of a Git repository:

- **`repoURL`**: Points to the Helm chart repository URL (not a Git repository)
- **`chart`**: Specifies the chart name (replaces the `path` field used for Git repos)
- **`targetRevision`**: Refers to the chart version (not a Git commit/branch/tag)

## 📚 Helpful Resources

- [Argo CD - Helm Chart Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [Kubernetes Dashboard Helm Chart](https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard)
- [Argo CD Application Specification](https://argo-cd.readthedocs.io/en/stable/operator-manual/application.yaml)

### Accessing the Dashboard

Port-forward to access the dashboard locally:

```bash
kubectl port-forward svc/k8s-dashboard-kong-proxy 8443:443 -n k8s-dashboard
```

Then access the dashboard at: `https://localhost:8443`

**Note:** You'll need to create a service account and token to log in:

```bash
# Create a service account
kubectl create serviceaccount k8s-dashboard-admin -n k8s-dashboard

# Create a cluster role binding
kubectl create clusterrolebinding k8s-dashboard-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=k8s-dashboard:k8s-dashboard-admin

# Get the token
kubectl create token k8s-dashboard-admin -n k8s-dashboard
```

Use the generated token to log in to the dashboard.

## 💭 Reflection Questions

1. What are the key differences between using the `chart` field versus the `path` field in an Argo CD Application, and when would you use each?
2. Why is it important to pin a specific `targetRevision` (chart version) when deploying from a public Helm repository instead of using "latest"?
3. How does deploying from a public Helm repository affect your GitOps principles, since the chart itself isn't stored in your Git repository?
