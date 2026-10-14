#argocd #gitops #projects #multitenancy #rbac

# Argo CD projects

An `AppProject` is a boundary around a set of Applications: which repositories they may deploy from, which clusters and namespaces they may deploy to, which kinds of objects they may create, and who may act on them. Every `Application` belongs to exactly one project, named in `spec.project`.

> *"Projects provide a logical grouping of applications, which is useful when Argo CD is used by multiple teams."*

## What a project is, and what it is not

| | |
|---|---|
| Kind | `AppProject`, `argoproj.io/v1alpha1` |
| Scope | **Namespaced** — always in `argocd`, Argo CD's own namespace |
| Short names | `appproj`, `appprojs` |

Argo CD looks a project up by name in its own namespace and nowhere else, so an `AppProject` created in any other namespace is never found. Applications are more flexible: with `sourceNamespaces` on the project and the matching `--application-namespaces` setting on the controller and API server, they may live in other namespaces — the projects they reference may not.

A project exists only inside Argo CD. It is not a cluster-wide boundary in the way a `Namespace` with Kubernetes RBAC is: it constrains what Argo CD does on behalf of an `Application`, and nothing else. Anyone with `kubectl` rights to the cluster bypasses it completely. Projects complement Kubernetes RBAC rather than replace it.

**The controller deploys with its own rights, and they are cluster-wide:** on this install the `argocd-application-controller` ClusterRole grants `*` on every API group, every resource and every verb.  
Whatever an Application's repository contains, the controller is able to apply — the project is what narrows that, team by team. This is why `sourceRepos` is a security control rather than bookkeeping: anyone allowed to point an Application at a repository can deploy its contents with the controller's privileges, within whatever the project permits. `destinationServiceAccounts`, listed further down, exists to sync with narrower credentials instead.

The documentation lists four jobs:

- *"restrict what may be deployed (trusted Git source repositories)"*
- *"restrict where apps may be deployed to (destination clusters and namespaces)"*
- *"restrict what kinds of objects may or may not be deployed (e.g. RBAC, CRDs, DaemonSets, NetworkPolicy etc...)"*
- *"defining project roles to provide application RBAC (bound to OIDC groups and/or JWT tokens)"*

## The `default` project

Created at install and wide open — any source, any destination, every kind:

```yaml
spec:
  sourceRepos:
  - '*'
  destinations:
  - namespace: '*'
    server: '*'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
```

*"The `default` project can be modified, but not deleted."* Until a dedicated project exists, every Application lands in it — on this cluster `headlamp` still does, while `guestbook` now belongs to `team-finance`.

The `spec.project` key itself is required by the CRD, as covered in **12 - The Application CRD**; it is an *empty* value that resolves to `default`. Locking `default` down has an immediate consequence: *"any application that attempts to use it will be denied until you explicitly move the application to a more permissive project."*

## The guardrails

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-finance
  namespace: argocd
spec:
  description: Project for the finance team
  sourceRepos:
    - 'https://github.com/<owner>/<repo>.git'
  destinations:
    - server: https://kubernetes.default.svc
      namespace: finance
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
```

### Where from: `sourceRepos`

A list of repository URLs, with globs and with `!` for exclusions:

```yaml
sourceRepos:
  - '!https://gitlab.com/group/**'   # nothing from this group
  - '*'                              # anything else
```

The rule for combining them is explicit — a source is valid when:

1. *"Any allow source rule (i.e. a rule which isn't prefixed with `!`) permits the source"*
2. *"AND no deny source (i.e. a rule which is prefixed with `!`) rejects the source"*

So a deny always beats an allow, regardless of order, and at least one allow rule must exist — *"`!*` is an invalid rule, since it doesn't make any sense to disallow everything."*

**Wildcards stop at `/`:** patterns are matched against the normalised repository URL (as in **41 - Connecting to private repositories**) with `/` as the segment boundary — `*` matches within one path segment, `**` across segments. The latest documentation adds a warning that holds for this version too: *"A wildcard placed on the organization segment admits lookalike organizations. Prefer an exact repository URL, or a pattern whose organization segment is bounded by `/`."*

```yaml
sourceRepos:
  - 'https://github.com/acme/*'    # repositories of acme only
  - 'https://github.com/acme*/*'   # also acme-evil — the * sits inside the organization segment
```

### Where to: `destinations`

Pairs of cluster and namespace, combined under exactly the same allow-and-no-deny rule:

```yaml
destinations:
  - namespace: '!kube-system'   # never here
    server: '*'
  - namespace: '*'              # anywhere else
    server: '*'
```

### What kinds: the four resource lists

| List | Applies to | Left out entirely |
|---|---|---|
| `clusterResourceWhitelist` | cluster-scoped kinds | **nothing** cluster-scoped is allowed |
| `clusterResourceBlacklist` | cluster-scoped kinds | nothing is denied |
| `namespaceResourceWhitelist` | namespaced kinds | **every** namespaced kind is allowed |
| `namespaceResourceBlacklist` | namespaced kinds | nothing is denied |

The asymmetry is deliberate: *"namespaced-scoped resources are restricted via a deny list, whereas cluster-scoped resources are restricted via allow list."*

```text
strict decoding error: unknown field "spec.namespaceResourceDenylist"
```

With validation switched off (`--validate=false`) it is worse: the API server drops the unknown field silently, and the project is stored without the restriction it was meant to carry.  
A kind is permitted only when it is whitelisted *and* not blacklisted.  
A new project that never mentions `clusterResourceWhitelist` can therefore deploy no Namespaces, ClusterRoles or CRDs at all — which is why the example above whitelists `Namespace` explicitly. On this cluster, `headlamp` could not join `team-finance` as configured: its chart renders a `ClusterRoleBinding`, and that project whitelists no cluster-scoped kind.

**Omitting `namespaceResourceWhitelist` is not the same as writing `[]`:** an absent whitelist allows every namespaced kind, but an explicit empty list allows none. Argo CD's permission check treats `nil` as "no restriction" and an empty list as a list with nothing in it.

**A namespace whitelist also shapes the resource tree:** *"Any child resource whose GroupKind is not permitted by the project is omitted from the tree, even if it exists and is healthy in the cluster."* Whitelisting only `apps/Deployment` and `Service` hides the ReplicaSets and Pods beneath them until those kinds are added as well. The effect is display only — it *"does not affect sync permissions for resources already deployed; it only affects observability in the resource tree."*

Keeping RBAC objects out of a project splits across both scopes. A `RoleBinding` is namespaced, so it is kept out with `namespaceResourceBlacklist`; a `ClusterRoleBinding` is cluster-scoped, so it is kept out simply by never whitelisting it.

The same deny list stops a team from overriding the limits set on its namespaces, while leaving every other namespaced kind available. Both kinds belong to the core group, hence `group: ''`:

```yaml
namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota
  - group: ''
    kind: LimitRange
```

Cluster-scoped kinds can also be narrowed by name, since names such as a Namespace's carry meaning of their own:

```yaml
clusterResourceWhitelist:
  - group: ''
    kind: Namespace
    name: team1-*
```

#### What `group` and `kind` refer to

Every entry in the four lists names a type of object by its API **group** and **kind** — read straight off a manifest's `apiVersion` and `kind` fields:

| Object | `apiVersion` | `group` | `kind` | Governed by |
|---|---|---|---|---|
| Service, ConfigMap | `v1` | `''` | `Service`, `ConfigMap` | namespace lists |
| Namespace | `v1` | `''` | `Namespace` | cluster lists |
| Deployment | `apps/v1` | `apps` | `Deployment` | namespace lists |
| Ingress, NetworkPolicy | `networking.k8s.io/v1` | `networking.k8s.io` | `Ingress`, `NetworkPolicy` | namespace lists |
| RoleBinding | `rbac.authorization.k8s.io/v1` | `rbac.authorization.k8s.io` | `RoleBinding` | namespace lists |
| ClusterRoleBinding | `rbac.authorization.k8s.io/v1` | `rbac.authorization.k8s.io` | `ClusterRoleBinding` | cluster lists |
| CustomResourceDefinition | `apiextensions.k8s.io/v1` | `apiextensions.k8s.io` | `CustomResourceDefinition` | cluster lists |
| Application | `argoproj.io/v1alpha1` | `argoproj.io` | `Application` | namespace lists |

- `group` is the part of `apiVersion` before the `/`. Core objects carry a bare `v1` with no slash, so their group is empty — written `group: ''`
- the version after the `/` is dropped entirely: an entry for group `apps` covers `apps/v1` and any other version of it alike
- `kind` is the manifest's `kind` — singular and CamelCase, `Deployment`, not the lowercase plural `deployments` from the `NAME` column of `kubectl api-resources`, which is what Kubernetes' own RBAC rules use
- which pair of lists applies is fixed by the kind's scope, not chosen. `RoleBinding` and `ClusterRoleBinding` share a group yet land in different lists, because one is namespaced and the other is not

Each entry is matched as a glob pattern, case-sensitively:

| Pattern | Matches |
|---|---|
| `group: '*'` | any group, the empty core group included |
| `group: ''` | the core group only — an empty pattern is not a wildcard |
| `group: '*.k8s.io'` | `rbac.authorization.k8s.io`, `networking.k8s.io` and the like, but not `apps` |
| `kind: deployment` | nothing — the case has to match |

`group` and `kind` are required in every entry; only the cluster-scoped lists accept the optional `name` shown above. The group, scope and kind of any type sit side by side in one listing:

```bash
kubectl api-resources | grep -E 'APIVERSION|rolebindings'
```

## Two checkpoints, two kinds of error

A project is enforced twice, at different moments, with different symptoms.

**Before rendering, against the Application spec:** a forbidden source or destination is recorded as an `InvalidSpecError` condition, and nothing is synced:

```text
application repo <url> is not permitted in project '<project>'
application destination server '<server>' and namespace '<ns>' do not match any of the allowed destinations in project '<project>'
```

**At sync, against every rendered resource:** each object's kind is checked against the four lists, and each namespaced object's *own* namespace against the destinations:

```text
resource <group>:<kind> is not permitted in project <project>
namespace <ns> is not permitted in project '<project>'
```

The second check closes a gap the first leaves open: a manifest that hardcodes `metadata.namespace` is judged by where it would actually land, not by the Application's `destination` — so the destination field cannot be used to smuggle objects elsewhere.

An `Application` applied with `kubectl` is admitted by the API server either way; the project is enforced by Argo CD afterwards, so a violation surfaces as a condition on the Application rather than as a `kubectl` error.

## Roles: RBAC scoped to one project

> *"Projects include a feature called roles that can be used to determine who and what can be done to the applications associated with the project."*

The documented example is a CI pipeline allowed to sync one application without being able to change its source or destination.

```yaml
spec:
  roles:
    - name: read-only
      description: Read-only privileges to my-project
      policies:
        - p, proj:my-project:read-only, applications, get, my-project/*, allow
      groups:
        - my-oidc-group
```

| Rule | Why it matters |
|---|---|
| the subject must be `proj:<project>:<role>` | *"It is very important that policy roles follow the pattern `proj:<project-name>:<role-name>` or they won't be effective"* |
| a role's policies stay inside its project | cross-project rules belong in `argocd-rbac-cm` instead |
| users join a role through `groups` | typically OIDC groups |
| machines join through JWT tokens | `argocd proj role create-token <project> <role>` |

**Give role tokens an expiry:** *"By default, the cli creates them without an expirations date."* Passing `-e` sets one. *"Since the JWT tokens aren't stored in Argo CD, they can only be retrieved when they are created"*, and a revoked token stops working even before it expires.

## Everything else a project holds

The installed CRD carries fourteen `spec` fields; the guardrails and roles above are seven of them.

| Field | Purpose |
|---|---|
| `description` | free text |
| `syncWindows` | *"controls when syncs can be run for apps in this project"* |
| `orphanedResources` | monitors objects in the project's namespaces that no Application tracks — the meaning of "orphaned" distinguished in **31 - Automated sync, pruning and self-healing** |
| `signatureKeys` | PGP key IDs that commits must be signed with before they may be synced |
| `sourceNamespaces` | namespaces outside `argocd` that may hold this project's Applications |
| `destinationServiceAccounts` | service accounts impersonated when syncing, per destination |
| `permitOnlyProjectScopedClusters` | restricts destinations to clusters registered to this project |

Repositories and clusters can be scoped to a project too — the optional `project` field on a repository Secret, from **41 - Connecting to private repositories**. That is how a team adds its own repository without an admin editing the project: when Argo CD loads a project, it appends every repository scoped to it to `sourceRepos`, and every cluster scoped to it to `destinations` — with every namespace on that cluster when the cluster lists none. The latest documentation states the consequence for clusters plainly: *"these implicit destinations are not reflected in the AppProject's declarative spec or the UI."* Permission to add a project-scoped repository or cluster is therefore permission to widen the project's guardrails — for clusters, without the change showing anywhere in the project — and deserves the same care as editing the project itself.

## Moving an Application into a project

```bash
argocd app set guestbook --project team-finance
```

or a change to `spec.project` followed by `kubectl apply`. Through the CLI or UI, *"the user must have permissions to access the new project"*.

The Application must also satisfy the new project's rules from the moment it moves. `guestbook` deployed from `maor-klir/argocd-example-apps` into `default`, so joining `team-finance` meant permitting that repository and changing the destination to `finance` in the same step — otherwise the move is an `InvalidSpecError`, with the messages above.

**A new manifest brings only its own fields:** the `team-finance` manifest carries no `syncPolicy`, so `guestbook` no longer has the automated sync, prune and self-heal settings from earlier. Re-created from that manifest, the Application simply never had them; re-applied over the earlier one, the result is the same, because the three-way merge from **14 - Deploy our first application** deletes fields the previous apply had set.

### Objects outside the project's rules drop out of sight

Before comparing, Argo CD runs every live object it found for an Application through the project's rules and discards the ones that are not permitted — the controller's own comment reads *"filter out all resources which are not permitted in the application project"*.

It surfaced here through a delete and re-create. `guestbook` was deleted from the UI without cascading — the **Non-Cascading** option, which *"Deletes only the Application resource"* and leaves everything it managed running — and then created anew in `team-finance` with the `finance` destination. The old Deployment and Service stayed in `default`, still carrying tracking ids that name `guestbook`:

```text
Deployment  guestbook-helm-guestbook  guestbook:apps/Deployment:default/guestbook-helm-guestbook
Service     guestbook-helm-guestbook  guestbook:/Service:default/guestbook-helm-guestbook
```

The new Application had the same name, so the tracking ids on those objects still identified them as its resources. It nevertheless reported `Synced` and listed only the two objects in `finance`. Because `default` is not a permitted destination of `team-finance`, Argo CD filtered the old objects out before comparing — they were absent from the Application's resource list and were not marked for pruning. `Synced` therefore described only the objects the project permits, not every object whose tracking id names the Application.

Two ways to avoid stranding objects like this:

- **When deleting:** keep the UI's default, **Foreground**, which *"Deletes the application and all its managed resources"* — the cascading delete from **13 - Argo CD Applications vs. Kubernetes primitive manifests**
- **When moving without deleting:** change the destination and sync with prune while the old namespace is still permitted, then tighten the project — after the switch, the filter hides the old objects in exactly the same way

## Verify

```bash
kubectl -n argocd get appproj                                         # every project
argocd proj get team-finance                                          # its rules, readably
kubectl -n argocd get app -o custom-columns='APP:.metadata.name,PROJECT:.spec.project'
kubectl -n argocd get app guestbook -o jsonpath='{.status.conditions}'   # InvalidSpecError lands here
```
