# Unified Operational Data Store

**Platform Context and Evaluation Criteria**

*Global Payments · Enterprise Data Platform · August 2026 · For vendor and technology evaluation · v2.1*

---

## 1. Purpose and background

Global Payments is establishing a unified Operational Data Store (ODS) as a shared capability of its enterprise data platform. The ODS is the operational serving tier of the data estate: it receives business events continuously from the enterprise streaming platform (Kafka) and serves current-state data to applications and APIs with predictable, very low read latency. It complements the open data lake, which serves historical and analytical consumption; the ODS holds a rolling hot window of operational data, typically 14 to 90 days per dataset.

No central ODS capability exists today. Teams that need real-time operational data either build a store specific to their use case, wait one or more delivery increments for one to be built, or go without. The consequences are visible across the company: business capabilities with genuine real-time needs are slow to reach market, infrastructure and licensing spend is duplicated across programs, and the estate accumulates divergent one-off solutions that each carry their own operational and compliance burden.

A central platform is justified by what sits around the database rather than the database itself. Each team-built store duplicates the same expensive surround: streaming ingestion with idempotency and replay, schema evolution handling, multi-region availability engineering, PCI and GDPR controls, audit integration, and lifecycle automation. Siloed stores also fragment the data: the highest-value operational use cases read across domains, combining transaction, merchant, account, and token data in a single low-latency path, which per-team stores cannot serve. The central ODS carries these concerns once, on shared infrastructure with per-tenant isolation, and every use case inherits them. It is not a replacement for application systems of record; it is the shared, read-optimized serving tier for streamed, current-state data.

The unified ODS replaces the siloed pattern with one consistent, cost-efficient, self-service platform. The selected technology will operate as a shared enterprise service hosting many use cases and teams as isolated tenants on common infrastructure. Sensitive card data is tokenized upstream before it reaches the ODS; the store serves high-concurrency lookups by business keys such as transaction, merchant, account, and token identifiers, for use cases including real-time fraud lookup, in-flight authorization enrichment, and merchant servicing.

## 2. Guiding principles

1. **One platform, many use cases.** The ODS is a shared enterprise service with isolated tenants, not a per-project build.
2. **Real time by default.** Data arrives continuously from the enterprise event stream and is readable within seconds of publication.
3. **Latency is the product.** Predictable low-latency reads under sustained load are the defining capability, not a best case.
4. **No single point of failure.** Availability is engineered across nodes, zones, regions and, where possible, cloud providers.
5. **Self-service or it does not ship.** Provisioning, ingestion, and access are automated; a capability that requires a manual ticket is a platform failure.
6. **Open and portable.** Operational data synchronizes continuously to the open data lake (Apache Iceberg), and the platform must never hold data in a closed format without an export path.
7. **Secure and compliant by design.** PCI DSS, GDPR, and SOX obligations are met once, by the platform, rather than re-solved by every use case.
8. **Transparent cost.** Every tenant and dataset carries cost attribution; unit economics are measured and managed.

## 3. What the platform provides

Once live, the platform gives publishing and consuming teams:

- Continuous ingestion from the enterprise Kafka estate, with events readable within seconds of publication.
- Current-state operational serving: high-concurrency reads by business keys with predictable low latency.
- Isolated tenants and workloads on shared infrastructure, provisioned through self-service automation.
- Automated data lifecycle, with policy-driven hot windows of 14 to 90 days per dataset.
- Continuous synchronization of operational data to the open data lake in Apache Iceberg format.
- Built-in observability, security controls, and per-tenant cost reporting.

## 4. Evaluation criteria

Vendors and technologies will be assessed against the criteria below. Each criterion states the requirement and, where applicable, the measurable expectation. Responses should address every criterion. Reference volumetrics (daily record counts, stored volume, and growth) are provided at vendor briefing; benchmark claims must state method, load profile, record size, and working set.

### A. Performance and throughput

| ID | Criterion | Requirement |
|----|-----------|-------------|
| A1 | Read and write latency under load | Indexed point lookups served at p95 ≤ 5 ms and p99 ≤ 10 ms, measured at the database under the sustained load defined in A3, with records of roughly 1 KB and a realistic working set. Single-record writes and upserts: p99 ≤ 20 ms under the same load. The end-to-end serving budget, including network and API layers, is p99 ≤ 50 ms. Provide benchmark evidence and method, including behaviour on cache miss. |
| A2 | Ingest latency, Kafka to readable | An event published to Kafka is readable in the ODS at p95 ≤ 2 seconds and p99 ≤ 5 seconds under sustained load. Architectures with a micro-batch latency floor above these figures do not meet the requirement. |
| A3 | Throughput | Sustain 100,000 writes per second and 100,000 reads per second per workload with 3× burst headroom. Demonstrate near-linear scale-out to 10× those rates with no downtime and no re-architecture. |
| A4 | Latency stability | p99 read latency must hold while ingest runs at full rate and during background activity: compaction, retention expiry, rebalancing, backup, and scaling events. State the known scenarios in which latency degrades. |

### B. Scalability and multi-tenancy

| ID | Criterion | Requirement |
|----|-----------|-------------|
| B1 | Elastic scaling | Online horizontal and vertical scaling with zero downtime, including scale-down. Describe automated or policy-driven capacity management, and whether compute and storage scale independently. |
| B2 | Workload isolation on shared data | Multiple workloads must query the same dataset with isolated compute or enforceable resource governance, so that no workload can degrade another’s latency or I/O. Describe the mechanism (for example isolated read replicas, analytics nodes, separate query services, or workspaces) and its limits. |
| B3 | Tenant model | Many use cases hosted as isolated tenants on shared infrastructure: per-tenant security boundaries, quotas, noisy-neighbour protection, and independent lifecycle (create, scale, retire) without impact to other tenants. |

### C. Availability and resilience

| ID | Criterion | Requirement |
|----|-----------|-------------|
| C1 | Availability | Designed to achieve 99.999% availability of the read path in a multi-region configuration. State the maximum contractual SLA offered, the topology required to earn it, and historical attainment over the past 24 months. |
| C2 | Active-active model | Describe the multi-region model precisely: are concurrent writes accepted in two or more regions against the same dataset, and under what conflict-resolution model? Or are writes routed to a primary region, and with what failover semantics? Reads must be served locally in every deployed region. |
| C3 | RPO and RTO | Node or zone failure: zero data loss, no service interruption. Region failure: reads continue uninterrupted from surviving regions; the write path recovers automatically within 60 seconds; data loss is bounded by cross-region replication lag of no more than 15 seconds; steady-state replication lag within a jurisdiction pair must hold below 1 second at p99. No human intervention in the failover path. |
| C4 | Backup and recovery | Point-in-time recovery and automated backups with cross-region copies, with stated restore-time expectations. Dataset re-hydration from Kafka and the lake is covered under J3. |
| C5 | Zero-downtime operations | Rolling upgrades, patching, and maintenance with no read or write interruption. No scheduled maintenance windows for production tenants. |

### D. Cloud, deployment, and residency

| ID | Criterion | Requirement |
|----|-----------|-------------|
| D1 | Dual cloud and parity | Offered as a managed service on both AWS and Google Cloud with capability parity. Provide a feature-parity statement across AWS, Google Cloud, and Azure, identifying any capability, SLA, or service-tier differences by provider. Describe support for cross-cloud replication, or for a single logical database spanning providers. |
| D2 | Regions and residency | Multi-region operation is required in AWS us-east-1, us-east-2, eu-west-1, and eu-west-2. Equivalent Google Cloud regions are to be confirmed and will be issued as an addendum. Datasets must be pinnable to a jurisdiction: EU-resident data, including replicas and backups, remains in EU regions. |
| D3 | Private connectivity | All data-plane and administrative access over private networking (AWS PrivateLink or VPC peering; Google Cloud Private Service Connect). No public endpoints. |

### E. Data model, consistency, and access

| ID | Criterion | Requirement |
|----|-----------|-------------|
| E1 | Access patterns | Efficient lookups by multiple business keys: secondary indexes, including compound and partial indexes, and query capability beyond primary-key access. State support for further access patterns as differentiators: full-text search, aggregation queries, and SQL-style interfaces. |
| E2 | Consistency | Tunable consistency per operation or dataset, read-your-own-writes available for operational experiences, and documented consistency behaviour during failover. |
| E3 | Drivers and APIs | Official, vendor-maintained drivers for at least Java, Python, Node.js, and Go, with connection pooling and client-side failover. State the full API surface (drivers, REST or gRPC services, query gateways) and the deprecation policy for each; a minimum of 12 months’ notice is expected for any retirement. |
| E4 | Change streams | Native change data capture so downstream systems can subscribe to inserts, updates, and deletes with low latency. |

### F. Streaming integration and lake synchronization

| ID | Criterion | Requirement |
|----|-----------|-------------|
| F1 | Kafka ingestion | A vendor-supported Kafka sink (Kafka Connect or native) with Avro and schema-registry support, at-least-once delivery with idempotent upserts, per-key ordering, dead-letter queue integration, tunable low-latency batching, and documented backpressure behaviour. The connector must be covered by the vendor’s support terms. |
| F2 | Lake synchronization | A supported path from the ODS to Apache Iceberg tables with no more than 5 minutes of lag: native Iceberg materialization, or change-stream egress to Kafka feeding the lake pipeline. Schema changes must propagate without manual intervention. State the catalog integrations supported (Iceberg REST catalog, Unity Catalog, AWS Glue). |

### G. Data lifecycle

| ID | Criterion | Requirement |
|----|-----------|-------------|
| G1 | Retention | Policy-driven time-to-live per dataset across a 14-to-90-day range, fully automated with no scheduled purge jobs. Describe the expiry mechanism: whether aged data is physically partitioned or segmented so that expiry avoids active data paths, and quantify the impact of expiry and deletion on p99 read and write latency under full load (see A4). |
| G2 | Targeted deletion | Deletion of individual records to meet data-subject requests under GDPR, within 30 days of a validated request and propagated to backups per policy. |

### H. Security and compliance

| ID | Criterion | Requirement |
|----|-----------|-------------|
| H1 | PCI DSS | Current PCI DSS attestation as a Level 1 service provider for the managed offering, with the Attestation of Compliance available under NDA and a published shared-responsibility matrix. Card data reaching the ODS is tokenized upstream; the attestation is required regardless. |
| H2 | Certifications | SOC 2 Type II and ISO 27001; GDPR-aligned data-processing terms; audit and change controls that support SOX obligations. |
| H3 | Encryption | TLS 1.2 or later in transit and AES-256 at rest; customer-managed keys through AWS KMS and Google Cloud KMS, with rotation. Field-level or client-side encryption is a differentiator. |
| H4 | Identity and audit | Role-based access integrated with enterprise identity (SAML/OIDC single sign-on, SCIM, and cloud IAM federation for workloads), with immutable audit logging of administrative and data-access events, exportable to the enterprise SIEM. |

### I. Operations and self-service

| ID | Criterion | Requirement |
|----|-----------|-------------|
| I1 | Infrastructure as code | A generally available, vendor-maintained Terraform provider and a complete administrative API; a Kubernetes operator where a self-managed option exists. |
| I2 | Provisioning speed | A new tenant or database provisioned entirely through automation, including access, ingestion wiring, and observability, in minutes and no more than 30 minutes end to end (automation scope detailed in L2). |
| I3 | Observability | Prometheus or OpenTelemetry metrics export, query-level diagnostics and slow-query insight, alerting integration, and per-tenant health visibility. |
| I4 | Support | 24×7 enterprise support with priority-1 response of 30 minutes or better, named technical account management, and professional services available for build acceleration and operational enablement. |

### J. Data architecture

| ID | Criterion | Requirement |
|----|-----------|-------------|
| J1 | Partitioning and distribution | Data must distribute predictably across nodes and regions under high-volume workloads. Describe partitioning strategy, shard allocation, hot-partition detection, skew mitigation, rebalancing behaviour, and maximum recommended partition size. Demonstrate sustained performance under uneven key distributions representative of payment and transaction-processing workloads. |
| J2 | Schema evolution | Support additive and non-breaking schema changes without service interruption. Describe schema versioning, backward and forward compatibility, deployment of schema changes, and the impact of evolving event structures across producers, consumers, and stored datasets. |
| J3 | Replay and re-hydration | Support complete or partial reconstruction of datasets from Kafka and/or Apache Iceberg. Demonstrate replay by tenant, dataset, partition, and time range. State achievable replay throughput, consistency guarantees during replay, operational procedures, and expected recovery characteristics for large-scale rebuild scenarios. |

### K. Governance and data management

| ID | Criterion | Requirement |
|----|-----------|-------------|
| K1 | End-to-end lineage | Provide lineage visibility from source systems through Kafka topics, transformation pipelines, ODS datasets, APIs, and synchronization to Apache Iceberg. Lineage must be automatically maintained and exportable for audit and compliance purposes. |
| K2 | Data ownership and stewardship | Every dataset must support assignment of business owner, technical owner, steward, classification, and lifecycle policy. Ownership information must be discoverable through platform metadata and administrative interfaces. |
| K3 | Policy enforcement | Governance policies for access control, retention, residency, classification, encryption, and lifecycle management must be centrally managed and programmatically enforced. Describe policy administration, auditability, and the mechanisms that prevent manual circumvention. |

### L. Platform operations and service management

| ID | Criterion | Requirement |
|----|-----------|-------------|
| L1 | FinOps and cost visibility | Provide cost visibility and attribution at tenant, workload, dataset, environment, and region levels. Costs must be measurable across storage, compute, networking, replication, backup, and operational services. Support forecasting, budgeting, and cost anomaly detection. |
| L2 | Tenant onboarding automation | New tenants must be provisioned entirely through automation, including infrastructure, security policies, observability, ingestion configuration, and cost reporting. State which onboarding activities require manual intervention and the expected provisioning times against the I2 target. |
| L3 | Recommended operating model | Alongside the recommended deployment architecture, propose the customer-side operating model to run the central ODS as a product: team structure, roles, skills, and FTE count to own automation, self-service, integration capabilities, compliance, upgrades, and business-as-usual operations. State the split of responsibilities between vendor and customer, and the professional-services and training path to steady state. |

### M. Commercial and vendor

| ID | Criterion | Requirement |
|----|-----------|-------------|
| M1 | Cost model | Transparent, predictable pricing with unit economics modelled at current and 10× volume, and per-tenant and per-dataset cost attribution to support chargeback. State storage unit pricing ($/GB-month) and request or capacity unit pricing, with on-demand versus committed rates. |
| M2 | References | Production references in payments or financial services at comparable scale and latency. |
| M3 | Vendor stability | Ownership and roadmap stability, disclosure of any material pending change of control, and roadmap alignment with these criteria. Capabilities still on the roadmap must be identified as such, with dates. |
| M4 | Portability and exit | Bulk data export in open formats at production scale, documented exit tooling, and contractual data-egress terms. No required capability may depend on a proprietary format without an export path. |

## 5. Response guidance

Respond to each criterion with a capability statement and supporting evidence: benchmark results with method, SLA documents, compliance attestations, and reference architectures. Distinguish clearly between what is generally available today and what is on the roadmap. Responses feed a weighted evaluation; shortlisted platforms proceed to a proof of concept against the Section A criteria and the demonstration items in Section J before a technology decision targeted for October 2026.
