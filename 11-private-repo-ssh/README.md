## Lab Goal

Connect Argo CD to a private Git repository using SSH authentication with deploy keys, implementing a production-grade security pattern.

## Overview & Concepts

While username/PAT authentication works well, SSH deploy keys are a common choice for production GitOps workflows. A deploy key is an SSH key that grants access to a single repository, read-only unless you explicitly allow write access. (GitHub itself recommends a GitHub App for the finest-grained control, and Argo CD supports that too.)

The advantages of deploy keys over PATs include:

- **Repository-specific**: Each key only grants access to one repository
- **No user account dependency**: Keys aren't tied to an individual's account
- **Read-only by default**: GitHub adds deploy keys read-only unless you tick **Allow write access** — a default, not an enforced restriction
- **No expiry**: Nothing to renew — though GitHub lists this as a drawback, so plan your own rotation

GitHub's documentation lists the trade-offs as well: a deploy key stays active after the person who created it leaves, never expires, and with write access can perform the same actions as a repository admin.

In this lab, you'll generate an SSH key pair specifically for Argo CD, configure your repository to accept this key, and update your application to use SSH authentication.

## Lab Tasks

1.  Create a new **private** repository on your Git provider (or use the one from the previous lab)
2.  If not already done, copy the `helm-guestbook` chart into this repository and push it
3.  Generate a new SSH key pair on your local machine:
    - Use a descriptive name (e.g., `argocd-deploy-key`)
    - **Do not use a passphrase** — press Enter at both prompts (Argo CD has no field for a passphrase)

    ```bash
    ssh-keygen -t ed25519 -C "argocd-deploy-key" -f ~/.ssh/argocd-deploy-key
    ```

    Ed25519 is the algorithm GitHub recommends. This creates the private key `argocd-deploy-key` and the public key `argocd-deploy-key.pub`.
4.  Add the **public key** to your repository as a Deploy Key:
    - Navigate to your repository's settings
    - Find the Deploy Keys section
    - Add the public key with an appropriate title
    - Leave **Allow write access** unchecked — a deploy key with write access can do anything a repository admin can
5.  In the Argo CD UI, add a new repository connection:
    - Connection method: SSH
    - Repository URL: The SSH URL of your private repository (starts with `git@`)
    - SSH private key: Paste the whole **private key** file (the one without `.pub`), including the `BEGIN` and `END` lines
6.  Test the connection to verify Argo CD can authenticate
7.  Update your Application manifest to use the SSH repository URL
8.  Apply the manifest and confirm the application syncs from the private repository

If you switch the Application to the SSH URL before the SSH credential exists, it reports `error creating SSH agent: "SSH agent requested but SSH_AUTH_SOCK not-specified"`.  
That means no SSH credential matched the URL — Argo CD looked for an SSH agent instead, and has none. An HTTPS credential from the previous lab does not help, because credentials are matched by URL and the SSH URL is a different string.

## Alternative: Connect from the CLI

Task 5 can also be done without the UI. Reading the key with `--from-file` keeps it off your screen and out of shell variables and history, and stores it byte for byte:

```bash
kubectl -n argocd create secret generic private-repo-ssh --dry-run=client -o yaml \
  --from-literal=type=git \
  --from-literal=url=<your-private-repo-ssh-url> \
  --from-file=sshPrivateKey="$HOME/.ssh/argocd-deploy-key" \
| kubectl label --local -f - argocd.argoproj.io/secret-type=repository -o yaml \
| kubectl apply --server-side -f -
```

Use `$HOME` rather than `~` — the shell does not expand a tilde inside `--from-file=sshPrivateKey=~/…`.  
Only the last command talks to the cluster: the first two build and label the Secret locally, and `apply --server-side` creates it — without the second copy of your private key that plain `kubectl apply` would store in a `last-applied-configuration` annotation.  
The `url` must match the Application's `repoURL` exactly, and as with HTTPS, the Secret must be in the `argocd` namespace and carry the `argocd.argoproj.io/secret-type: repository` label, or it is silently ignored.

To check that GitHub accepts the deploy key itself, offering only that key:

```bash
ssh -i ~/.ssh/argocd-deploy-key -o IdentitiesOnly=yes -T git@github.com
```

## Helpful Resources

- [Argo CD - Private Repositories Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/)
- [GitHub - Managing Deploy Keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)
- [GitHub - Generating a New SSH Key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) (the key generation part only — Argo CD does not use an `ssh-agent`)

## Reflection Questions

1. What do SSH deploy keys offer over Personal Access Tokens for production GitOps workflows — and why does GitHub itself recommend a GitHub App over both?
2. What security advantages come from deploy keys being repository-specific rather than user-account-wide, and how does this align with the principle of least privilege?
3. Why is it important that deploy keys used by Argo CD don't have a passphrase, and how should you compensate for this security consideration?
