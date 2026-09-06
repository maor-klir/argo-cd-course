#argocd #kubernetes #gitops #sync #health

# Sync and health checks

Argo CD reports two statuses side by side for every application, and they answer different questions about different things.  
Confusing them wastes debugging time, because each has a failure mode the other cannot see.

| | Sync status | Health status |
|---|---|---|
| Question | does the live state match the desired state? | are the deployed resources actually working? |
| Compares | desired state ↔ live state | live state only |
| Source of truth | the manifests in Git | the resources' own status in the cluster |
| Fails when | someone edited the cluster, or Git moved on | the workload is broken, whatever Git says |

Official definitions: sync status is *"whether or not the live state matches the target state — is the deployed application the same as Git says it should be?"*, while health is *"the health of the application, is it running correctly? Can it serve requests?"*

The shapes differ:
- Sync is a **two-sided comparison** — Argo CD renders the manifests from the repository, reads the live objects, and diffs them.
- Health is a **one-sided inspection** — it looks only at what is running and asks whether each resource reports itself as working.

## Sync status

Three possible values:

| Value | Meaning |
|---|---|
| `Synced` | the live state matches the desired state |
| `OutOfSync` | the live state does not match — configuration drift, or a newer commit in Git |
| `Unknown` | the comparison could not be made at all, e.g. the repository is unreachable |

**`Progressing` is not a sync status:** it is a *health* value. While a sync is running you will see movement in the UI, but that comes from the operation phase and from health transitions — the sync status field itself only ever holds one of the three values above.  
Argo CD's built-in notification trigger `on-sync-status-unknown` fires on *"Application status is 'Unknown'"*, which is the third value in the same set.

Read it directly:

```bash
kubectl -n argocd get app guestbook -o jsonpath='{.status.sync.status}'
```

## Health status

Six possible values:

| Value | Meaning |
|---|---|
| `Healthy` | the resource is healthy |
| `Progressing` | not healthy yet, but still making progress and may yet become healthy |
| `Degraded` | the resource is degraded |
| `Suspended` | suspended, waiting on an external event to resume |
| `Missing` | should exist according to the source, but is absent from the cluster |
| `Unknown` | health could not be determined |

**Application health is the worst status among its immediate children**, ranked from healthiest to least:  
`Healthy → Suspended → Progressing → Missing → Degraded → Unknown`. One `Degraded` Deployment makes the whole application `Degraded`.

**Health is calculated per resource, from that resource alone:** it is *"not inherited from child resources — it is calculated using only information about the resource itself"*. A Deployment reports its own status conditions; it does not average its Pods.

Argo CD ships built-in health assessments for the standard kinds:

```text
Deployment · ReplicaSet · StatefulSet · DaemonSet
Service · Ingress
Job · CronJob
PersistentVolumeClaim
Argo CD Application
```

Anything outside that list — most custom resources — reports `Healthy` by default unless you teach Argo CD how to judge it.  
Custom checks are written in **Lua** and configured in the `argocd-cm` ConfigMap under `resource.customizations.health.<group>_<kind>`, or contributed upstream into `resource_customizations/`.  
Standard Lua libraries are disabled unless enabled per kind via `resource.customizations.useOpenLibs.<group>_<kind>`.

## The two axes are independent

Every combination is reachable, and each means something specific:

| | `Healthy` | `Degraded` |
|---|---|---|
| **`Synced`** | the normal state | **the cluster matches Git, and Git is wrong** — a bad image tag, a broken config |
| **`OutOfSync`** | the running version works; it just isn't the committed one | drifted *and* broken |

The top-right cell is the one worth internalising. A perfect sync guarantees only that Argo CD applied what the repository asked for. If the repository asks for a container image that crash-loops, the application is `Synced` and `Degraded` at the same time, and syncing again will not help — the fix belongs in Git.

## Not every resource has a sync status

Only resources defined in the desired state can be compared against it. Resources created by *other controllers* have nothing to compare to, so they carry a health status and no sync status.

The guestbook app makes this concrete. The repository holds two manifests, so Argo CD tracks two resources:

```bash
kubectl -n argocd get app guestbook -o jsonpath='{range .status.resources[*]}{.kind}/{.name}{"\n"}{end}'
```

```text
Service/guestbook-ui
Deployment/guestbook-ui
```

But three workload objects exist in the cluster:

```bash
kubectl -n default get deploy,rs,pod
```

```text
deployment.apps/guestbook-ui
replicaset.apps/guestbook-ui-6595f948db
pod/guestbook-ui-6595f948db-jrjh7
```

The ReplicaSet and Pod are absent from the tracked set.  
Nobody wrote a ReplicaSet manifest — the Deployment controller created it, and the ReplicaSet controller created the Pod.  
In the UI's resource tree they appear as children with a **health** indicator only; the Deployment and Service carry both a sync check mark and a health heart.

This is the reconciliation handoff made visible: Argo CD owns the two objects it applied, Kubernetes owns everything downstream of them.

## Refresh: recomputing the comparison

Sync status is not worked out at the moment you look at it — it is the stored result of the last comparison. **Refresh** is the operation that re-runs that comparison: officially, *"compare the latest code in Git with the live state. Figure out what is different."*

**A refresh changes nothing in the cluster:** it only updates what Argo CD believes about it. Refresh looks; sync acts. An app can be refreshed a hundred times and stay `OutOfSync` forever.

### What triggers one

| Trigger | Detail |
|---|---|
| Polling | on a `timeout.reconciliation` cycle — this install sets `120s` with `60s` of jitter, so every two to three minutes |
| A tracked resource changes | *"an Argo CD Application is refreshed every time a resource that belongs to it changes"* |
| Git webhook | a push notifies Argo CD immediately instead of waiting for the next poll |
| Manual | the UI's **Refresh** button, or `--refresh` on the CLI |

### Reconciliation and jitter

**Reconciliation** is the control-loop idea underneath all of this: keep comparing what is desired against what exists, and close the gap. Kubernetes controllers work exactly this way — the Deployment controller reconciles a Deployment towards its replica count, forever. Argo CD's application controller does the same thing one level up, reconciling the cluster towards Git.

`timeout.reconciliation` is the timer for that loop when nothing else has woken it:

> *"Application reconciliation timeout is the amount of time spent before Argo tries to discover if a new manifests version got published to the repository. Reconciliation by timeout is disabled if timeout is set to 0. Two minutes by default with additional jitter."*

Setting it to `0` disables timed reconciliation altogether — refreshes would then happen only when a tracked resource changes or a webhook arrives.

**Jitter** is a random offset added to a repeating interval so that many clients do not all fire at the same moment. Without it, all the Applications created in the same rollout would refresh together, cycle after cycle, hitting the repo-server in bursts rather than a steady trickle:

> *"With a large number of applications, the periodic refresh for each application can cause a spike in the refresh queue and can cause a spike in the repo-server component. To avoid this, you can set a jitter to the sync timeout, which will spread out the refreshes and give time to the repo-server to catch up. The jitter is the maximum duration that can be added to the sync timeout."*

It is a **maximum**, not a fixed addition — each cycle draws a fresh random value between zero and the jitter, so the applications drift apart instead of staying synchronised.

| Key | Default | Effect |
|---|---|---|
| `timeout.reconciliation` | `120s` | how long before Argo CD checks the repository again; `0` disables timed refresh |
| `timeout.reconciliation.jitter` | `60s` | upper bound on the random delay added to each cycle; `0` disables jitter |

Together they put each application's next refresh somewhere between two and three minutes after the last.

The same key does double duty: for the repo-server, `timeout.reconciliation` sets the expiration on **cached git revisions** — which is the cache a hard refresh discards.

### Refresh vs. hard refresh

| | `--refresh` | `--hard-refresh` |
|---|---|---|
| Documented as | *"Refresh application data when retrieving"* | *"Refresh application data as well as target manifests cache"* |
| Re-reads the live cluster | yes | yes |
| Re-renders manifests from the repository | reuses the cached render when it is considered valid | **always** — the cache is discarded first |
| Cost | cheap | a fresh clone and a full Helm/Kustomize render |

The difference is the **repo-server's manifest cache**. Rendering means cloning a repository and running Helm or Kustomize over it, which is expensive enough that results are cached rather than recomputed on every poll. A plain refresh will reuse that cache; a hard refresh throws it away first.

So a plain refresh answers *"has the cluster or the revision changed?"*, while a hard refresh also answers *"does this revision still render to what we think it does?"*

### When to use which

**Use a plain refresh** for essentially everything — you pushed a commit and do not want to wait for the poll, or you changed something in the cluster and want the status updated now. This is the UI's **Refresh** button.

**Reach for a hard refresh** only when you have reason to believe the cached render no longer matches what the repository would produce:

| Situation | Why a plain refresh is not enough |
|---|---|
| `Manifest generation error (cached)` | *"the error message has been cached to avoid runaway retries"* — a plain refresh re-reads the cached error |
| A chart dependency republished under the same version | the revision did not change, so nothing signals that the render should |
| A Git tag moved to a different commit | same name, different content |
| Values or a chart pulled from outside the tracked repository | changes there are invisible to the revision Argo CD compares against |

The FAQ is blunt about the limits: *"Doing a hard refresh (ignoring the cached error) can overcome transient issues. But if there's an ongoing reason manifest generation is failing, a hard refresh will not help."* If manifests are genuinely broken, hard refresh just reproduces the error more expensively.

Neither is a substitute for sync. If the app is `OutOfSync` because the cluster really does differ from Git, refreshing confirms that repeatedly and changes nothing.

```bash
argocd app get guestbook --refresh          # recompare now
argocd app get guestbook --hard-refresh     # discard the manifest cache and recompare
```

## While a sync is running

The sync status does not change to reflect progress. A separate field tracks the operation:

| `status.operationState.phase` | Meaning |
|---|---|
| `Running` | the sync is in progress |
| `Succeeded` | it completed |
| `Failed` | a resource failed to apply or a hook failed |
| `Error` | the operation errored before completing |
| `Terminating` | it is being cancelled |

Within a sync, resources are applied in ordered phases:

| Phase | When it runs |
|---|---|
| `PreSync` | before the manifests are applied |
| `Sync` | after all `PreSync` hooks succeed, alongside applying the manifests |
| `Skip` | marks a manifest not to be applied at all |
| `PostSync` | after all `Sync` hooks succeed, the apply succeeds, and all resources are `Healthy` |
| `SyncFail` | when the sync operation fails |
| `PreDelete` | before the application's resources are deleted |
| `PostDelete` | after the application's resources are deleted |

A `PreSync` failure stops the whole sync and marks it failed — later phases never run. Within a phase, Argo CD orders resources by **phase → wave (lower first) → kind → name**, which is why namespaces are created before the resources that go in them.

Each resource's result is recorded as `Synced`, `SyncFailed`, `Pruned` or `PruneSkipped`.

So a sync in flight looks like: sync status still `OutOfSync`, operation phase `Running`, health moving through `Progressing` — three fields telling three parts of one story.

## Where to read each

| Field | Holds |
|---|---|
| `status.sync.status` | the application's sync status |
| `status.sync.revision` | the revision the comparison was made against |
| `status.health.status` | the application's health status |
| `status.resources[]` | per-resource **sync** status for the tracked set — see the note below on health |
| `status.resourceHealthSource` | where per-resource health is kept: `appTree` (the default) or `inline` |
| `status.operationState.phase` | the in-flight or last sync operation |
| `status.conditions[]` | errors and warnings — check here first when something is wrong |

**Per-resource health is not in `status.resources` by default:** each entry there carries a sync status and nothing else.

```text
{"kind":"Service","name":"guestbook-ui","namespace":"default","status":"Synced","version":"v1"}
```

The reason is `status.resourceHealthSource`, documented as *"indicates where the resource health status is stored: inline if not set or appTree"*. With the default `appTree`, per-resource health lives in the application tree instead — which is why the CLI and UI can show a health column that the raw field does not contain.

```bash
kubectl -n argocd get app                              # both statuses, one line per app
kubectl -n argocd get app guestbook -o yaml            # the full status subtree
kubectl -n argocd get app guestbook \
  -o jsonpath='{.status.resourceHealthSource}'         # appTree or inline
argocd app get guestbook --core                        # per-resource table, no server login
```

`argocd app get` shows both statuses per resource:

```text
Sync Status:        Synced to HEAD (0d521c6)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME          STATUS  HEALTH   MESSAGE
       Service     default    guestbook-ui  Synced  Healthy  service/guestbook-ui created
apps   Deployment  default    guestbook-ui  Synced  Healthy  deployment.apps/guestbook-ui created
```

Note the two columns: `STATUS` is sync, `HEALTH` is health, reported independently per resource — the same split this whole note is about.

The `--core` flag there makes the CLI talk to Kubernetes directly instead of `argocd-server`, so this works with no port-forward and no login. How the CLI connects, what core mode does to your config, and how to switch back: **16 - The Argo CD CLI and core mode**.
