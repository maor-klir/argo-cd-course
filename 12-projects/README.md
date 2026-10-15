## Lab Goal

Create a custom Argo CD Project with security guardrails and migrate an existing application into it.

## Overview & Concepts

To scale Argo CD safely, we must move away from the unrestricted `default` project. In this lab, you will define a new `AppProject` Custom Resource. This project, named `team-finance`, will enforce two critical security policies: it will only allow deploying applications from a specific, whitelisted Git repository, and it will only allow deployments into a specific, whitelisted namespace.

After creating the project, you will deploy your `guestbook` application and make it belong to `team-finance`, demonstrating how to associate applications with your new, more secure configuration.

## Lab Tasks

1.  Create a new YAML manifest file for your `AppProject` resource named `team-finance-project.yaml`.
2.  In the manifest, define an `AppProject` named `team-finance`.
3.  Configure the project's `spec` to enforce guardrails:
    - Whitelist your `argocd-example-apps` fork — the repository your `guestbook` Application already deploys from — as the only allowed `sourceRepos`.
    - Whitelist the `finance` namespace on `https://kubernetes.default.svc` as the only allowed `destinations`.
4.  Create the `finance` namespace with `kubectl apply -f finance-ns.yaml`. The project leaves `clusterResourceWhitelist` out, which allows no cluster-scoped kinds, so its Applications cannot create a Namespace themselves.
5.  Apply the `AppProject` manifest to your cluster to register it with Argo CD.
6.  Point your `guestbook` Application at the new project, as in `application.yaml`: set `spec.project` to `team-finance` **and** `spec.destination.namespace` to `finance`. The Application has to satisfy both of the project's rules, or it is rejected with an `InvalidSpecError`.
7.  Apply the updated application manifest and sync.
8.  Verify in the Argo CD UI that the `guestbook` application now belongs to the `team-finance` project.
9.  (Bonus) Test the project's restrictions by temporarily changing the application's destination namespace to a disallowed one and observing the error.

## Helpful Resources

- [Argo CD Projects Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
- [The `AppProject` Specification](https://argo-cd.readthedocs.io/en/stable/operator-manual/project-specification/)

## Reflection Questions

1. Why is the `default` project in Argo CD considered insecure for multi-tenant environments?
2. What happens if you try to deploy an application to a namespace that is not whitelisted in the `AppProject`?
3. How do `AppProjects` help in segregating permissions for different teams (for example, Team Finance vs. Team Marketing)?
