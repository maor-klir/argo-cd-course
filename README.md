# Argo CD and Argo Rollouts for GitOps: The Definitive Guide

This repository contains the code and materials for the **Argo CD** section of my **Argo CD and Argo Rollouts** course.

### 🔗 Further Course Materials

- [Argo CD Example Apps](https://github.com/lm-academy/argocd-example-apps)
- [Argo CD Example Apps Labs](https://github.com/lm-academy/argocd-example-apps-labs)
- [Argo Rollouts Repository (Course Companion)](https://github.com/lm-academy/argo-rollouts-course)

## 🚀 Getting Started

To run the code examples, you'll need a Kubernetes cluster and some CLI tools installed.

### Prerequisites

1.  **Kubernetes Cluster**: You can use a local cluster like [Minikube](https://minikube.sigs.k8s.io/docs/start/), [Kind](https://kind.sigs.k8s.io/), or [Docker Desktop](https://docs.docker.com/desktop/kubernetes/).
2.  **kubectl**: The Kubernetes command-line tool. [Installation guide](https://kubernetes.io/docs/tasks/tools/).
3.  **Helm**: The package manager for Kubernetes. [Installation guide](https://helm.sh/docs/intro/install/).
4.  **Argo CD CLI**: Command-line interface for Argo CD. [Installation guide](https://argo-cd.readthedocs.io/en/stable/cli_installation/).

## 📚 Repository Structure

This repository covers the following topics:

- **01-installing-argocd**: Getting Started with Argo CD. Covers installing Argo CD via Helm and setting up the namespace.
- **02-access-argocd**: Accessing the Argo CD Web UI and CLI, including retrieving the initial admin password.
- **03-deploy-first-application**: Core Argo CD Concepts. Deploying your first "Guestbook" application and understanding the Application CRD.
- **04-sync-process**: Understanding the Sync Process, Configuration Drift, and Application Health status.
- **05-intro-helm-charts**: Working with Helm Charts. Learn how Argo CD integrates with Helm as a template engine.
- **06-public-helm-charts**: Deploying Public Helm Charts from external repositories.
- **07-setting-chart-values**: Customizing Helm Values & Precedence using `values.yaml` and parameters.
- **08-automated-sync-pruning**: Advanced Sync & Automation. Configuring automated syncing and pruning of orphaned resources.
- **09-self-healing**: Implementing Self-Healing to automatically correct configuration drift.
- **10-private-repo-https**: Access Management. Connecting to private Git repositories using HTTPS and Personal Access Tokens.
- **11-private-repo-ssh**: Access Management. Connecting to private Git repositories using SSH and Deploy Keys.
- **12-projects**: Organizing & Orchestrating Applications. Using Argo CD Projects for multi-tenancy and access control.
- **13-sync-phases-hooks**: Using Sync Phases and Hooks to run custom logic during the deployment lifecycle.
- **14-sync-waves**: Orchestrating complex deployments using Sync Waves to define precise deployment order.
