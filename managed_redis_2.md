# Azure Managed Redis

Platform Service Design - Requirements and Solution Intent

**DRAFT v3 - For Architecture Review Board**

## 1. Document Control

| Field | Value |
| --- | --- |
| Title | Azure Managed Redis - Requirements and Solution Intent |
| Version | 3 (Draft) |
| Author | Krishnan Subramanian, Platform Engineering |
| Status | Draft - pending Architecture Review Board |
| Reviewers | ARB, Security Architecture, Network Engineering, FinOps |
| Related artifacts | Terraform module, Azure Policy initiative, service catalog entry, operational runbooks |

## 2. Executive Summary

Platform Engineering will offer Azure Managed Redis (AMR) as a standardized, self-service caching and temporary-data capability. Application teams will consume the service through a certified Terraform module that deploys an AMR instance into the application subscription.

The platform baseline is:

- Public network access disabled without an override.
- Private Endpoint connectivity.
- Central Private DNS integration with a single declared owner.
- TLS-encrypted client connections.
- Microsoft Entra authentication using managed identities or service principals.
- Access-key authentication disabled unless an approved exception exists.
- High availability and zone redundancy for production.
- Mandatory metrics, connection auditing, alerts, and tags.
- Explicit workload choices for SKU, clustering, eviction, persistence, and Redis modules.

Azure Managed Redis is selected over Azure Cache for Redis because the classic service is on a published retirement path. Enterprise and Enterprise Flash tiers retire on 31 March 2027. Basic, Standard, and Premium tiers retire on 30 September 2028.

The service is not a general-purpose system of record. Every consuming team must identify an authoritative data source or explicitly accept data loss. Persistence improves recovery of the same AMR instance but is not a backup or point-in-time recovery mechanism.

## 3. Verified Platform Constraints

The following constraints are part of the platform contract and must be reflected in module validation and documentation:

1. Azure Managed Redis supports one logical database per instance.
2. Microsoft Entra groups are not supported for Redis data-plane authentication. Assign individual users, service principals, or managed identities. An object ID is an opaque GUID, so Terraform can reject a declared `Group` principal type but cannot prove the directory object's actual type without a Microsoft Graph lookup.
3. A standard access-policy assignment grants broad access to commands and keys. Fine-grained Redis ACL access strings are a preview capability and require separate approval.
4. Entra-authenticated clients must acquire, refresh, and reapply tokens to long-lived Redis connections.
5. High availability is required for the availability SLA and for data persistence.
6. Zone redundancy is automatic only when HA is enabled and the selected region supports availability zones for AMR.
7. RDB and AOF persistence are mutually exclusive.
8. Persistence cannot be combined with active geo-replication.
9. Persistence files are service-managed and restore only to the same cache. Portable recovery uses export/import.
10. `OSSCluster` clients connect initially on port `10000` and then connect to service-discovered shard ports in the `85xx` range. These shard ports can change and must not be hardcoded.
11. `NoCluster` is supported only for caches of 25 GB or less and can restrict later scaling choices.
12. RediSearch requires `EnterpriseCluster` and `NoEviction`.
13. Redis modules and clustering-policy changes can recreate the database and lose cached data.
14. Approximately 20 percent of provisioned memory is reserved for service operations and is not usable for application keys.
15. CMK protects data written to disk. AMR does not service-encrypt data held in memory.
16. If access keys are enabled, the AzureRM provider records them as sensitive attributes in Terraform state even when the module does not output them.

## 4. Scope

### 4.1 In Scope

- Azure Managed Redis using the `Microsoft.Cache/redisEnterprise` resource family.
- Single-region instances with optional HA and automatic zone distribution where supported.
- A reusable Terraform module published to the internal module registry.
- Private Endpoint creation and one selected Private DNS integration mode.
- Microsoft Entra data-plane access assignments for managed identities, service principals, and named break-glass users.
- Access-key exception support with restricted Terraform-state handling.
- SKU, clustering, eviction, persistence, Redis module, and CMK configuration with compatibility validation.
- Diagnostic settings, baseline alerts, tagging, documentation, and runbooks.
- Environment profiles for development, test, and production.

### 4.2 Out of Scope for Initial Production Release

- Active geo-replication and multi-region failover.
- Automated migration from Azure Cache for Redis.
- Shared multi-tenant caches across unrelated application teams.
- Self-hosted Redis on AKS or virtual machines.
- Fine-grained custom Redis ACLs unless the ARB separately approves the preview API.
- Scheduled maintenance configuration unless the ARB separately approves the preview feature and its implementation path.
- A centrally managed recurring export service. The backup requirement and ownership are an open decision.

## 5. Requirements

### 5.1 Functional Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-01 | App teams can provision AMR using name, resource group, approved region, environment, workload profile, and Private Endpoint subnet. | Must |
| FR-02 | The module supports an allow-listed set of Balanced, Memory Optimized, Compute Optimized, and Flash Optimized SKUs. Preview SKUs are excluded unless explicitly approved. | Must |
| FR-03 | The module creates a Private Endpoint and supports exactly one DNS mode: central policy-managed zone group or module-managed zone group. | Must |
| FR-04 | Public network access is always disabled by the module and has no consumer override. | Must |
| FR-05 | The module creates Entra access-policy assignments for managed identities, service principals, and individual users. Declared group principals are rejected. Actual object-type verification requires the optional Microsoft Graph validation mode. | Must |
| FR-06 | Access-key authentication is disabled by default and enabled only through an approved exception input. | Must |
| FR-07 | The module exposes clustering policy with validation for `OSSCluster`, `EnterpriseCluster`, and `NoCluster`. | Must |
| FR-08 | The module exposes eviction policy and supplies documented workload defaults. | Must |
| FR-09 | The module supports either RDB or AOF persistence when HA is enabled. Invalid combinations fail during Terraform validation. | Should |
| FR-10 | The module supports approved Redis modules and validates SKU, clustering, eviction, and geo-replication compatibility. | Could |
| FR-11 | The module supports CMK disk encryption using a Key Vault key and user-assigned managed identity. | Should |
| FR-12 | The module creates diagnostic settings for supported metrics and connection/audit events and sends them to the central Log Analytics workspace. | Must |
| FR-13 | The module creates baseline metric and activity-log alerts with an application-supplied action group. | Must |
| FR-14 | The module applies mandatory tags and naming rules to all resources that support tags. | Must |
| FR-15 | Outputs include resource ID, hostname, port, database ID, Private Endpoint ID, and connection guidance. Outputs never include access keys or connection strings containing secrets. | Must |
| FR-16 | The module documents required client behaviour for TLS, Entra token refresh, cluster discovery, reconnects, retries, and failover. | Must |

### 5.2 Non-Functional Requirements

| ID | Category | Requirement | Priority |
| --- | --- | --- | --- |
| NFR-01 | Availability | Production uses HA in an AMR zone-capable region and is designed to qualify for the 99.99 percent connectivity SLA. | Must |
| NFR-02 | Security | Private Endpoint is the only ingress path. Public access is hard-disabled. | Must |
| NFR-03 | Security | Client protocol is `Encrypted`; TLS 1.2 or later is required. | Must |
| NFR-04 | Identity | Entra authentication is the default. App identities are assigned per instance and are not shared between applications. | Must |
| NFR-05 | Secrets | If access keys are exceptionally enabled, Terraform state is treated as a secret store with encryption, least-privilege access, audit logging, and a dedicated restricted workspace where required. | Must |
| NFR-06 | Encryption | Microsoft-managed disk encryption is the default. CMK is available for approved classifications and does not imply in-memory encryption. | Must |
| NFR-07 | Performance | Each production profile is performance-tested using representative payload, concurrency, clustering, TLS, and client topology. No universal sub-millisecond guarantee is made. | Must |
| NFR-08 | Capacity | Sizing accounts for approximately 20 percent reserved memory, expected key growth, TTLs, connection limits, throughput, and failover headroom. | Must |
| NFR-09 | Observability | Metrics, connection events, Resource Health/activity events, and a synthetic connectivity check form the monitoring baseline. | Must |
| NFR-10 | Compliance | Azure Policy denies public access and disallowed regions/SKUs, enforces tags, and audits or remediates missing diagnostics and Private Endpoint integration. | Must |
| NFR-11 | Lifecycle | The module documents every input that can replace the instance or recreate the database. Production pipelines require explicit approval for destructive plans. | Must |
| NFR-12 | Recoverability | Each workload declares RPO, RTO, authoritative data source, and cache rehydration method. Persistence and export are selected against those targets. | Must |
| NFR-13 | Supportability | Examples, client matrix, troubleshooting flow, ownership model, and escalation path ship with the module. | Must |

## 6. Solution Intent

### 6.1 Service Selection

**Decision: Azure Managed Redis.**

| Option | Assessment | Verdict |
| --- | --- | --- |
| Azure Managed Redis | Strategic Azure service using Redis Enterprise, current Redis capabilities, Private Link, Entra authentication, HA, persistence, and modules. | Selected |
| Azure Cache for Redis | Published retirement path creates a forced near-term migration for new deployments. | Rejected for new builds |
| Self-hosted Redis | Platform team would own patching, replication, failover, scaling, and on-call operation. | Rejected |
| Redis Cloud | Capable service but introduces a separate commercial, governance, and networking model. | Not selected |

Price-performance statements remain hypotheses until verified with an application-representative benchmark.

### 6.2 SKU and Sizing Strategy

The module maps workload profiles to tested SKU allow-lists. An explicit SKU override is available only within the allow-list.

| Tier | Suitable workload | Important constraints |
| --- | --- | --- |
| Balanced | General caching and session state | Default starting family |
| Memory Optimized | Large memory footprint with moderate compute needs | Lower compute-to-memory ratio |
| Compute Optimized | High operation rate or CPU-heavy commands | Higher cost for compute |
| Flash Optimized | Large, read-heavy hot/cold datasets | Cold reads have higher latency; limited module and DR support |

Sizing uses usable memory rather than nominal memory. The capacity worksheet must include:

- Peak and projected key/value memory.
- Approximately 20 percent service reservation.
- Fragmentation and failover headroom.
- Key count and key-name size.
- Connection count and connection-creation rate.
- Operations per second and bandwidth.
- TTL and eviction behaviour.
- Persistence overhead, especially AOF.

### 6.3 Compatibility Rules

The module must reject invalid or operationally unsafe combinations before deployment.

| Feature | Required validation |
| --- | --- |
| Persistence | HA enabled; exactly one of RDB or AOF; no active geo-replication |
| NoCluster | Cache size no greater than 25 GB; consumer acknowledges restricted scaling |
| RediSearch | `EnterpriseCluster`; `NoEviction`; unsupported on Flash Optimized |
| RedisBloom / RedisTimeSeries | Unsupported on Flash Optimized |
| RedisJSON | Validate against selected tier and any other enabled modules |
| Flash Optimized | Reject unsupported modules, non-clustered mode, and active geo-replication |
| Preview SKU or feature | Explicit platform allow-list and ARB approval |

### 6.4 Network Architecture

The data path is:

```text
Application
-> application VNet or hybrid network
-> Private DNS resolution
-> Private Endpoint private IP
-> Azure Managed Redis
```

Controls:

- Public network access is `Disabled` and cannot be overridden.
- The module creates the Private Endpoint in a consumer-supplied subnet.
- The application connects using `<cache-name>.<region>.redis.azure.net`, not a raw IP or the `privatelink` hostname.
- The central zone is `privatelink.redis.azure.net`.
- `private_dns_integration_mode` is either `PolicyManaged` or `ModuleManaged`, never both.
- Module-managed DNS requires permission to attach the central cross-subscription zone.
- Policy-managed DNS requires a managed identity with the necessary cross-subscription role and an operational remediation SLA.
- On-premises clients require VPN or ExpressRoute routing plus conditional DNS forwarding to Azure DNS Private Resolver.
- `OSSCluster` firewall rules must allow the initial endpoint port and the required service-discovered `85xx` shard-port range. Individual shard ports can change; do not pin them. Cluster-aware clients must use Redis cluster discovery.
- If NSGs or UDRs are expected to govern Private Endpoint traffic, Private Endpoint network policies must be enabled on the subnet by its owner.

Azure Policy denies public access. A missing Private Endpoint is audited or remediated after resource creation because the endpoint cannot reference an AMR resource that does not yet exist.

### 6.5 Identity and Access

Data-plane access is separate from Azure control-plane RBAC.

Supported data-plane identities:

- Application managed identity.
- Application service principal where managed identity is unavailable.
- Individually assigned, time-bounded break-glass user.

Unsupported:

- Entra group assignment to the Redis data plane.
- Shared application identities across unrelated workloads.

For the initial production release, each assigned identity receives broad access to the single application-owned Redis database. Fine-grained command and key-pattern restrictions require the custom ACL preview API and are subject to a separate ARB decision.

Each access assignment includes an object ID and a declared principal type. The module rejects a declared `Group` during plan. In the default validation mode, the module cannot independently prove the type represented by an opaque object ID; Azure remains the final enforcement point and rejects unsupported group assignments during deployment. An optional strict mode uses the `azuread_directory_object` data source to resolve the actual type during plan when the object ID is already known. Terraform can defer that lookup until apply when the ID is produced by another resource in the same run. Strict mode adds the `hashicorp/azuread` provider and requires approved Microsoft Graph read permissions for the pipeline identity.

Client requirements:

- In Azure public cloud, acquire a token for `https://redis.azure.com/.default`; Gate 3 confirms the scope for the selected cloud and client library.
- Use the identity object ID as the Redis username and the token as the password.
- Refresh and reapply the token before expiry.
- Reauthenticate every pooled or shard connection.
- Reconnect and retry with exponential backoff after failover or maintenance.

### 6.6 Access-Key Exception

Access keys are disabled by default. Enabling them requires an approved, time-bounded exception.

The module does not output keys. This does not prevent the provider from storing computed keys in Terraform state. Therefore the exception requires:

- Encrypted remote state.
- Strict state RBAC and audit logging.
- No broad remote-state readers.
- A defined key-rotation owner and runbook.
- Key Vault delivery where the application requires a secret reference.
- A migration date for moving the application to Entra authentication.

Changing access-key authentication can terminate existing client connections. The change plan must include reconnect testing.

### 6.7 Encryption

- TLS 1.2 or later protects data in transit.
- Microsoft-managed keys encrypt OS, temporary/export, and persistence disks by default.
- CMK is an optional disk-encryption control for regulated workloads.
- Data held in Redis memory is not service-encrypted. Restricted data requires application-level encryption or exclusion from Redis.

CMK prerequisites:

- RSA key in Azure Key Vault.
- Soft delete and purge protection enabled.
- User-assigned managed identity.
- `Key Vault Crypto Service Encryption User` role, or equivalent key permissions.
- Documented key rotation, expiry, disablement, and recovery runbooks.

The platform must decide whether the module creates the CMK role assignment or requires it as a prerequisite.

### 6.8 Availability and Recoverability

Production uses HA. In supported regions, AMR distributes HA nodes across availability zones by default. This qualifies the instance for the higher connectivity SLA, subject to the Azure SLA terms.

The SLA does not guarantee that cached data cannot be lost. Clients must tolerate connection interruption and retry failed operations.

Recovery options:

| Mechanism | Purpose | Limitations |
| --- | --- | --- |
| HA | Node and zone availability | Replication is not a backup |
| RDB | Periodic same-instance recovery | Snapshot interval is not an exact RPO |
| AOF | Lower potential data loss for same-instance recovery | Higher compute/throughput impact |
| Export/import | Portable recovery copy | Requires Storage and external scheduling automation |
| Application rehydration | Rebuild cache from authoritative store | Can overload the source during a cache stampede |

Because regional DR is out of scope, every application must demonstrate that its authoritative store can sustain controlled cache rehydration. Use request coalescing, rate limiting, backoff, circuit breakers, and staged warm-up where appropriate.

### 6.9 Observability and Operations

Diagnostic settings send supported AMR metrics and connection events to the central Log Analytics workspace.

Baseline alerts include:

- Used-memory percentage and memory growth.
- Evicted keys.
- CPU and server load.
- Connected-client count and connection spikes.
- Operations per second and bandwidth.
- Latency where the metric is supported.
- Resource Health and maintenance activity events.
- Synthetic DNS, TLS, authentication, and `PING` connectivity.

An availability alert must not assume a platform metric that does not exist. The synthetic check covers the complete client path.

Because the AMR endpoint is private, the probe must run from a VNet-connected or hybrid execution environment that can resolve the private DNS path. It must exercise DNS, TCP/TLS, Entra token acquisition and refresh, authentication, and `PING`, then publish a health signal to the central monitoring service. D11 must assign the runtime, deployment scope, operational owner, identity, frequency, alert route, and cost owner before Gate 7 can pass.

Runbooks cover:

- Authentication failure and Entra token refresh.
- DNS and Private Endpoint troubleshooting.
- Failover/reconnect behaviour.
- Memory pressure and eviction.
- Scaling and replacement.
- Access-key rotation.
- CMK failure and rotation.
- Cache flush, rehydration, export, and import.
- Redis major-version compatibility, deferral, and upgrade coordination.

### 6.10 Governance Guardrails

The central Azure Policy initiative should use:

- **Deny:** public network access enabled.
- **Deny:** disallowed regions and SKUs.
- **Deny or Modify:** missing mandatory tags, following the landing-zone standard.
- **AuditIfNotExists:** missing Private Endpoint.
- **DeployIfNotExists:** diagnostic settings and policy-owned DNS zone group where selected.
- **Audit:** access-key authentication enabled outside the exception process.

Policy and module ownership must be unambiguous. A resource is managed by one mechanism, not both.

## 7. Terraform Module Design Intent

### 7.1 Consumption Model

App teams pin a module version and deploy into their application subscription. The module owns AMR, its default database, Private Endpoint, access assignments, diagnostics, and alerts. Central platform services such as Private DNS, Log Analytics, DNS Private Resolver, policy assignments, and optionally Key Vault remain externally owned dependencies. Optional strict directory-object validation adds the `hashicorp/azuread` provider and Microsoft Graph read-permission prerequisites. Synthetic probe ownership is governed by D11 and is not implicitly included merely because the module creates alert rules.

### 7.2 Provider Strategy

- **Primary:** `hashicorp/azurerm`. Provider release history indicates `>= 4.60.0, < 5.0.0` as the minimum candidate constraint for the native capabilities listed below; the supported constraint and lock-file test baseline must be confirmed by implementation acceptance tests.
- **Native support:** AMR, database, `NoCluster`, RDB/AOF persistence, CMK, modules, access-policy assignments, Private Endpoint, DNS zone group, diagnostics, and alerts.
- **AzAPI:** allowed only for an explicitly approved preview capability, such as custom ACL access strings or maintenance scheduling, and isolated behind a feature flag.
- **No unnecessary AzAPI:** persistence and `NoCluster` use AzureRM directly.

`NoCluster` and persistence are present in AzureRM `4.54.0`; the native Managed Redis access-policy assignment was introduced in `4.60.0`. The production module pins and tests its selected provider version. Dependabot/Renovate updates are tested but are not automatically merged for production modules.

### 7.3 Draft Module Interface

| Variable | Type | Default | Notes |
| --- | --- | --- | --- |
| `name`, `resource_group_name`, `location` | string | required | Region must be allow-listed. |
| `environment` | string | required | `dev`, `test`, or `prod`. |
| `workload_profile` | string | `general_cache` | Drives documented defaults, not hidden irreversible choices. |
| `size` | string | `small` | Maps to an allow-listed SKU. |
| `sku_override` | string | `null` | Must still pass the allow-list and compatibility checks. |
| `private_endpoint_subnet_id` | string | required | Consumer-owned subnet. |
| `private_dns_integration_mode` | string | required | `PolicyManaged` or `ModuleManaged`. |
| `private_dns_zone_id` | string | conditional | Required only for `ModuleManaged`. |
| `high_availability_enabled` | bool | environment-based | Enforced `true` in production. Replacement impact documented. |
| `clustering_policy` | string | `OSSCluster` | `OSSCluster`, `EnterpriseCluster`, or `NoCluster`. |
| `eviction_policy` | string | profile-based | Explicit and validated. |
| `persistence` | object | disabled | `RDB` or `AOF`; requires HA. |
| `redis_modules` | map(object) | `{}` | Validated compatibility matrix. |
| `access_policy_assignments` | map(object) | `{}` | Each entry supplies `object_id` and declared `principal_type`; `Group` is rejected. |
| `directory_object_validation_mode` | string | `DeclaredType` | `Graph` verifies the actual type when the object ID is known; lookups can defer to apply. Requires the AzureAD provider and Graph read permissions. |
| `access_keys_enabled` | bool | `false` | Exception-only and state-sensitive. |
| `cmk` | object | `null` | Key ID, UAMI ID, and declared RBAC ownership. |
| `log_analytics_workspace_id` | string | required | Central workspace. |
| `action_group_ids` | list(string) | required in prod | Alert destinations. |
| `tags` | map(string) | required | Mandatory tag keys validated. |

Outputs contain no secrets:

- AMR resource ID and hostname.
- Database ID and port.
- Private Endpoint ID and network interface ID.
- Connection guidance for TLS, Entra, and clustering.
- Diagnostic setting and alert IDs.

### 7.4 Destructive-Change Contract

| Change | Expected effect |
| --- | --- |
| Name, resource group, or region | Replace AMR instance |
| HA setting through current AzureRM resource | Replace AMR instance |
| Clustering policy | Recreate database; cached data lost; interruption expected |
| Redis module selection or arguments | Recreate database; cached data lost; interruption expected |
| Some SKU reductions or tier transitions | Unsupported, constrained, or replacement-prone |
| Enabling/disabling access-key authentication | Existing connections can be terminated |
| Azure-managed or manually initiated Redis major-version upgrade | Irreversible out-of-band service operation; client compatibility testing and upgrade coordination required |

Production controls:

- CI detects and labels replacement or database recreation.
- Production apply requires explicit approval.
- `prevent_destroy` is considered for the AMR resource, with a documented break-glass process.
- Upgrade notes describe data and availability impact.
- Example tests cover existing-resource upgrades, not only clean deployment.

The current AzureRM resource does not expose a configurable `redis_version` or upgrade-deferral field, so the module intentionally has no `redis_version` input. The Platform service owner monitors version announcements, validates clients and workloads, records the approved deferral mechanism, and coordinates manual or automatic major upgrades. Minor and patch updates remain service-managed.

### 7.5 Engineering Standards

- Semantic versioning and CHANGELOG.
- Provider lock file committed for examples/tests.
- `terraform fmt`, `validate`, `tflint`, security scanning, and Terraform tests.
- Real Azure tests for Private Endpoint DNS, Entra authentication, diagnostics, HA, persistence, and destructive-plan detection.
- Test matrix for allowed SKU and feature combinations.
- Examples: minimal dev, production cache, persistence, CMK, module-managed DNS, and policy-managed DNS.
- Module support path and operational ownership documented.

## 8. Environment Profiles

| Setting | Dev | Test | Prod |
| --- | --- | --- | --- |
| SKU | Smallest approved GA Balanced SKU | Workload-representative Balanced SKU | Performance-test result |
| HA | Off by default | On | On and enforced |
| Zone-capable region | Preferred | Required for prod-like tests | Required |
| Persistence | Off | Optional | Based on declared RPO/RTO |
| Access keys | Off | Off | Off; approved exception only |
| Public access | Off | Off | Off |
| Alerts | Minimal | Baseline | Baseline plus action group and synthetic probe |
| Diagnostics | On | On | On |
| Preview features | Off | Isolated evaluation only | Off unless ARB-approved |

Changing an environment label must not silently change HA, clustering, persistence, or modules on an existing instance. Derived defaults are resolved at initial creation and subsequent changes require explicit values and approval.

## 9. Cost Considerations

Cost estimates include more than the AMR SKU:

- AMR node count and HA.
- Private Endpoint.
- Log Analytics ingestion and retention.
- Synthetic monitoring runtime, network integration, identity, and telemetry ingestion.
- Key Vault, key operations, and optional CMK rotation.
- Export Storage and automation if backup is enabled.
- Shared DNS Private Resolver and hybrid connectivity allocation.
- Temporary parallel environments required for replacement or migration.

Mandatory cost-center and application tags support chargeback. Reservations are evaluated after the stable production fleet and region/tier demand are known.

## 10. Risks and Mitigations

| ID | Risk | Mitigation |
| --- | --- | --- |
| R1 | Custom least-privilege ACLs are preview and absent from the stable AzureRM assignment resource. | Use instance-per-app isolation and full per-instance access in the initial production release; require ARB approval for preview ACLs. |
| R2 | Client libraries fail to refresh Entra tokens or reauthenticate shard connections. | Publish a tested client matrix and integration tests for long-lived connections. |
| R3 | Access keys appear in Terraform state. | Default keys off; exception-only restricted state; audit and rotation controls. |
| R4 | Clustering or module changes recreate the database. | Destructive-plan detection, approval gates, migration procedure, and cache rehydration. |
| R5 | DNS policy and Terraform both manage a zone group. | Mandatory mutually exclusive DNS ownership mode. |
| R6 | Hybrid/on-premises clients resolve public DNS or cannot route to the PE. | DNS Private Resolver, conditional forwarding, route/firewall tests, and runbook. |
| R7 | Invalid SKU/feature combinations fail late or enter preview unexpectedly. | Versioned compatibility matrix and Terraform validations. |
| R8 | Cache loss overloads the authoritative database. | Rehydration load test, request coalescing, backoff, rate limits, and staged warm-up. |
| R9 | Persistence is mistaken for backup. | RPO/RTO declaration, export option, and restore testing. |
| R10 | CMK is disabled, deleted, or inaccessible. | Purge protection, least-privilege role, monitoring, rotation, and recovery runbook. |
| R11 | Performance target is assumed from marketing guidance. | Representative benchmark and measurable workload SLOs. |
| R12 | Correctness-critical distributed locking is implemented without failure analysis. | Require architecture review for locks that protect financial, safety, or consistency invariants. |
| R13 | A principal object ID is mislabeled and its actual directory type is not checked during plan. | Offer strict Graph-backed validation; otherwise document that Azure service enforcement can fail the deployment at apply. |

## 11. ARB Decisions

| ID | Decision | Recommendation |
| --- | --- | --- |
| D1 | Ratify AMR as the standard Redis offering for new builds. | Approve. |
| D2 | Access-key exception owner and expiry. | Security Architecture approval; maximum 12 months; migration plan required. |
| D3 | Private DNS zone-group ownership. | Use landing-zone policy by default; module-managed only where policy is unavailable. Never both. |
| D4 | Fine-grained custom ACL preview in the initial production release. | Defer; use instance-per-app isolation and stable full-access assignments. |
| D5 | Redis modules in the initial production release. | Allow only demonstrated demand with compatibility validation; otherwise defer to a later release. |
| D6 | Production region allow-list. | Approve only regions validated for AMR zone redundancy and required SKUs. |
| D7 | CMK ownership. | CMK mandatory only for approved restricted classifications; Security owns Key Vault policy, Platform documents RBAC boundary. |
| D8 | Scheduled maintenance preview. | Defer until GA unless a pilot requirement justifies preview. |
| D9 | Portable backup/export ownership. | Decide whether Platform supplies recurring export automation or consumers own it. |
| D10 | Access-key Terraform state isolation. | Require restricted state workspace and audited access for every exception deployment. |
| D11 | Synthetic probe runtime and ownership. | Select a VNet-connected shared or per-application runtime; record deployment, identity, monitoring, support, and cost ownership. |

## 12. Implementation Acceptance Gates

The module is not approved for production until these tests pass:

1. Public endpoint is inaccessible and Private Endpoint connectivity succeeds.
2. Azure and on-premises test clients resolve the normal AMR hostname to the private path where hybrid access is required.
3. Entra managed identity connects, survives token refresh, and reconnects after a forced connection interruption.
4. A declared Entra `Group` principal fails during plan. When strict directory validation is enabled, the actual object type is resolved through Microsoft Graph during plan where possible or during apply when the ID was initially unknown. Otherwise testing confirms that an unsupported group object ID cannot be successfully provisioned.
5. Invalid clustering, module, eviction, persistence, SKU, and HA combinations fail during plan.
6. Diagnostics arrive in the intended Log Analytics tables.
7. The D11-owned synthetic probe runs from the private network path, and its baseline alert triggers and resolves correctly.
8. Production plan blocks or requires approval for replacement/database recreation.
9. Persistence recovery and application rehydration are tested against declared RPO/RTO.
10. CMK deployment, rotation, and denied-key recovery are tested when CMK is enabled.
11. Access-key exception proves state encryption, access restrictions, Key Vault delivery, and rotation.
12. Representative load test validates memory, throughput, latency, connections, and failover headroom.
13. The Redis major-version monitoring, compatibility-test, deferral, and upgrade-approval process is exercised.

## 13. Next Steps

- ARB reviews and resolves D1-D11.
- Platform Engineering builds the module skeleton and executable validations.
- Network Engineering confirms central DNS, policy identity, hybrid resolver, and PE subnet requirements.
- Security Architecture confirms stable access scope, preview ACL position, key exception, state, and CMK controls.
- FinOps validates the profile cost model including ancillary services.
- Pilot with one non-production application and one production-like load test.
- Publish the production module only after the implementation acceptance gates pass.

## 14. References

- [Azure Managed Redis overview](https://learn.microsoft.com/en-us/azure/redis/overview)
- [Azure Managed Redis architecture](https://learn.microsoft.com/en-us/azure/redis/architecture)
- [Azure Cache for Redis retirement FAQ](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/retirement-faq)
- [Microsoft Entra authentication](https://learn.microsoft.com/en-us/azure/redis/entra-for-authentication)
- [AzureAD directory-object data source and permissions](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/data-sources/directory_object)
- [Custom data access permissions - preview](https://learn.microsoft.com/en-us/azure/redis/configure-access-permissions)
- [AzureRM `azurerm_managed_redis_access_policy_assignment`](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/managed_redis_access_policy_assignment)
- [Azure Managed Redis Private Link](https://learn.microsoft.com/en-us/azure/redis/private-link)
- [Private Endpoint DNS integration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns-integration)
- [Private Endpoint network policies](https://learn.microsoft.com/en-us/azure/private-link/disable-private-endpoint-network-policy)
- [Azure DNS Private Resolver architecture](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/azure-dns-private-resolver)
- [Azure Managed Redis persistence](https://learn.microsoft.com/en-us/azure/redis/how-to-persistence)
- [Redis modules](https://learn.microsoft.com/en-us/azure/redis/redis-modules)
- [Flash Optimized best practices](https://learn.microsoft.com/en-us/azure/redis/best-practices-flash-optimized)
- [Azure Managed Redis security](https://learn.microsoft.com/en-us/azure/redis/secure-azure-managed-redis)
- [Azure Managed Redis metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-cache-redisenterprise-metrics)
- [Application Insights availability tests and private endpoints](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability)
- [Azure Managed Redis version upgrades](https://learn.microsoft.com/en-us/azure/redis/how-to-upgrade)
- [Scheduled maintenance - preview](https://learn.microsoft.com/en-us/azure/redis/scheduled-maintenance)
- [Azure Managed Redis pricing and SLA conditions](https://azure.microsoft.com/en-us/pricing/details/managed-redis/)
- [AzureRM `azurerm_managed_redis`](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/managed_redis)
- [AzureRM 4.54.0 release notes](https://github.com/hashicorp/terraform-provider-azurerm/releases/tag/v4.54.0)
- [AzureRM 4.60.0 release notes](https://github.com/hashicorp/terraform-provider-azurerm/releases/tag/v4.60.0)
- [Terraform sensitive-data guidance](https://developer.hashicorp.com/terraform/language/manage-sensitive-data)
