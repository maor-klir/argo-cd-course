#argocd #helm #gitops #sync #tracking

# Deploying Helm charts

Switching a running `Application` from a directory of plain manifests to a Helm chart is a one-field edit. Everything interesting that follows comes from a side effect of that edit: the chart renders its objects under **different names**, and names are what Argo CD tracks by.

## The edit

```yaml
spec:
  source:
    repoURL: https://github.com/maor-klir/argocd-example-apps.git
    targetRevision: HEAD
-   path: guestbook
+   path: helm-guestbook
+   helm:
+     valueFiles:
+       - values.yaml
```

Nothing else changes — same repository, same revision, same destination.

## The chart is the same two objects, templated

| Path | Holds |
|---|---|
| `Chart.yaml` | `name: helm-guestbook`, `version: 0.1.0`, `appVersion: "1.0"` |
| `values.yaml` | `replicaCount: 1`, the image, `service.type: ClusterIP`, `service.port: 80` |
| `values-production.yaml` | one key only — `service.type: LoadBalancer` |
| `templates/deployment.yaml` | the Deployment |
| `templates/service.yaml` | the Service |
| `templates/_helpers.tpl` | the name-building templates every other file calls |

Rendered, it produces a Deployment and a Service — the same two objects the plain `guestbook/` directory produced. Only the syntax differs.

## Why the names change

Object names come from one template:

```gotemplate
{{- define "helm-guestbook.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}
```

The release name is not something the chart chooses: *"By default, the Helm release name is equal to the Application name to which it belongs."* The Application is named `guestbook`, so:

| Input | Value |
|---|---|
| `.Release.Name` | `guestbook` — the Application's name |
| `.Chart.Name` | `helm-guestbook` |
| `contains "helm-guestbook" "guestbook"` | false — so no collapse |
| Result | `guestbook-helm-guestbook` |

The collapse branch exists for the common case where the release is named after the chart. Here it is the other way round — the chart name contains the release name, not the reverse — so both get concatenated.

| | Plain manifests | Chart |
|---|---|---|
| Deployment | `guestbook-ui` | `guestbook-helm-guestbook` |
| Service | `guestbook-ui` | `guestbook-helm-guestbook` |

## Four resources, not two

Because the tracking id encodes the object's **name**, renaming an object makes it a different object as far as Argo CD is concerned. Apply the path change and refresh without syncing, and the app reports four:

```bash
kubectl -n argocd get app guestbook \
  -o jsonpath='{range .status.resources[*]}{.kind}/{.name}  {.status}{"\n"}{end}'
```

```text
Service/guestbook-helm-guestbook     OutOfSync
Service/guestbook-ui                 OutOfSync
Deployment/guestbook-helm-guestbook  OutOfSync
Deployment/guestbook-ui              OutOfSync
```

Two of them do not exist yet — rendered by the chart, absent from the cluster. Two of them exist and are no longer in the source, so the UI flags each with a prune warning — the resource is not present in the application source, and will be deleted from the cluster only if prune is enabled on the sync.

Crucially Argo CD does not tear down the Application and rebuild it. It compares object by object, finds two missing and two orphaned, and reports exactly that. The tracking ids show why they are treated as unrelated:

```text
guestbook:/Service:default/guestbook-ui                  ← on the live object
guestbook:/Service:default/guestbook-helm-guestbook      ← what the chart renders
```

Same app, same kind, same namespace — only the final segment differs, and that is enough. The mechanism is in **13 - Argo CD Applications vs. Kubernetes primitive manifests**.

## Making the names line up: `fullnameOverride`

The template's first branch short-circuits all of that. Feed it the old name and the rendered objects land on top of the existing ones:

```yaml
spec:
  source:
    path: helm-guestbook
    helm:
      valuesObject:
        fullnameOverride: guestbook-ui
```

`valuesObject` is an inline values file — the same thing a `values.yaml` would say, embedded in the Application. Its place in the precedence chain is in **21 - Managing Helm charts with Argo CD**.

With this applied the four resources collapse back to two, the tracking ids match, and the diff shrinks to what the chart genuinely adds:

- **labels** — `chart`, `release` and `heritage`, which the plain manifests never carried
- **ports** — the chart's Service uses a named `targetPort: http` where the plain Service used `targetPort: 80`
- **replicas** — the chart's `replicaCount: 1` against the three currently running

## The Deployment still fails, and the reason is worth understanding

Syncing at this point succeeds for the Service and fails for the Deployment:

```text
The Deployment "guestbook-ui" is invalid:
* spec.selector: Invalid value: {"matchLabels":{"app":"helm-guestbook","release":"guestbook"}}:
  field is immutable
```

`fullnameOverride` changes the **name** only. Labels and selectors come from a *different* template — `helm-guestbook.name`, which reads `nameOverride` or falls back to `.Chart.Name`:

| | Live object | Chart renders |
|---|---|---|
| `spec.selector.matchLabels` | `app: guestbook-ui` | `app: helm-guestbook`, `release: guestbook` |

A Deployment's `spec.selector` is immutable once created, so no apply can move it. The Service survives the same change because a Service's selector is an ordinary mutable field.

**`nameOverride` does not rescue it either:** setting it to `guestbook-ui` would fix the `app` label, but the chart's selector also carries `release: guestbook`, which the original never had. The selector would still differ, and it would still be immutable. For this particular pair the Deployment simply has to be recreated.

## Replace and Force

The way through is to stop patching and start replacing:

| Option | What it does |
|---|---|
| `Replace=true` | *"Argo CD will use `kubectl replace` or `kubectl create` command to apply changes."* |
| `Replace=true` + `Force=true` | *"the resources will be synchronized using the 'kubectl delete/create' command"* |

Both carry the same warning, and it is not decorative: *"This sync option has the potential to be destructive and might lead to resources having to be recreated, which could cause an outage for your application."*

**`Replace` on its own does not clear an immutable field:** `kubectl replace` sends a whole object rather than a patch, but it is still an *update*, so the same validation runs. Substituting the chart's selector into the live Deployment and submitting it both ways fails identically:

```text
kubectl apply   --dry-run=server  →  spec.selector: ... field is immutable
kubectl replace --dry-run=server  →  spec.selector: ... field is immutable
```

Only `Force` changes the outcome, because delete-then-create is not an update at all. That is why this particular sync needs both options set, not just `Replace`.

Treat it as the last resort it is. Reach for it when a field that cannot be patched has genuinely changed — an immutable selector, a Job's pod template — and not as a general fix for a stubborn sync.

## The other route: let it rename, and prune

Dropping `fullnameOverride` gives up on reusing the old objects. The chart renders `guestbook-helm-guestbook`, and the old `guestbook-ui` pair becomes garbage to collect:

| | Keep the names (`fullnameOverride`) | Let them change (prune) |
|---|---|---|
| Old objects | reused in place | deleted during sync |
| Needs | `Replace` + `Force` for the Deployment | **Prune** enabled on the sync |
| Downtime | the Deployment is recreated | the Deployment is recreated |
| Ends up with | objects named for the old scheme | objects named by the chart |
| Worth it when | the names are referenced elsewhere — Ingresses, Services, dashboards | nothing outside the app depends on the names |

Without prune the old objects stay behind, running and unmanaged, and have to be removed by hand.

Neither route avoids recreating the Deployment. The choice is only about which name it ends up with.

## `valueFiles`

The alternative to embedding values is to point at files inside the chart:

```yaml
    helm:
      valueFiles:
        - values.yaml
```

**The block nests under `helm`, not directly under `source`:** it is a Helm-specific setting, and `source.valueFiles` is silently not a field.

Listing `values.yaml` changes nothing on its own — a chart's own `values.yaml` is always the base layer, so naming it explicitly is a no-op that documents intent. Swapping in the other file is what has an effect:

```yaml
      valueFiles:
        - values-production.yaml
```

That file sets exactly one key, `service.type: LoadBalancer`, so the app goes `OutOfSync` on the Service type alone. It does not change the replica count — worth checking what a values file actually contains before assuming what it will move.

Paths resolve inside the repository, so the file has to live in the chart for Argo CD to find it.

## Where this cluster stands

The second route was taken — a sync with prune enabled — and it completed:

```text
guestbook   Synced   Healthy

Service/guestbook-helm-guestbook     Synced
Deployment/guestbook-helm-guestbook  Synced
```

The old `guestbook-ui` pair is gone, and the chart's two objects run under the names the fullname template produces. Every difference the diff predicted landed:

| | Before | After |
|---|---|---|
| Names | `guestbook-ui` | `guestbook-helm-guestbook` |
| Replicas | 3 | 1 — the chart's `replicaCount` |
| Service `targetPort` | `80` | `http`, a named port |
| Selector | `app: guestbook-ui` | `app: helm-guestbook`, `release: guestbook` |
| Labels | none | `app`, `chart`, `release`, `heritage` |

The Deployment is a new object rather than a patched one — which it had to be, given the immutable selector. The tracking ids moved with the names:

```text
guestbook:apps/Deployment:default/guestbook-helm-guestbook
guestbook:/Service:default/guestbook-helm-guestbook
```

That is what made the old pair prunable: nothing in the rendered output claimed them any more.

## Verify

```bash
kubectl -n argocd get app guestbook -o jsonpath='{.spec.source}'      # path + helm block
kubectl -n default get deploy,svc                                      # which names exist
kubectl -n default get deploy guestbook-ui -o jsonpath='{.spec.selector}'

argocd app manifests guestbook                                         # what the chart renders
argocd app diff guestbook                                              # rendered vs live
```

The clearest single check is the last: it shows the rename as a set of additions and deletions rather than as a modification, which is exactly how Argo CD sees it.
