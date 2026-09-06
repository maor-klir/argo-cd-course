#argocd #kubernetes #gitops #sync #manifests

# Deploy our first application

Bootstrapping one `Application`, watching it report `OutOfSync`, and syncing it. The whole point of the exercise is what happens *between* those two steps — Argo CD detects the gap and then waits, because manual sync is the default.

## What actually gets deployed

Three layers, and it is worth keeping them straight:

```text
guestbook-app.yaml              ← you apply this once, by hand
  └── points at: lm-academy/argocd-example-apps @ HEAD, path guestbook/
        ├── guestbook-ui-deployment.yaml   393 B
        └── guestbook-ui-svc.yaml          141 B
              └── become: Deployment + Service in the `default` namespace
```

The two manifests in the repo are deliberately minimal:

| File | Produces |
|---|---|
| `guestbook-ui-deployment.yaml` | `Deployment` — 1 replica, image `gcr.io/google-samples/gb-frontend:v5`, `containerPort: 80` |
| `guestbook-ui-svc.yaml` | `Service` — ClusterIP, port 80 → targetPort 80, selector `app: guestbook-ui` |

You never apply those two yourself. That is Argo CD's job from here on.

## The Application manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/lm-academy/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

Field by field:

| Field | Why this value |
|---|---|
| `metadata.namespace: argocd` | the `Application` must live in a namespace Argo CD watches — by default only its own |
| `spec.project: default` | required; `default` is created at install and constrains nothing |
| `source.repoURL` | any repo Argo CD can reach. Public here, so no credentials needed |
| `source.targetRevision: HEAD` | a **rolling** pointer — see below |
| `source.path: guestbook` | deploy only this subdirectory, not the whole repo |
| `destination.server` | `https://kubernetes.default.svc` is the in-cluster address — the same cluster Argo CD runs in |
| `destination.namespace: default` | where the Deployment and Service land, *not* where the `Application` lives |

Those last two rows are the distinction worth re-reading: two different namespaces in one manifest, doing two different jobs.

Validate it before applying — this checks against the real API server without creating anything:

```bash
kubectl apply --dry-run=server -f guestbook-app.yaml
```

### Deep dive: client-side vs. server-side dry-run

`--dry-run` takes a mode, and both names mislead. Neither is offline, and for a valid manifest they frequently print exactly the same thing. What differs is *what gets checked*.

| | `--dry-run=client` | `--dry-run=server` |
|---|---|---|
| Reaches the API server | yes | yes |
| Runs schema validation | no | yes |
| Runs admission webhooks | no | yes |
| Applies server-side defaulting | no | yes |
| Persists to etcd | no | no |

**Client mode is not offline:** point kubectl at a dead endpoint and it fails before it can tell you anything about your file.

```text
error: error validating "app.yaml": failed to download openapi:
Get "https://127.0.0.1:1/openapi/v2": dial tcp 127.0.0.1:1: connect: connection refused
```

It needs the cluster for two things: the OpenAPI schema it validates shape against, and the live object it merges your file into. `--validate=false` skips the first.

**The output is often identical, so it is not the differentiator:** running both modes against the already-deployed `guestbook` app produced byte-identical YAML, 111 lines each. Both fetch the live object and show the merge result.

**Validation is the differentiator:** the same manifest, missing the server-required `spec.project`, is waved through by one mode and rejected by the other.

```text
client:  application.argoproj.io/dryrun-test created (dry run)
server:  The Application "dryrun-test" is invalid: spec.project: Required value
```

Client mode reported **success** for a manifest the API server rejects outright. Only server mode runs the full request pipeline — schema validation, mutating and validating admission webhooks, quota, immutable-field checks — stopping just short of writing to etcd.

**Neither mode catches an unknown kind any differently:** both fail the same way, because mapping kind to resource happens through API discovery either way.

```text
no matches for kind "Frobnicator" in version "example.com/v1"
ensure CRDs are installed first
```

This is exactly why Argo CD ships the `SkipDryRunOnMissingResource=true` sync option: when a sync installs a CRD *and* a resource of that kind in one pass, the dry-run of the second object fails against a cluster that does not yet know the kind.

### Deep dive: dry-run is not the same axis as server-side apply

`--dry-run=server` and `--server-side` are unrelated flags that get conflated because both contain the word "server":

- **`--dry-run=server`** decides *whether* the change is persisted
- **`--server-side`** decides *how* the merge is computed, and who owns which fields

They combine freely. Everything below was checked with `--server-side --dry-run=server`, which changes nothing.

#### The two strategies compared

| | Client-side apply (the default) | Server-side apply |
|---|---|---|
| Where the merge happens | in kubectl | in the API server |
| Prior state tracked in | the `last-applied-configuration` annotation | `metadata.managedFields` |
| Granularity | whole object, as one blob | per field, per manager |
| Sent over the wire | a computed patch | your entire object |
| Another writer owns the field | silently overwritten | rejected, naming field and owner |
| Removing a field from your manifest | deleted, if the annotation recorded you setting it | deleted or reset to default, if no other manager owns it |
| Recorded operation | `Update` | `Apply` |

#### Client-side: a three-way merge inside kubectl

kubectl computes the patch locally from three inputs — your file, the live object, and an annotation recording what you last applied. That annotation is a **complete copy of the manifest, stored inside the object it describes**:

```bash
kubectl -n argocd get app guestbook \
  -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' | wc -c
```

```text
358
```

Small here, but it scales with the manifest, and as an annotation it is size-bounded — large CRDs are the usual casualty.

The subtler problem is that it records only what `kubectl apply` did. A `kubectl edit` or `kubectl patch` changes the object **without** updating the annotation, so the next apply computes its diff against a baseline that no longer reflects reality.

#### Server-side: ownership recorded per field

The API server records which manager set which field, and keeps that alongside the object:

```bash
kubectl -n argocd get app guestbook -o json --show-managed-fields
```

```text
manager=kubectl-client-side-apply     op=Update   owns: metadata, spec
    under spec: destination, project, source
manager=argocd-server                 op=Update   owns: status
manager=argocd-application-controller op=Update   owns: status
```

Three writers on one object: the bootstrap `kubectl apply`, the API server acting on a UI sync, and the controller writing `status`. The two Argo CD managers **co-own** `status`, which is legal — shared ownership is only a problem when they disagree on a value.

Every operation reads `Update`, not `Apply` — nothing here used server-side apply. That column is how you tell the two strategies apart on a live object.

**`-o json` hides `managedFields` unless you ask:** `--show-managed-fields` is required, since kubectl started suppressing the block to keep output readable.

#### Conflicts are the difference you actually feel

Applying as a *different* manager, with a *different* value for a field someone else owns:

```bash
kubectl apply --server-side --field-manager=note-test --dry-run=server -f gb-changed.yaml
```

```text
error: Apply failed with 1 conflict: conflict with "kubectl-client-side-apply"
using argoproj.io/v1alpha1: .spec.source.targetRevision
```

It names the exact field and the manager that owns it. The same change through client-side apply:

```text
application.argoproj.io/guestbook configured (server dry run)
```

**Silently accepted:** no warning, no indication that another writer was overwritten. That is the case for server-side apply in one line.

Three ways out of a conflict:

| Resolution | How |
|---|---|
| Take ownership | re-run with `--force-conflicts` — ownership transfers and the field is removed from other managers' entries |
| Give up the field | remove it from your manifest so you stop claiming it |
| Co-own it | change your value to match the server's — you then share ownership, and any later change by either party conflicts |

Matching values is also why an identical re-apply produces no conflict at all: setting a field to the value it already has is never a conflict, whoever owns it.

#### Argo CD's use of both

Argo CD defaults to client-side apply and can be switched per application:

```yaml
syncPolicy:
  syncOptions:
    - ServerSideApply=true
```

Worth reaching for when manifests are large enough to strain the annotation, or when another controller legitimately co-owns fields on the same resource — an HPA setting `replicas`, a mutating webhook injecting a sidecar. Under client-side apply Argo CD would fight those writers silently; under server-side apply the contention becomes visible.

## Apply it

```bash
kubectl apply -f guestbook-app.yaml
```

```text
application.argoproj.io/guestbook created
```

This is a **one-time bootstrap**. It is the last time you hand-apply anything for this app: from now on you change Git, and Argo CD moves the cluster. The `Application` manifest itself belongs in Git too, even though it was applied by hand — that is what makes the setup reproducible on a rebuilt cluster.

## It lands `OutOfSync`, and that is correct

```bash
kubectl -n argocd get app
```

```text
NAME        SYNC STATUS   HEALTH STATUS
guestbook   OutOfSync     Missing
```

Nothing has been deployed. The `default` namespace still holds only the `kubernetes` service:

```bash
kubectl -n default get deploy,svc
```

**`OutOfSync` is a comparison result, not an error:** Argo CD has fetched the manifests, compared them against the live cluster, found two resources that should exist and do not, and stopped there. With no `syncPolicy` in the manifest, sync is manual — Argo CD reports drift and waits to be told to act.

## Sync status and health status are different axes

The two columns above answer different questions. **Sync status** compares desired state against live state — does the cluster match Git? **Health status** inspects the live state alone — are the resources actually working?

They move independently, which is why the pairing matters:

| | Meaning |
|---|---|
| `Synced` + `Healthy` | the normal state |
| `Synced` + `Degraded` | the cluster matches Git, and Git is wrong — syncing again will not help |
| `OutOfSync` + `Healthy` | what is running works; it just is not the committed version |

`OutOfSync` + `Missing` is where this app starts: nothing deployed yet, so there is a difference and the resources do not exist.

Full treatment — both value sets, why `Progressing` is a health status and never a sync status, and which resources get a sync status at all — in **15 - Sync and health checks**.

## Inspect before syncing

```bash
kubectl -n argocd describe application guestbook
```

The `status` subtree is where Argo CD writes what it has worked out — none of it is yours to edit:

| Field | Holds |
|---|---|
| `status.sync` | current sync status and the revision compared against |
| `status.health` | current health status |
| `status.resources` | every resource the app manages, each with its own sync and health |
| `status.conditions` | errors and warnings — the first place to look when something is wrong |
| `status.operationState` | the result of the last sync, including failure messages |
| `status.history` | past revisions, for rollback |

The UI's **App Diff** view shows the same comparison visually: the desired manifest on one side, what exists in the cluster on the other. Before the first sync both resources show as entirely absent.

## Sync it

Three routes to the same operation:

```bash
argocd app sync guestbook               # CLI
```

```yaml
spec:                                    # declarative — no manual step ever again
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

...and the UI's **Sync** button, whose dialog exposes one-off overrides:

| Option | Effect |
|---|---|
| **Revision** | sync a different branch, tag or SHA than `targetRevision`, just this once |
| **Prune** | also delete resources that no longer exist in Git |
| **Dry Run** | run the comparison and report, change nothing |
| **Apply Only** | skip sync hooks (`PreSync`/`PostSync`) |
| **Force** | use `kubectl replace` instead of `apply` — recreates rather than patches |
| **Resource selection** | sync only some of the app's resources |

For this lab the defaults are correct — click Synchronize, or run the CLI command.

## Verify

```bash
kubectl -n default get deploy,svc,pod
```

```text
deployment.apps/guestbook-ui   1/1     1            1
service/guestbook-ui           ClusterIP   10.43.x.x   <none>   80/TCP
pod/guestbook-ui-...           1/1     Running
```

Argo CD created the `Deployment`, not the Pod. The Deployment controller created a ReplicaSet, which created the Pod — Argo CD's involvement ended at `apply`.

Reach the UI through a port-forward:

```bash
kubectl -n default port-forward svc/guestbook-ui 8081:80
```

Then open `http://localhost:8081`. The `svc/` prefix matters — `port-forward` accepts `pod/`, `svc/` or `deployment/`, and defaults to a pod if you give a bare name.

## `HEAD` is a rolling pointer

`targetRevision: HEAD` resolves to the latest commit on the repository's default branch, *at the moment Argo CD checks*. Push a commit and the app goes `OutOfSync` on its own — nothing was applied, but the target moved underneath it.

That is convenient in a lab and wrong in production, where you want a deployment to be reproducible:

| `targetRevision` | Resolves to | Reproducible? |
|---|---|---|
| `HEAD` | latest commit on the default branch | no — moves with every push |
| `main` | latest commit on that branch | no — same problem, named explicitly |
| `v1.4.0` (tag) | the tagged commit | mostly — unless someone moves the tag |
| `9f3c1a2…` (commit SHA) | exactly that commit | yes — immutable |
| `1.2.*` (Helm chart) | newest chart matching the range | no — semver range |

Argo CD polls Git roughly every three minutes by default, so "immediately" means "within a few minutes" unless a webhook is configured.

## The same Application, built from the UI

The Argo CD web UI has a **+ NEW APP** button above the applications list. It opens a form — a panel of input boxes — that assembles exactly the same `Application` object written by hand above, and creates it in the cluster on submit.

Nothing new is available there; it is a different way to fill in the same fields. Worth knowing what each control maps to, so the UI and the YAML stop feeling like two separate systems:

| Field in the form | Manifest field | Notes |
|---|---|---|
| Application Name | `metadata.name` | |
| Project | `spec.project` | only `default` exists on a fresh install |
| Sync Policy | `spec.syncPolicy.automated` | Manual or Automatic, plus prune and self-heal toggles |
| Sync Options | `spec.syncPolicy.syncOptions` | including `CreateNamespace=true` |
| Source type | — | **Git**, **Helm** or **OCI**, which changes the fields below |
| Repository URL | `source.repoURL` | |
| Revision | `source.targetRevision` | branch, tag or SHA |
| Path | `source.path` | Git sources only; Helm sources use `chart` instead |
| Cluster URL / Name | `destination.server` / `destination.name` | one or the other, never both |
| Namespace | `destination.namespace` | not created unless `CreateNamespace=true` |
| Directory Recurse | `source.directory.recurse` | **off by default** — only files at the root of `path` are read |

**Directory Recurse is worth understanding before you need it:** with it off, only manifests sitting directly in `path` are picked up, and anything in a subdirectory is silently ignored. The guestbook path is flat, so it makes no difference here. Note also that directory-type sources are for *plain* manifests only — Argo CD fails to render if it finds Helm or Kustomize files while `directory:` is set.

**The form does not persist a YAML file:** it creates the `Application` in the cluster directly, which leaves nothing in Git and defeats the point. If you do build one in the UI, export it afterwards:

```bash
kubectl -n argocd get app guestbook -o yaml > guestbook-app.yaml
```

That output carries `status` and cluster-added metadata, so strip those before committing.

## Cleaning up

```bash
kubectl delete -f guestbook-app.yaml
```

Without the `resources-finalizer.argocd.argoproj.io` finalizer this deletes only the `Application` — the `Deployment` and `Service` keep running in `default`, now unmanaged. Add the finalizer for a cascading delete, or remove them by hand:

```bash
kubectl -n default delete deploy,svc -l app=guestbook-ui
```
