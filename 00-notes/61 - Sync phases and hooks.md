#argocd #sync #hooks #webhooks #jobs

# Sync phases and hooks

So far a sync has looked like a single step: apply the manifests. Argo CD actually runs it as an ordered sequence of **phases**, and lets us place our own resources — **hooks** — into those phases to run code before, during or after the manifests are applied.

> *"Sync phases and hooks define when resources are applied such as before or after the main sync operation."*

A hook is an ordinary Kubernetes manifest in the same Git source as the application, marked with one annotation:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
```

## What a hook is

**A hook, in its plainest form:** a fixed point in a program's own sequence of steps where the program runs code that someone else supplied, then carries on.  
The program decides *when* the code runs; we decide *what* it does. This reverses the usual direction of a call.  
Normally we call the program: we run `git commit`. With a hook, the program calls our code: we never run the `pre-commit` script ourselves — `git commit` runs it as one of its own steps, every time.  
Git's documentation describes its hooks in exactly these terms: *"Hooks are programs you can place in a hooks directory to trigger actions at certain points in git's execution."*

**Two roles, whatever the form:** in this section, *program* and *code* name roles, not kinds of software.

- **The program:** the tool that owns the sequence of steps and defines the hook points — Git, the kubelet, Argo CD, the Kubernetes API server, GitHub.
- **Our code:** whatever is handed to the program to run at a hook point, whether we wrote it or installed it — a script for Git, a command in the container for `PreStop`, a Kubernetes resource such as a Job for Argo CD, an HTTP server for a webhook.

Our code is often a program in its own right, which is why Git's documentation calls hooks *"programs"*. What makes it a hook is its role: it runs because the program reached that point, not because we started it.

Three things make something a hook:

- the point is defined and named in advance by the program — `pre-commit`, `PreStop`, `PreSync`
- the code is ours, registered with the program **rather** than written into it
- the program runs it whenever that point is reached, without any request from us

Many hooks can also change the outcome of the step they are attached to.  
For Git's `pre-commit` hook: *"Exiting with a non-zero status from this script causes the `git commit` command to abort before creating a commit."* A failing `PreSync` hook stops a sync in the same way.

**A webhook is a hook delivered over HTTP:** instead of running the code itself, the program sends an HTTP request describing the event to a URL registered with it, and whichever server answers at that URL does the work.  
The server runs independently of the program, which knows only its URL. It can be code we wrote or software we installed.  
*"Webhooks let you subscribe to events happening in a software system and automatically receive a delivery of data to your server whenever those events occur."*  
The reason for the design is to avoid asking repeatedly: *"Webhooks are used to receive data as it happens, as opposed to polling an API (calling an API intermittently) to see if data is available."*

**Admission webhooks show the separation:** the Kubernetes API server is the program; the webhook server is a separate process, usually a Pod in the cluster behind a Service, registered with a `ValidatingWebhookConfiguration` or `MutatingWebhookConfiguration`.  
The API server never runs the webhook's code — all it holds is where to send the request. For each matching request it sends an `AdmissionReview` (*"Webhooks are sent as POST requests"*) and waits for the answer, which must contain *"`allowed`, either set to `true` or `false`"*.  
Because the server is separate, the API server can fail to reach it, and the webhook's `failurePolicy` decides what happens then.  
The API describes the field as *"Defaults to Fail."*, and *"`Fail` means that an error calling the webhook causes the admission to fail and the API request to be rejected."* A webhook server that is down therefore blocks every request it is registered for, while the API server itself is working normally.

| | Our code runs | Called by, and when | Can it stop the step? |
|---|---|---|---|
| Git `pre-commit` hook | as a script on our machine | git, before it creates a commit | yes — a non-zero exit aborts the commit |
| Kubernetes `PreStop` container hook | inside the container | the kubelet, before it stops the container | no — *"regardless of the outcome of the handler, the container will eventually terminate within the Pod's termination grace period"* |
| Argo CD `PreSync` hook | as a resource Argo CD creates in the cluster | Argo CD, at the start of a sync | yes — a failure stops the sync |
| Kubernetes validating admission webhook | on a separate HTTP server, usually a Pod in the cluster | the API server, for each matching request | yes — they *"can reject requests to enforce custom policies"* |
| GitHub webhook | on the server at the registered URL | GitHub, after an event such as a push | no — the event has already happened |

### Where Argo CD meets webhooks

**The hooks in this note are not webhooks:** Argo CD runs them itself, as Kubernetes resources, at fixed points in a sync.  
Webhooks do appear around Argo CD, in both directions:

| | Program — sends the request | Server — receives it |
|---|---|---|
| Git webhook | the Git provider, after a push | Argo CD's API server, at `/api/webhook` |
| Notifications webhook | Argo CD's notifications controller, after an event such as a successful sync | any URL we configure — the documentation's examples are a Jenkins job and a GitHub commit status |
| The documentation's `PostSync` Slack example | the hook's own container, running `curl` | Slack |

**A Git webhook only triggers a refresh:** without one, Argo CD checks each repository on a timer, every two to three minutes on this install (**15 - Sync and health checks**); with one, the Git provider calls `/api/webhook` after a push and Argo CD refreshes at once.  
A refresh compares Git with the cluster but does not sync — *"the contents of webhook payloads are considered untrusted, and will only result in a refresh of the application"* — so a webhook never runs the hooks in this note; only a sync does.  
On this cluster `argocd-server` is a `ClusterIP` Service with no Ingress, so GitHub cannot reach it and this install relies on polling.

## The hook types

| Hook | Runs |
|---|---|
| `PreSync` | *"Executes prior to the application of the manifests."* |
| `Sync` | *"Executes after all `PreSync` hooks completed and were successful, at the same time as the application of the manifests."* |
| `PostSync` | *"Executes after all `Sync` hooks completed and were successful, a successful application, and all resources in a `Healthy` state."* |
| `SyncFail` | *"Executes when the sync operation fails."* |
| `PreDelete` | *"Executes before Application resources are deleted."* Never during a sync. |
| `PostDelete` | *"Executes after all Application resources are deleted."* Never during a sync. Available since v2.10. |
| `Skip` | *"Indicates to Argo CD to skip the application of the manifest."* Not a phase. |

The first four belong to a sync and the two `Delete` hooks to deleting the Application. `Skip` runs nothing; it marks a manifest that Argo CD should not apply. Our ordinary manifests always go into the `Sync` phase: *"Pre-sync and post-sync can only contain hooks."*

*"Hooks can be any type of Kubernetes resource kind, but tend to be Pod, Job or Argo Workflows. Multiple hooks can be specified as a comma separated list."* — `argocd.argoproj.io/hook: PreSync,PostSync` runs the same resource in both phases.

## How a sync moves through the phases

```text
sync triggered
  │
  ├─ PreSync    run PreSync hooks ─────────────────────── any fails? ──┐
  ├─ Sync       apply manifests, run Sync hooks ── fails / Degraded? ──┤
  ├─ PostSync   run PostSync hooks ────────────────────── any fails? ──┤
  │                                                                    ▼
  └─ Succeeded                            SyncFail: run SyncFail hooks,
                                          operation ends as Failed
```

A failing `PreSync` hook stops the sync before any manifest is applied: *"If it fails then the whole sync operation will stop preventing the deployment from taking place."* `SyncFail` is reached from any phase; in the sync engine's source, a failed task in `PreSync`, `Sync` or `PostSync` leads to the same `SyncFail` step. It is the place for cleanup — for example, reverting what a failed `PreSync` migration left behind. If the cleanup itself fails, *"Argo CD does not take any additional action beyond marking the overall operation as failed."*

**What "succeeded" means for a hook:** Argo CD judges a hook by its health, not by its exit code (sync engine source, `getOperationPhase`):

| Hook's health | Treated as |
|---|---|
| `Healthy` | succeeded |
| `Progressing`, `Suspended` | still running — the phase waits |
| `Degraded`, `Unknown` | failed |
| kind without a health check (e.g. ConfigMap) | succeeded as soon as it is created |

This is why hooks are usually Jobs: a Job has a defined end — it either completes or fails — and Argo CD's built-in Job health check maps those to `Healthy` and `Degraded`. A Deployment as a hook would count as done once it is `Healthy`, which says nothing about whether any task finished.

**A failing Job hook takes time to fail:** Argo CD sees `Degraded` only once the Job itself has failed, after `backoffLimit` retries — *"the number of retries before considering a Job as failed"*, 6 by default.  
Failed Pods are recreated *"with an exponential back-off delay (10s, 20s, 40s ...) capped at six minutes"*, so with the defaults a failing hook holds the phase for roughly ten minutes before the sync fails.  
A small `backoffLimit` shortens that: `backoffLimit: 2` means three attempts in total, the first run and two retries.

**A later step makes the sync wait for health:** Argo CD waits for resources to become `Healthy` only when another phase or wave follows them (sync engine source). A sync whose last step is the `Sync` phase succeeds as soon as the manifests are applied, even if the objects degrade afterwards — a `PreSync` hook alone does not change that. A `PostSync` hook adds a step after the `Sync` phase, so the operation stays `Running` until every resource in it is `Healthy`. The operation message names what it is waiting for, as in the documentation's example:

```text
waiting for completion of hook batch/Job/ingress-nginx-admission-create
```

In that case a Deployment that never becomes `Healthy` keeps the operation `Running` until its progress deadline marks it `Degraded`, which fails the sync. `argocd app terminate-op <app>` ends a stuck operation at once.

## How hooks differ from ordinary resources

**Only hooks in the Application's own manifests run:** the sync engine separates hooks out of the manifests Argo CD renders from the Application's source, and runs only those (sync engine source, `Reconcile` → `splitHooks`). An object that already exists in the cluster is never run as a hook, whatever its annotations: a Job created with `kubectl` and annotated `argocd.argoproj.io/hook: PreSync` is ignored by every sync, even in the Application's namespace. Kubernetes in turn ignores the annotations. It runs that Job as soon as it is created, as it does every Job, and nothing ever applies its `HookSucceeded` policy.

**Hooks do not affect the sync status:** the controller skips them when it computes the Application's sync status — in its own words, hooks and skipped resources *"don't store sync status, and do not affect overall sync status"* (controller source, `state.go`). Committing a new hook to Git therefore leaves the Application `Synced`. Automated sync only starts when the Application is `OutOfSync` — *"Only perform auto-sync if we detect OutOfSync status"* (controller source) — so a hook-only change runs at the next sync, whether we trigger it ourselves or another change makes the Application `OutOfSync`. Automated sync, pruning and self-healing are covered in **31 - Automated sync, pruning and self-healing**.

**Once the sync is over, Argo CD stops looking at a hook:** it creates the hook for the operation and judges it while it runs, but afterwards the hook sits outside everything Argo CD does with ordinary resources (controller source, `setApplicationHealth`; sync engine source, `getSyncTasks`):

| | Ordinary resource | Hook, once the sync is over |
|---|---|---|
| Drift | compared continuously; a change makes the app `OutOfSync` | ignored by the comparison |
| Self-healing | a live edit is reverted | never triggered — self-healing acts only on `OutOfSync` |
| Application health | counts toward it | skipped — a failed hook Job left in the cluster does not make the app `Degraded` |
| Removal | pruned once dropped from Git | never pruned — only its delete policy removes it |

**A hook removed from Git can be left behind:** if its delete policy has not already removed it, nothing else will. Argo CD never creates a prune task for a hook, and with the manifest gone there is no next run to trigger `BeforeHookCreation`. The object stays in the cluster until we delete it by hand.

**Selective syncs skip hooks:** *"Hooks do not run during a selective sync operation."* Syncing chosen resources in the UI, or `argocd app sync --resource`, applies those resources and runs no hooks.

**Each hook gets a finalizer while it runs:** Argo CD adds `argocd.argoproj.io/hook-finalizer` to every hook it creates and removes it once the hook has completed (sync engine source). The source describes its purpose: *"to ensure they are deleted only after the sync phase is completed."* A Job's `ttlSecondsAfterFinished`, or someone deleting the Job by hand, cannot remove it before Argo CD has read its result — the object waits in `Terminating`, as described under **Finalizers** in **52 - Propagation policies**.

## Hook cleanup

Hooks are created anew for each sync, and the `argocd.argoproj.io/hook-delete-policy` annotation decides when Argo CD deletes them:

| Policy | Deletes the hook |
|---|---|
| `HookSucceeded` | *"The hook resource is deleted when the sync succeeds. For completion-based hooks such as `Job` or `Workflow`, this typically happens after the hook completes successfully."* |
| `HookFailed` | *"The hook resource is deleted after the hook failed."* |
| `BeforeHookCreation` | *"Any existing hook resource is deleted before the new one is created (since v1.3). It is meant to be used with `/metadata/name`."* |

*"Note that if no deletion policy is specified, Argo CD will automatically assume `BeforeHookCreation` rules."*

**`HookSucceeded` waits for the whole sync:** the sync engine deletes `HookSucceeded` hooks only when the operation as a whole succeeds. A `PreSync` Job that completed is still there if the `Sync` phase fails afterwards.

**Why the lab combines two policies:** hooks are applied, not created — *"In Argo CD, hooks are created by using `kubectl apply`, rather than `kubectl create`. This means that if the hook is named and already exists, it will not change unless you have annotated it with `before-hook-creation`."* A Job left over from an earlier run, completed and `Healthy`, would be re-applied unchanged and counted as succeeded without running again. `BeforeHookCreation,HookSucceeded` covers both cases: the Job is removed after a successful sync, and any leftover is deleted before the next run creates a fresh one.

**`name` or `generateName`:** with a fixed `metadata.name`, each run replaces the previous object, which is what `BeforeHookCreation` expects. With `metadata.generateName`, the sync engine builds a unique name per run — the prefix, the first seven characters of the revision, the phase and a timestamp — so earlier runs are never replaced and need `HookSucceeded` or `HookFailed` to stop them accumulating.

**Deleting a hook deletes its logs:** *"Deleting a Job will clean up the Pods it created."* — and the Pods' logs with them. `HookSucceeded` therefore leaves nothing to inspect after a successful run.  
With `BeforeHookCreation` alone, the last run's Job and Pods stay until the next sync replaces them: `kubectl logs job/<name>` still shows the output, and `kubectl delete job <name>` removes them by hand.

**Argo CD deletes hooks with the prune propagation policy:** the sync engine uses the same delete options for hooks as for pruning — `PrunePropagationPolicy`, foreground unless set otherwise (sync engine source, `getDeleteOptions`).  
That is why a hook Job's Pods are deleted with it, although a Job deleted through the API without a policy leaves its Pods behind (**Jobs are the exception** in **52 - Propagation policies**). With `PrunePropagationPolicy=orphan` (**53 - Argo CD propagation policies**), hook Pods would be left behind as well.

## Deletion hooks

`PreDelete` and `PostDelete` hooks run only when the Application itself is deleted: *"not during normal sync operations, even when pruning is enabled."* The controller keeps them out of every sync (controller source).

| | `PreDelete` | `PostDelete` |
|---|---|---|
| Runs | before any of the Application's resources are deleted | after all of them are gone |
| Waits for | all `PreDelete` hooks to become `Healthy` | all `PostDelete` hooks to become `Healthy` |
| On failure | *"the Application deletion is blocked"* | *"The Application is already deleted from the cluster, but the Application CR remains with a `DeletionError` condition"* |

Both run on our v3.4.6 cluster. A failed hook leaves the Application with a `DeletionError` condition; the fix is either a corrected hook in Git, which Argo CD retries automatically, or deleting the failing hook resource by hand.

**The Application gets finalizers for them:** when the manifests contain deletion hooks, the controller adds `pre-delete-finalizer.argocd.argoproj.io` and/or `post-delete-finalizer.argocd.argoproj.io` (each with a `/cleanup` variant) to the Application, and removes them once the hooks have completed (controller source). These are separate from the `resources-finalizer` described in **53 - Argo CD propagation policies**; they hold the Application while its deletion hooks run.

## Skip

A resource annotated `argocd.argoproj.io/hook: Skip` is not applied, and does not count toward the sync status. The sync engine treats it as skipped only when `Skip` is the sole value — `Skip,PreSync` is a `PreSync` hook. The documentation's example is a Helm chart whose admission-webhook Job left syncs stuck *"waiting for completion of hook"*; setting `Skip` on that Job through the chart's values let the sync complete.

## Helm hooks

Helm charts declare hooks with their own `helm.sh/hook` annotation, and Argo CD maps them onto its phases:

| Helm | Argo CD |
|---|---|
| `pre-install`, `pre-upgrade` | `PreSync` |
| `post-install`, `post-upgrade` | `PostSync` |
| `pre-delete` / `post-delete` | `PreDelete` / `PostDelete` |
| `hook-weight` | `sync-wave` |
| `hook-delete-policy` | supported, with Argo CD's timing |
| `pre-rollback`, `post-rollback`, `test-success`, `test-failure` | not supported, ignored |

Argo CD renders charts with `helm template` and never installs a release (**22 - Deploying Helm charts**), so it cannot tell an install from an upgrade. Every operation is a sync, and *"apps that have `pre-install` and `pre-upgrade` will have those hooks run at the same time."* A `pre-install` hook runs on every sync, not just the first. On a resource that carries both annotations, the Argo CD hook wins and the Helm one is ignored.

## Hooks compared with init containers

A `PreSync` Job and a Pod's init container both run something before the application starts, at different scopes:

| | `PreSync` hook | Init container |
|---|---|---|
| Runs | once per sync | in every Pod: *"If the Pod restarts, or is restarted, all init containers must execute again."* |
| Blocks | the whole sync — no manifest is applied | only its own Pod's app containers |
| On failure | the sync fails and `SyncFail` runs | *"the kubelet repeatedly restarts that init container until it succeeds"* (subject to the Pod's `restartPolicy`) |
| Known to | Argo CD only | Kubernetes — works with any deployment tool |

A database migration belongs in a hook: it should run once per release, not once per replica and again on every restart. Either way the code must tolerate running twice — the Kubernetes documentation's rule for init containers, *"init container code should be idempotent"*, applies equally to a hook that a sync retries.

## Sync waves

Within a phase, the `argocd.argoproj.io/sync-wave` annotation orders resources into steps, each waiting for the previous one to become healthy — covered in **62 - Sync waves**.

## Verify

```bash
kubectl -n <namespace> get jobs -w                                               # the hook Job appears, completes, and is deleted
kubectl -n <namespace> get job <name> -o jsonpath='{.metadata.finalizers}'       # argocd.argoproj.io/hook-finalizer while it runs
kubectl -n argocd get app <app> -o jsonpath='{range .status.operationState.syncResult.resources[*]}{.syncPhase}{"\t"}{.hookType}{"\t"}{.kind}/{.name}{"\t"}{.hookPhase}{"\t"}{.message}{"\n"}{end}'
argocd app terminate-op <app>                                                    # end an operation stuck waiting on a hook
```

The third command lists the last sync's results with the phase each resource ran in. From the `guestbook` sync with the `PreSync` Job:

```text
PreSync   PreSync   Job/db-migration-job                  Succeeded   Reached expected number of succeeded pods
Sync                Service/guestbook-helm-guestbook      Running     service/guestbook-helm-guestbook unchanged
Sync                Deployment/guestbook-helm-guestbook   Running     deployment.apps/guestbook-helm-guestbook configured
```

Hooks show their type in the second column; ordinary resources leave it empty. In the fourth column, ordinary resources show `Succeeded` once Argo CD has waited for their health, and `Running` if they were in the last step, which Argo CD does not wait for — as for the Service and Deployment here. The results remain after a `HookSucceeded` hook has been deleted.
