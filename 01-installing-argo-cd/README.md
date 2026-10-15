## Lab Goal

Install the Argo CD controller and its components into the cluster using the official Helm chart, ensuring a specific, repeatable version is deployed.

## Overview & Concepts

We will use Helm, the standard Kubernetes package manager, to install Argo CD. This is the official and recommended method as it correctly handles all of Argo CD's various Kubernetes components. The process involves adding the official Argo Project Helm repository, creating a dedicated `argocd` namespace for organizational hygiene, and then using a `helm install` command with a pinned version to deploy the chart.

## Lab Tasks

1.  Add the official Argo Project Helm repository to your local Helm client.
2.  Update your Helm repositories to ensure you have the latest chart information.
3.  Create a dedicated `argocd` namespace in your cluster.
4.  Install the `argo-cd` Helm chart version `10.2.2` into the `argocd` namespace, as a release named `argocd`.
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

1. Add the Argo Helm repository and refresh it:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

2. List the available chart versions:

```bash
helm search repo argo/argo-cd --versions
```

3. Create the namespace from the manifest in this directory:

```bash
kubectl apply -f argocd-ns.yaml
```

4. Install a pinned chart version — release `argocd` into namespace `argocd`:

```bash
helm upgrade argocd argo/argo-cd --version 10.2.2 --install -n argocd
```

Both names matter. The chart prefixes its resources with the release name unless that name already contains `argocd`, so a release called `argo-cd` produces `argo-cd-argocd-server` and friends instead of `argocd-server`. And every later lab assumes the `argocd` namespace: Applications are only honoured in Argo CD's own namespace, and the commands use `-n argocd`.

```bash
helm -n argocd list
```

```text
NAME  	NAMESPACE	REVISION	UPDATED                                 	STATUS  	CHART         	APP VERSION
argocd	argocd   	1       	2026-09-05 12:27:13.741501396 +0200 CEST	deployed	argo-cd-10.2.2	v3.4.6
```

The chart's install notes, as shown again by `helm -n argocd get notes argocd`:

```text
In order to access the server UI you have the following options:

1. kubectl port-forward service/argocd-server -n argocd 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate

2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

(You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)
```

5. Verify that all the pods are running:

```bash
kubectl -n argocd get pods
```

```text
NAME                                               READY   STATUS    RESTARTS      AGE
argocd-application-controller-0                    1/1     Running   3 (13d ago)   40d
argocd-applicationset-controller-958cbb657-knk7w   1/1     Running   4 (13d ago)   40d
argocd-dex-server-ffb674c4-fm95l                   1/1     Running   1 (13d ago)   21d
argocd-notifications-controller-84b67cd59c-b72gg   1/1     Running   3 (13d ago)   40d
argocd-redis-557bb4bb-2s8qv                        1/1     Running   4 (13d ago)   40d
argocd-repo-server-7bf9495ffc-bnhrp                1/1     Running   3 (13d ago)   40d
argocd-server-5fb8b6457d-6vq6r                     1/1     Running   2 (13d ago)   36d
```
