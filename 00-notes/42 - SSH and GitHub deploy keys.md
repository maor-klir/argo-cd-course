#argocd #gitops #ssh #github #credentials

# SSH and GitHub deploy keys

SSH replaces the username and token with a key pair. Argo CD holds the private half in a repository Secret; on GitHub, the public half is usually registered as a **deploy key** on the repository itself. Everything about how Argo CD finds the Secret — the `argocd` namespace, the label, the URL as the join key — is unchanged from **41 - Connecting to private repositories**.

## Switching an Application to SSH

The repository URL changes form, and the `Application` has to change with it:

| | URL |
|---|---|
| HTTPS | `https://github.com/<owner>/<private-repo>.git` |
| SSH | `git@github.com:<owner>/<private-repo>.git` — the **SSH** tab under the repository's **Code** button |

**An HTTPS credential for the same repository does not carry over:** credentials are matched by URL, and the two URLs are different strings. Repointing the `Application` at the SSH form while only an HTTPS Secret exists leaves it with no credential at all:

```text
Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = Unknown desc = failed to list refs: error creating SSH agent: "SSH agent requested but SSH_AUTH_SOCK not-specified"
```

With no SSH key matched, Argo CD's git client falls back to asking an **SSH agent** for one — the default behaviour of the git library underneath. The repo-server runs no agent, so `SSH_AUTH_SOCK` is unset and the attempt fails before any connection is made.

The same mistake therefore reads differently by transport, but means the same thing:

| Transport | The no-credential message ends with | Where it failed |
|---|---|---|
| HTTPS | `authentication required: Repository not found.` | at GitHub, which answered `401` |
| SSH | `error creating SSH agent: "SSH agent requested but SSH_AUTH_SOCK not-specified"` | inside the repo-server, before reaching GitHub |

Either way: no Secret matched this URL.

## What a deploy key is

GitHub defines it as *"an SSH key that grants access to a single repository"*, and is specific about where each half lives: GitHub attaches *"the public part of the key directly to your repository instead of a personal account, and the private part of the key remains on your server"* — the server here being Argo CD.

It is a **repository** setting, not an account one: **Settings → Deploy keys → Add deploy key** on the repository, with a title and the contents of the `.pub` file.

**Leave "Allow write access" unchecked:** Argo CD only ever reads the repository, and the alternative is far broader than it sounds — *"Deploy keys with write access can perform the same actions as an organization member with admin access, or a collaborator on a personal repository."*

**One key per repository, without exception:** *"You can't reuse a deploy key for multiple repositories."* Ten private repositories means ten key pairs and ten Secrets. It also rules out the credential-template shortcut from note 41 — a `repo-creds` Secret holds a single `sshPrivateKey`, and a deploy key opens exactly one repository.

## The advantages, against GitHub's own list

The usual case for deploy keys over personal access tokens holds up — but GitHub's documentation files several of the same properties under **cons**:

| Property | What GitHub says |
|---|---|
| repository-specific | confirmed — and also a con: *"More complex projects may have many repositories to pull to the same server."* |
| no user-account dependency | confirmed, as a con: *"If the user who created the deploy key is removed from the repository, the deploy key will still be active as it isn't tied to the specific user, but rather to the repository."* The key survives its creator leaving — useful for Argo CD, and also a credential nobody may remember to revoke. One exception sits on GitHub's token page: *"If you delete a personal access token that was used to create a deploy key, the deploy key will also be deleted."* A key created by automation through the REST API is only as durable as the token that created it |
| read-only by default | confirmed — *"read-only by default, but you can give them write access"*: a default, not a restriction |
| long-lived | confirmed, as a con: *"Deploy keys are credentials that don't have an expiry date."* Rotation is entirely on the owner |
| auditable | not mentioned anywhere on GitHub's deploy-key page |

GitHub's own recommendation sits above all of this: *"For enhanced security and fine-grained control over repository access and permissions, we recommend using a GitHub App instead."* Argo CD supports that too, through the `githubAppID`, `githubAppInstallationID` and `githubAppPrivateKey` fields of the same repository Secret.

## Generating the key pair

```bash
ssh-keygen -t ed25519 -C "argocd-deploy-key" -f ~/.ssh/argocd-deploy-key
```

| Flag | Purpose |
|---|---|
| `-t ed25519` | the algorithm GitHub recommends; RSA 4096 is its fallback *"If you are using a legacy system that doesn't support the Ed25519 algorithm"* |
| `-C "argocd-deploy-key"` | a comment stored with the key — a label, nothing more |
| `-f ~/.ssh/argocd-deploy-key` | the private key's path; the public half lands beside it as `argocd-deploy-key.pub` |

**Press Enter at both passphrase prompts:** Argo CD has nowhere to put a passphrase. The only SSH credential field in a repository Secret is `sshPrivateKey`, and neither the documentation nor `argocd repo add` — whose only key option is `--ssh-private-key-path` — offers one. GitHub names the cost plainly: *"Deploy keys are usually not protected by a passphrase, making the key easily accessible if the server is compromised."* The Secret holding the key is its only protection.

GitHub's key-generation guide goes on to add the new key to an `ssh-agent`. That half does not apply here — Argo CD reads the key from its Secret, and the absence of an agent is exactly what the error above is about.

## Connecting it to Argo CD

**From the UI:** **Settings → Repositories → Connect Repo using SSH**, with the `git@` URL exactly as the `Application` uses it, and the whole private key — including the `BEGIN` and `END` lines — pasted in. The docs warn to *"make sure there are no unintended line breaks or additional characters in the text area"*.

**From the command line:**

```bash
kubectl -n argocd create secret generic private-repo-ssh --dry-run=client -o yaml \
  --from-literal=type=git \
  --from-literal=url=git@github.com:<owner>/<private-repo>.git \
  --from-file=sshPrivateKey="$HOME/.ssh/argocd-deploy-key" \
| kubectl label --local -f - argocd.argoproj.io/secret-type=repository -o yaml \
| kubectl apply --server-side -f -
```

`--from-file` reads the key straight from disk, so it is never printed, held in a shell variable, or recorded in history, and it is stored byte for byte. Reading it through `$(cat …)` instead drops the trailing newline — 399 bytes become 398 for an Ed25519 key. Use `$HOME` rather than `~`: inside `--from-file=sshPrivateKey=~/…` the shell does not expand the tilde, and `kubectl` receives it literally.

The remaining stages work as explained in note 41: the Secret is built and labelled locally, then written once with `apply --server-side` — which, unlike plain `apply`, does not keep a second copy of the private key in a `last-applied-configuration` annotation.

Either route produces the same labelled Secret, and each shows up in the other — the UI lists a `kubectl`-created repository alongside its own.

Nothing needs adding for GitHub's host key: `github.com` is among the known hosts Argo CD ships with, as covered in note 41.

## Verify

```bash
ssh -i ~/.ssh/argocd-deploy-key -o IdentitiesOnly=yes -T git@github.com   # does GitHub accept this key?
kubectl -n argocd get secret -l argocd.argoproj.io/secret-type            # the Secret exists, labelled
kubectl -n argocd get app guestbook-private -o jsonpath='{.status.sync.status}'
```

GitHub answers the first with *"You've successfully authenticated, but GitHub does not provide shell access"* when the key is accepted, and `Permission denied (publickey).` when it is not. `IdentitiesOnly=yes` is what makes the test meaningful: without it, `ssh` may also offer a personal key from `~/.ssh` or an agent, and a successful greeting would prove nothing about the deploy key.
