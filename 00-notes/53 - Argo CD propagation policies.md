#argocd #deletion #finalizers #pruning #kubernetes

# Argo CD propagation policies

Argo CD reuses the Kubernetes propagation policies from **52 - Propagation policies** whenever it deletes something on an Application's behalf — when the Application itself is deleted, and when a sync prunes. Two differences from native Kubernetes run through everything below: the cascade does not come from owner references, and the default is **foreground**, not background.

## The Application owns nothing, in Kubernetes' sense

The objects Argo CD manages carry no owner reference to their Application — only the tracking annotation from **13 - Argo CD Applications vs. Kubernetes primitive manifests**:

```text
Deployment  guestbook-helm-guestbook   ownerReferences: none   tracking-id: guestbook:apps/Deployment:finance/guestbook-helm-guestbook
Service     guestbook-helm-guestbook   ownerReferences: none   tracking-id: guestbook:/Service:finance/guestbook-helm-guestbook
```

So the Kubernetes garbage collector never cascades from an Application to its resources. Every cascading delete Argo CD performs has two layers:

1. Argo CD deletes the Application's resources — the Deployment, the Service
2. Kubernetes deletes *their* dependents — the ReplicaSet and Pods, the EndpointSlice — through owner references, using the policy Argo CD passed along

## The finalizer is the switch

Whether deleting an Application cascades at all is decided by a finalizer on the Application:

```yaml
metadata:
  finalizers:
    # The default behaviour is foreground cascading deletion
    - resources-finalizer.argocd.argoproj.io
    # Alternatively, background cascading deletion
    # - resources-finalizer.argocd.argoproj.io/background
```

*"When deleting an Application with this finalizer, the Argo CD application controller will perform a cascading delete of the Application's resources."* Without it, deleting the Application deletes only the Application.

The controller's handling, in its source, maps directly onto the Kubernetes mechanism:

- the finalizer variant selects a Kubernetes `propagationPolicy` — foreground, unless it is the `/background` finalizer
- each managed resource is deleted with that policy — the controller logs *"Deleting application's resources with %s propagation policy"*
- the finalizer is removed, and the Application with it, only once none of those objects remain in the API

That last point is what makes the two policies differ at the Application level. Under **foreground**, each Deployment stays visible in *deletion in progress* until its ReplicaSets and Pods are gone, so the Application waits for whole trees — *"Waits for all managed resources to be deleted before the Application is removed."* Under **background**, the top-level objects disappear at once, so the Application goes almost immediately while the garbage collector finishes behind it — *"The Application is removed immediately, and resources are deleted in the background."*

## Same Application, different tools, different outcomes

| Tool | Cascades by default? | Choosing otherwise |
|---|---|---|
| `argocd app delete` | **yes** — `--cascade` defaults to true and `--propagation-policy` to foreground. *"When you invoke `argocd app delete` with `--cascade`, the finalizer is added automatically."* | `--cascade=false` for no cascade, `-p background` for background |
| the UI | **yes** — the dialog defaults to **Foreground**, with **Background** and **Non-Cascading (Orphan)** offered | the dialog |
| `kubectl delete app` | **only if the finalizer is already set** | patch the finalizer in or out first |

Neither Application on this cluster carries the finalizer. `kubectl delete app guestbook` would therefore leave its Deployment and Service running, while `argocd app delete guestbook` would remove them — the same request, opposite results, decided by the tool.

**`kubectl delete --cascade` does not control Argo CD's cascade:** that flag sets the Kubernetes policy for the *Application object's* own dependents by owner reference, and it has none. The documented `kubectl` route for a non-cascading delete accordingly changes the finalizer, not the flag:

```bash
kubectl patch app APPNAME -p '{"metadata": {"finalizers": null}}' --type merge
kubectl delete app APPNAME
```

## Compared with native Kubernetes

| | Kubernetes | Argo CD |
|---|---|---|
| What links owner and dependents | `ownerReferences`, set by controllers | the tracking annotation |
| What performs the cascade | the garbage collector | the application controller, triggered by a finalizer |
| Default policy | Background — Jobs excepted | Foreground — for Application deletion and pruning alike |
| Where the policy is chosen | per delete request, `propagationPolicy` or `--cascade` | finalizer variant, `--propagation-policy`, the UI dialog, `PrunePropagationPolicy` |
| What "orphan" leaves behind | dependents with the owner reference **removed** — free to be adopted | resources with the tracking annotation **still in place** |
| Foreground vs background decides | when the owner object disappears | when the Application disappears |

The orphan row has a practical edge. Kubernetes strips the link when it orphans, so a new owner can adopt cleanly. Argo CD's **Non-Cascading** delete — *"The finalizer is removed automatically before deletion"* — leaves the tracking annotation untouched, so a later Application with the same name considers those objects its own again. That is what happened when `guestbook` was re-created, in **51 - Argo CD projects**.

## What a cascading delete skips

Not everything an Application manages is deleted with it:

| Skipped | Why |
|---|---|
| resources annotated `argocd.argoproj.io/sync-options: Delete=false` | *"you might want to retain them even after your application is deleted, e.g. for Persistent Volume Claims"*; also settable for the whole Application in `syncOptions` |
| resources annotated `helm.sh/resource-policy: keep` | *"Supported as equivalent to `argocd.argoproj.io/sync-options: Delete=false`."* |
| resources marked `Delete=confirm` | held until approved in the UI, the CLI, or with an `argocd.argoproj.io/deletion-approved` annotation — meant for objects such as Namespaces |
| CustomResourceDefinitions | excluded by the controller's own deletion filter |
| objects outside the project's permitted destinations and kinds | the controller only deletes live objects the project permits — the same filter as in note 51 |

*"Note that setting a Delete sync option on the resource will always override a Delete sync policy defined in the Application."*

## Pruning during a sync

> *"By default, extraneous resources get pruned using the foreground deletion policy. The propagation policy can be controlled using the `PrunePropagationPolicy` sync option. Supported policies are background, foreground, and orphan."*

```yaml
spec:
  syncPolicy:
    syncOptions:
      - PrunePropagationPolicy=foreground
```

Here `orphan` reaches Kubernetes itself. Application deletion only ever passes foreground or background down — its *Non-Cascading (Orphan)* option skips the resources altogether. Pruning a Deployment with `orphan` removes the object Git no longer declares while its ReplicaSet and Pods keep running — with no owner, and no Application tracking the ReplicaSet either. Pruning itself is in **31 - Automated sync, pruning and self-healing**.

Prune order follows sync waves in reverse: *"Resources in higher waves are pruned first."*

## Verify

```bash
kubectl -n argocd get app -o custom-columns='APP:.metadata.name,FINALIZERS:.metadata.finalizers'   # will deleting it cascade?
kubectl -n argocd get app <name> -o jsonpath='{.spec.syncPolicy.syncOptions}'                      # PrunePropagationPolicy, Delete=…
kubectl -n <ns> get deploy,svc -o custom-columns='NAME:.metadata.name,OWNERS:.metadata.ownerReferences,TRACKING:.metadata.annotations.argocd\.argoproj\.io/tracking-id'
```

The first is the one to run before any `kubectl delete app`: an empty `FINALIZERS` column means the delete will leave everything running.
