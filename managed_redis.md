  
**Azure Managed Redis**

Platform Service Design — Requirements & Solution Intent

**DRAFT v0.1 — For Architecture Review Board**

Prepared by: Platform Engineering  |  Date: 17 July 2026

# **1\. Document Control**

| Field | Value |
| :---- | :---- |
| Title | Azure Managed Redis — Requirements & Solution Intent |
| Version | 0.1 (Draft) |
| Author | Krishnan Subramanian, Platform Engineering |
| Status | Draft — pending Architecture Review Board |
| Reviewers | ARB, Security Architecture, Network Engineering, FinOps |
| Related artifacts | Terraform module (to follow), Azure Policy set, service catalog entry |

# **2\. Executive Summary**

Platform Engineering will offer Azure Managed Redis (AMR) as a standardized, self-service caching capability. Application teams will consume it through a certified Terraform module that provisions a secure-by-default, zone-redundant Redis instance into their own subscription. The module encodes organizational guardrails — private-endpoint-only networking, Entra ID authentication, mandatory diagnostics and tagging — so that every instance is compliant on day one without per-team security review.

Azure Managed Redis is chosen over the classic Azure Cache for Redis because Microsoft has announced retirement of the classic service: **Enterprise/Enterprise Flash tiers retire 31 March 2027 and Basic/Standard/Premium tiers retire 30 September 2028**. AMR is the strategic replacement, offering better price-performance, Redis 7.4+ features, and zone redundancy by default. Building on AMR avoids a forced migration within the module's first two years of life.

This document defines the requirements (functional and non-functional), the solution intent, the Terraform module design intent, and the open decisions the ARB is asked to ratify.

# **3\. Background & Drivers**

* **Service retirement:** Azure Cache for Redis (all tiers) is on a published retirement path; new designs must not target it.

* **Inconsistent adoption:** App teams currently provision caches ad hoc with divergent security postures (public endpoints, access keys in config, no diagnostics).

* **Platform strategy:** Landing-zone model — central guardrails, decentralized ownership. Redis should follow the same certified-module pattern as our other data services.

* **Demand:** Caching, session state, distributed locking, rate limiting, and increasingly AI workloads (semantic caching, vector search) require a low-latency in-memory store.

# **4\. Scope**

## **4.1 In Scope**

* Azure Managed Redis (Microsoft.Cache/redisEnterprise resource family), single region, zone redundant.

* A reusable Terraform module published to the internal module registry, consumable by app teams in their own subscriptions.

* Secure-by-default networking (private endpoint), identity (Entra ID), encryption, diagnostics, tagging, and Azure Policy guardrails.

* Environment profiles (dev/test/prod) with right-sized defaults.

## **4.2 Out of Scope (this iteration)**

* Geo-replication / active-active multi-region. DR beyond zone redundancy is handled at application level; geo-replication is a candidate v2 feature.

* Migration tooling from existing Azure Cache for Redis instances (separate workstream; Microsoft CLI migration tooling is rolling out from Feb 2026).

* Self-hosted Redis (OSS on AKS/VMs) — explicitly rejected; see Section 6.1.

* Central shared multi-tenant cache. Model is one instance per app team, in their subscription.

# **5\. Requirements**

## **5.1 Functional Requirements**

| ID | Requirement | Priority |
| :---- | :---- | :---- |
| FR-01 | App teams can provision an AMR instance in their own subscription via the Terraform module with minimal required inputs (name, resource group, VNet/subnet for private endpoint, environment, SKU size). | Must |
| FR-02 | Module supports selection of AMR performance tiers: Balanced (B), Memory Optimized (M), Compute Optimized (X), and Flash Optimized (F), across approved memory sizes. | Must |
| FR-03 | Module deploys a private endpoint and integrates with the central private DNS zone (privatelink.redis.azure.net) via policy-driven or module-driven DNS zone group. | Must |
| FR-04 | Module enables Microsoft Entra ID authentication and supports assignment of access policies to app managed identities and Entra groups. | Must |
| FR-05 | Module configures diagnostic settings to the central Log Analytics workspace (metrics \+ audit/connection logs where supported). | Must |
| FR-06 | Module enforces mandatory tags (owner, cost-center, environment, data-classification, app-id) and applies org naming convention. | Must |
| FR-07 | Module supports optional data persistence (RDB snapshots and/or AOF) for workloads that require warm restart. | Should |
| FR-08 | Module supports clustering policy selection (OSS cluster / Enterprise cluster / non-clustered) to accommodate client-library constraints. | Should |
| FR-09 | Module supports Redis modules where approved (e.g., RediSearch, RedisJSON) as opt-in flags. | Could |
| FR-10 | Module exposes outputs required for app integration: hostname, port, resource ID, private endpoint IP, and (if keys enabled) access to keys via secure means only. | Must |
| FR-11 | Module supports customer-managed key (CMK) encryption as an opt-in for regulated workloads. | Should |
| FR-12 | Module supports maintenance window configuration where the platform exposes it. | Could |

## **5.2 Non-Functional Requirements**

| ID | Category | Requirement | Priority |
| :---- | :---- | :---- | :---- |
| NFR-01 | Availability | Production instances are highly available (multi-replica) and zone redundant by default in AZ-enabled regions; target 99.99% SLA. | Must |
| NFR-02 | Security | No public network access. Private endpoint is the only ingress path. Public access is hard-disabled in the module (no variable override). | Must |
| NFR-03 | Security | TLS 1.2 minimum enforced; non-TLS ports disabled. | Must |
| NFR-04 | Security | Entra ID authentication preferred; access keys disabled by default. Where keys are unavoidable, they are never output in plain text and rotation guidance applies. | Must |
| NFR-05 | Security | Encryption at rest with platform-managed keys by default; CMK opt-in for regulated data. | Must |
| NFR-06 | Performance | Sub-millisecond median latency for cache operations within region; SKU guidance ensures teams size for throughput (connections, ops/sec), not memory alone. | Should |
| NFR-07 | Observability | Diagnostics to central Log Analytics mandatory; baseline alert pack (CPU, memory %, connections, cache latency, availability) deployed with the instance. | Must |
| NFR-08 | Compliance | All instances pass the org Azure Policy initiative (deny public access, require PE, require diagnostics, require tags) with zero exemptions by default. | Must |
| NFR-09 | Cost | Non-prod defaults use smallest viable Balanced SKUs without HA where acceptable; module surfaces estimated cost class in documentation. | Should |
| NFR-10 | Operability | Module upgrades are non-destructive for minor versions; breaking changes follow semver major with migration notes. | Must |
| NFR-11 | Recoverability | Where persistence is enabled, recovery from node/zone failure restores data per RDB/AOF configuration; export/import supported for backup beyond persistence. | Should |
| NFR-12 | Supportability | Module documented with examples, input/output reference, and a support path (platform team owns module defects; Azure issues via org support plan). | Must |

# **6\. Solution Intent**

## **6.1 Service Selection**

**Decision: Azure Managed Redis (AMR).** Alternatives considered:

| Option | Assessment | Verdict |
| :---- | :---- | :---- |
| Azure Managed Redis | Strategic Microsoft offering; Redis Enterprise engine; zone redundant by default; Redis 7.4+; all HA/DR features on all SKUs; better price-performance than classic. | Selected |
| Azure Cache for Redis (classic) | Retiring — Enterprise tiers 31 Mar 2027, Basic/Standard/Premium 30 Sep 2028\. New builds would face forced migration. | Rejected |
| Self-hosted Redis OSS (AKS/VM) | Full control but platform team absorbs patching, HA, scaling, and on-call burden; violates managed-first principle. | Rejected |
| Redis Cloud (Redis Inc. SaaS) | Capable, but adds a vendor, marketplace billing, and network egress complexity; no org precedent. | Rejected |

## **6.2 SKU & Sizing Strategy**

AMR SKUs combine a performance tier and memory size. The module exposes an allow-listed set, with guidance:

| Tier | When to use | Module default |
| :---- | :---- | :---- |
| Balanced (B) | General caching, session state — the default for most teams. | Yes — B-series |
| Memory Optimized (M) | Large datasets, low compute demand (1:8 vCPU:memory). | Opt-in |
| Compute Optimized (X) | High throughput / ops-heavy workloads (1:2 vCPU:memory). | Opt-in |
| Flash Optimized (F) | Very large (up to \~1.5 TB) cost-sensitive datasets on NVMe flash. | Opt-in, prod only |

The module maps a simple T-shirt input (e.g., small / medium / large per environment) to concrete SKUs, so app teams do not hand-pick SKU strings. An escape-hatch variable allows explicit SKU selection for advanced teams, still validated against the allow-list.

## **6.3 Network Architecture**

* **Private endpoint only.** Public network access is disabled and not overridable. The module creates the private endpoint into a consumer-supplied subnet in the spoke VNet.

* **Private DNS.** A record registered in the central privatelink DNS zone (hub-managed). The module supports both central policy-driven DNS zone groups and explicit zone group configuration, per landing-zone standard.

* **No service endpoints, no firewall-IP allow-listing.** These weaker patterns are excluded to keep the posture uniform.

* **East-west control.** Access to the PE subnet is governed by NSGs owned by the app team within landing-zone rules.

## **6.4 Identity & Access**

* **Data plane:** Microsoft Entra ID authentication with AMR access policies. App workloads authenticate with managed identities; humans (break-glass/debug) via Entra groups. Default access policy grants are least-privilege.

* **Access keys:** disabled by default. An opt-in variable exists for legacy clients that cannot use Entra ID; enabling it requires a documented exception and keys must be consumed via Key Vault reference, never Terraform outputs in state-readable plain text.

* **Control plane:** Azure RBAC per landing-zone standard; the module performs no custom role assignments beyond access policies.

## **6.5 Encryption**

* In transit: TLS 1.2+ only; plaintext port disabled.

* At rest: platform-managed keys by default; opt-in CMK (Key Vault \+ user-assigned managed identity) for regulated data classifications.

## **6.6 Availability & Recoverability**

* **High availability:** prod instances run with replication enabled; AMR distributes nodes across availability zones by default in AZ regions — this satisfies the zone-redundancy requirement with no extra configuration.

* **Non-prod:** HA optional (off by default in dev) to reduce cost.

* **Persistence:** optional RDB (snapshot) or AOF (append-only) persistence for workloads needing warm restarts. Cache-aside patterns should treat Redis as ephemeral; persistence is not a backup substitute.

* **DR:** out of scope this iteration (per requirements). Apps must tolerate regional cache loss (rehydrate from source of truth). Geo-replication is a planned v2 module feature.

## **6.7 Observability & Operations**

* Diagnostic settings to the central Log Analytics workspace are created unconditionally by the module.

* Baseline metric alerts (CPU %, memory %, connection count, latency, availability) deployed with sensible thresholds; action group supplied by consumer.

* Dashboards/workbook published centrally by Platform Engineering; instances discoverable via tags.

* Patching/maintenance is Microsoft-managed; module exposes maintenance configuration where the API supports it.

## **6.8 Governance Guardrails (Azure Policy)**

The module makes instances compliant by construction; policy makes non-compliance impossible outside the module:

* Deny: public network access enabled on Microsoft.Cache/redisEnterprise.

* Deny/Audit: missing private endpoint; missing mandatory tags; disallowed SKUs or regions.

* DeployIfNotExists: diagnostic settings; private DNS zone group.

# **7\. Terraform Module Design Intent**

## **7.1 Consumption Model**

One versioned module (terraform-azurerm-managed-redis) in the internal registry. App teams pin a version and supply a small set of inputs; everything security-relevant is either hard-coded or validated. Reference architecture: consumer pipeline (app team) → module → resources in app subscription.

## **7.2 Provider Strategy**

The azurerm provider now offers a first-class azurerm\_managed\_redis resource (superseding azurerm\_redis\_enterprise\_cluster/database), plus access policy assignment support (v4.60+). Known gaps at time of writing: data persistence configuration and the NoCluster clustering policy are not yet exposed and require the azapi provider. 

* **Primary:** azurerm (pinned \~\> 4.x) for the cluster, database, private endpoint, DNS, diagnostics, alerts.

* **Fallback:** azapi resource/patch only for persistence and NoCluster policy until azurerm closes the gap (tracked upstream: hashicorp/terraform-provider-azurerm \#30940, \#30941). Encapsulated inside the module so consumers never see it.

* **Reference:** align structure and interfaces with Azure Verified Modules (avm-res-cache-redisenterprise) where practical, without taking a hard dependency.

## **7.3 Module Interface (Draft)**

Key inputs (abridged — full variable reference will ship with the module):

| Variable | Type | Default | Notes |
| :---- | :---- | :---- | :---- |
| name / resource\_group / location | string | — | Required. Name validated against org convention. |
| environment | string | — | dev | test | prod. Drives profile defaults below. |
| size | string | small | T-shirt size mapped to allow-listed AMR SKU. |
| sku\_override | string | null | Escape hatch; validated against allow-list. |
| subnet\_id (private endpoint) | string | — | Required. Consumer spoke subnet. |
| private\_dns\_zone\_ids | list | \[\] | Empty when central DNS policy manages zone groups. |
| high\_availability | bool | env-based | true in prod (non-overridable), false default in dev. |
| clustering\_policy | string | OSSCluster | OSSCluster | EnterpriseCluster | NoCluster. |
| access\_policy\_assignments | map | {} | Entra object IDs → policy (least-privilege default). |
| access\_keys\_enabled | bool | false | Exception-only; forces Key Vault integration. |
| persistence | object | disabled | RDB or AOF options (via azapi until native). |
| cmk | object | null | Key Vault key \+ UAMI for customer-managed keys. |
| log\_analytics\_workspace\_id | string | — | Required. Central workspace. |
| tags | map | — | Mandatory org tags validated by the module. |

Outputs: resource id, hostname, port, private endpoint IP/FQDN, principal-facing connection guidance. No secrets in outputs.

## **7.4 Engineering Standards**

* Semantic versioning; CHANGELOG; upgrade notes for majors. Provider versions pinned with pessimistic constraints.

* CI: fmt/validate, tflint, checkov/trivy policy scan, terraform test (mock \+ real plan), example deployments per profile.

* Examples directory: minimal, prod-grade, with-persistence, with-cmk.

* Ownership: Platform Engineering owns the module; contributions via inner-source PRs.

# **8\. Environment Profiles**

| Setting | Dev | Test | Prod |
| :---- | :---- | :---- | :---- |
| Default SKU | Balanced B0/B1 (smallest allow-listed) | Balanced B1/B3 | Balanced B5+ / per sizing |
| High availability (replicas) | Off (opt-in) | On | On (enforced) |
| Zone redundancy | N/A when HA off | Default with HA | Default with HA (enforced) |
| Persistence | Off | Off (opt-in) | Opt-in per workload |
| Access keys | Off | Off | Off (exception process) |
| Alerts | Minimal | Baseline | Baseline \+ consumer action group |
| Diagnostics | On | On | On |

# **9\. Cost Considerations**

* AMR bills per SKU (tier \+ memory size); HA doubles effective node count in-price. Non-prod defaults deliberately small and HA-off.

* Flash Optimized offers large capacity at lower per-GB cost for suitable workloads.

* FinOps: mandatory cost-center tag enables chargeback; module docs include an indicative cost class per T-shirt size; reserved-capacity evaluation once fleet size justifies it.

# **10\. Risks & Mitigations**

| \# | Risk | Mitigation |
| :---- | :---- | :---- |
| R1 | azurerm provider gaps (persistence, NoCluster) force azapi usage; API-version churn risk. | Encapsulate azapi in module internals; track upstream issues; swap to native resource in a minor release. |
| R2 | Entra ID data-plane auth maturity — feature availability differences across SKUs/regions. | Verify per-region during module build; keys-based exception path exists but is governed. |
| R3 | Client library incompatibility with clustering policy (e.g., libraries without cluster-mode support). | Expose clustering\_policy including NoCluster; publish client guidance matrix. |
| R4 | Teams treat cache as source of truth and are exposed on regional failure (DR out of scope). | Document ephemerality contract; persistence opt-in; geo-replication in v2 roadmap. |
| R5 | SKU allow-list too restrictive, driving shadow IT. | Escape-hatch override with validation; quarterly allow-list review. |
| R6 | Cost surprise from oversized prod SKUs. | T-shirt sizing defaults, cost class docs, FinOps tagging and review. |

# **11\. Open Decisions for the ARB**

| \# | Decision | Options / Recommendation |
| :---- | :---- | :---- |
| D1 | Ratify AMR as the org-standard Redis offering (classic Azure Cache for Redis blocked for new builds). | Recommend: approve. |
| D2 | Access-key exception process — who approves, and time-boxing of exceptions. | Recommend: security architecture approval, 12-month expiry. |
| D3 | Private DNS integration pattern — central policy-driven zone groups vs module-managed. | Recommend: follow existing landing-zone DNS standard (policy-driven). |
| D4 | Redis modules (RediSearch/RedisJSON/vector) — approved set for v1 or defer. | Recommend: defer to v1.1 pending app demand. |
| D5 | Baseline alert thresholds and whether alerts are mandatory in non-prod. | Recommend: mandatory prod, optional non-prod. |
| D6 | Region allow-list for AMR (AZ-enabled regions only?). | Recommend: primary \+ secondary org regions, both AZ-enabled. |
| D7 | CMK: opt-in (recommended) vs mandatory for specific data classifications. | Recommend: mandatory only for 'restricted' classification. |

# **12\. Next Steps**

* ARB review of this document; incorporate feedback; ratify decisions D1–D7.

* Build Terraform module skeleton (interfaces per Section 7.3) \+ CI pipeline.

* Implement environment profiles, policy set, and alert pack; validate in sandbox subscription.

* Pilot with one app team; harden docs; publish v1.0 to the internal registry.

* Plan v2: geo-replication, migration guidance from classic Azure Cache for Redis.

# **13\. References**

* Microsoft Learn — What is Azure Managed Redis: https://learn.microsoft.com/azure/redis/overview

* Azure Cache for Redis retirement FAQ: https://learn.microsoft.com/azure/azure-cache-for-redis/retirement-faq

* Terraform azurerm\_managed\_redis resource: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/managed\_redis

* Azure Verified Module (Redis Enterprise): https://registry.terraform.io/modules/Azure/avm-res-cache-redisenterprise/azurerm/latest

* Deploy AMR with AzAPI (Redis blog): https://redis.io/blog/deploy-azure-managed-redis-with-azapi-using-terraform/

* azurerm provider gaps: hashicorp/terraform-provider-azurerm issues \#30940 (NoCluster), \#30941 (persistence).
