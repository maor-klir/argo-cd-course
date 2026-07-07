## 🎯 Lab Goal

Define and deploy your first GitOps-managed application using the core Argo CD resource: the `Application` CRD.

## 📝 Overview & Concepts

The heart of Argo CD is a Custom Resource Definition (CRD) called `Application`. This resource is a declarative manifest that acts as a contract, telling Argo CD three main things: **where** the desired state is defined (a Git repository), **what** to deploy (a specific path and version in that repo), and **where** to deploy it (a destination cluster and namespace).

In this lab, you will write an `Application` manifest from scratch. You will commit this file to your Git repository. Then, you will perform a one-time `kubectl apply` of this manifest to "bootstrap" the application and register it with Argo CD. From that moment on, Argo CD will take over, pulling the manifests from the source repository and deploying them to the destination namespace.

## 📋 Lab Tasks

1.  Create a new YAML file for your `Application` resource.
2.  Define the `metadata` for the `Application`, giving it the name `guestbook` and ensuring it lives in the `argocd` namespace.
3.  Define the `spec.project` as `default` and the `spec.source` to point to a public Git repository containing plain Kubernetes manifests. You can use the following for a demo application:
    - **Repo URL:** `https://github.com/lm-academy/argocd-example-apps.git`
    - **Revision:** `HEAD`
    - **Path:** `guestbook`
4.  Define the `spec.destination` field to deploy the application to the local cluster (`https://kubernetes.default.svc`) into the `default` namespace.
5.  Apply the `Application` manifest to your cluster using `kubectl apply`.
6.  Verify in the Argo CD UI that the `guestbook` application appears and successfully syncs.
7.  Use `kubectl` to verify that the `guestbook` application's resources (Deployments, Services) are running in the `default` namespace.

## 📚 Helpful Resources

- [Argo CD Application CRD Specification](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
- [Declarative Setup in Argo CD](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
- [The `argocd-example-apps` Repository](https://github.com/argoproj/argocd-example-apps)
- [My Fork of the `argocd-example-apps` Repository](https://github.com/lm-academy/argocd-example-apps)

## 💭 Reflection Questions

1. Why do you think the `Application` resource itself lives in the `argocd` namespace, but the application it deploys goes to the `default` namespace?
2. What is the significance of using `HEAD` as the `targetRevision`, and what are alternative approaches you might use in production?
3. How does storing the `Application` manifest in Git align with GitOps principles, and what benefits does this provide?
