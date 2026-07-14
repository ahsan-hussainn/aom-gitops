# aom-gitops

GitOps source of truth for the **AOM** platform. A single [ArgoCD](https://argo-cd.readthedocs.io/)
_app-of-apps_ reconciles the entire cluster state from the manifests in this
repo — every push to `dev` is applied automatically, with no manual `kubectl`
after the initial bootstrap.

The workloads here stand up an end-to-end **Change Data Capture (CDC)** pipeline:

```
┌──────────────┐   logical    ┌──────────────────┐   produce   ┌───────────────┐
│  PostgreSQL  │  replication  │  Debezium (Kafka │   topics    │     Kafka     │
│  (inventory) │ ────────────► │  Connect + PG    │ ──────────► │  (Strimzi,    │
│   WAL=logical│               │   connector)     │             │   KRaft)      │
└──────────────┘               └──────────────────┘             └──────┬────────┘
                                                                       │
                                                                 ┌─────▼──────┐
                                                                 │  Kafka UI  │
                                                                 │ (observe)  │
                                                                 └────────────┘
```

Row-level changes in Postgres are streamed off the write-ahead log by Debezium
and published as Kafka topics; Kafka UI provides a browser view of clusters,
topics, and the connector.

---

## Architecture

Everything is declared as an ArgoCD `Application`. The **root** Application
(`apps/root`) is an app-of-apps: it globs `apps/*/application.yaml` and manages
every other Application — including itself. Ordering across Applications is
controlled with `argocd.argoproj.io/sync-wave` annotations so dependencies come
up before their consumers.

| Wave | Application        | Source                          | What it deploys |
|-----:|--------------------|---------------------------------|-----------------|
| -100 | `root`             | `apps/` (recurse)               | App-of-apps — manages all Applications below |
|  -10 | `strimzi-operator` | Helm `strimzi-kafka-operator` 0.51.0 | The Strimzi operator that reconciles `Kafka` / `KafkaConnect` CRs (`watchAnyNamespace`) |
|    0 | `postgres-source`  | raw manifests                   | Postgres 16 with `wal_level=logical`, seeded DB `inventory` — the CDC source |
|    5 | `kafka`            | raw manifests                   | A `Kafka` CR + `KafkaNodePool` — single combined broker/controller in **KRaft** mode, ephemeral storage |
|   10 | `debezium-connect` | raw manifests                   | `KafkaConnect` with an in-cluster image build bundling the Debezium Postgres connector 2.7.3 |
|   10 | `kafka-ui`         | Helm `kafka-ui` (Provectus) 0.7.6 | Web UI wired to the Kafka cluster and the Debezium connect endpoint |

### Component notes

- **Kafka** runs in KRaft mode (no ZooKeeper) via Strimzi 0.51 on Kafka 4.1.1,
  as a single node combining the controller and broker roles. Storage is
  ephemeral and replication factors are `1` — this is a demo/lab footprint, not
  a production topology.
- **Debezium** is delivered as a `KafkaConnect` resource with
  `strimzi.io/use-connector-resources: "true"`, so connectors can be managed as
  `KafkaConnector` CRs. Strimzi builds the Connect image in-cluster, layering the
  Debezium Postgres connector plugin on top of the base Connect image.
- **Postgres** is configured for logical decoding (`wal_level=logical`,
  replication slots and WAL senders raised) so Debezium can tail the WAL.
- **Kafka UI** is exposed via an NGINX ingress at `kafka-ui.local`.

---

## Repository layout

```
apps/
  root/
    application.yaml        # app-of-apps root — manages every other Application (including itself)
  <app>/
    application.yaml        # ArgoCD Application CR — discovered by root via the apps/*/application.yaml glob
    values.yaml             # Helm values overlay (chart-source apps), OR
    manifests/              # raw Kubernetes manifests (directory-source apps)
```

Two source styles are used:

- **Chart apps** (`strimzi-operator`, `kafka-ui`) reference an upstream Helm
  chart plus a `values.yaml` overlay from this repo, using ArgoCD multi-source
  (`$values`).
- **Directory apps** (`postgres-source`, `kafka`, `debezium-connect`) point at a
  `manifests/` folder of raw YAML applied verbatim.

---

## Bootstrap (once per cluster)

Apply the root Application. Everything else follows.

```bash
kubectl apply -f apps/root/application.yaml
```

From here the root reconciles every other `apps/*/application.yaml` — no further
manual `kubectl apply` is ever needed.

> **Prerequisites:** a Kubernetes cluster with ArgoCD installed, an NGINX
> ingress controller (for Kafka UI), and network access from the cluster to the
> upstream Helm/OCI registries and Maven Central (for the Debezium plugin
> download during the Connect image build).

Point `kafka-ui.local` at your ingress controller (e.g. via `/etc/hosts` or DNS)
to reach the UI.

---

## Workflow

1. Edit an app's `values.yaml`, its `manifests/`, or the `Application` CR itself.
2. `git push origin dev`.
3. The root re-applies any Application-CR changes; each child Application then
   reconciles its own resources. End-to-end reconciliation is usually within
   ~3 minutes (faster if you click **Sync** in the ArgoCD UI).

All Applications run with `automated` sync (`prune: true`, `selfHeal: true`) —
drift is corrected automatically and removed resources are pruned.

**Adding an app:** drop a new folder under `apps/` with its own
`application.yaml` (and supporting `manifests/` or `values.yaml`). On the next
sync the root picks it up automatically — no `kubectl apply` step. Set a
`sync-wave` annotation if it must come up before or after existing apps.

**Removing an app:** `git rm -r apps/<app>/` and push. The root prunes the
Application CR, whose `resources-finalizer.argocd.argoproj.io` finalizer
cascade-deletes the underlying workload.

---

## Notes

- The tracked branch is **`dev`**; every Application's `targetRevision` is `dev`.
- The seeded Postgres credentials in `apps/postgres-source/manifests/secret.yaml`
  are plaintext demo values (`postgres` / `postgres`). Do not reuse them for
  anything real — replace with a proper secret store before any non-lab use.
