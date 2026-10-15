#argocd #sync #syncwaves #hooks

# Sync waves

Sync phases order a sync coarsely — `PreSync`, `Sync`, `PostSync` (**61 - Sync phases and hooks**). Inside a phase, Argo CD applies resources in a fixed order, but it does not wait for one resource to become healthy before applying the next. **Sync waves** split a phase into numbered steps, and Argo CD waits for each step to become healthy before starting the next one.

> *"This way you can use sync phases for coarse grained ordering and sync waves for defining the exact order of a resource within an individual phase."*

## The annotation

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "5"
```

*"The value is an integer that defines the ordering (ArgoCD will start deploying from the lowest number and finish with the highest number)."* *"Hooks and resources are assigned to wave 0 by default. The wave can be negative, so you can create a wave that runs before all other resources."* Only the order of the numbers matters: waves `-5`, `-2`, `2` and `10` are four steps in that order, with nothing in between.

**Leave gaps between wave numbers:** numbering in tens — `10`, `20`, `30` — leaves room to insert a step later without renumbering every manifest.

**The value must be quoted:** annotation values are strings — *"The keys and the values in the map must be strings. In other words, you cannot use numeric, boolean, list or other types for either the keys or the values."* An unquoted `sync-wave: 5` is a YAML number, and the manifest is rejected before it reaches the cluster:

```text
json: cannot unmarshal number into Go struct field ObjectMeta.metadata.annotations of type string
```

**A value that is not an integer becomes wave 0 without any error:** the sync engine parses the value with `strconv.Atoi`; if that fails it falls back to Helm's `helm.sh/hook-weight`, and without one to `0` (sync engine source, `syncwaves.Wave`). `"1.5"` or `"first"` put the resource in wave 0, silently.

## The order inside a wave

When a sync starts, Argo CD sorts everything by:

1. *"The phase"*
2. *"The wave they are in (lower values first)"*
3. *"By kind (e.g. namespaces first and then other Kubernetes resources, followed by custom resources)"*
4. *"By name"*

The kind order is a fixed list in the sync engine source, beginning `Namespace`, `NetworkPolicy`, `ResourceQuota`, `LimitRange`, … `ServiceAccount`, `Secret`, `ConfigMap`, … `PersistentVolumeClaim`, `CustomResourceDefinition`, the RBAC kinds, `Service`, … `Deployment`, `HorizontalPodAutoscaler`, `StatefulSet`, `Job`, `CronJob`, `IngressClass`, `Ingress`, `APIService`. Kinds not on the list — custom resources among them — come last.

**Within a wave, Argo CD waits for API calls, not for health:** resources of the same kind are applied in parallel, and the next kind starts once those calls have returned (sync engine source, `runTasks`). A ConfigMap therefore exists before a Job in the same wave is created, but nothing in a wave waits for anything else in it to become `Healthy` or to finish. A Deployment is even applied *before* a Job in the same wave, since `Deployment` precedes `Job` in the kind order.

**Namespaces and CRDs are moved forward automatically:** if a Namespace in the Application's manifests sits in a later wave than a resource placed in it, the sync engine moves the Namespace into that resource's wave and phase, ahead of it; a CustomResourceDefinition is moved ahead of its custom resources the same way (sync engine source, `adjustDeps`). A `sync-wave: "-1"` on a Namespace is not needed to create it first.

## How a sync moves from wave to wave

```text
wave -1   apply ── 2 s ── wait until all Healthy ──┐
wave 0    apply ── 2 s ── wait until all Healthy ──┤   ← every resource without the annotation
wave 1    apply ── 2 s ── wait until all Healthy ──┤
wave 2    apply ───────────────────────────────────┴─► last step: the sync succeeds once applied
```

**What "Healthy" means here:** each resource's own health assessment, as in **15 - Sync and health checks**. A kind without a health check, such as a ConfigMap or a Secret, counts as done as soon as it is applied. A Job is `Progressing` until it completes, so the next wave waits for the Job to finish. A `Degraded` resource fails the sync. Argo CD waits for health only when another wave or phase follows; after the last step it does not (see **A later step makes the sync wait for health** in **61 - Sync phases and hooks**).

**One unhealthy resource holds every later wave:** *"Because an application can have resources that are unhealthy in the first wave, it may be that the app can never get to healthy."* A Deployment stuck on a missing image in wave 1 keeps waves 2 and up from being applied; once its progress deadline passes it turns `Degraded`, and the sync fails without them.

**The pause between waves:** *"There is currently a delay between each sync wave in order to give other controllers a chance to react to the spec change that was just applied. This also prevents Argo CD from assessing resource health too quickly (against the stale object), causing hooks to fire prematurely."* It comes after each wave is applied and before its health is checked. The delay is 2 seconds, set by `controller.sync.wave.delay.seconds` in `argocd-cmd-params-cm`, which the controller reads as the environment variable `ARGOCD_SYNC_WAVE_DELAY`. The key is unset on this cluster, so the default applies. There is no delay after the final wave (controller source, `delayBetweenSyncWaves`).

## Waves inside phases

Waves never cross a phase boundary: the phase is sorted first and the wave only within it.

```text
PreSync    wave -5 → wave -2 → wave 2 → wave 10
Sync       wave -2 → wave -1 → wave 1 → wave 2
PostSync   wave -5 → wave 3  → wave 4 → wave 5
```

`PostSync` wave `-5` runs after `Sync` wave `2`, despite the lower number. To order hooks within their phase, we give them a wave as well — the documentation's database-migration example is a `PreSync` hook in wave `-1`, so it runs before any other `PreSync` hook in wave `0`.

## A resource in a wave compared with a hook

Waves and hooks both put work before other work, but the resource behaves differently:

| | Ordinary resource with a wave | Hook |
|---|---|---|
| Part of the sync status | yes — diffed, shown as `Synced` or `OutOfSync`, pruned | no (**61 - Sync phases and hooks**) |
| On each sync | applied again; an unchanged object is left as it is | created anew (with the default `BeforeHookCreation` policy) |
| A Job runs | once, when it is first created | on every sync |
| Holds later steps until `Healthy` | yes, when a later wave or phase follows | yes, always — until the hook completes |

**A Job in a wave runs once:** re-applying an unchanged Job that has completed changes nothing, so Kubernetes does not run it again. Changing it in Git does not rerun it either: a Job's pod template cannot be changed after creation, and the sync fails with `spec.template: Invalid value: …: field is immutable` (server dry run on this cluster). Getting past that takes deleting the Job, or `Replace=true` with `Force=true` as in **Replace and Force** in **22 - Deploying Helm charts**. Only the pod template is fixed: changing a Job's annotations, its wave among them, updates the Job in place without running it again. A Job that must run on every sync, such as a database migration, belongs in a hook; a Job in a wave fits a one-time step, or a check that only has to pass before the first rollout.

## Pruning runs in reverse wave order

*"During pruning, the wave order is reversed from the creation order. Resources in higher waves are pruned first."* With the `PruneLast` sync option, pruning instead happens *"as a final, implicit wave of a sync operation, after the other resources have been deployed and become healthy, and after all other waves completed successfully."* Pruning and its propagation policy are covered in **53 - Argo CD propagation policies**.

## Verify

```bash
kubectl -n argocd get app <app> -o jsonpath='{range .status.resources[*]}{.syncWave}{"\t"}{.kind}/{.name}{"\n"}{end}'   # each resource's wave
kubectl -n argocd get app <app> -o jsonpath='{range .status.operationState.syncResult.resources[*]}{.syncPhase}{"\t"}{.hookType}{"\t"}{.kind}/{.name}{"\t"}{.hookPhase}{"\n"}{end}'   # the last sync, in processing order
kubectl -n <namespace> get configmap,job,deployment -w                                                                 # watch the waves arrive during a sync
kubectl -n argocd get cm argocd-cmd-params-cm -o jsonpath='{.data.controller\.sync\.wave\.delay\.seconds}'               # empty means the 2 s default
```

The first command lists resources by kind, not by wave, and leaves the wave column empty for wave `0`, the default. The second lists the last sync's results in the order the sync processed them (sync engine source, the result's `Order` field). From the `guestbook` chart with a `PreSync` hook, an unannotated Service, and a ConfigMap, Job and Deployment in waves `10`, `20` and `30`:

```text
PreSync   PreSync   Job/db-migration-job                  Succeeded
Sync                Service/guestbook-helm-guestbook      Succeeded
Sync                ConfigMap/db-config                   Succeeded
Sync                Job/db-check-job                      Succeeded
Sync                Deployment/guestbook-helm-guestbook   Running
```

The hook runs first, then wave `0` — the Service — and then each wave in turn. The Deployment, in the last step, shows `Running` because Argo CD does not wait for the health of the last step.
