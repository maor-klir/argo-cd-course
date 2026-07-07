## 🎯 Lab Goal

Learn how to customize Helm chart deployments in Argo CD by setting custom values using different override methods, and understand the order of precedence when values conflict.

## 📝 Overview & Concepts

One of the most powerful features of Helm charts is their ability to be configured through values. When deploying charts with Argo CD, you need to know how to override these default values to customize your deployments for different environments and use cases.

Argo CD provides three main methods to override Helm chart values within the `spec.source.helm` block:

1. **`valueFiles`**: Reference one or more values files from your Git repository
2. **`valuesObject`**: Define value overrides directly in the Application manifest as structured YAML
3. **`parameters`**: Override individual values using key-value pairs

### Understanding Value Precedence

When the same value is defined in multiple places, Argo CD follows a strict order of precedence (from lowest to highest priority):

1. `values.yaml` in the chart itself (the defaults)
2. `valueFiles`
3. `valuesObject` / `values`
4. `parameters` (highest priority)

This means that `parameters` will always win over `valuesObject`, which wins over `valueFiles`, which wins over the chart's default `values.yaml`.

## 📋 Lab Tasks

### Part 1: Explore the Default Values

1. Navigate to your fork of the `argocd-example-apps` repository
2. Open the `helm-guestbook/values.yaml` file to see the default chart values
3. Note the default `replicaCount` value

### Part 2: Override Using `valueFiles`

1. In your `argocd-example-apps` repository, create a new file called `values-custom.yaml` in the `helm-guestbook` directory
2. In this file, set:
   - `replicaCount: 3`
3. Commit and push this file to your repository
4. Update your `guestbook-app.yaml` Application manifest to use this values file:
   ```yaml
   helm:
     valueFiles:
       - values-custom.yaml
   ```
5. Apply the updated manifest and sync the application in Argo CD UI
6. Verify with `kubectl` that 3 replicas are now running

### Part 3: Override Using `valuesObject`

1. Update your `guestbook-app.yaml` to replace the `valueFiles` with `valuesObject`:
   ```yaml
   helm:
     valuesObject:
       replicaCount: 2
       service:
         type: NodePort
   ```
2. Apply the manifest and sync in Argo CD
3. Verify that now only 2 replicas are running and the service type changed to NodePort
4. Notice how `valuesObject` completely replaced the previous configuration

### Part 4: Override Using `parameters`

1. Update your `guestbook-app.yaml` to add `parameters` while keeping `valuesObject`:
   ```yaml
   helm:
     valuesObject:
       replicaCount: 2
       service:
         type: NodePort
     parameters:
       - name: replicaCount
         value: '1'
   ```
2. Apply the manifest and sync
3. Verify that only 1 replica is running now
4. Observe that the `parameters` value (1) overrode the `valuesObject` value (2)

## 📚 Helpful Resources

- [Argo CD - Helm Values Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#values-files)
- [Helm Values Documentation](https://helm.sh/docs/chart_template_guide/values_files/)
- [Argo CD Application Specification](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications)

## 💡 Best Practices

- **Use `valueFiles`** for environment-specific configurations (dev, staging, prod)
- **Use `valuesObject`** for structured overrides that should be visible in the Application manifest
- **Use `parameters`** sparingly, typically for single values that need to override everything else
- Remember the precedence order when troubleshooting unexpected values

## 💭 Reflection Questions

1. Why is understanding the precedence order (chart defaults → valueFiles → valuesObject → parameters) critical when troubleshooting why a particular value isn't being applied as expected?
2. In what scenarios would you choose `valueFiles` over `valuesObject`, and what are the trade-offs between storing configuration in Git files versus inline in the Application manifest?
3. Given that `parameters` has the highest precedence, why should you use this method sparingly rather than for all your value overrides?
