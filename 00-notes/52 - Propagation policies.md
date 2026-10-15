#kubernetes #propagation #policies #garbagecollection #finalizers

# Propagation policies

A propagation policy is a Kubernetes setting. When an object that has dependents is deleted, it decides what happens to those dependents and in which order. The same terms — foreground, background, orphan — appear in the Argo CD UI when deleting an Application; this note covers what they mean in Kubernetes itself.

## Owners and dependents

Kubernetes links objects through **owner references**:

> *"Owner references tell the control plane which objects are dependent on others."*

The chain behind `guestbook`, read from the cluster:

```text
Deployment  guestbook-helm-guestbook                    ownerReferences: none
ReplicaSet  guestbook-helm-guestbook-79b75cdbfb         owner: Deployment  (controller, blockOwnerDeletion)
Pod         guestbook-helm-guestbook-79b75cdbfb-68xzn   owner: ReplicaSet  (controller, blockOwnerDeletion)
```

Nobody wrote those references: *"Kubernetes sets the value of this field automatically for objects that are dependents of other objects like ReplicaSets, DaemonSets, Deployments, Jobs and CronJobs"*. A controller that sets an owner reference also sets `blockOwnerDeletion: true`, which matters for foreground deletion below.

**Ownership is not the same as a label selector:** a Service finds its endpoints through labels, yet each `EndpointSlice` it produces also carries an owner reference — the one here points at `Service/guestbook-helm-guestbook`. Labels decide what the Service routes to; the owner reference decides what is cleaned up with it.

Owner references never cross namespaces: *"A namespaced owner **must** exist in the same namespace as the dependent."*

## The three policies

The policy travels with the delete request, as `propagationPolicy` in the API or `--cascade` in `kubectl`. Two of its three values cascade to the dependents; the third deliberately does not:

| Policy | The owner | The dependents | `kubectl delete` |
|---|---|---|---|
| **Foreground** | stays visible in *deletion in progress* until the dependents are gone | deleted first | `--cascade=foreground` |
| **Background** | deleted immediately | deleted afterwards by the garbage collector | `--cascade=background` |
| **Orphan** | deleted | left in place — *"the dependents left behind are called orphan objects"* | `--cascade=orphan` |

**Background is the Kubernetes default:** *"By default, Kubernetes uses background cascading deletion unless you manually use foreground deletion or choose to orphan the dependent objects."* `kubectl delete` matches it — `--cascade` *"Defaults to background."*

**Jobs are the exception:** *"unmanaged jobs (Jobs that you created directly, and not indirectly through other workload APIs such as CronJob) have a default deletion policy of `orphanDependents` causing Pods created by an unmanaged Job to be left around after that Job is fully deleted."* A delete request that names no policy therefore orphans a Job's Pods. `kubectl` hides this — it always sends its own `--cascade` value, background unless told otherwise — so the gap shows up with other API clients. Setting `ttlSecondsAfterFinished` avoids it, because the TTL controller deletes Jobs cascadingly.

### Foreground: the owner waits

The owner is not removed at once. Instead:

- *"The Kubernetes API server sets the object's `metadata.deletionTimestamp` field to the time the object was marked for deletion."*
- *"The Kubernetes API server also sets the `metadata.finalizers` field to `foregroundDeletion`."*
- *"The object remains visible through the Kubernetes API until the deletion process is complete."*

The garbage collector then deletes the dependents, and only after them the owner. Not every dependent holds the owner back: *"the only dependents that block owner deletion are those that have the `ownerReference.blockOwnerDeletion=true` field"* — which, as above, controllers set by default.

The waiting is inherited down the chain. When an object's owner is being deleted in the foreground and the object has dependents of its own, the garbage collector deletes it in the foreground as well — its source logs exactly that: *"at least one owner of item has FinalizerDeletingDependents, and the item itself has dependents, so it is going to be deleted in Foreground"*.

**A foreground delete can stall:** *"finalizers can block the deletion of dependent objects, which can cause the targeted owner object to remain for longer than expected without being fully deleted."* An owner stuck in *deletion in progress* is waiting on a dependent; the dependents' finalizers and owner references show which one. Why, and what not to do about it, is under **Finalizers** below.

For a Deployment, the order is therefore bottom-up:

1. the Deployment enters *deletion in progress*
2. its ReplicaSet enters *deletion in progress* too, because it has Pods
3. the Pods are deleted
4. the ReplicaSet disappears, then the Deployment

### Background: the owner goes first

The order reverses. The Deployment is removed immediately; the garbage collector then finds a ReplicaSet whose owner no longer exists and deletes it, and the same happens to its Pods. Nothing waits — `kubectl delete` returns as soon as the owner is gone, while the cleanup is still running.

### Orphan: the owner goes alone

Only the named object is deleted. Deleting a Deployment with `--cascade=orphan` leaves its ReplicaSet and Pods running — and the ReplicaSet, still a working controller, keeps its Pods alive. Deleting that ReplicaSet afterwards with either cascading policy removes the Pods; deleting it as an orphan once more leaves them behind on their own.

During an orphan delete, the owner carries an `orphan` finalizer in place of `foregroundDeletion`, and the garbage collector removes the owner's reference from each dependent before the owner goes. The dependents are left with no controller at all — and that is what makes them adoptable.

**Orphaning is how a controller is replaced without restarting its Pods:** *"Once the original is deleted, you can create a new ReplicaSet to replace it. As long as the old and new `.spec.selector` are the same, then the new one will adopt the old Pods."* A ReplicaSet takes over any Pod without a controller owner that its selector matches — *"they will immediately be acquired by it."* Adoption does not reconcile content, though: the new ReplicaSet *"will not make any effort to make existing Pods match a new, different pod template."*

## Finalizers

Every foreground or orphan delete above runs on the same mechanism: a **finalizer**, a key on an object that holds its deletion until someone has done something first.

> *"You can use finalizers to control garbage collection of objects by alerting controllers to perform specific cleanup tasks before deleting the target resource."*

A finalizer contains no logic: *"Finalizers don't usually specify the code to execute. Instead, they are typically lists of keys on a specific resource similar to annotations."*  
The key is a claim — a component announcing it has work to finish before the object may go. The custom-resource documentation puts it in one line: *"Finalizers allow controllers to implement asynchronous pre-delete hooks."*

### How a deletion with finalizers proceeds

When a `DELETE` arrives for an object whose `metadata.finalizers` is not empty, the API server does not delete it. Instead it:

- *"Modifies the object to add a `metadata.deletionTimestamp` field with the time you started the deletion."*
- *"Prevents the object from being removed until all items are removed from its `metadata.finalizers` field"*
- *"Returns a `202` status code (HTTP "Accepted")"*

The object stays readable, now marked as being deleted — `Terminating` in `kubectl get`. Each controller that owns one of the keys sees the `deletionTimestamp`, does its cleanup, and removes its own key: *"It is the responsibility of each controller to remove its finalizer from the list."* When the list is empty, *"an object with a `deletionTimestamp` field set is automatically deleted."*

### Finalizers met in practice

| Finalizer | Field | Holds the deletion until |
|---|---|---|
| `foregroundDeletion` | `metadata.finalizers` | the garbage collector has deleted the dependents — a foreground delete |
| `orphan` | `metadata.finalizers` | the garbage collector has removed the owner reference from every dependent — an orphan delete |
| `kubernetes.io/pvc-protection` | `metadata.finalizers` | no Pod is using the PersistentVolumeClaim any more |
| `kubernetes.io/pv-protection` | `metadata.finalizers` | the PersistentVolume is no longer bound to a claim |
| `kubernetes` | the Namespace's **`spec.finalizers`** | the namespace's contents have been removed |

The storage pair exists for a reason stated outright: to ensure that *"PersistentVolumeClaims (PVCs) in active use by a Pod and PersistentVolume (PVs) that are bound to PVCs are not removed from the system, as this may result in data loss."*

The last row uses a different field. A Namespace carries its own list under `spec`, which the API describes as *"an opaque list of values that must be empty to permanently remove object from storage"*. All seven namespaces on this cluster carry `spec.finalizers: [kubernetes]`, and no `metadata.finalizers` at all.

### Rules the API server enforces

| Rule | In the documentation's words |
|---|---|
| once deletion starts, keys can only be removed | *"You can remove existing finalizers (deleting an entry from the `finalizers` list) but you cannot add a new finalizer."* |
| the deletion cannot be undone | *"After the deletion is requested, you can not resurrect this object. The only way is to delete it and make a new similar object."* |
| no ordering between keys | *"Finalizers may be processed and removed in any order. Order is NOT enforced because it introduces significant risk of stuck finalizers."* |
| anyone may add one | *"Any controller can add a finalizer to any object's list of finalizers."* |
| custom names must be qualified | *"Custom finalizer names **must** be publicly qualified finalizer names, such as `example.com/finalizer-name`."* |

The naming rule is enforced on write, verified with a server dry run:

```text
metadata.finalizers[0]: Invalid value: "my-finalizer": name is neither a standard finalizer name nor is it fully qualified
```

`example.com/my-finalizer` is accepted. The standard names — `foregroundDeletion`, `orphan`, `kubernetes` — are the only ones allowed without a domain.

Finalizers also work without any deletion pending: *"You can also use finalizers to prevent deletion of unmanaged resources."* A key that no controller will ever remove keeps the object from being deleted, deliberately.

### Gotchas

**Nobody left to remove the key:** a finalizer only clears when its controller removes it. If that controller has been uninstalled, is crashing, or never existed, the object waits in `Terminating` indefinitely. The namespace documentation names the simplest case: *"if you specify a nonexistent finalizer, the namespace will be created but will get stuck in the `Terminating` state if the user tries to delete it."*

**One stuck object holds its whole namespace:** a namespace's own finalizer clears only once everything inside it is gone, so a single object waiting on a finalizer inside keeps the namespace in `Terminating` — *"the namespace is being deleted, and cannot be used for new objects"* — for as long as it waits.

**`kubectl delete` appears to hang:** its `--wait` flag defaults to true — *"wait for resources to be gone before returning. This waits for finalizers."* `--wait=false` returns at once; the deletion stays pending either way.

**`--force` does not get past a finalizer:** *"While any finalizers remain it is also impossible to force the deletion of an object."* `--force` skips graceful termination, not finalizers.

**The name stays taken:** a `Terminating` object still exists under its name, so a new object with the same name cannot be created until the old one is fully gone.

**Removing a finalizer by hand skips the work it guarded:** the documentation is explicit — *"In cases where objects are stuck in a deleting state, avoid manually removing finalizers to allow deletion to continue. Finalizers are usually added to resources for a reason, so forcefully removing them can lead to issues in your cluster. This should only be done when the purpose of the finalizer is understood and is accomplished in another way (for example, manually cleaning up some dependent object)."* Deleting the `kubernetes.io/pvc-protection` key from a claim still mounted by a Pod, for example, removes exactly the protection against data loss described above.

**Diagnosing a stuck deletion:** the documentation's advice is to *"check finalizers and owner references on the target owner and dependent objects to troubleshoot the cause."*

```bash
kubectl get <kind> <name> -o jsonpath='{.metadata.deletionTimestamp}{"  "}{.metadata.finalizers}{"\n"}'
kubectl get ns <name> -o jsonpath='{.status.phase}{"  "}{.spec.finalizers}{"\n"}'
```

The first shows what an object is waiting for; the key's prefix usually names the component responsible for removing it.

## Verify

```bash
kubectl get rs,pod -o custom-columns='KIND:.kind,NAME:.metadata.name,OWNER:.metadata.ownerReferences[0].kind'
kubectl get deploy <name> -o jsonpath='{.metadata.deletionTimestamp} {.metadata.finalizers}'   # during a foreground delete
kubectl delete deployment <name> --cascade=orphan                                              # the owner goes, its dependents stay
```

The first command shows the chain a cascade follows; the second catches a foreground delete in its *deletion in progress* state.
