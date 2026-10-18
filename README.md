# ECS Observability: Grafana Dashboards and Prometheus Monitoring on Huawei Cloud Stack
 
Prometheus and Grafana observability for the Elastic Cloud Server (ECS) fleet on Huawei Cloud Stack, combining ManageOne inventory metrics with in-guest `node_exporter` telemetry, delivered entirely through an Azure DevOps pipeline.
 
---
 
## Table of Contents
 
- [Overview](#overview)
- [Real-World Business Value](#real-world-business-value)
- [Skills Demonstrated](#skills-demonstrated)
- [Project Folder Structure](#project-folder-structure)
- [Tasks and Implementation Steps](#tasks-and-implementation-steps)
- [Core Implementation Breakdown](#core-implementation-breakdown)
- [IAM Role and Permissions](#iam-role-and-permissions)
- [Project Features (Detailed Breakdown)](#project-features-detailed-breakdown)
- [Design Decisions and Highlights](#design-decisions-and-highlights)
- [Local Testing and Validation](#local-testing-and-validation)
- [Errors Encountered and Resolved](#errors-encountered-and-resolved)
- [Conclusion](#conclusion)
---
 
## Overview
 
This component delivers end-to-end observability for the Elastic Cloud Server (ECS) estate that underpins a private-cloud Database-as-a-Service (DBaaS) platform on Huawei Cloud Stack, built for a regulated banking environment. It answers two distinct operational questions that were previously unanswerable from a single place: what the platform has been allocated, and what the platform is actually consuming.
 
Allocation and inventory data are sourced from the ManageOne Operation Centre through a purpose-built Python exporter, `oc_exporter.py`, which publishes the `oc_vm_*` metric family on port 9199. Guest operating system utilisation is sourced from Prometheus `node_exporter` running directly on each ECS instance and scraped on port 9100. Prometheus correlates both sources through a shared `vm_name` label, and three Grafana dashboards present the result as an overview, a per-instance detail view, and a deep operating system view.

<img width="3024" height="1724" alt="image" src="https://github.com/user-attachments/assets/b6fe2bd8-a106-465a-8f46-f99dc6850b2a" />

 
The solution is composed of the following main components:
 
- **Metrics generation.** The Operation Centre exporter and its systemd unit live in `5-vdc-shared/0-dev/oc-exporter`.
- **Metrics collection and alerting rules.** The Prometheus scrape configuration and rule files live in `5-vdc-shared/0-dev/prometheus`.
- **Visualisation.** The dashboard definitions and the Grafana import helper live in `5-vdc-shared/0-dev/grafana`.
- **Notification.** The Alertmanager configuration and Microsoft Teams templates live in `5-vdc-shared/0-dev/alertmanager`.
- **Delivery.** The Azure Pipelines definition that validates and deploys every artefact is `pipelines/0-dev-monitoring.yml`.
### Scope boundaries
 
Included in scope:
 
- ECS inventory, allocation and state metrics derived from the ManageOne `cloud_vm` resource API.
- Guest operating system utilisation metrics for ECS instances that run `node_exporter`.
- Automatic Prometheus target discovery for the ECS fleet using file-based service discovery.
- Three Grafana dashboards with bidirectional drill-down navigation.
- Continuous validation and automated deployment of all configuration through Azure Pipelines.
Intentionally excluded from scope:
 
- PostgreSQL Relational Database Service monitoring. That estate is covered separately by `postgres_exporter` and the `oc_rds_*` metric family.
- Application-level and database-level metrics from workloads hosted on the ECS instances.
- Long-term metric retention and federation. Prometheus is configured for local retention only, bounded at 32 days or 12 GiB, whichever limit is reached first.
- Environments beyond development. The test, user acceptance test, pre-production and production equivalents are not yet provisioned.
---
 
## Real-World Business Value
 
- **Elimination of manual capacity reporting.** Before this work, ECS allocation figures were obtained by manually paging through the ManageOne Operation Centre console. The exporter now discovers the fleet automatically on every scrape interval and publishes allocation as queryable time series, which removes a recurring manual reporting task.
- **Detection of resource waste.** Correlating allocated vCPU and memory against measured guest utilisation makes over-provisioned instances visible. This converts an opinion-based right-sizing discussion into a data-based one and creates a direct path to reduced consumption of the shared resource pools.
- **Reduced mean time to identify.** Operators move from a fleet-wide overview to a single instance and then to full operating system detail through in-dashboard drill-down links, without changing tools or writing ad hoc PromQL.
- **Standardisation across the estate.** Every dashboard, scrape job and alert rule is defined as code in a single repository, reviewed through pull request, and applied by one pipeline. Configuration drift between what is documented and what is running is therefore structurally prevented.
- **Auditability and change control.** Each change to a dashboard or a scrape target is a commit with an author, a reviewer and a pipeline run. This satisfies the change evidence expectations that apply to a regulated banking platform.
- **Security posture.** No credential is stored in the repository. Every secret is injected at deployment time from an Azure DevOps variable group and is masked in pipeline logs.
---
 
## Skills Demonstrated
 
- **Prometheus engineering.** Design of a multi-source scrape topology that unifies a custom exporter and a fleet of `node_exporter` agents, including file-based service discovery, relabelling, recording rules and alerting rules validated by `promtool`.
- **Custom exporter development in Python.** Implementation of an authenticated collector against the Huawei ManageOne Operation Centre REST API, including RSA-OAEP password encryption, Central Authentication Service single sign-on, session cookie and cross-site request forgery token handling, paginated resource discovery, exponential authentication back-off and Prometheus client instrumentation.
- **Grafana dashboard design as code.** Authoring of dashboard JSON by hand with template variables, threshold-driven colour mapping, value mappings, and cross-dashboard data links that propagate the selected instance as a URL variable.
- **Azure Pipelines engineering.** A two-stage pipeline with a validate stage that runs on every pull request and a deploy stage gated to merges into `main`, parameterised per component, executed on a self-hosted agent inside the private cloud.
- **Secure secret handling.** Consumption of an Azure DevOps variable group and secure files, environment variable substitution at deployment time, and strict SSH host key verification.
- **Infrastructure as Code on Huawei Cloud Stack.** Terraform module authoring for `hcs_ecs_compute_instance` with pinned provider versions, remote state in Object Storage Service, and cross-stack data lookups.
- **Linux service operations.** Hardened systemd unit design applying `ProtectSystem`, `ProtectHome`, `PrivateTmp`, `NoNewPrivileges` and memory accounting limits.
- **Least privilege identity design.** Separation of the monitoring service account from the infrastructure and role-based access control service accounts, each with distinct credentials and distinct group membership.
---
 
## Project Folder Structure
 
```text
infra-live/
├── pipelines/
│   ├── 0-dev-monitoring.yml            # Validate and deploy the whole monitoring stack
│   └── README.md                       # Pipeline conventions, gates and templates
├── 5-vdc-shared/
│   └── 0-dev/
│       ├── grafana/
│       │   ├── hcs-ecs-overview-dashboard.json         # Fleet overview, uid ecs-overview-001
│       │   ├── hcs-ecs-instance-detail-dashboard.json  # Single instance, uid ecs-detail-001
│       │   ├── node-exporter-full-dashboard.json       # Operating system detail, uid rYdddlPWk
│       │   └── prepare_import_payload.py               # Wraps dashboard JSON for the Grafana API
│       ├── prometheus/
│       │   ├── dev-configuration.yaml                  # Scrape jobs, service discovery, Alertmanager
│       │   ├── dev-alerting-rules.yaml                 # Recording rules and alert rules
│       │   └── metrics-and-thresholds.md               # Metric catalogue and threshold rationale
│       ├── oc-exporter/
│       │   ├── oc_exporter.py                          # ManageOne collector, listens on 9199
│       │   ├── oc-exporter.service                     # Hardened systemd unit for the collector
│       │   ├── node-exporter.service                   # Hardened systemd unit for node_exporter
│       │   └── ECS_FLEET_RECONCILER.md                 # Controlled fleet rollout procedure
│       └── alertmanager/
│           ├── dev-alertmanager.yml                    # Routing, grouping and receivers
│           └── templates/teams.tmpl                    # Microsoft Teams message rendering
├── 2-vdc-infra/
│   └── 0-dev/
│       └── ecs/                                        # Terraform root module for the ECS estate
└── images/
    └── node-exporter-full-dashboard.png                # Dashboard screenshots referenced by this document
```
 
Purpose of each major folder:
 
- **`pipelines`** holds every Azure Pipelines definition for the platform. It exists so that delivery logic is versioned alongside the configuration it deploys rather than being maintained in the Azure DevOps user interface.
- **`5-vdc-shared/0-dev/grafana`** holds the dashboard JSON that is the single source of truth for what operators see. Dashboards are never edited directly in the Grafana user interface, because the pipeline overwrites them on the next deployment.
- **`5-vdc-shared/0-dev/prometheus`** holds the scrape configuration and rule files. Keeping the rules beside the configuration allows `promtool check config` to resolve the `rule_files` references during validation.
- **`5-vdc-shared/0-dev/oc-exporter`** holds the collector source and the systemd units for both the collector and `node_exporter`. It exists because the metrics required for this platform are not published by any off-the-shelf exporter.
- **`5-vdc-shared/0-dev/alertmanager`** holds notification routing and message templates, separated from rule evaluation so that changes to notification behaviour do not require changes to alert logic.
- **`2-vdc-infra/0-dev/ecs`** holds the Terraform root module that provisions the ECS instances being monitored.
Key files:
 
| Artefact                        | Path                                     |
| ------------------------------- | ---------------------------------------- |
| ECS overview dashboard          | `hcs-ecs-overview-dashboard.json`        |
| ECS instance detail dashboard   | `hcs-ecs-instance-detail-dashboard.json` |
| Node Exporter Full dashboard    | `node-exporter-full-dashboard.json`      |
| Grafana import payload helper   | `prepare_import_payload.py`              |
| Prometheus scrape configuration | `dev-configuration.yaml`                 |
| Recording and alerting rules    | `dev-alerting-rules.yaml`                |
| Operation Centre exporter       | `oc_exporter.py`                         |
| Exporter systemd unit           | `oc-exporter.service`                    |
| Node exporter systemd unit      | `node-exporter.service`                  |
| Alertmanager configuration      | `dev-alertmanager.yml`                   |
| Monitoring pipeline             | `0-dev-monitoring.yml`                   |
| Fleet rollout procedure         | `ECS_FLEET_RECONCILER.md`                |
| ECS Terraform root module       | `2-vdc-infra/0-dev/ecs`                  |
 
---
 
## Tasks and Implementation Steps
 
1. **Established the metric source for ECS inventory.**
   The ManageOne Operation Centre exposes a `cloud_vm` resource collection that lists every virtual machine with its flavour, host placement, availability zone, guest operating system and power state. A `discover_vms()` routine was added to the existing Operation Centre exporter to page through that collection, tolerating the several alternative field names that the API returns for the same attribute. The result is published as the `oc_vm_*` metric family. Artefact: `oc_exporter.py`.
2. **Selected `node_exporter` as the guest utilisation source.**
   The Operation Centre performance API was evaluated and rejected for ECS, because its schema for guest-level counters could not be verified and because ECS instances, unlike the managed Relational Database Service, are ordinary virtual machines reachable over SSH. Running `node_exporter` inside the guest therefore yields authoritative utilisation without a dependency on an unverified upstream contract.
3. **Automated Prometheus target discovery for the fleet.**
   A static target list would fall out of date every time an instance was created or destroyed. The `discover_and_write_ecs_targets_json()` routine was added to the exporter so that, on each discovery cycle, it writes `/etc/prometheus/targets/ecs_node_exporter.json` in Prometheus file-based service discovery format. Each entry is `<ip>:9100` and carries the `vm_id`, `vm_name`, `host_name`, `cluster`, `azone`, `flavor` and `os_type` labels. Instances without a resolvable address are skipped, and the file is written to a temporary path and moved into place so that Prometheus never observes a partially written file. Artefact: `oc_exporter.py`.
4. **Added the ECS scrape jobs to Prometheus.**
   The `oc-exporter` job scrapes the collector on `localhost:9199`. The `ecs-node-exporter` job consumes the generated file-based service discovery list with a 60 second refresh, and additionally pins a static `localhost:9100` target for the monitoring bastion itself so that the host running the stack is never invisible to the stack. Artefact: `dev-configuration.yaml`.
5. **Built the fleet overview dashboard.**
   A dashboard with unique identifier `ecs-overview-001` was authored to answer the fleet-level questions: how many instances exist, how many are running, how much vCPU and memory are allocated, how the estate is distributed across clusters and guest operating systems, and whether the collector itself is healthy. Cluster, availability zone and guest operating system template variables allow the whole dashboard to be filtered. Artefact: `hcs-ecs-overview-dashboard.json`.
6. **Built the per-instance detail dashboard.**
   A dashboard with unique identifier `ecs-detail-001` was authored to present a single instance: its power state, allocated vCPU, memory and root volume, its uptime derived from the launch timestamp, its full metadata, its allocation history, and gauges for guest processor, memory and root filesystem utilisation. Artefact: `hcs-ecs-instance-detail-dashboard.json`.
7. **Imported and cross-linked the community Node Exporter Full dashboard.**
   Rather than reimplement more than one hundred operating system panels, the Grafana Labs Node Exporter Full dashboard, identifier 1860 and unique identifier `rYdddlPWk`, was downloaded from the Grafana catalogue API, committed to the repository, and cross-linked with the instance detail dashboard in both directions. Artefact: `node-exporter-full-dashboard.json`.
8. **Extracted the Grafana import payload logic into a committed script.**
   The Grafana dashboard API requires the raw dashboard object to be wrapped in an envelope with `overwrite` and an optional folder identifier, and rejects payloads that retain a numeric `id`. That transformation was implemented as a standalone script rather than as inline Python inside the pipeline YAML, because indented here-documents inside YAML introduce leading whitespace into the script body. Artefact: `prepare_import_payload.py`.
9. **Added the dashboards to the pipeline validate and deploy loops.**
   Dashboards are enumerated explicitly rather than discovered by glob, so each of the three ECS dashboards was registered in both the validation loop and the deployment loop. Artefact: `0-dev-monitoring.yml`.
10. **Automated the `node_exporter` installation.**
    A `deployNodeExporter` pipeline parameter was added that downloads the pinned `node_exporter` v1.8.2 release archive, creates a dedicated unprivileged `node_exporter` system user and group, installs the binary, and applies the hardened systemd unit. Artefacts: `0-dev-monitoring.yml` and `node-exporter.service`.
11. **Defined a controlled fleet rollout procedure.**
    Enabling `node_exporter` across the fleet in a single step was rejected as too high risk. A staged procedure was documented that proves the agent on the bastion first, verifies target file freshness, confirms live scraping, and only then widens the rollout. Artefact: `ECS_FLEET_RECONCILER.md`.
---
 
## Core Implementation Breakdown
 
### Data flow
 
```text
Huawei ManageOne Operation Centre
   │  HTTPS, CAS single sign-on session cookie
   ▼
oc_exporter.py  (systemd, port 9199)
   ├── publishes  oc_vm_info, oc_vm_vcpu, oc_vm_memory_mb, oc_vm_disk_gb,
   │              oc_vm_running, oc_vm_launched_at_timestamp
   └── writes     /etc/prometheus/targets/ecs_node_exporter.json  (file_sd)
                             │
ECS instance guest OS        │ 60 s refresh
   node_exporter (port 9100) │
        │                    │
        ▼                    ▼
   Prometheus  (port 9090, scrape_interval 30 s, retention 32 d / 12 GiB)
        ├── recording rules and alerting rules
        ├──► Alertmanager (port 9093) ──► Microsoft Teams and SMTP
        └──► Grafana (port 3000)
                 ├── ecs-overview-001   Fleet overview
                 ├── ecs-detail-001     Instance detail
                 └── rYdddlPWk          Node Exporter Full
```
 
### Prometheus collection
 
Two jobs serve the ECS estate. Both use the global 30 second scrape interval and a 10 second scrape timeout.
 
| Job                 | Target source                                                                                                                         | Purpose                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `oc-exporter`       | Static, `localhost:9199`                                                                                                              | Inventory, allocation and power state for every virtual machine, plus exporter health metrics |
| `ecs-node-exporter` | File-based service discovery, `/etc/prometheus/targets/ecs_node_exporter.json`, plus a static `localhost:9100` target for the bastion | Guest processor, memory, filesystem, disk and network counters                                |
 
The correlation between the two sources is the `vm_name` label. The exporter emits it on every `oc_vm_*` series, and the generated service discovery file attaches it to every `node_exporter` target, so a single dashboard variable selects both the allocation panels and the utilisation panels for the same instance.
 
### Grafana dashboards
 
**Fleet overview, unique identifier `ecs-overview-001`.** Template variables `cluster`, `azone` and `os_type` are applied to every query using regular expression matching, so an operator can scope the entire view to a cluster or a guest operating system family. The counter row uses eight equally sized stat cards. Representative queries:
 
```promql
count(oc_vm_info{cluster=~"$cluster", azone=~"$azone", os_type=~"$os_type"})
sum(oc_vm_vcpu{cluster=~"$cluster", azone=~"$azone", os_type=~"$os_type"})
sum(oc_vm_memory_mb{cluster=~"$cluster", azone=~"$azone", os_type=~"$os_type"}) / 1024
count(count by (host_id) (oc_vm_info{cluster=~"$cluster", azone=~"$azone", os_type=~"$os_type"}))
time() - oc_infra_scrape_last_success_timestamp_seconds
```
 
The instance inventory table carries a data link to `/d/ecs-detail-001/ecs-instance-detail`, which passes the selected row through as the `vm_name` variable.
 
**Instance detail, unique identifier `ecs-detail-001`.** A single `vm_name` variable drives the whole dashboard. Status is rendered as a mapped stat card where zero is DOWN in red and one is UP in green. Uptime is computed as `time() - oc_vm_launched_at_timestamp`. Allocation panels read the `oc_vm_*` family, while the guest utilisation gauges read `node_exporter` counters:
 
```promql
100 - (avg by (vm_name) (rate(node_cpu_seconds_total{vm_name="$vm_name", mode="idle"}[5m])) * 100)
(1 - (node_memory_MemAvailable_bytes{vm_name="$vm_name"} / node_memory_MemTotal_bytes{vm_name="$vm_name"})) * 100
(1 - (node_filesystem_avail_bytes{vm_name="$vm_name", mountpoint="/"} / node_filesystem_size_bytes{vm_name="$vm_name", mountpoint="/"})) * 100
```
 
Two navigation links are present: one returns to the overview, and one opens the Node Exporter Full dashboard pre-filtered with `var-job=ecs-node-exporter`.
 
**Node Exporter Full, unique identifier `rYdddlPWk`.** The unmodified community dashboard provides pressure stall information, virtual memory statistics, per-device disk saturation and time synchronisation panels that would be uneconomic to reimplement.
 
### Deployment pipeline
 
The pipeline in `0-dev-monitoring.yml` triggers on pushes and pull requests targeting `main`, restricted to changes under `5-vdc-shared/0-dev/**` or to the pipeline file itself. It runs on the self-hosted pool `<agent-pool-name>` with a demand that pins execution to the agent named `<bastion-host>`, which is the monitoring bastion. Five boolean parameters, all defaulting to `true`, allow each component to be deployed independently: `deployPrometheus`, `deployAlertmanager`, `deployGrafana`, `deployOCExporter` and `deployNodeExporter`.
 
The validate stage runs on every push and every pull request:
 
- `promtool check config` and `promtool check rules` against a staged directory layout that mirrors the deployed layout, so that `rule_files` references resolve exactly as they will in production.
- A JSON parse of every dashboard file, which catches truncated or malformed dashboard commits before they reach Grafana.
- `python3 -m py_compile` against the exporter source.
The deploy stage is skipped for pull requests and runs only on merges into `main`:
 
- Alertmanager configuration and templates are rendered with `envsubst` to inject the Teams webhook and SMTP settings, then copied into `/etc/alertmanager`.
- Prometheus configuration and rules are copied to `/etc/prometheus` and the service is restarted.
- Each dashboard is passed through `prepare_import_payload.py` and posted to `http://localhost:3000/api/dashboards/db` with `overwrite` set to true.
- The exporter secrets file, source and systemd unit are installed, the service is restarted, and port 9199 is probed for reachability.
- The pinned `node_exporter` binary and unit are installed and the service is restarted.
Representative commands:
 
```bash
# Validate the Prometheus configuration and rules exactly as the pipeline does
promtool check config ./staged/prometheus.yml
promtool check rules ./staged/alerting-rules.yaml
 
# Validate every dashboard parses as JSON
for f in 5-vdc-shared/0-dev/grafana/*.json; do python3 -c "import json,sys; json.load(open(sys.argv[1]))" "$f"; done
 
# Build and post a single dashboard import payload
python3 5-vdc-shared/0-dev/grafana/prepare_import_payload.py \
  5-vdc-shared/0-dev/grafana/hcs-ecs-overview-dashboard.json /tmp/payload.json
curl -sS -u "admin:${GRAFANA_ADMIN_PASSWORD}" \
  -H 'Content-Type: application/json' \
  -X POST http://localhost:3000/api/dashboards/db -d @/tmp/payload.json
```
 
---
 
## IAM Role and Permissions
 
### Identity model
 
Three dedicated service accounts are used, one per pipeline responsibility, so that a compromise or misconfiguration in one delivery path cannot be used to act in another.
 
| Service account        | Consumer                                          | Credentials                                                   | Purpose                                                                                            |
| ----------------------- | -------------------------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `svc-hcs-za-mon-dev`   | Monitoring pipeline and the exporter systemd unit | `MON_OC_PASSWORD`, `MON_HCS_ACCESS_KEY`, `MON_HCS_SECRET_KEY` | Operation Centre single sign-on for read-only resource discovery, and Huawei Cloud Stack API reads |
| `svc-hcs-za-infra-dev` | Infrastructure pipeline                           | `INFRA_HCS_ACCESS_KEY`, `INFRA_HCS_SECRET_KEY`                | Terraform provider authentication and Object Storage Service state access                          |
| `svc-hcs-za-rbac-dev`  | Role-based access control pipeline                | `RBAC_HCS_ACCESS_KEY`, `RBAC_HCS_SECRET_KEY`                  | Provisioning of roles, groups and policy as code                                                   |
 
The monitoring account is a member of the `OC-MONITORING-ADMIN` group in the Operation Centre console. Its access is confined to reading resource inventory and performance views. It does not create, modify or delete any compute, storage or network resource, and it is not used by any Terraform stack.
 
### Permissions required, and why
 
- **Operation Centre resource read.** Required to enumerate the `cloud_vm` collection that produces the ECS inventory and the `node_exporter` target list. Without it there is no fleet discovery at all.
- **Huawei Cloud Stack API read.** Required for supplementary service queries performed by the same exporter process. Write scopes are not requested.
- **Grafana administrator.** Required only by the pipeline, in order to create and overwrite dashboards through the dashboard API. Human operators are provisioned as Editors rather than Administrators.
- **SSH access to the bastion.** Required by the deployment steps to place configuration files and restart services. Access is by key pair only. The ECS Terraform module enforces this with a precondition that fails the plan when `ecs_keypair_name` is unset, which structurally prevents password authentication from being introduced.
Role and group resources are declared in a separate Terraform module maintained outside this repository, which creates `hcs_vdc_user` and `hcs_vdc_group_membership` resources against an existing group looked up as a data source.
 
### Secret handling
 
- No credential, key, password or webhook URL is committed to the repository. All such values are declared as secret variables in an Azure DevOps variable group and are masked in pipeline logs.
- The SSH private key is stored as an Azure DevOps secure file, downloaded to the agent temporary directory for the duration of the run only.
- SSH host key verification is strict. Host keys are collected with `ssh-keyscan` into a dedicated known-hosts file rather than disabling verification.
- The exporter reads its credentials from `/etc/oc-exporter/secrets.env`, which is rendered on the target host at deployment time from the injected pipeline variables and referenced by the systemd unit through `EnvironmentFile`. The rendered file is never returned to the agent or the repository.
- The Grafana administrator password is supplied to the deployment step as the secret variable `GRAFANA_ADMIN_PASSWORD` and is used only for the loopback API call on the bastion itself.
---
 
## Project Features (Detailed Breakdown)
 
### 1. Automatic ECS fleet discovery
 
**Purpose.** Keep the monitored inventory correct without manual maintenance as instances are created and destroyed.
 
**Implementation.** `discover_vms()` pages through the ManageOne `cloud_vm` resource collection at fifty records per page, bounded at fifty pages. It normalises the several alternative field names the API uses for the same attribute, for example `imageOsType` against `osType` and `privateIps` against `externalRelayIpAddress`, and returns a normalised record per instance. Results are published as `oc_vm_info`, `oc_vm_vcpu`, `oc_vm_memory_mb`, `oc_vm_disk_gb`, `oc_vm_running` and `oc_vm_launched_at_timestamp`.
 
**Operational considerations.** Discovery failures are observable through `oc_infra_scrape_success` and `oc_infra_scrape_last_success_timestamp_seconds`, both of which are rendered as stat cards on the overview dashboard, so a silent collector is visible within one scrape interval. Authentication outcomes are counted by `oc_auth_success_total`, `oc_auth_failures_total` and `oc_auth_consecutive_failures`, and repeated failures trigger an exponential back-off reported by `oc_auth_backoff_seconds` in order to avoid a lockout of the service account.
 
**Artefact.** `oc_exporter.py`.
 
### 2. File-based service discovery for `node_exporter`
 
**Purpose.** Give Prometheus an always-current target list without coupling Prometheus directly to the ManageOne API.
 
**Implementation.** `discover_and_write_ecs_targets_json()` renders the discovered fleet into `/etc/prometheus/targets/ecs_node_exporter.json`, one `<ip>:9100` target per instance with the `vm_id`, `vm_name`, `host_name`, `cluster`, `azone`, `flavor` and `os_type` labels attached. The file is written to a temporary path and then moved into place atomically. Prometheus reloads it every sixty seconds.
 
**Operational considerations.** Instances without a resolvable address are omitted rather than emitted as broken targets, which keeps the `up` metric meaningful. Because the file is regenerated from live discovery, an instance that is deleted disappears from the target list on the next cycle without operator action.
 
**Artefacts.** `oc_exporter.py` and `dev-configuration.yaml`.
 
### 3. Fleet overview dashboard
 
**Purpose.** Provide a single screen that reports estate size, running state, total allocation, distribution across clusters and guest operating systems, and collector health.
 
**Implementation.** Eight stat cards of equal width form the counter row, followed by an instance inventory table and allocation breakdowns by cluster and by guest operating system. All queries are filtered by the `cluster`, `azone` and `os_type` template variables. Instant bar gauges are preferred over stacked time series for inventory panels, because the operator intent is a current-state comparison rather than a trend.
 
**Operational considerations.** The `Scrape Status` and `Data Freshness` cards mean that a stale dashboard is self-evident rather than mistaken for a healthy but idle estate.
 
**Artefact.** `hcs-ecs-overview-dashboard.json`.
 
### 4. Instance detail dashboard with allocation against utilisation
 
**Purpose.** Allow an operator to judge whether a specific instance is correctly sized, by presenting what it was given beside what it is using.
 
**Implementation.** A `vm_name` variable scopes the whole dashboard. Status is rendered as a mapped stat card where zero is DOWN in red and one is UP in green. Uptime is computed as `time() - oc_vm_launched_at_timestamp`. Guest gauges compute processor, memory and root filesystem utilisation from `node_exporter` counters.
 
**Operational considerations.** The launch timestamp gauge is published only when a real value resolves from the API. Publishing a placeholder of zero would render as an uptime of more than fifty years, so the metric is deliberately left absent and Grafana displays no data instead of a plausible but false value. The guest utilisation panels remain empty for any instance that does not yet run `node_exporter`, which is stated in the dashboard description so that an empty panel is not misread as an idle instance.
 
**Artefact.** `hcs-ecs-instance-detail-dashboard.json`.
 
### 5. Operating system drill-down
 
**Purpose.** Support deep diagnosis of a single instance without leaving Grafana.
 
**Implementation.** The Grafana Labs Node Exporter Full dashboard, catalogue identifier 1860, was retrieved through the Grafana catalogue API, committed unmodified as `node-exporter-full-dashboard.json`, and cross-linked with the instance detail dashboard in both directions. Its `job` variable defaults to `ecs-node-exporter`, and a `Back to ECS Instance Detail` link returns the operator to the instance view.
 
**Operational considerations.** The dashboard is treated as a vendored third-party artefact. It is refreshed by downloading a new revision rather than by hand editing, which keeps future upstream updates a one-command operation.
 
### 6. Automated `node_exporter` deployment
 
**Purpose.** Install the guest agent consistently and repeatably rather than by hand on each instance.
 
**Implementation.** The `deployNodeExporter` pipeline step downloads the pinned v1.8.2 release archive, creates the unprivileged `node_exporter` user and group if absent, installs the binary, and applies a systemd unit that listens on port 9100 with a textfile collector directory at `/var/lib/node_exporter/textfile_collector`. The unit applies `ProtectSystem=full`, `ProtectHome=true` and `PrivateTmp=true`, restarts on failure, and is bounded by `MemoryHigh=128M` and `MemoryMax=256M`.
 
**Operational considerations.** Copying a new binary directly over a running one fails with `Text file busy`, so the deployment copies to `node_exporter.new` and renames it into place before restarting the service. Memory limits ensure that a misbehaving agent cannot destabilise the host it is meant to observe.
 
**Artefacts.** `node-exporter.service` and `0-dev-monitoring.yml`.
 
### 7. Validated, idempotent dashboard delivery
 
**Purpose.** Guarantee that the dashboards running in Grafana are exactly the dashboards committed to the repository.
 
**Implementation.** Every dashboard is parsed during the validate stage, then wrapped by `prepare_import_payload.py` and posted with `overwrite` set to true during the deploy stage. The helper removes the numeric `id` field, which the Grafana API rejects on import, and preserves folder placement through an optional folder identifier argument.
 
**Operational considerations.** Because delivery is an overwrite, any change made directly in the Grafana user interface is lost at the next deployment. This is intentional and is the mechanism that prevents drift. Dashboards are enumerated explicitly in both pipeline loops, so a new dashboard file must be registered in both places to be validated and deployed.
 
**Artefacts.** `prepare_import_payload.py` and `0-dev-monitoring.yml`.
 
### 8. Alert routing to Microsoft Teams and email
 
**Purpose.** Deliver actionable notifications to the channel the on-call team already uses.
 
**Implementation.** Alertmanager groups by `alertname` and `datname`, waits 30 seconds before the first notification for a group, sends updates at 5 minute intervals and repeats at 4 hours. A sub-route reduces low-urgency alerts labelled `notify_profile: warn_24h` to a 24 hour repeat interval. The default receiver posts to a Microsoft Teams incoming webhook and sends email through the configured SMTP smart host, both rendering through shared templates.
 
**Operational considerations.** Resolved notifications are enabled, so a channel that received an alert also receives the recovery. The webhook receiver sets `insecure_skip_verify` because the private cloud presents an internally signed certificate chain; this is a deliberate and documented trade-off, not an oversight.
 
**Artefact.** `dev-alertmanager.yml`.
 
---
 
## Design Decisions and Highlights
 
**In-guest `node_exporter` was chosen over the ManageOne performance API for ECS utilisation.** The alternative was to poll the Operation Centre monitor view, which is the approach used for the Relational Database Service. That approach was rejected for ECS because its guest-level counter schema could not be verified, and because it is unnecessary: ECS instances are ordinary virtual machines the team provisions and can reach over SSH, whereas the Relational Database Service is a managed service with no shell access. The trade-off accepted is an agent to deploy and maintain on every instance, in exchange for authoritative metrics and independence from an unverified upstream contract.
 
**Target discovery was placed inside the existing exporter rather than in a separate process.** A standalone discovery agent would have required its own Operation Centre login. Multiple independent logins originating from the same bastion address risk triggering a CAPTCHA challenge and locking the shared service account. Reusing the already-authenticated session inside the exporter removes that failure mode entirely. The trade-off is that target discovery is coupled to exporter availability, which is acceptable because a failed exporter is itself an alerting condition.
 
**File-based service discovery was chosen over static targets or a Prometheus service discovery integration.** Static targets go stale with every fleet change and there is no supported native Prometheus discovery mechanism for this Huawei Cloud Stack version. File-based discovery is simple, auditable on disk, and requires no Prometheus restart when targets change.
 
**The community Node Exporter Full dashboard was vendored rather than reimplemented.** Reproducing more than one hundred operating system panels would consume significant effort for no differentiated value. Committing the upstream artefact keeps the repository the source of truth while allowing the dashboard to be refreshed from upstream in a single command. Critically, the dashboard was retrieved through the Grafana catalogue download API rather than copied from rendered page text, because rendered text elides content and silently corrupts the JSON.
 
**Dashboard files are enumerated explicitly in the pipeline rather than discovered by glob.** Explicit enumeration means that adding a dashboard is a deliberate, reviewable act, and that an unfinished dashboard committed for discussion is not deployed by accident. The cost is that a new dashboard must be registered in two loops, which is documented in this file and in the pipeline comments.
 
**Grafana payload preparation lives in a committed Python file rather than inline in the pipeline YAML.** Multi-line Python embedded in an indented YAML block acquires the surrounding indentation as literal leading whitespace and fails with an `IndentationError` on the first line. Extracting the logic to a file makes it lintable, testable and reusable.
 
**Systemd units are hardened by default.** Both units apply filesystem and privilege restrictions and declare explicit memory ceilings. Observability components must not become the cause of the incident they exist to detect.
 
**Deployment is an overwrite, and the repository is the source of truth.** Making the pipeline authoritative over live Grafana state guarantees reproducibility and makes every visual change traceable to a commit and a reviewer.
 
**Provider and tool versions are pinned.** The Terraform module for the ECS estate constrains the `huaweicloud/hcs` provider to `~> 2.4.0`, `node_exporter` is pinned to v1.8.2, and `promtool` v2.53.0 is used for validation. Pinning removes the class of failure where an unannounced upstream release changes behaviour between two otherwise identical pipeline runs.
 
**A metric is omitted rather than defaulted when a value cannot be resolved.** Setting a gauge to zero when a field is genuinely missing produces confidently wrong output, such as an uptime of more than fifty years. Leaving the series absent causes Grafana to display no data, which is an honest representation of the state of knowledge.
 
---
 
## Local Testing and Validation
 
### Prerequisites
 
- Python 3.6 or later, available on the CentOS 7 bastion.
- `promtool` v2.53.0, obtained from the Prometheus release archive.
- `jq` and `curl` for interacting with the Grafana API.
- Network access to the monitoring bastion, and an SSH key pair authorised for it.
### Validating the Prometheus configuration and rules
 
The pipeline stages the configuration so that `rule_files` references resolve exactly as they do on the target host. The same layout can be reproduced locally.
 
```bash
mkdir -p /tmp/promstage
cp 5-vdc-shared/0-dev/prometheus/dev-configuration.yaml /tmp/promstage/prometheus.yml
cp 5-vdc-shared/0-dev/prometheus/dev-alerting-rules.yaml /tmp/promstage/alerting-rules.yaml
 
promtool check config /tmp/promstage/prometheus.yml
promtool check rules  /tmp/promstage/alerting-rules.yaml
```
 
Expected outcome: `SUCCESS` for the configuration file, confirmation that both rule files were found, and a reported count of parsed rules with no errors.
 
### Validating the dashboards
 
```bash
for f in 5-vdc-shared/0-dev/grafana/*.json; do
  python3 -c "import json,sys; json.load(open(sys.argv[1])); print('OK', sys.argv[1])" "$f"
done
```
 
Expected outcome: one `OK` line per dashboard. A truncated or hand-edited dashboard fails here rather than in Grafana.
 
### Validating the exporter
 
```bash
python3 -m py_compile 5-vdc-shared/0-dev/oc-exporter/oc_exporter.py
```
 
Expected outcome: no output and a zero exit status.
 
### Verifying the running stack
 
```bash
# The exporter is publishing ECS inventory
curl -sk http://localhost:9199/metrics | grep -c '^oc_vm_info'
 
# The generated target file is present and recent
sudo jq 'length' /etc/prometheus/targets/ecs_node_exporter.json
sudo stat -c '%y %n' /etc/prometheus/targets/ecs_node_exporter.json
 
# Prometheus considers the ECS targets healthy
curl -s 'http://localhost:9090/api/v1/targets?state=active' \
  | jq -r '.data.activeTargets[] | select(.labels.job=="ecs-node-exporter") | "\(.labels.vm_name) \(.health)"'
 
# node_exporter responds on the bastion
curl -s http://localhost:9100/metrics | head -n 5
 
# The deployed dashboard matches the committed dashboard
curl -sS -u "admin:${GRAFANA_ADMIN_PASSWORD}" \
  http://localhost:3000/api/dashboards/uid/ecs-overview-001 | jq '.dashboard.version, .meta.updated'
```
 
Expected outcomes: a non-zero count of `oc_vm_info` series, a target file whose modification time is within the last few minutes, every `ecs-node-exporter` target reporting `up`, `node_exporter` metric output on port 9100, and a dashboard version number that increments after each deployment.
 
### Pull request behaviour
 
Opening a pull request runs the validate stage only. The deploy stage is conditioned on the build reason and is skipped, so no configuration reaches the bastion until the change is reviewed and merged into `main`.
 
### Fleet rollout verification
 
Widening `node_exporter` beyond the bastion follows the staged procedure in `ECS_FLEET_RECONCILER.md`: prove the agent on the bastion, confirm the freshness of the inventory cache, verify live scraping, and only then enable further instances.
 
---
 
## Errors Encountered and Resolved
 
| Symptom                                                                                            | Root cause                                                                                                                                                             | Fix                                                                                                                                                         | Preventative measure                                                                                                                                        |
| --------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Deployment of a new `node_exporter` binary failed with `Text file busy` (`ETXTBSY`)                | The destination path was the executable of a currently running process, and a direct write to a busy inode is refused by the kernel                                    | Copy to `node_exporter.new` and then `mv -f` it into place, because a rename does not require write access to the busy inode, before restarting the service | The two-step copy and rename is now the standard pattern in the deployment step                                                                             |
| The Grafana deployment step failed with `IndentationError` on the first line of the payload script | A multi-line `python3 -c` block nested inside an indented YAML and shell context acquired the surrounding indentation as literal leading whitespace in the script body | The logic was extracted into the committed file `prepare_import_payload.py` and invoked as a script                                                         | Grafana payload preparation must live in a committed `.py` file and must never be written inline in pipeline YAML                                           |
| A committed community dashboard would not parse as JSON                                            | The dashboard had been captured from rendered page text, which elides long content with an ellipsis and silently truncates the structure                               | The dashboard was re-downloaded from the Grafana catalogue download API and committed unmodified                                                            | Third-party dashboards are retrieved through the catalogue API only, and the validate stage parses every dashboard on every pull request                    |
| The instance detail dashboard reported an uptime of more than fifty years                          | The launch timestamp gauge was being set to zero when the API did not return the field, so `time() - 0` evaluated against the Unix epoch                               | The exporter now publishes the metric only when a real value resolves, and leaves the series absent otherwise                                               | A gauge is never set to a placeholder value for a field that was genuinely not found                                                                        |
| A new dashboard was committed but never appeared in Grafana                                        | Dashboards are enumerated explicitly in the pipeline and the new file had been added to the validate loop only                                                         | The file was added to the deploy loop as well                                                                                                               | Both loops are documented as a single required change, in this file and in the pipeline comments                                                            |
| The Grafana administrator account became locked during manual dashboard pushes                     | Grafana brute-force protection blocks the account after five failed authentication attempts against the dashboard API                                                  | The failed attempts were cleared and the correct credential used; automated deployment through the pipeline replaced manual pushes                          | Dashboard delivery is performed by the pipeline using the secret variable `GRAFANA_ADMIN_PASSWORD`, so manual authentication attempts are no longer routine |
 
---
 
## Conclusion
 
This work delivers a complete, code-managed observability capability for the ECS estate on Huawei Cloud Stack. It unifies two fundamentally different data sources, a proprietary cloud management API and a standard open-source guest agent, behind a single label convention, and presents them through three linked dashboards that support a natural investigative path from fleet to instance to operating system. Every artefact is validated on every pull request and deployed by a single pipeline from a single source of truth, using dedicated least-privilege service accounts and secrets that never enter the repository.
 
The most valuable lessons were architectural rather than technical. Choosing an agent over an unverified vendor API traded a small operational burden for correctness that can be defended. Reusing an existing authenticated session rather than creating a second one eliminated a lockout failure mode that would have been discovered only in production. Declining to publish a placeholder value when a field could not be resolved kept the dashboards honest, which matters more than keeping them full. Together these decisions show a preference for systems that fail visibly and recover predictably.
 
