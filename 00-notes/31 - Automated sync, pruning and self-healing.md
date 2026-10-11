#argocd #gitops #selfhealing #automation #pruning

# Automated sync, pruning and self-healing

Three switches sit under `spec.syncPolicy.automated`, and all three are off by default. Each closes a different gap between what Git says and what the cluster does, and they are independent — turning one on says nothing about the others:

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
```

## The default is to report, not to act

Out of the box Argo CD detects drift and stops there. It marks resources `OutOfSync` and waits to be told to act, which is a deliberate safety position rather than an omission: it leaves room to read the planned change before anything reaches the cluster.

The cost is that every deployment needs a human at the end of it, however well reviewed the commit already was. The three switches below buy that back, one gap at a time.

| Switch | Closes the gap when | Default |
|---|---|---|
| `automated` | a commit lands in Git and nothing applies it | off |
| `prune` | a manifest is deleted from Git and the object survives in the cluster | off |
| `selfHeal` | someone changes the cluster directly and Git never notices | off |

## Automated sync

> *"Argo CD has the ability to automatically sync an application when it detects differences between the desired manifests in Git, and the live state in the cluster."*

This collapses the two-step rhythm — Argo CD reports, a person syncs — into one. What it does not do is remove review from the process: approval moves upstream to the pull request, where a reviewer reads the change before it is merged. After the merge there is no second gate.

**One behaviour is easy to be caught by:** *"Automatic sync will not reattempt a sync if the previous sync attempt against the same commit-SHA and parameters had failed."* A failed sync is not retried in a loop against the same revision. Pushing a new commit, or syncing by hand, is what clears it — otherwise a broken revision sits failed and automation appears to have stopped working.

### The presence of the key is the switch

`automated` needs no contents to take effect. An empty object is a complete, working policy:

```yaml
spec:
  syncPolicy:
    automated: {}
```

That enables automatic syncing with `prune` and `selfHeal` left at their defaults of false — commits deploy themselves, nothing is deleted, manual edits survive. The value is stored exactly as written, so `automated: {}` and an absent `syncPolicy` are genuinely different states rather than two spellings of the same thing.

| Goal | How |
|---|---|
| Enable | add `automated` — `{}` is enough |
| Add pruning or self-healing | set the respective field to `true` |
| Suspend without losing the block | `enabled: false` — *"Enable allows apps to explicitly control automated sync"* |
| Disable entirely | remove the block, or `argocd app set <app> --sync-policy manual` (`none` is an alias) |

The CLI covers all three switches: `--sync-policy automated`, `--auto-prune`, `--self-heal`.

## Pruning

> *"By default (and as a safety mechanism), automated sync will not delete resources when Argo CD detects the resource is no longer defined in Git."*

Deleting a manifest from the repository is therefore not enough to remove the object. The resource keeps running, Argo CD keeps reporting the application `OutOfSync`, and the state persists until pruning is enabled or the object is removed by hand.

**Pruning is an action taken during a sync, never a background sweep:** nothing reaps orphaned objects on a timer. Setting `prune: true` does not reach back and delete what is already orphaned — it changes what the *next* sync is permitted to do. Until a sync actually runs, the object stays and the application stays `OutOfSync`.

**"Orphaned" means something narrower in Argo CD:** a resource left behind by a skipped prune is still *tracked* — it carries this application's tracking id, which is exactly how Argo CD knows it is a prune candidate in the first place. An `AppProject` separately carries `orphanedResources`, which *"specifies if controller should monitor orphaned resources of apps in this project"* — resources sitting in the project's namespaces that **no** application claims at all. The two are different problems: one is an object Argo CD owns and has not been authorised to delete, the other is an object nobody owns.

That sync can arrive either way, and the two paths read their prune setting from different places:

| Setting | Governs | Set by |
|---|---|---|
| `spec.syncPolicy.automated.prune` | automated syncs — *"as part of automated sync"* | the `Application` manifest |
| `operation.sync.prune` | a single manual sync — *"delete resources from the cluster that are no longer tracked in git"* | `argocd app sync --prune`, or the **Prune** checkbox in the UI |

They do not stand in for one another. `automated.prune: true` is scoped to automated sync by its own definition, so a hand-run `argocd app sync` on that same application still needs `--prune` to remove anything.

The practical consequence: with `automated` off, enabling `prune` changes nothing on its own, because no sync is ever triggered to act on it.

Pruning only reaches resources Argo CD knows are its own, which is what the tracking id decides — the mechanism in **13 - Argo CD Applications vs. Kubernetes primitive manifests**. An object with no tracking id, or another application's, is never a prune candidate no matter what the policy says.

### Guards worth knowing

| Setting | Effect |
|---|---|
| `allowEmpty: true` | permits a sync that would leave the application with zero resources. Off by default — *"automated sync with prune have a protection from any automation/human errors when there are no target resources"* |
| `Prune=false` | marks one resource as never prunable. The sync-status panel reports that pruning was skipped and why, and the application stays `OutOfSync` |
| `Prune=confirm` | holds the prune until approved, the operation staying in `Syncing` until someone confirms in the UI, the CLI, or by adding the `argocd.argoproj.io/deletion-approved` annotation |
| `PruneLast=true` | defers pruning until everything else has synced and become healthy |
| `PrunePropagationPolicy` | `foreground` (the default), `background` or `orphan` |

`Prune=false` and `Prune=confirm` go on the resource as an `argocd.argoproj.io/sync-options` annotation, or on the application in `spec.syncPolicy.syncOptions`:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Prune=false
```

A Namespace is the usual candidate for `Prune=confirm` — deleting one takes everything inside it, including objects no application tracks.

## Self-healing

Automated sync watches the repository. It does not watch the cluster:

> *"By default, changes that are made to the live cluster will not trigger automated sync."*

So a `kubectl edit` against a live Deployment leaves the application `OutOfSync` and nothing acts on it, because no new revision ever appeared. `selfHeal: true` closes that: a change to a tracked resource is reverted to what the repository says.

**Retries are spaced by exponential backoff, not a fixed interval:** repeated self-heal attempts would otherwise fight a failing resource at full speed. This install's controller exposes the timings:

| Flag | Default here |
|---|---|
| `--self-heal-backoff-timeout-seconds` | `2` — the first wait |
| `--self-heal-backoff-factor` | `3` — each wait triples |
| `--self-heal-backoff-cap-seconds` | `300` — the ceiling |
| `--self-heal-timeout-seconds` | unset |

Documentation for the current release describes a flat *"self-heal timeout (5 seconds by default)"* on `--self-heal-timeout-seconds`. The binary installed here carries the backoff trio instead, with that flag left without a default — worth reading from the controller rather than the docs when the exact timing matters.

**Self-healing and live debugging do not mix:** a value changed in the cluster to test something is reverted within seconds. The practical pattern is to turn it off for the duration and back on afterwards, rather than fighting it.

## The switches are independent

Each combination is a real position, not a stepping stone to the next:

| `automated` | `prune` | `selfHeal` | Behaviour |
|---|---|---|---|
| off | — | — | report only; every change is applied by hand |
| on | off | off | commits deploy themselves; deletions and manual edits do not |
| on | on | off | Git is authoritative for what exists, but manual edits survive |
| on | off | on | manual edits are reverted, but deleted manifests leave objects behind |
| on | on | on | the cluster tracks the repository in both directions |

The middle rows are where most applications sit. Full automation is a choice about blast radius, not a maturity level — an application sharing a namespace with resources other tools manage is a reason to leave `prune` off, or to scope it with `Prune=false`.

## Verify

```bash
kubectl -n argocd get app guestbook -o jsonpath='{.spec.syncPolicy}'   # which switches are on
argocd app set guestbook --self-heal                                    # enable one
argocd app set guestbook --auto-prune
kubectl -n argocd get app guestbook \
  -o jsonpath='{.status.operationState.phase}{" "}{.status.operationState.message}'
```

An empty `syncPolicy` is the giveaway that nothing is automated — it is absent rather than set to false. Sync and health status are read separately, as covered in **15 - Sync and health checks**.
