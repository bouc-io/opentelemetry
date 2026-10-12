# OpenTelemetry GitOps Integration

This directory captures the shared configuration that backs the FluxCD deployment of the OpenTelemetry Operator and the Bouc.io collector pipeline.

## Repository Layout

```
infrastructure/opentelemetry/
├── base.values.yaml       # Shared Helm values leveraged by every environment
├── lcl.values.yaml        # Local-only overrides (e.g. clusterDomain)
├── snbx.values.yaml       # Sandbox-only overrides
├── base/                  # Source collector manifests synced by Flux
│   └── boucio-otel-collector.yaml
└── README.md              # (this file)
```

Environment overlays consume `base.values.yaml` plus the relevant `*.values.yaml` file through `configMapGenerator` entries (see `clusters/*/config/kustomization.yaml`).

## FluxCD GitOps

The Flux stack installs and manages OpenTelemetry via the manifests below:

- `fluxcd/fluxcdboucio/clusters/base/infrastructure/fluxcd-opentelemetry.yaml`
  - Creates the `opentelemetry` namespace.
  - Registers the upstream Helm repository.
  - Defines the `HelmRelease` that pulls its values from the generated ConfigMaps.
  - Includes the `OpenTelemetryCollector` CR used across environments.
- `fluxcd/fluxcdboucio/clusters/local/infrastructure/kustomization.yaml`
- `fluxcd/fluxcdboucio/clusters/sandbox/infrastructure/kustomization.yaml`
  - Reference the base manifest above so it is reconciled for each environment.
- `fluxcd/fluxcdboucio/clusters/local/config/kustomization.yaml`
- `fluxcd/fluxcdboucio/clusters/sandbox/config/kustomization.yaml`
  - Generate `opentelemetry-base-values` and `opentelemetry-level-values` ConfigMaps pointing at `clusters/components/opentelemetry/*.values.yaml`.
- `fluxcd/fluxcdboucio/clusters/local/flux-system/otel-collectors-base-kustomization.yaml`
- `fluxcd/fluxcdboucio/clusters/sandbox/flux-system/otel-collectors-base-kustomization.yaml`
  - Reconcile the collector definitions from `clusters/components/opentelemetry/collectors/base`.

> **Note:** The `clusters/components/opentelemetry` folder is expected to be sourced via Git submodule. Create or update the submodule before reconciling Flux so the Kustomizations above can locate the manifests.

## Secret Management

The collector references `datadog-secret` (key `api-key`) for the Datadog API key.
It is delivered by External Secrets, not created by hand: the
`boucio-datadog-secret` ExternalSecret in
`infrastructure/external-secrets/externalsecrets/infra/<env>/` writes it into this
namespace. The `datadog` namespace gets its own copy from the same source value,
because Secrets are namespace-scoped.

Set the source value once (`boucio-datadog-api-key`): on local in the gitignored
`boucio-local-external-credentials.yaml`, on sandbox in GCP Secret Manager. See
`infrastructure/external-secrets/README.md`.

These ExternalSecrets are applied by the `<env>-eso-infra-externalsecrets`
Kustomization, which is deliberately `wait: false` with nothing depending on it, so
a missing Datadog key degrades Datadog only and never blocks other reconciliation.

For a standalone install outside this GitOps setup, create it manually:

```shell
kubectl create secret generic datadog-secret \
  -n opentelemetry \
  --from-literal api-key=<API KEY>
```

## Local

1. Ensure the Datadog secret exists in your cluster.
2. Push the updated OpenTelemetry submodule and this directory to Git.
3. Trigger reconciliation:
   ```
   flux reconcile kustomization flux-system --with-source
   ```
4. Confirm resources:
   ```
   kubectl get pods -n opentelemetry
   kubectl get otelcol -n opentelemetry
   ```

## Accessing Jaeger V2 UI

The Jaeger V2 UI is available at `http://localhost:30686`.

## Legacy Helm Commands

For historical reference, the original imperative install steps were:

```
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
kubectl create namespace opentelemetry --cluster docker-desktop
helm upgrade -install opentelemetry-operator -f local.operator.values.yaml open-telemetry/opentelemetry-operator -n opentelemetry --kube-context docker-desktop
kubectl apply -f collectors/base/boucio-otel-collector.yaml --cluster docker-desktop
```

These commands are no longer required once Flux is in place but are kept here for context.

## References

1. https://opentelemetry.io/docs/what-is-opentelemetry/
2. https://opentelemetry.io/docs/
3. https://opentelemetry.io/docs/platforms/kubernetes/helm/

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
