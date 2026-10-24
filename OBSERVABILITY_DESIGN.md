# Bouc.io Observability & Monitoring — Design

> **Status:** Implemented. All phases (1–4) applied and reconciling; Grafana is also wired to
> Keycloak OIDC login. See the implementation note below for as-built details.
> **Scope:** the whole-system telemetry, monitoring, and observability solution, spanning four
> components: OpenTelemetry, Jaeger, Prometheus, Grafana, and Datadog.
>
> This file lives in `infrastructure/opentelemetry/` because the OpenTelemetry collector is the
> hub of the design, but it describes the entire observability stack — not just OTel.

> **Implementation status (2026-06):**
> - **Done — Phase 1 (OTel collector):** `environment` tagged via `${ENVIRONMENT_ID}`
>   (Flux-substituted); Datadog trace/metrics/logs exporters commented out (`# DATADOG-PAID`); the
>   logs pipeline is kept receiving via a `debug` sink. Required adding `substituteFrom: cluster-vars`
>   to `otel-collectors-base-kustomization.yaml` (local + sandbox) and escaping runtime-expanded
>   collector vars as `$${...}` (see §7 caveat).
> - **Done — Phase 2 (Datadog):** the agent is the sole infra feeder; an `env:${ENVIRONMENT_ID}`
>   tag was added to `datadog/lcl.values.yaml` and `snbx.values.yaml` for consistency with OTel.
> - **Done — Phase 3:** this document updated.
> - **Done — Phase 4a (Grafana onboarding + datasource):** Grafana onboarded into FluxCD
>   (HelmRepository, HelmRelease `grafana-helmrelease`, `base`/`lcl`/`snbx` overlays, config +
>   infrastructure wiring, `grafana` namespace) and wired to Prometheus via a provisioned datasource
>   (`prometheus-server.prometheus.svc.cluster.local`). Reconciles once `infrastructure/grafana` is
>   committed and its `components/grafana` submodule pointer bumped. Note: it does **not** need a
>   `cluster-vars` entry in `infra-kustomization` (its values carry no `${...}` — same as Prometheus/Kiali).
> - **Done — Phase 4b (dashboards):** the 10 dashboard JSONs live in
>   `infrastructure/grafana/dashboards/base/` as **raw files only** (no `kustomization.yaml` in the
>   component data folder). They are turned into 4 label-tagged ConfigMaps (`grafana_dashboard: "1"`,
>   all < 1 MB) by `configMapGenerator` entries in `clusters/{local,sandbox}/config/kustomization.yaml`
>   — the same pattern as `keycloak-users-realm`. Each entry carries the
>   `kustomize.toolkit.fluxcd.io/substitute: "disabled"` annotation so Flux postBuild **skips** them
>   (the JSONs carry 470+ `${datasource}` / `${DS_PROMETHEUS}` / `$__rate_interval` vars that
>   substitution would blank out). Grafana's dashboards sidecar is enabled in `base.values.yaml`.
>   **No dedicated Flux Kustomization** and no component-folder `kustomization.yaml`; the
>   `components/grafana` submodule needs no re-bump (it already holds just the JSONs).
>   Convention confirmed via the Keycloak realm load (which escapes its own `${}` as `$${}`).
> - **Done — Grafana auth + cleanup:** Grafana wired to Keycloak OIDC against the `master` realm,
>   reusing Kiali's `kiali-client` (client secret written by the `keycloak-kiali-sync` hook Job in
>   `infrastructure/keycloak` — see the Grafana README); the duplicate Istio Mesh dashboard
>   (`7639_rev217`) was removed.
> - **Deferred (known backlog):** Grafana role mapping (`realm_access.roles` isn't in Keycloak's
>   userinfo, so all OAuth users land as Viewer), Grafana persistence (ephemeral SQLite), durable log
>   store (Loki), Jaeger persistence, Alertmanager, production overlay, and secret management.

---

## 1. Purpose

Define what each observability component is **for**, what telemetry it owns, and how signals are
routed between them — so that configuration is a matter of executing an agreed design rather than
rediscovering intent each time.

The design is shaped by one hard constraint and one forward-looking goal:

- **Constraint — free-tier Datadog.** We run Datadog on the free plan and intend to keep doing so
  for now. Free-tier Datadog is **infrastructure monitoring only** (≈5 hosts, ~1-day metric
  retention, dashboards, host/container/cluster-state metrics and events). **APM/tracing and Log
  Management are separate paid products and are not available.** We accept short retention and
  potential data loss on the free tier.
- **Goal — Datadog-ready.** Eventually everything may move to Datadog (paid). The design must make
  that flip **trivial**: keep apps OTLP-native, and keep the Datadog export paths present but
  **commented out**, not deleted. Turning Datadog into the primary backend should be an
  uncomment-and-reconcile operation, not a re-architecture.

---

## 2. Design principles

1. **Instrument once, route centrally.** Applications emit OTLP (traces **and** metrics) to the
   OpenTelemetry collector, which is the one place backends are swapped — never the apps. Apps
   **also** expose a Prometheus `/metrics` endpoint for direct scrape; both paths are intended.
   The OTLP-routed metrics arrive in Prometheus `otel_`-prefixed and `environment`-stamped by the
   collector, while the direct-scraped business metrics keep their native `*_total`/`*_seconds`
   names and carry an `environment` label sourced from `ENVIRONMENT_ID` — so the two families line
   up in Grafana without colliding. Trace/metric backends are still swapped by editing the
   collector alone.
2. **One job per component.** Each backend owns exactly one signal class it is best at. No
   component is the system of record for a signal another component already owns.
3. **Own your durable data, rent the convenience.** Self-hosted Jaeger and Prometheus are the
   durable stores we control. Datadog (free) is a convenient hosted *glance* layer for infra that
   we accept losing quickly.
4. **Reversibility toward Datadog.** Every Datadog integration that is disabled for cost reasons is
   left in place as commented configuration with a clear marker, so the paid migration is a flip.
5. **Environment-aware via `cluster-vars`.** Environment identity comes from the manually-created
   `cluster-vars` ConfigMap (substituted by Flux `postBuild.substituteFrom`), never hardcoded.
6. **Keep the FluxCD delivery model as-is.** Same Flux → Helm → values-from-ConfigMap wiring, same
   `base.values.yaml` + `lcl`/`snbx` overlays. The design changes *what* the values say, not *how*
   they are delivered.

---

## 3. Component responsibilities

| Component | Single purpose | Owns (at rest) | Rationale |
|---|---|---|---|
| **OpenTelemetry Collector** | Ingestion pipeline / router — receive, enrich (k8s metadata), sample, fan out | nothing at rest; all signals in transit | Vendor-neutral. The one place backends are wired. Makes the Datadog flip trivial. |
| **Jaeger** | Trace store + UI | traces | Datadog APM is not free; Jaeger is the real tracing backend. |
| **Prometheus** | Durable metrics store | metrics | Self-hosted, free, longer retention than Datadog free. System of record for metrics and basis for future alerting. |
| **Grafana** | Visualization for self-hosted metrics | none (reads Prometheus) | Prometheus has no usable UI. Grafana is the dashboard layer for app + infra metrics. |
| **Datadog (free)** | Best-in-class hosted infra dashboards | infra/cluster metrics + events (~1 day) | Polished infra view at zero cost. A premium glance layer, **not** a system of record. |

Mental model: **OTel is plumbing; Jaeger + Prometheus + Grafana are the durable stack we own;
Datadog is a free hosted view of infrastructure we accept losing quickly.**

---

## 4. Signal routing

| Signal | Primary destination | Datadog (free) today | Future paid Datadog |
|---|---|---|---|
| **Traces** | Jaeger (via OTel) | not sent — exporter **commented out** | uncomment OTel → Datadog trace exporter |
| **Metrics** | Prometheus (durable) → Grafana | infra metrics via Datadog agent only | OTel metrics exporter to Datadog uncommented |
| **Logs** | **received, not stored yet** (pipeline kept receiving; store plugged later) | not sent | OTel logs exporter to Datadog uncommented |
| **Infra / cluster state** | Datadog agent → Datadog | yes — agent's native job | unchanged (already there) |

Notes:

- **Traces → Jaeger only.** The OTel collector's Datadog trace exporter is kept but **commented
  out**. Jaeger is the trace home until we pay for APM.
- **Metrics → Prometheus is the record.** The Datadog *agent* feeds Datadog its own infra metrics
  natively. The OTel collector does **not** also export metrics to Datadog (avoids double-counting
  against free-tier limits). The OTel → Datadog metrics exporter is kept but commented out.
- **Logs are received but not stored yet.** Today the collector ships logs only to Datadog, which
  won't retain them on the free tier. We **stop sending logs to Datadog** but **keep the logs
  pipeline receiving** (OTLP receiver + processors stay active). The only remaining task to make
  logs durable is plugging a store exporter into the already-receiving pipeline — see §6.1 and §8.
- **Datadog has a single feeder: the Datadog agent.** All Datadog ingestion comes from the agent,
  not from OTel, while we are on the free tier. This keeps infra metrics from being counted twice.

---

## 5. Architecture

```
        ┌─────────────────────┐        ┌──────────────────────────┐
        │  Instrumented apps  │        │  Cluster nodes + kubelet │
        │ traces·metrics·logs │        │   host & pod metrics     │
        └──────────┬──────────┘        └───────┬──────────┬───────┘
                   │ OTLP                       │          │ infra
                   ▼                            ▼          ▼
        ┌─────────────────────────────┐   ┌─────────────────────┐
        │   OpenTelemetry collector   │   │    Datadog agent    │
        │ receive·enrich·sample·route │   │  infra autodiscovery│
        └───┬───────────┬─────────────┘   └──────────┬──────────┘
   traces   │   metrics │  (logs: received, no store)  │ infra metrics
            ▼           ▼                              ▼
     ┌───────────┐  ┌──────────────┐          ┌──────────────────┐
     │  Jaeger   │  │  Prometheus  │          │  Datadog (free)  │
     │ traces+UI │  │ metrics store│          │ infra · ~1d ret. │
     └───────────┘  └──────┬───────┘          └──────────────────┘
                           │ datasource
                           ▼
                    ┌──────────────┐
                    │   Grafana    │
                    │  dashboards  │
                    └──────────────┘

  Commented-out (Datadog-ready) paths, enabled when we go paid:
    OTel ──traces──▶ Datadog APM
    OTel ──metrics─▶ Datadog
    OTel ──logs────▶ Datadog Logs
```

Legend: self-hosted = Jaeger, Prometheus, Grafana (durable, owned). Datadog = free hosted glance
layer for infra. OTel = the router that makes the paid-Datadog flip a one-line change.

---

## 6. Per-component intent (what configuration will do)

This section states the intended end-state for each component. Concrete values land in the
configuration phase; this is the contract that phase implements.

### 6.1 OpenTelemetry collector
- **Receivers:** keep OTLP (apps), plus `hostmetrics` and `kubeletstats` for node/pod metrics.
  Jaeger/Zipkin receivers stay (cheap, allow non-OTLP trace sources).
- **Processors:** keep `memory_limiter` → `batch` → `resource` → `k8sattributes` → `tail_sampling`.
  - **`resource` processor:** replace the hardcoded `environment: production` with the
    `ENVIRONMENT_ID` value from `cluster-vars` (Flux substitution: `${ENVIRONMENT_ID}`). Every
    signal is then tagged with the real environment (local / sandbox / prod).
- **Exporters & pipelines:**
  - `traces` → Jaeger (`otlp/jaeger`). **Datadog trace exporter: keep but comment out.**
  - `metrics` → Prometheus exporter (`:8889`, scraped by Prometheus). **Datadog metrics export:
    keep but comment out.**
  - `logs` → **keep the pipeline receiving.** The OTLP receiver and processors stay active so the
    collector keeps accepting logs. **Comment out the Datadog logs exporter.** Because an OTel
    pipeline must reference at least one exporter to be valid, attach a temporary no-op sink
    (`nop`, or `debug` for visibility) so the pipeline stays healthy with nowhere durable to send.
    The **single remaining task** to make logs durable later is swapping that placeholder for a
    real store exporter (Loki, or uncommenting Datadog) — nothing else in the pipeline changes.
- **Datadog-ready markers:** every commented Datadog export block carries a clear comment (e.g.
  `# DATADOG-PAID: uncomment to route <signal> to Datadog`) so the paid flip is greppable.

### 6.2 Jaeger
- Stays as the trace store + UI (NodePort `30686`).
- **Known limitation:** in-memory storage (10k-trace cap) — traces do not survive a pod restart.
  Acceptable for local/sandbox; revisit persistence if tracing becomes load-bearing (§8).

### 6.3 Prometheus
- Stays as the durable metrics store; scrapes the OTel collector's `:8889` Prometheus endpoint and
  anything annotated `prometheus.io/scrape`.
- Retention per overlay (shorter local, longer sandbox). System of record for metrics and the
  future basis for alerting (Alertmanager currently disabled — a later decision).

### 6.4 Grafana — **wire to Prometheus** (currently disconnected)
- Today `datasources: {}` is empty — Grafana is deployed but has no data source.
- **Wire Grafana to Prometheus** by provisioning a Prometheus datasource, e.g.:
  ```yaml
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          access: proxy
          url: http://prometheus-server.prometheus.svc.cluster.local
          isDefault: true
  ```
  > Confirmed against the running cluster: service `prometheus-server` in namespace `prometheus`,
  > `ClusterIP` on port `80` → targetPort `9090` (release `prometheus`, Flux HelmRelease
  > `prometheus-helmrelease`). The URL above resolves correctly; no port needed (defaults to 80).
- Provision the existing dashboard JSONs in `infrastructure/grafana/` (k8s views, nodes, pods,
  namespaces, etc.) against that datasource via the dashboards sidecar.
- **Migrate Grafana to the standard overlay pattern** (`base.values.yaml` + `lcl`/`snbx`) to match
  the other components and the FluxCD delivery model. It currently only has `lcl.values.yaml`.
- **Auth: Keycloak OIDC (master/root realm), reusing Kiali's `kiali-client`.** `grafana.ini` →
  `[auth.generic_oauth]` in `base.values.yaml` points at `https://sso.${CLUSTER_DOMAIN}/realms/master`
  with `client_id: kiali-client` (the same confidential client Kiali uses); the client secret is
  injected as `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET` via `envValueFrom` from the `grafana-oauth`
  secret, which the `keycloak-kiali-sync` post-install hook Job in `infrastructure/keycloak` writes
  (along with Kiali's `kiali` secret) straight from Keycloak. `lcl` adds
  `tls_skip_verify_insecure` for local self-signed certs. No manual prereqs: the same Job creates
  the client with both redirect URIs when it is absent. The chart ingress stays
  disabled — external exposure (Istio VirtualService) lives in a separate GitOps repo; `server.root_url`
  is `https://grafana.${CLUSTER_DOMAIN}/` so the OAuth redirect matches it.

### 6.5 Datadog
- **Single feeder = the Datadog agent.** Keep the agent doing infra/cluster/container metrics and
  events (its native, free-tier-appropriate job).
- The OTel → Datadog export paths remain **commented out** (see §6.1) so nothing is sent to Datadog
  from OTel while on the free tier.
- Identity (`clusterName`, `clusterDomain`, environment) sourced from `cluster-vars`.

---

## 7. cluster-vars usage

`cluster-vars` is created manually during cluster bootstrap and is assumed present. Flux substitutes
its keys via `postBuild.substituteFrom`. Observability components consume:

| Key | Used by | Purpose |
|---|---|---|
| `ENVIRONMENT_ID` | OTel `resource` processor | Tags all telemetry with the environment (replaces hardcoded `production`). |
| `CLUSTER_NAME` | Datadog | Scopes hosts / cluster checks. |
| `CLUSTER_DOMAIN` | OTel operator / others | Cluster DNS domain. |

All components must read cluster identity from the same `cluster-vars` keys rather than defining
their own.

> **Flux substitution caveat (as-built).** Enabling `substituteFrom: cluster-vars` on the collector
> Kustomization makes Flux process *every* `${...}` token in the collector manifest. Runtime-expanded
> collector vars (`${K8S_NODE_NAME}`, and the commented `${DD_API_KEY}`) are therefore written as
> `$${...}` so Flux leaves them intact for the collector to expand at runtime; only `${ENVIRONMENT_ID}`
> is single-`$` for Flux to resolve. The values-based components (Datadog, etc.) are unaffected — their
> `${...}` tokens are already substituted by the existing `config`/`infra` Kustomizations.

---

## 8. Out of scope / future (named, not built)

- **Durable logs.** No durable log store today, but the OTel logs pipeline is **already receiving**
  (with a temporary no-op/debug sink). Making logs durable is a single wiring step: plug a store
  exporter into the existing pipeline — Loki (self-hosted, parallel to Prometheus/Jaeger) or paid
  Datadog Logs. Backend choice to be decided later; the receiving side is ready.
- **Jaeger persistence.** Move off in-memory storage if traces need to survive restarts.
- **Alerting.** Prometheus Alertmanager is disabled; alerting strategy (and whether it lives in
  Prometheus or Grafana) is a later decision.
- **Production overlay.** Prometheus/OTel/Datadog overlays cover local + sandbox today; production
  is to be defined.
- **Secret management.** Secrets (e.g. `datadog-secret`) are managed manually for now; SOPS /
  SealedSecrets integration is deferred — left as-is per current direction.
- **The paid-Datadog flip.** When moving to paid Datadog: uncomment the OTel Datadog export blocks
  (`# DATADOG-PAID` markers) for traces, metrics, and logs; decide whether Datadog supplements or
  replaces Jaeger/Prometheus/Grafana.

---

## 9. Summary of changes vs. current state

| Area | Current | Target (this design) |
|---|---|---|
| OTel traces | → Datadog **and** Jaeger | → Jaeger only (Datadog exporter commented out) |
| OTel metrics | → Datadog **and** Prometheus | → Prometheus only (Datadog exporter commented out) |
| OTel logs | → Datadog | kept receiving; Datadog exporter commented; temporary no-op sink, store exporter TBD |
| Environment tag | hardcoded `production` | `${ENVIRONMENT_ID}` from `cluster-vars` |
| Datadog feeder | OTel **and** Datadog agent | Datadog agent only |
| Grafana | deployed, no datasource | wired to Prometheus + dashboards provisioned |
| Grafana values layout | `lcl.values.yaml` only | `base` + `lcl`/`snbx` overlays |
