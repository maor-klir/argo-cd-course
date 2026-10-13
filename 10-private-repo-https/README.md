## Lab Goal

Connect Argo CD to a private Git repository using HTTPS authentication with a username and Personal Access Token (PAT).

## Overview & Concepts

In production environments, your GitOps repositories will almost always be private. Argo CD needs credentials to access these repositories. The simplest authentication method is using HTTPS with a username and Personal Access Token (PAT).

A Personal Access Token is like a password, but it's designed for applications rather than humans. It can be scoped with specific permissions (like read-only access to repositories) and can be easily revoked if compromised. This makes PATs much safer than using your actual password.

In this lab, you'll create a private repository, generate a PAT from your Git provider, and configure Argo CD to authenticate using these credentials.

## Lab Tasks

1.  Create a new **private** repository on your Git provider (GitHub, GitLab, etc.)
2.  Clone the private repository to your local machine
3.  Copy the `helm-guestbook` chart from your `argocd-example-apps` fork into the new private repository
4.  Commit and push the chart to your private repository
5.  Generate a Personal Access Token (PAT) from your Git provider with appropriate permissions:
    - The token needs read access to repositories
    - Set an appropriate expiration date
    - **Save the token immediately** - you won't be able to see it again
6.  In the Argo CD UI, navigate to Settings → Repositories
7.  Connect a new repository using:
    - Connection method: HTTPS
    - Repository URL: The HTTPS URL of your private repository
    - Username: Your Git provider username
    - Password: Your Personal Access Token
8.  Test the connection to ensure Argo CD can access the repository
9.  Create a new Application manifest (e.g., `guestbook-private.yaml`) that references your private repository
10. Apply the manifest and verify the application syncs successfully from the private repository

## Alternative: Connect from the CLI 

Tasks 6–8 can also be done without the UI, by creating the credential Secret directly. Avoid writing it to a YAML file — a manifest holding your token is one `git add` away from being committed, and this repository is a public fork:

```bash
read -rs TOKEN   # paste your PAT; it is not echoed or saved to shell history
kubectl -n argocd create secret generic private-repo-https --dry-run=client -o yaml \
  --from-literal=type=git \
  --from-literal=url=<your-private-repo-https-url> \
  --from-literal=username=<your-git-username> \
  --from-literal=password="$TOKEN" \
| kubectl label --local -f - argocd.argoproj.io/secret-type=repository -o yaml \
| kubectl apply -f -
unset TOKEN
```

The `url` must be the same URL your `Application` uses in `repoURL`. Two more details decide whether Argo CD can use the Secret, and getting either wrong fails silently — the Secret is created, but the Application's error does not change:

- it must be in the `argocd` namespace
- it must carry the label `argocd.argoproj.io/secret-type: repository`

If you created the Secret without the label, add it afterwards and refresh the Application — there is no need to recreate it:

```bash
kubectl -n argocd label secret private-repo-https argocd.argoproj.io/secret-type=repository
```

Either way, `kubectl -n argocd get secret -l argocd.argoproj.io/secret-type=repository` lists the result — a repository connected through the UI shows up here too, because the UI creates the same kind of Secret.

## Key Concepts

**Why use a PAT instead of a password?**

- PATs can be scoped with minimal permissions (principle of least privilege)
- PATs can be revoked individually without changing your password
- PATs can have expiration dates for automatic rotation
- Many Git providers now require PATs for API access

**Security Best Practices:**

- Use read-only tokens when possible
- Set reasonable expiration dates
- Store tokens securely (never commit them to Git!)
- Rotate tokens periodically
- Revoke tokens when they're no longer needed

## Helpful Resources

- [Argo CD - Private Repositories Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/)
- [GitHub - Creating a Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

## Reflection Questions

1. Why are Personal Access Tokens considered more secure than passwords for service-to-service authentication, even though they function similarly?
2. What permissions should you grant to a PAT used by Argo CD following the principle of least privilege, and why is read-only access typically sufficient?
3. How does token expiration improve security, and what operational considerations does it introduce for your GitOps workflow?
