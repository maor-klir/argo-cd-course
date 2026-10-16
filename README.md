# GitOps with Argo CD: labs and notes

Labs and notes from the Argo CD section of Lauro Müller's course [Argo CD and Argo Rollouts for GitOps: The Definitive Guide](https://www.udemy.com/course/argo-cd-rollouts-gitops) on Udemy.

The labs come from the course repository, [lm-academy/argocd-course](https://github.com/lm-academy/argocd-course).  
This copy adds notes on every topic in [`00-notes`](00-notes/), lab READMEs corrected and extended with real output, and fixes found while checking everything against the official documentation for Argo CD v3.4.6.

**Further course materials:**

- [Argo CD example apps - the fork the labs deploy from](https://github.com/maor-klir/argocd-example-apps)
- [Argo Rollouts repository (course companion)](https://github.com/lm-academy/argo-rollouts-course)

## Lab environment

The labs ran on a [k3d](https://k3d.io) cluster on an always-on home server, driven from a laptop over the LAN. Setup, TLS SANs and kubeconfig handling are in notes 01–03.

| Component | Version |
|---|---|
| Argo CD | v3.4.6 (Helm chart `argo-cd` 10.2.2) |
| k3d / k3s | v5.9.0 / v1.35.5+k3s1 |
| kubectl | v1.37.1 |
| Helm | v4.3.0 |
| argocd CLI | v3.5.5 - one minor version ahead of the server |

## Getting started

To run the labs, you'll need a Kubernetes cluster, a few CLI tools, and a GitHub account.

### Prerequisites

1. **Kubernetes cluster**: [k3d](https://k3d.io) is what these labs used. It runs [k3s](https://k3s.io) ([documentation](https://docs.k3s.io)) inside Docker containers. [Kind](https://kind.sigs.k8s.io/), [Minikube](https://minikube.sigs.k8s.io/docs/start/) and [Docker Desktop](https://docs.docker.com/desktop/kubernetes/) work too. k3d and Kind need [Docker](https://docs.docker.com/engine/install/).
2. **kubectl**: the Kubernetes command-line tool. [Installation guide](https://kubernetes.io/docs/tasks/tools/).
3. **Helm**: the package manager for Kubernetes. [Installation guide](https://helm.sh/docs/intro/install/).
4. **Argo CD CLI**: the command-line interface for Argo CD. [Installation guide](https://argo-cd.readthedocs.io/en/stable/cli_installation/).
5. **A fork of [lm-academy/argocd-example-apps](https://github.com/lm-academy/argocd-example-apps)**: from lab 04 on, the labs deploy from your own copy, so you can push changes and watch Argo CD react.
6. **For labs 10 and 11**: a private Git repository, plus a fine-grained personal access token or an SSH deploy key.

> **Never commit repository credentials.** This repository is public. Labs 10 and 11 create the repository Secrets from the command line without writing them to disk.

## Repository structure

### Notes

- [**00-notes**](00-notes/): notes on every topic, written while working through the labs and checked against the official documentation. See the index below.

### Getting started

- [**01-installing-argo-cd**](01-installing-argo-cd/): installing Argo CD with Helm into the `argocd` namespace.
- [**02-access-argo-cd**](02-access-argo-cd/): reaching the web UI and the CLI, retrieving the initial admin password, and replacing it.
- [**03-deploy-first-application**](03-deploy-first-application/): the Application CRD, and deploying the first application.
- [**04-sync-process**](04-sync-process/): the sync process, configuration drift and health status.

### Helm

- [**05-intro-helm-charts**](05-intro-helm-charts/): deploying a Helm chart from a Git repository, and what changes when an application switches from plain manifests to a chart.
- [**06-public-helm-charts**](06-public-helm-charts/): deploying a public chart (Headlamp) from a Helm repository.
- [**07-setting-chart-values**](07-setting-chart-values/): overriding chart values with `valueFiles`, `valuesObject` and `parameters`, and their order of precedence.

### Sync automation

- [**08-automated-sync-pruning**](08-automated-sync-pruning/): automated sync, and pruning resources removed from Git.
- [**09-self-healing**](09-self-healing/): self-healing - reverting changes made directly in the cluster.

### Private repositories

- [**10-private-repo-https**](10-private-repo-https/): connecting to a private repository over HTTPS with a personal access token.
- [**11-private-repo-ssh**](11-private-repo-ssh/): connecting to a private repository over SSH with a deploy key.

### Projects

- [**12-projects**](12-projects/): projects - restricting where applications deploy from and to, and what they may create.

### Sync phases and waves

- [**13-sync-phases-hooks**](13-sync-phases-hooks/): sync phases and hooks - running jobs before, during and after a sync.
- [**14-sync-waves**](14-sync-waves/): sync waves - ordering resources within a phase.

## Notes index

| Labs | Notes |
|---|---|
| lab cluster | 01–03: k3d on a home server, TLS SANs, kubeconfigs |
| 01–04 | 11–16: architecture, the Application CRD, Applications vs. Kubernetes manifests, the first deploy, sync and health, the CLI and core mode |
| 05–07 | 21–23: Helm in Argo CD, deploying charts, overriding values |
| 08–09 | 31: automated sync, pruning and self-healing |
| 10–11 | 41–42: private repositories, SSH and GitHub deploy keys |
| 12 | 51–53: projects, Kubernetes propagation policies, Argo CD propagation policies |
| 13–14 | 61–62: sync phases and hooks, sync waves |
