## 🎯 Lab Goal

Connect Argo CD to a private Git repository using SSH authentication with deploy keys, implementing a production-grade security pattern.

## 📝 Overview & Concepts

While username/PAT authentication works well, SSH deploy keys represent the gold standard for production GitOps workflows. A deploy key is a dedicated SSH key pair that grants access to a single repository, typically with read-only permissions.

The advantages of deploy keys over PATs include:

- **Repository-specific**: Each key only grants access to one repository
- **No user account dependency**: Keys aren't tied to an individual's account
- **Read-only by default**: Most Git providers enforce read-only access for deploy keys
- **Auditable**: Deploy key usage is logged separately from user activity
- **Long-lived**: No expiration dates to manage

In this lab, you'll generate an SSH key pair specifically for Argo CD, configure your repository to accept this key, and update your application to use SSH authentication.

## 📋 Lab Tasks

1.  Create a new **private** repository on your Git provider (or use the one from the previous lab)
2.  If not already done, copy the `helm-guestbook` chart into this repository and push it
3.  Generate a new SSH key pair on your local machine:
    - Use a descriptive name (e.g., `argocd-deploy-key`)
    - **Do not use a passphrase** (Argo CD can't handle passphrase-protected keys)
4.  Add the **public key** to your repository as a Deploy Key:
    - Navigate to your repository's settings
    - Find the Deploy Keys section
    - Add the public key with an appropriate title
    - Ensure it's set to **read-only** access
5.  In the Argo CD UI, add a new repository connection:
    - Connection method: SSH
    - Repository URL: The SSH URL of your private repository (starts with `git@`)
    - SSH private key: Paste the contents of your **private key** file
6.  Test the connection to verify Argo CD can authenticate
7.  Update your Application manifest to use the SSH repository URL
8.  Apply the manifest and confirm the application syncs from the private repository

## 📚 Helpful Resources

- [Argo CD - Private Repositories Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/)
- [GitHub - Managing Deploy Keys](https://docs.github.com/en/developers/overview/managing-deploy-keys)
- [SSH Key Generation Guide](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

## 💭 Reflection Questions

1. Why are SSH deploy keys considered the "gold standard" for production GitOps workflows compared to Personal Access Tokens, despite being more complex to set up?
2. What security advantages come from deploy keys being repository-specific rather than user-account-wide, and how does this align with the principle of least privilege?
3. Why is it important that deploy keys used by Argo CD don't have a passphrase, and how should you compensate for this security consideration?
