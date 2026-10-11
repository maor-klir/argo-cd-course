#argocd #helm #values #precedence #templating

# Overriding Helm chart values

A chart ships defaults; an `Application` overrides them. There are five places an override can come from, and when two of them set the same key the winner is decided by a fixed order — not by position in the manifest.

## The five mechanisms

All of them nest under `source.helm`, and each maps onto something `helm template` already does:

| Key | Shape | `helm template` equivalent |
|---|---|---|
| `valueFiles` | list of paths inside the repository | `--values` |
| `values` | a YAML document embedded as a **string** | — |
| `valuesObject` | the same content embedded as real YAML | — |
| `parameters` | `name` / `value` pairs, both strings | `--set` |
| `fileParameters` | `name` / `path` pairs, value read from the file | `--set-file` |

```yaml
spec:
  source:
    repoURL: https://kubernetes-sigs.github.io/headlamp/
    chart: headlamp
    targetRevision: 0.45.0
    helm:
      valueFiles:
        - values.yaml
        - custom-values.yaml
      valuesObject:
        config:
          oidc:
            secret:
              create: false
      parameters:
        - name: replicaCount
          value: '2'
```

**Watch the singular and the plural:** `valueFiles` is plural, `valuesObject` is singular, and they are unrelated fields. Mistyping one produces a schema validation error rather than silent acceptance, because the CRD declares the exact set of keys the `helm` block allows.

## Precedence

Officially:

> *"Order of precedence is `parameters > valuesObject > values > valueFiles > helm repository values.yaml`."*

Read as a ladder, weakest at the bottom:

| | Source | Beats everything below it |
|---|---|---|
| 5 | `parameters` | the highest authority |
| 4 | `valuesObject` | |
| 3 | `values` | |
| 2 | `valueFiles` | |
| 1 | the chart's own `values.yaml` | the fallback when nothing else sets the key |

The chart's own defaults are not a *file we listed* — they are the base layer Helm always starts from. Listing `values.yaml` in `valueFiles` is therefore close to a no-op: it names the file Helm would have read anyway.

**Precedence is per key, not per block:** a `parameters` entry setting `replicaCount` does not discard the rest of a values file. Every layer is merged; only the keys that collide are resolved by rank.

## Ordering inside a list

Two of the five are lists, and within each one the **last entry wins**:

> *"When multiple valueFiles are specified, the last file listed has the highest precedence."*

The same applies to duplicate keys inside a single file — *"When multiple of the same key are found the last one wins"* — and to repeated `parameters` entries naming the same key.

So there are two independent rules working together: rank decides between mechanisms, position decides within one.

## Confirmed by rendering

Every rule above, checked against a real chart by moving one key (`replicaCount`, which the chart defaults to `1`):

| What was passed | Renders |
|---|---|
| nothing — chart defaults only | `1` |
| `-f a(2)` | `2` |
| `-f a(2) -f b(3)` | `3` — later file wins |
| `-f b(3) -f a(2)` | `2` — same files, order reversed |
| one file setting the key twice, `4` then `5` | `5` — last occurrence wins |
| `-f b(3) --set 9` | `9` — `parameters` outrank `valueFiles` |
| `--set 7 --set 8` | `8` — later entry wins |

The middle rungs of the ladder cannot be reproduced with the Helm CLI alone, because `values` and `valuesObject` have no CLI equivalent — Argo CD writes them to a temporary file before invoking Helm. Their rank comes from the documented order above.

## `values` and `valuesObject` hold the same thing

The difference is only how it is written: `values` is a string containing YAML, `valuesObject` is YAML.

```yaml
      values: |                      # a string that happens to look like YAML
        config:
          enableHelm: true

      valuesObject:                  # structured YAML the schema understands
        config:
          enableHelm: true
```

Prefer `valuesObject`. Editors, schema validation and diff tools can all see into it, where `values` is an opaque block that only Helm ever parses.

The schema is explicit about what happens if both are set: *"ValuesObject takes precedence over Values, so use one or the other."* Setting both is accepted rather than rejected, which makes it a quiet way to have an override that never takes effect.

## `parameters` are strings, always

Both `name` and `value` are typed `string` in the CRD. A YAML number or boolean fails validation:

```text
spec.source.helm.parameters[0].value: Invalid value: "integer":
spec.source.helm.parameters[0].value in body must be of type string: "integer"
```

Quote the value — `value: '2'`, `value: 'false'` — and Helm converts it back when rendering. Nested keys use dots, the same as `--set`:

```yaml
      parameters:
        - name: config.oidc.secret.create
          value: 'false'
```

**`forceString` stops that conversion:** *"ForceString determines whether to tell Helm to interpret booleans and numbers as strings"*. Without it `value: '2'` reaches the template as the number `2`; with it, as the text `"2"`. It matters for keys a chart puts into a label or an annotation, where a number would be invalid.

## Two fields worth knowing

| Field | Effect |
|---|---|
| `fileParameters` | *"file parameters to the helm template"* — reads the value from a file rather than inlining it, mapping to `--set-file`. Useful for a certificate or a script that would be unreadable inline |
| `ignoreMissingValueFiles` | *"prevents helm template from failing when valueFiles do not exist locally by not appending them to helm template --values"*. Turns a missing optional overlay from a render failure into a no-op |

`ignoreMissingValueFiles` is what makes a per-environment pattern safe: list `values.yaml` and `values-$ENV.yaml`, and the second is simply skipped where it does not exist.

## Which to reach for

| Situation | Use |
|---|---|
| Settings shared by every environment | a values file in the chart |
| Settings that differ per environment | a second entry in `valueFiles` |
| A handful of keys specific to this `Application` | `valuesObject` |
| One key, often changed by automation or a CI step | `parameters` |
| A value too large or too awkward to inline | `fileParameters` |

The general rule is to keep overrides as low on the ladder as they can go. A key set in `parameters` cannot be overridden by anything, which is exactly what makes it useful for a one-off and awkward for anything a later layer should be able to adjust.

## Verify

```bash
kubectl -n argocd get app <name> -o jsonpath='{.spec.source.helm}'   # every override in force
argocd app manifests <name>                                          # the rendered result
kubectl explain application.spec.source.helm.valuesObject            # schema, from the cluster
```

The decisive check is `argocd app manifests`: it shows what the layers actually produced, which is the only place a precedence mistake becomes visible. An override that lost to a higher rank leaves no trace anywhere else.
