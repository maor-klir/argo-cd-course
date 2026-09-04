# Lab Goal

Install the Argo CD controller and its components into the cluster using the official Helm chart, ensuring a specific, repeatable version is deployed.

## Overview & Concepts

We will use Helm, the standard Kubernetes package manager, to install Argo CD. This is the official and recommended method as it correctly handles all of Argo CD's various Kubernetes components. The process involves adding the official Argo Project Helm repository, creating a dedicated `argocd` namespace for organizational hygiene, and then using a `helm install` command with a pinned version to deploy the chart.

## Lab Tasks

1.  Add the official Argo Project Helm repository to your local Helm client.
2.  Update your Helm repositories to ensure you have the latest chart information.
3.  Create a dedicated `argocd` namespace in your cluster.
4.  Install the `argo-cd` Helm chart version `8.6.0` into the `argocd` namespace.
5.  Verify that all the Argo CD pods have been created and are in a `Running` state.

## Helpful Resources

- [Argo CD - Getting Started Guide](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Helm `install` Command Documentation](https://helm.sh/docs/helm/helm_install/)
- [Helm `repo` Command Documentation](https://helm.sh/docs/helm/helm_repo/)
- [Kubectl `create namespace` Command Documentation](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create-namespace)

## Reflection Questions

1. Why is it important to pin a specific version when installing Argo CD (or any critical infrastructure component) rather than using the latest version?
2. Why do we create a dedicated namespace for Argo CD instead of installing it in the `default` namespace?
3. What are the advantages of using Helm to install Argo CD compared to applying raw YAML manifests directly with `kubectl`?

---

## Installing Argo CD

1. Adding the Argo Helm repository and make sure it's at its latest version (a no-op in this case)

```bash
~/repositories/github/maor-klir/argo-cd-course main*​
helm repo add argo https://argoproj.github.io/argo-helm
"argo" has been added to your repositories

~/repositories/github/maor-klir/argo-cd-course main*​
❯ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "argo" chart repository
Update Complete. ⎈Happy Helming!⎈
```
2. Checking which Helm charts versions are available:

```bash
~/repositories/github/maor-klir/argo-cd-course main*​
❯ helm search repo argo/argo-cd --versions
```

3. Applying the namespace manifest onto the cluster:

```bash
~/repositories/github/maor-klir/argo-cd-course main*​
❯ k apply -f argocd-ns.yaml
namespace/argo-cd created
```

4. Installing a specific chart version:

```bash
~/repositories/github/maor-klir/argo-cd-course main*​
❯ helm upgrade argo-cd argo/argo-cd --version 10.2.2 --install -n argo-cd
Release "argo-cd" does not exist. Installing it now.
NAME: argo-cd
LAST DEPLOYED: Fri Sep  4 19:22:55 2026
NAMESPACE: argo-cd
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
In order to access the server UI you have the following options:

1. kubectl port-forward service/argo-cd-argocd-server -n argo-cd 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate

2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

kubectl -n argo-cd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

(You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)
```
5. Verifying all pods are running:

```bash
~/repositories/github/maor-klir/argo-cd-course main*​ 2s
❯ k -n argo-cd get po -w
NAME                                                        READY   STATUS    RESTARTS   AGE
argo-cd-argocd-application-controller-0                     1/1     Running   0          2m9s
argo-cd-argocd-applicationset-controller-64b8db7fbd-bq8rw   1/1     Running   0          2m9s
argo-cd-argocd-dex-server-59777968dd-6vmhr                  1/1     Running   0          2m9s
argo-cd-argocd-notifications-controller-76d465d5b7-ghx6t    1/1     Running   0          2m9s
argo-cd-argocd-redis-75759d788f-xstsr                       1/1     Running   0          2m9s
argo-cd-argocd-repo-server-787659dc89-jjm9k                 1/1     Running   0          2m9s
argo-cd-argocd-server-678bdcfd7d-988fp                      1/1     Running   0          2m9s
^C
```