## 🎯 Lab Goal

Install the Argo CD controller and its components into the cluster using the official Helm chart, ensuring a specific, repeatable version is deployed.

## 📝 Overview & Concepts

We will use Helm, the standard Kubernetes package manager, to install Argo CD. This is the official and recommended method as it correctly handles all of Argo CD's various Kubernetes components. The process involves adding the official Argo Project Helm repository, creating a dedicated `argocd` namespace for organizational hygiene, and then using a `helm install` command with a pinned version to deploy the chart.

## 📋 Lab Tasks

1.  Add the official Argo Project Helm repository to your local Helm client.
2.  Update your Helm repositories to ensure you have the latest chart information.
3.  Create a dedicated `argocd` namespace in your cluster.
4.  Install the `argo-cd` Helm chart version `8.6.0` into the `argocd` namespace.
5.  Verify that all the Argo CD pods have been created and are in a `Running` state.

## 📚 Helpful Resources

- [Argo CD - Getting Started Guide](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Helm `install` Command Documentation](https://helm.sh/docs/helm/helm_install/)
- [Helm `repo` Command Documentation](https://helm.sh/docs/helm/helm_repo/)
- [Kubectl `create namespace` Command Documentation](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create-namespace)

## 💭 Reflection Questions

1. Why is it important to pin a specific version when installing Argo CD (or any critical infrastructure component) rather than using the latest version?
2. Why do we create a dedicated namespace for Argo CD instead of installing it in the `default` namespace?
3. What are the advantages of using Helm to install Argo CD compared to applying raw YAML manifests directly with `kubectl`?
