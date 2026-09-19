#argocd #cli #kubernetes #rbac

# The Argo CD CLI and core mode

The `argocd` CLI is normally a **client of `argocd-server`** — it speaks gRPC to the API server, which means it needs a reachable endpoint and a login. On a lab cluster with no ingress, that endpoint does not exist until we make one.

There are three ways to give it one, and the third bypasses `argocd-server` altogether.

## Three ways the CLI reaches Argo CD

| Mode | How it connects | Needs |
|---|---|---|
| **Direct** | gRPC to a server address we supply | a reachable `argocd-server` — ingress, LoadBalancer, or a port-forward we run ourselves |
| **`--port-forward`** | the CLI opens its own port-forward to `argocd-server` | cluster access; no manual tunnel, no address to remember |
| **`--core`** | talks to the Kubernetes API directly, no `argocd-server` involved | cluster access and RBAC on Argo CD's resources |

The middle one is the least known and often the most convenient:

```bash
argocd app list --port-forward --port-forward-namespace argocd
```

*"Connect to a random argocd-server port using port forwarding"* — the CLI sets up and tears down the tunnel itself, so there is no second terminal to keep alive.

## Connecting to the API server

```bash
argocd login localhost:8080                    # username / password
argocd login cd.argoproj.io --sso              # single sign-on
```

Useful flags when the server is not plainly reachable:

| Flag | For |
|---|---|
| `--plaintext` | the server is serving HTTP, not HTTPS |
| `--insecure` | skip certificate and domain verification — self-signed certs |
| `--grpc-web` | the server sits behind a proxy without HTTP/2 support |
| `--name` | store the context under a name of our choosing |

`argocd login` stores the server details and the resulting auth token in the CLI's own config and makes that context current.

## Core mode

Officially: *"If set to true then the Argo CD CLI talks directly to Kubernetes instead of talking to Argo CD API server."*

With `--core` the CLI **spawns a local API server process transparently** and reads the cluster directly. Nothing to port-forward, nothing to log into.

**Two different things share the name "core":**

| | What it is |
|---|---|
| `--core` (the flag) | a **client-side** choice to bypass `argocd-server`. Works against any install, including a completely normal one |
| Argo CD Core (the install) | a **server-side** install mode that omits the API server, Dex, notifications and the Argo CD RBAC model entirely |

The flag does not require the install mode. Everything documented here was read with `--core` against a standard chart install running a perfectly healthy `argocd-server`.

**The current kube context namespace must be the Argo CD namespace:** this is the part that bites, because there is no `-n` flag for it and the failure names the wrong problem.

```text
FATA[0000] configmap "argocd-cm" not found
```

That is core mode looking for Argo CD's config in whatever namespace our context points at. The documented fix:

```bash
kubectl config set-context --current --namespace=argocd
```

If we would rather not move our active context, point `KUBECONFIG` at a copy with the namespace set:

```bash
cp ~/.kube/config /tmp/kc-argocd.yaml
KUBECONFIG=/tmp/kc-argocd.yaml kubectl config set-context --current --namespace=argocd
KUBECONFIG=/tmp/kc-argocd.yaml argocd app get guestbook --core
```

**Authorisation falls back to Kubernetes RBAC:**  
Bypassing `argocd-server` also bypasses Argo CD's own RBAC model and any SSO.  
The docs are explicit that *"the user (or the process) invoking the CLI needs to have access to the Argo CD namespace with the proper permission in the Application and ApplicationSet resources."*  
Whatever our kubeconfig can do to those resources is what we can do — no more, no less. Worth remembering before assuming a project restriction protects something.

### Core mode is a saved context, not a one-way switch

Typing `--core` on every command gets old, so the CLI can remember it:

```bash
argocd login --core
```

```text
Context 'kubernetes' updated
```

That wording is the important part. It **adds** a context rather than reconfiguring anything, writing to the CLI's own config at `~/.config/argocd/config`:

```yaml
contexts:
- name: pi-hole            # an earlier server login — untouched
  server: localhost:8080
- name: kubernetes         # added by --core
  server: kubernetes
current-context: kubernetes
servers:
- core: true               # this is what marks the context as core mode
  server: kubernetes
```

Core mode is an ordinary context with `core: true` set. Nothing is lost, and going back is one command:

```bash
argocd context                 # list contexts, * marks the current one
argocd context pi-hole         # back to talking to argocd-server
argocd context kubernetes      # back to core mode
```

```text
CURRENT  NAME        SERVER
*        pi-hole     localhost:8080
         kubernetes  kubernetes
```

The full set of ways to move between the two:

| Goal | Command |
|---|---|
| Switch the default back to the API server | `argocd context <name>` |
| Switch the default to core mode | `argocd context kubernetes` |
| One command against the API server, without switching | `argocd app get <app> --server localhost:8080` |
| One command in core mode, without switching | `argocd app get <app> --core` |
| Remove the core context entirely | `argocd context kubernetes --delete` |
| Re-login to a server, recreating its context | `argocd login localhost:8080` |

The `--core` flag works per command regardless of which context is current, so the context only sets the default. **A context pointing at `localhost:8080` depends on a live port-forward:** switching back to it with no tunnel running gives a connection error, which means the tunnel is missing, not that the context is broken.

**That config file holds bearer tokens in plaintext:** one per server login. It is mode 600, and should stay that way.

## The local Web UI

`argocd admin dashboard` starts the Web UI locally, with no port-forward to `argocd-server` at all:

```bash
argocd admin dashboard -n argocd            # http://localhost:8080
argocd admin dashboard --port 9090 --address 127.0.0.1
```

Together with `--core`, this makes the whole of Argo CD reachable from a laptop holding nothing but cluster credentials — useful against a remote cluster, and the quickest recovery when a port-forward dies mid-session.
