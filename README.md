> **Provenance**: Maintained by IBM Client Engineering (github.com/ibmclientengineering). Derived from cloud-native-toolkit/multi-tenancy-gitops-services (Apache-2.0); modernized for CP4I 16.x / OpenShift 4.18+ (July 2026).

# multi-tenancy-gitops-services

The **shared-services layer** of the GitOps deployment: the Cloud Pak for Integration operators and the cluster-wide middleware that every tenant application builds on, expressed as version-pinned Helm charts and reconciled by Argo CD.

Part of the **IBM Client Engineering Cloud Pak for Integration production-deployment demo** — the four `multi-tenancy-gitops*` repos deploy a full CP4I estate from Git. This one owns everything between the platform and the apps: install the operators, stand up the shared instances, hand a ready runtime to the workloads.

## What this is

A catalog of Helm charts that Argo CD renders into two kinds of objects: OLM **operator subscriptions** and the **custom resources** those operators reconcile. It is owned by the platform-services team and reconciled **after `-infra`, before `-apps`** — so by the time an application syncs, its operators are installed and its shared middleware is already running.

## What's inside

- **`operators/`** — one chart per operator, each creating an OLM `Subscription` (and, via `ibm-catalogs`, the `CatalogSource`s) on a pinned channel. Covers the CP4I estate — `ibm-cp4i-operators`, `ibm-platform-navigator`, `ibm-mq-operator`, `ibm-ace-operator`, `ibm-apic-operator`, `ibm-datapower-operator`, `ibm-eventstreams-operator`, `ibm-foundations` (common services), `ibm-automation-foundation` — plus the platform primitives `cert-manager`, `sealed-secrets`, `openshift-gitops`, and `openshift-pipelines`.
- **`instances/`** — one chart per custom resource, materialized once the CRDs exist: `ibm-platform-navigator-instance` (the CP4I console), `mq-quickstart` (a `QueueManager`), the API Connect subsystem instances (`mgmt`, `gtw`, `ptl`, `a7s`), `ibm-eventstreams-instance`, `instana-agent`, plus `cert-manager` and `sealed-secrets` runtimes.
- Most charts are thin wrappers that pin a versioned subchart from `charts.cloudnativetoolkit.dev`; the platform primitives ship raw `operator.yaml` + Kustomize overlays. Values carry the **2026 CR model** — Platform Navigator `16.2.0` on operator channel `v8.3-sc2`, CP4I `v1.1-eus`, ACE `v13.3`, MQ `v4.0-sc2`, common services `v4.19` — not the fields that were removed in older CRDs.

## Why it's built this way

- **Separation of duties.** Operators and shared middleware live in their own repo with their own owners. The platform-services team moves an operator channel or an instance version without touching infrastructure or application code — and vice versa.
- **Everything is a version-pinned chart.** Each subscription and CR is a chart with an explicit channel and subchart version, so the cluster's operator estate is reproducible and auditable from Git rather than assembled by hand in the console.
- **Reusable factories.** `mq-quickstart`, the Platform Navigator instance, and the APIC subsystem charts are parameterized once and reused across tenants and clusters — change the values, not the templates.
- **Promotion is a pull request.** Bumping MQ to a new channel or Platform Navigator to a new version is a diff a reviewer can read and Argo CD can roll back. No `oc apply`, no drift.
- **Current, not deprecated.** Values track the 2026 CP4I 16.x CR shapes (e.g. Platform Navigator `16.2.0` with the retired `mqDashboard` field pruned), so a fresh sync lands on supported resources instead of failing against changed CRDs.
- **SealedSecrets is the secrets backend** for this demo — secrets are encrypted in Git and decrypted only in-cluster. The `ibm-vault` charts are present as the enterprise follow-on (Vault via External Secrets Operator); they are not the active backend here.

## How it fits the bigger picture

The demo splits one CP4I deployment across four repos, synced in order:

1. **[`multi-tenancy-gitops`](https://github.com/ibmclientengineering/multi-tenancy-gitops)** — the bootstrap repo; its Argo CD `Application`s point at the other three.
2. **[`multi-tenancy-gitops-infra`](https://github.com/ibmclientengineering/multi-tenancy-gitops-infra)** — cluster infrastructure (namespaces, RBAC, storage, networking).
3. **`multi-tenancy-gitops-services`** *(this repo)* — operators and shared instances.
4. **[`multi-tenancy-gitops-apps`](https://github.com/ibmclientengineering/multi-tenancy-gitops-apps)** — the tenant workloads and integration overlays that consume the queue managers, integration servers, and API gateways stood up here.

The operators and instances in this repo are the runtime contract the `-apps` overlays depend on: this layer installs MQ, App Connect, API Connect, Event Streams and DataPower and brings up their shared instances; `-apps` deploys the flows, queues, and APIs on top.

Maintained by IBM Client Engineering.
