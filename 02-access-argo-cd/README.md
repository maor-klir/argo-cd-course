## Lab Goal

Access the Argo CD web UI and install the command-line interface (CLI) to gain full control over your Argo CD installation.

## Overview & Concepts

Argo CD is now running in our cluster, but by default, it's not exposed to the outside world. To interact with it, we need to connect securely. In this lab, we'll first retrieve the auto-generated initial admin password from a Kubernetes secret. Then, we'll use `kubectl port-forward` to create a secure tunnel from our local machine to the Argo CD server pod. This will allow us to log in via a web browser. Finally, we'll install the powerful `argocd` CLI for command-line operations and log in with it as well.

## Lab Tasks

1.  Retrieve the initial admin password for the `admin` user from the `argocd-initial-admin-secret` Kubernetes secret and decode it.
2.  Use `kubectl port-forward` to make the Argo CD server UI accessible on `localhost:8080`.
3.  Log in to the Argo CD web UI in your browser using the `admin` username and the retrieved password.
4.  Install the `argocd` CLI tool on your local machine.
5.  Log in to the Argo CD API server using the `argocd` CLI.
6.  Change the `admin` password.
7.  Delete the `argocd-initial-admin-secret` Secret. The Getting Started guide is explicit: *"You should delete the `argocd-initial-admin-secret` from the Argo CD namespace once you changed the password. The secret serves no other purpose than to store the initially generated password in clear and can safely be deleted at any time."*

## Commands

```bash
argocd admin initial-password -n argocd                       # task 1 — prints the generated password
kubectl -n argocd port-forward svc/argocd-server 8080:443     # task 2 — leave it running; the UI is at https://localhost:8080
argocd login localhost:8080 --username admin --insecure       # task 5 — --insecure because the server's certificate is self-signed
argocd account update-password                                # task 6
kubectl -n argocd delete secret argocd-initial-admin-secret   # task 7
```

## Helpful Resources

- [Argo CD Docs - Log In To The UI / CLI](https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)
- [Argo CD CLI Installation Guide](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
- [Kubectl `port-forward` Command Documentation](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#port-forward)

## Reflection Questions

1. Why is `kubectl port-forward` an option to access Argo CD during development, and why might it not be suitable for production environments?
2. What security implications should you consider after retrieving the initial admin password, and what's the best practice for managing it?
3. In what scenarios would you prefer using the Argo CD CLI over the web UI, and vice versa?
