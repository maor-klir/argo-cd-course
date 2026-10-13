#argocd #gitops #git #credentials #secrets

# Connecting to private repositories

An `Application` pointing at a private repository looks exactly like one pointing at a public repository. Nothing in it carries credentials:

```yaml
spec:
  source:
    repoURL: https://github.com/<owner>/<private-repo>.git
    path: helm-guestbook
    targetRevision: HEAD
```

The credentials live somewhere else entirely — in Secrets in the Argo CD namespace — and Argo CD pairs them with the `Application` by URL. Making a private repository work is never a change to the `Application`; it is creating the right Secret.

## Without credentials

An `Application` pointing at a private repository with no matching credential records a `ComparisonError` condition:

```text
Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = Unknown desc = failed to list refs: authentication required: Repository not found.
```

Each segment is added by a different layer, outermost first:

| Segment | Comes from |
|---|---|
| `Failed to load target state:` | the application controller, unable to obtain the desired state |
| `failed to generate manifest for source 1 of 1:` | the controller's request to the repo-server, numbered per source — a single `source` is 1 of 1 |
| `rpc error: code = Unknown desc =` | the gRPC call between the two |
| `failed to list refs:` | the repo-server's first step — resolving `targetRevision` by listing the remote's branches and tags |
| `authentication required:` | the server's HTTP `401` |
| `Repository not found.` | GitHub's own response body, passed through verbatim |

`failed to list refs` places the failure precisely: the revision was never resolved, so nothing was fetched. `kubectl -n argocd get app -o wide` shows it too — the `REVISION` column reads `HEAD` instead of a commit SHA.

**The message cannot tell a missing credential from a wrong URL:** GitHub answers both identically. Asking for the private repository with no credentials, and for a repository that does not exist at all, returns the same response byte for byte:

```text
HTTP 401
Repository not found.
```

GitHub does not confirm that a private repository exists to a caller who cannot see it. A typo in `repoURL` therefore produces exactly this error, and no credential will fix it — both are worth checking before assuming either.

**`Healthy` means nothing here:** the same `Application` reports sync status `Unknown` and health `Healthy`, with zero resources. *"Argo CD App health is inferred from the health of its immediate child resources as represented in the application source"* — and with the source unreadable, there are no child resources to be anything but healthy. The sync status is what flags the problem — `Unknown`, because there is no desired state to compare against — as covered in **15 - Sync and health checks**.

Adding the repository from the CLI fails at the same point, before anything is saved — the documentation's own example:

```text
FATA[0000] rpc error: code = Unknown desc = authentication required
```

**The end of the message says whether a credential was sent at all:** GitHub's `401` carries a different body in each case, and Argo CD passes the body through verbatim:

| Message ends with | What happened |
|---|---|
| `authentication required: Repository not found.` | no credential was sent — no Secret matched the URL, or the URL itself is wrong |
| `Invalid username or token. Password authentication is not supported for Git operations.` | a Secret matched and its credential was sent, but GitHub rejected it |

Both bodies were checked against GitHub directly, with and without a credential attached. The first row is what every Secret mistake in the next section produces; the second points at the token itself. Its second sentence matters on its own: an account password is refused outright, so the credential has to be a token.

## Credentials are Secrets, found by label and URL

> *"Repository details are stored in secrets. To configure a repo, create a secret which contains repository details."*

Three things have to be right for Argo CD to find one:

| Requirement | Why |
|---|---|
| it lives in the `argocd` namespace | that is where Argo CD looks — not the application's destination namespace |
| it carries `argocd.argoproj.io/secret-type: repository` | the label is how these Secrets are told apart from every other Secret in the namespace; without it the Secret is not considered |
| its `url` matches the `Application`'s `repoURL` | the URL is the join key between the two |

**Get any of the three wrong and nothing complains:** `kubectl` creates the Secret without objection, and the `Application`'s error does not change — it stays the no-credential message, because to Argo CD a Secret in another namespace, or one without the label, does not exist. A missing label can be added after the fact, with no need to recreate the Secret; it takes effect on the next refresh:

```bash
kubectl -n argocd label secret private-repo-https argocd.argoproj.io/secret-type=repository
```

The label's value says what kind of credential the Secret holds:

| `secret-type` | Holds |
|---|---|
| `repository` | one repository — its URL and, optionally, its credentials |
| `repo-creds` | a credential template applied to every repository under a URL prefix |
| `cluster` | a destination cluster's connection details — the same mechanism, applied to clusters |

**`type` says what is at the URL:** `git` for a Git repository, `helm` for a Helm chart repository. A Helm repository Secret *"must specify the `url`, `type` and `name` fields"*, so the same labelled-Secret pattern covers private chart repositories as well.

**Argo CD does not follow redirects on repository URLs:** some hosts, *"notably GitLab"*, redirect a URL without the `.git` suffix, and *"Argo CD will **not** follow these redirects"*. Write repository URLs with `.git`, and use the same string in the Secret and the `Application`.

## HTTPS: a username and a token

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo-https
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/<owner>/<private-repo>.git
  username: <git-username>
  password: <personal-access-token>
```

The token goes in `password` — there is no separate token field. The username matters less than it looks: *"connect the repository using any non-empty string as username and the access token value as a password"*. The exceptions are spelled out too — *"For some services, you might have to specify your account name as the username"*, and Bitbucket requires `x-token-auth`.

The same Secret can be created without writing YAML:

```bash
argocd repo add https://github.com/<owner>/<private-repo>.git --username <user> --password <token>
```

or from the UI under **Settings → Repositories → Connect Repo using HTTPS**. Both test the connection before saving — *"Click `Connect` to test the connection and have the repository added"* — which is why a bad credential fails at that step rather than later inside an `Application`.

**Without a manifest on disk:** a file holding the token is one `git add` away from being committed. Creating the Secret straight from the command line avoids writing one at all:

```bash
read -rs TOKEN          # paste the token: not echoed, not recorded in shell history
kubectl -n argocd create secret generic private-repo-https --dry-run=client -o yaml \
  --from-literal=type=git \
  --from-literal=url=https://github.com/<owner>/<private-repo>.git \
  --from-literal=username=<git-username> \
  --from-literal=password="$TOKEN" \
| kubectl label --local -f - argocd.argoproj.io/secret-type=repository -o yaml \
| kubectl apply -f -
unset TOKEN
```

The pipeline sets the namespace and the label in the same step — the two things a bare `kubectl create secret` leaves out unless told. Typing the token straight into `--from-literal=password=…` works as well, but records it in shell history.

**Scope the token to the one repository, read-only:** syncing only ever reads from the repository. A fine-grained token limited to that repository's contents gives Argo CD exactly what it uses; anything broader is exposure with no benefit.

## SSH: a private key, and a deploy key on the other side

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:<owner>/<private-repo>.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

`sshPrivateKey` replaces `username` and `password`, and the URL changes form — `git@` or `ssh://` instead of `https://`. Because matching is by URL, a repository registered under its HTTPS address and the same repository under its SSH address are two different entries; the `Application` has to use the form the Secret was created for.

The private half of the key sits in the Secret. The public half is registered with the Git provider, usually as a **deploy key** on the repository itself — a key that grants access to that one repository rather than to a user account. On GitHub a deploy key is read-only unless write access is explicitly allowed, which is exactly the access Argo CD needs.

Two practical details:

- **The server's host key is checked too:** Argo CD verifies the SSH host key before trusting the connection. This install ships known hosts for `github.com`, `gitlab.com`, `bitbucket.org`, `ssh.dev.azure.com`, `vs-ssh.visualstudio.com` and `[ssh.github.com]:443`; any other host needs its key added with `argocd cert add-ssh`, fed from `ssh-keyscan`. `--insecure-skip-server-verification` exists, but the docs limit it to *"non-production setups, as it imposes a serious security issue through possible man-in-the-middle attacks"*
- **A non-standard port needs the `ssh://` form:** *"The scp-style `git@yourgit.com:yourrepo` URLs do **not** support port specification, and will treat any port number as part of the repository's path."*

## `stringData` or `data`

The examples above use `stringData`, but that is never what Argo CD reads. Kubernetes stores only `data`:

> *"It is provided as a write-only input field for convenience. All keys and values are merged into the data field on write, overwriting any existing values. The stringData field is never output when reading from the API."*

A `stringData` repository Secret submitted through a server-side dry run comes back with `data` alone, every value base64-encoded. Writing `data` directly is therefore equivalent — it skips the conversion, nothing more:

```yaml
data:
  type: Z2l0                                                     # git
  url: aHR0cHM6Ly9naXQuaW50ZXJuYWwuZXhhbXBsZS90ZWFtL2FwcC5naXQ=  # https://git.internal.example/team/app.git
  username: YXJnb2Nk                                             # argocd
  password: czNjcjN0LXRva2Vu                                     # the token
```

Only the values are encoded; the `argocd.argoproj.io/secret-type` label stays plain text.

| Pitfall | Effect |
|---|---|
| `echo 'token' \| base64` | `echo` appends a newline, encoding 13 bytes instead of 12 — the password silently gains a trailing `\n` and authentication fails |
| encoding only the password | every value under `data` must be base64, `type` and `url` included |
| the same key in both fields | `stringData` wins — it is merged in *"overwriting any existing values"* |

**base64 is an encoding, not encryption:** one `base64 -d` recovers the token, so a `data` Secret is exactly as unsafe to commit as a `stringData` one.

Letting `kubectl` do the encoding avoids the newline trap entirely — the command-line route in the HTTPS section produces correct `data` with no hand-encoding.

## Self-hosted and air-gapped servers

Nothing about the Secret changes for an internal Git server — the same keys, in `stringData` or `data`. What changes is everything involved in reaching the server:

| Concern | What to do |
|---|---|
| a certificate from an internal CA, or self-signed | the repository is refused, *"indicated by an error message such as `x509: certificate signed by unknown authority`"*. Add the server's certificate, or the CA's, to `argocd-tls-certs-cm` keyed by *"the repository server's hostname part (not the complete URL)"*, or use `argocd cert add-tls`. It is configured *"on a per-server, not on a per-repository basis"* |
| an expired certificate, or one issued for another name | *"adding a CA certificate will not help"* — the only workaround is `--insecure-skip-server-verification`, which the docs limit to non-production |
| SSH to an internal host | its host key is not among the shipped known hosts — add it with `argocd cert add-ssh`, as in the SSH section above |
| a proxy between Argo CD and the server | `proxy` and `noProxy` keys in the same Secret — base64-encoded as well when using `data` |
| self-hosted GitLab | write the `.git` suffix, since the redirect to it is not followed |
| Helm charts with no internet access | *"Even if the chart uses only dependencies from internal repos Helm might decide to refresh `stable` repo."* The documented workaround points the `stable` name at an internal Helm repository in `argocd-cm` |

`argocd-tls-certs-cm` is empty on this install, so only *"the system's default trust store"* applies — enough for public hosts and certificates from well-known CAs, and nothing more. The ConfigMap is mounted into `argocd-server` and `argocd-repo-server` at `/app/config/tls`, and changes *"might take a while"* to reach the pods.

## Credential templates: one Secret for many repositories

A `repo-creds` Secret holds credentials once, for every repository under a URL prefix:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-owner-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  type: git
  url: https://github.com/<owner>
  username: <git-username>
  password: <personal-access-token>
```

Every repository whose URL starts with that prefix borrows these credentials, under two conditions:

- the repository must either not be configured at all, or *"if configured, must not contain any credential information"*
- the template's URL must *"match as prefix for the repository URL"*, and when several match, *"the longest (best) match will take precedence"*

The first condition has a sharp edge, shown in the documentation's own CLI session: a repository added with *invalid* credentials of its own fails even when a valid template covers it. Its own credentials are tried, and the template never is.

From the CLI: `argocd repocreds add <prefix> --username <user> --password <token>`.

## Scoping a credential to a project

> *"Credentials can be scoped to a project using the optional `project` field. When omitted, the credential will be used as the default for all projects without a scoped credential."*

Without the field, a credential is available to every project in the install — worth knowing before projects are used to separate teams.

## The one manifest that should not go to Git as-is

Everything else in a GitOps setup belongs in the repository. A credential Secret is the exception: its `stringData` is the token itself, and committing it hands the credential to everyone who can read the repository. 

The docs point to an encrypted alternative — *"Consider using bitnami-labs/sealed-secrets to store an encrypted secret definition as a Kubernetes manifest"* — with a caveat that matters here: *"the labels will be removed and have to be readded"*. The label is what makes the Secret visible to Argo CD at all, so a sealed Secret without it fails exactly as if it were missing.

Beyond that, the docs' secret-management guidance favours secrets *"populated on the destination cluster"*, naming Sealed Secrets, External Secrets Operator and the Kubernetes Secrets Store CSI Driver: *"We strongly recommend this style of secret management."* With an operator such as External Secrets Operator the token lives in an external store, and nothing about it is committed or typed by hand. The requirements do not move with it — whatever writes the repository Secret still has to put it in the `argocd` namespace, with the label.

In a test setup, the command-line route in the HTTPS section or the UI keeps the Secret out of Git entirely.

## Verify

```bash
kubectl -n argocd get secret -l argocd.argoproj.io/secret-type   # every repository and credential Secret
argocd repo list                                                  # each repository's connection status
kubectl -n argocd get app <name> \
  -o jsonpath='{.status.conditions}'                              # a ComparisonError means the repository was not reached
```

Whichever route created the repository — a manifest, `kubectl`, the CLI or the UI — the first command lists it, because all of them produce the same kind of labelled Secret. Connecting through the UI was checked this way: it creates a labelled Secret holding `type`, `url`, `username` and `password`, plus `project` when one is chosen — and **Disconnect** deletes it again.
