## 🎯 Lab Goal

Create a custom Argo CD Project with security guardrails and migrate an existing application into it.

## 📝 Overview & Concepts

To scale Argo CD safely, we must move away from the unrestricted `default` project. In this lab, you will define a new `AppProject` Custom Resource. This project, named `team-finance`, will enforce two critical security policies: it will only allow deploying applications from a specific, whitelisted Git repository, and it will only allow deployments into a specific, whitelisted namespace.

After creating the project, you will deploy your `guestbook` application and make it belong to `team-finance`, demonstrating how to associate applications with your new, more secure configuration.

## 📋 Lab Tasks

1.  Create a new YAML manifest file for your `AppProject` resource named `team-finance-project.yaml`.
2.  In the manifest, define an `AppProject` named `team-finance`.
3.  Configure the project's `spec` to enforce guardrails:
    - Whitelist your private Helm chart repository (`argocd-example-apps-labs`) as the only allowed `sourceRepos`.
    - Whitelist the `guestbook` namespace as the only allowed `destinations`.
4.  Apply the `AppProject` manifest to your cluster to register it with Argo CD.
5.  Modify your `guestbook-app.yaml` manifest, changing the `spec.project` from `default` to `team-finance`.
6.  Apply the updated application manifest.
7.  Verify in the Argo CD UI that the `guestbook` application now belongs to the `team-finance` project.
8.  (Bonus) Test the project's restrictions by temporarily changing the application's destination namespace to a disallowed one and observing the error.

## 📚 Helpful Resources

- [Argo CD Projects Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
- [The `AppProject` CRD Specification](https://argo-cd.readthedocs.io/en/stable/operator-manual/api-docs/#argoproj.io/v1alpha1.AppProject)

## 💭 Reflection Questions

1. Why is the `default` project in Argo CD considered insecure for multi-tenant environments?
2. What happens if you try to deploy an application to a namespace that is not whitelisted in the `AppProject`?
3. How do `AppProjects` help in segregating permissions for different teams (for example, Team Finance vs. Team Marketing)?
