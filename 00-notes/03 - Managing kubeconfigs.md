#kubectl #kubeconfig #kubernetes #tls

# Managing kubeconfigs

How every cluster gets into `kubectl` without the config file turning into an unmaintainable pile. Verified against kubectl v1.37.0; the merge semantics below were tested empirically rather than taken from the docs.

## What a kubeconfig contains

Three independent lists plus a pointer. Everything else follows from this shape:

| Section | Holds | Example key |
|---|---|---|
| `clusters` | where the API server is, and the CA to trust it by | `server`, `certificate-authority-data` |
| `users` | how to authenticate | `client-certificate-data`, `token`, `exec` |
| `contexts` | a **named pairing** of one cluster, one user, and optionally a namespace | `cluster`, `user`, `namespace` |
| `current-context` | which context is active | a single string |

```yaml
contexts:
  - name: k3d-argocd          # the name we pass to --context
    context:
      cluster: k3d-argocd     # -> an entry in clusters:
      user: admin@k3d-argocd  # -> an entry in users:
      namespace: argocd       # optional; defaults to "default"
```

A context owns nothing — it is three references by name. That single fact explains why deleting a context leaves credentials behind, and why a name collision during a merge can orphan a cluster.

**Which file kubectl reads**, in precedence order:

1. `--kubeconfig <file>` on the command line
2. `$KUBECONFIG` — a **colon-separated list**, merged
3. `~/.kube/config`

## The model

Two possible arrangements:

| | Source of truth | Adding a cluster | Removing one |
|---|---|---|---|
| **In-place merge** | `~/.kube/config` itself | merge into it, compounding forever | hand-edit YAML |
| **Derived config** (used here) | per-cluster files in `~/.kube/cluster_configs/` | drop in a file, re-merge | `rm` the file, re-merge |

The second treats `~/.kube/config` as a **build artifact** we can delete and regenerate at any time. Layout follows [this netdevops.me post](https://netdevops.me/2024/managing-multiple-kubeconfigs-by-merging-them/), with fixes for three hazards it doesn't cover.

```text
~/.kube/
├── config                 # GENERATED — never hand-edit
├── config.new             # transient, during merge
├── merge.sh               # the generator
├── bak/                   # timestamped backups, one per merge
│   └── config_2026-09-04_12-24-10_bak
└── cluster_configs/       # SOURCE OF TRUTH — one file per cluster
    ├── k3d-argocd.yaml
    ├── k3s-prod.yaml
    └── k3s-qa.yaml
```

## How kubectl actually merges

`kubectl config view --flatten` collapses every file in `KUBECONFIG` into one document. Four rules matter.

**1. First-wins on name collisions, and it is silent:** verified with two files both defining a context named `dup`:

```bash
KUBECONFIG=a.yaml:b.yaml kubectl config view --flatten   # dup -> cluster c-a
KUBECONFIG=b.yaml:a.yaml kubectl config view --flatten   # dup -> cluster c-b
```

Exit code 0, nothing on stderr, no warning. The losing context vanishes without a trace. `find` returns no guaranteed order, so **always `sort`** — otherwise which cluster we reach can change between runs.

**2. Collisions are per-entry, not per-file:** in that same test both *clusters* (`c-a`, `c-b`) survived — only the colliding *context* was dropped, leaving one cluster present in the file but unreachable, because nothing pointed at it any more. A name clash discards individual entries, never a whole file.

**3. `--flatten` inlines every certificate and key:** the output is self-contained and portable — and holds every cluster credential we own in one file. `chmod 600` is not optional.

**4. `current-context` comes from the first file that sets one:** since `--minify` writes it into every extracted file, a plain re-merge silently repoints us at whatever sorts first — here `k3d-argocd.yaml`. The script below captures and restores it.

## Step 1 — one-time migration

Split an existing multi-context `~/.kube/config` into per-cluster files:

```bash
mkdir -p ~/.kube/cluster_configs ~/.kube/bak
cp ~/.kube/config ~/.kube/bak/config_$(date +%F_%H-%M-%S)_bak

for ctx in $(kubectl config get-contexts -o name); do
  kubectl config view --minify --flatten --context="$ctx" > ~/.kube/cluster_configs/"$ctx".yaml
  chmod 600 ~/.kube/cluster_configs/"$ctx".yaml
done
```

`--minify` restricts output to one context plus the cluster and user it references; `--flatten` inlines their certificates. The result is one self-contained file per cluster, each around 3 KB.

Verify before trusting it — each file should report exactly one context:

```bash
for f in ~/.kube/cluster_configs/*.yaml; do
  printf '%-28s %s\n' "$(basename "$f")" "$(KUBECONFIG=$f kubectl config get-contexts -o name | tr '\n' ' ')"
done
```

## Step 2 — the merge script

`~/.kube/merge.sh`, `chmod +x`. Run it after any change to `cluster_configs/`:

```bash
#!/usr/bin/env bash
set -euo pipefail
CUR="$(kubectl config current-context 2>/dev/null || true)"
mkdir -p ~/.kube/bak
[ -f ~/.kube/config ] && cp ~/.kube/config ~/.kube/bak/config_$(date +%F_%H-%M-%S)_bak
KUBECONFIG="$(find ~/.kube/cluster_configs -type f -name '*.yaml' | sort | tr '\n' ':')" \
  kubectl config view --flatten > ~/.kube/config.new
mv ~/.kube/config.new ~/.kube/config
chmod 600 ~/.kube/config
if [ -n "$CUR" ]; then kubectl config use-context "$CUR" >/dev/null 2>&1 || true; fi
kubectl config get-contexts
```

Four deliberate choices, each fixing a real failure:

- **`> config.new` then `mv`** — never redirect straight into `~/.kube/config`. The shell truncates the target *before* kubectl runs, so if that file is ever itself in `KUBECONFIG`, the redirect destroys it and kubectl reads an empty file. `mv` on the same filesystem is atomic.
- **`| sort`** — deterministic precedence, given first-wins.
- **`CUR` capture and restore** — stops a re-merge from silently switching our active cluster.
- **backup every run** — cheap, and the only thing standing between a typo and losing prod credentials.

## Step 3 — adding a cluster

Always fetch to a temp file, validate, *then* install. `>` creates the destination before the command runs, so a failure leaves a 0-byte file that silently pollutes the next merge.

**From k3d on a remote host:**

```bash
ssh pi-hole '/home/linuxbrew/.linuxbrew/bin/k3d kubeconfig get argocd' > /tmp/new.yaml
sed -i 's|https://0.0.0.0:6443|https://192.168.0.2:6443|' /tmp/new.yaml
```

k3d writes `https://0.0.0.0:6443`, which is meaningless from another machine — rewrite it to an address that appears in the certificate's SAN list (**02 - TLS SANs**). The absolute binary path is required over ssh, for reasons in **01 - K3d lab cluster on pi-hole**.

**From native k3s on a remote host:**

```bash
ssh host 'sudo cat /etc/rancher/k3s/k3s.yaml' > /tmp/new.yaml
sed -i 's|https://127.0.0.1:6443|https://<host-ip>:6443|' /tmp/new.yaml
```

k3s names its context `default` — **rename it before merging** or it will collide with every other k3s cluster we own. See Step 4.

**From a cloud provider or any other source:** whatever the tool hands us, the same three steps apply — validate, rename if generic, install.

Then, for any source:

```bash
KUBECONFIG=/tmp/new.yaml kubectl config get-contexts        # parses? one context?
KUBECONFIG=/tmp/new.yaml kubectl get nodes                  # actually reachable?

install -m 600 /tmp/new.yaml ~/.kube/cluster_configs/<name>.yaml
rm /tmp/new.yaml
~/.kube/merge.sh
```

Testing with `KUBECONFIG=` pointed at the standalone file proves the cluster works **before** it can affect `~/.kube/config`. If it fails here, nothing has been touched.

## Step 4 — rename generic context names

Name the file after the context, and keep both distinctive. `default`, `kubernetes-admin@kubernetes` and `k3s-default` all collide the moment we own two clusters.

```bash
KUBECONFIG=/tmp/new.yaml kubectl config rename-context default k3s-lab
```

Renaming the *context* leaves the cluster and user entries under their old names. That's harmless — they're referenced by name from the context — but for readable output rename all three. kubectl has no `rename-cluster` or `rename-user`, so the other two are a text edit on the standalone file:

```bash
KUBECONFIG=/tmp/new.yaml kubectl config rename-context default k3s-lab
sed -i 's/^\( *name: \)default$/\1k3s-lab/' /tmp/new.yaml
```

## Step 5 — verify the merge

```bash
kubectl config get-contexts                       # all clusters present?
kubectl config current-context                    # still where we expect?
kubectl --context=<new> get nodes                 # reachable THROUGH the merged file
```

The third matters: passing `--context` proves the merged config works, not just the standalone file that was already tested.

## Switching contexts

```bash
kubectl config use-context k3d-argocd     # switch
kubectl config current-context            # confirm
kubectl --context=k3s-qa get pods         # one-off, without switching
```

`--context` for a single command is safer than switching and forgetting to switch back — especially with a prod cluster in the same config.

Pin a default namespace to the active context so `-n argocd` stops being something we have to remember:

```bash
kubectl config set-context --current --namespace=argocd
```

This edits `~/.kube/config`, which is a generated file here — so make the same change in `cluster_configs/<name>.yaml` if we want it to survive the next merge.

## Removing a cluster

In this layout, deletion is a file operation:

```bash
rm ~/.kube/cluster_configs/k3d-argocd.yaml
~/.kube/merge.sh
```

If we ever edit `~/.kube/config` directly instead, note that **removal takes three commands, and `delete-context` alone is not enough.** Verified: after deleting only the context, its cluster and user entries — credentials included — were still on disk.

```bash
kubectl config delete-context k3s-qa
kubectl config delete-cluster k3s-qa
kubectl config delete-user    k3s-qa-admin
```

That is the three-lists model from the top of this note showing through: a context is a pointer, so removing it removes a pointer.

## Command reference

| Command | Purpose |
|---|---|
| `kubectl config get-contexts` | table of all contexts, `*` marks active |
| `kubectl config get-contexts -o name` | bare names, for scripting and loops |
| `kubectl config current-context` | just the active one |
| `kubectl config use-context <n>` | switch active context |
| `kubectl config set-context --current --namespace=<ns>` | pin a default namespace |
| `kubectl config view --flatten` | merge `KUBECONFIG` files, inline all certs |
| `kubectl config view --minify --context=<n>` | isolate one context |
| `kubectl config view --raw` | include credential data instead of redacting it |
| `kubectl config rename-context <old> <new>` | rename, in place |
| `kubectl config delete-context` / `-cluster` / `-user` | remove — all three needed |
| `KUBECONFIG=f1:f2 kubectl ...` | temporary merge, no files written |

## Recovery

Every merge writes a timestamped backup. To roll back:

```bash
ls -lt ~/.kube/bak/ | head
cp ~/.kube/bak/config_2026-09-04_12-24-10_bak ~/.kube/config
```

And because `cluster_configs/` is the real source of truth, the nuclear option is safe — delete `~/.kube/config` entirely and run `merge.sh` to rebuild it from scratch.
