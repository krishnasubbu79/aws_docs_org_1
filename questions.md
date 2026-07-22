# Azure Managed Redis Consumer Application Requirements Questionnaire

## Document Control

| Field | Value |
| --- | --- |
| Application / service | TBD |
| Application owner | TBD |
| Technical contact | TBD |
| Business owner | TBD |
| Environment(s) | Dev / Test / Production |
| Requirements workshop date | TBD |
| Document owner | TBD |
| Status | Draft |

## Primary Question

> What does the application use Redis for, and what happens to the application if Redis becomes unavailable or all cached data is lost?

**Application-team response:**

TBD

## 1. Business and Application Context

1. What business capability does the application provide?
2. Which application components will connect to Azure Managed Redis?
3. Is the application customer-facing, employee-facing, or an internal platform service?
4. Which environments require Redis?
5. Who owns the application and provides production support?
6. Are there regulatory, contractual, or data-residency requirements?

**Response:**

TBD

## 2. Redis Usage

1. What will Redis store?
   - General cache entries
   - User sessions
   - Authentication or authorization data
   - Rate-limit counters
   - Distributed locks
   - Queues or streams
   - Search indexes
   - Other
2. Is another system the authoritative source of this data?
3. Can all Redis data be reconstructed from the authoritative source?
4. How long would reconstruction take?
5. Are Redis transactions, Lua scripts, Pub/Sub, Streams, or distributed locks used?
6. Are RedisJSON, RediSearch, RedisBloom, or RedisTimeSeries required?
7. Are specific Redis commands required or prohibited?

**Response:**

TBD

## 3. Data and Recovery

1. Can all Redis data be lost without permanent business-data loss?
2. What happens if Redis returns with an empty database?
3. Does the application automatically repopulate the cache?
4. Could mass repopulation overload the authoritative database or dependent services?
5. Is cache warming required before application traffic is restored?
6. Is Redis persistence required? If yes, explain the business reason.
7. What are the required recovery time objective (RTO) and recovery point objective (RPO)?
8. Who owns and tests the cache-rehydration procedure?

**Response:**

TBD

## 4. Failure Behaviour and Availability

1. What happens if Redis is unavailable for:
   - 30 seconds?
   - 5 minutes?
   - 15 minutes?
   - 60 minutes?
2. Can the application continue without Redis?
3. Can it fall back to the authoritative database?
4. Is a degraded mode available?
5. Does the client retry failed operations?
6. What retry interval, retry limit, timeout, and backoff strategy are used?
7. Does the application use a circuit breaker?
8. Can it reconnect automatically following maintenance or failover?
9. Is high availability mandatory in every environment or only production?
10. Is cross-region disaster recovery required?

**Response:**

TBD

## 5. Capacity and Performance

| Requirement | Current | Peak | 12-month forecast | 24-month forecast |
| --- | ---: | ---: | ---: | ---: |
| Stored data size | TBD | TBD | TBD | TBD |
| Operations per second | TBD | TBD | TBD | TBD |
| Read operations per second | TBD | TBD | TBD | TBD |
| Write operations per second | TBD | TBD | TBD | TBD |
| Concurrent connections | TBD | TBD | TBD | TBD |
| Largest key size | TBD | TBD | TBD | TBD |
| Largest value size | TBD | TBD | TBD | TBD |

Additional questions:

1. What latency is required at the median, 95th percentile, and 99th percentile?
2. Are there predictable traffic spikes or seasonal events?
3. What is the normal TTL range?
4. Are any keys created without a TTL?
5. What should happen when memory is full?
6. Which eviction policy is required?
7. Is scale-up expected, and how much notice is available before peak events?

**Response:**

TBD

## 6. Client Compatibility

| Component | Language/runtime | Redis client and version | Hosting platform | Connection count |
| --- | --- | --- | --- | ---: |
| TBD | TBD | TBD | TBD | TBD |

For every client, confirm:

- [ ] TLS is supported and enabled.
- [ ] Microsoft Entra authentication is supported.
- [ ] Entra tokens are refreshed before expiration.
- [ ] Connection pooling is configured.
- [ ] Connections recover automatically after interruption.
- [ ] Redis Cluster topology discovery is supported if `OSSCluster` is selected.
- [ ] Dynamic shard endpoints and ports are handled correctly.
- [ ] DNS changes are handled without restarting the application.
- [ ] The client has been tested against Azure Managed Redis.

**Compatibility concerns or exceptions:**

TBD

## 7. Authentication and Authorization

1. Which managed identity, service principal, or user will each component use?
2. Does each application component have its own identity?
3. Who owns identity creation and lifecycle management?
4. Who approves Redis data-plane access?
5. Are Entra groups currently expected? Azure Managed Redis access assignments do not support groups.
6. Does any legacy component require access-key authentication?
7. If access keys are required:
   - What prevents Entra authentication?
   - Who approved the exception?
   - When does the exception expire?
   - Who owns migration to Entra authentication?
8. Where will authentication failures be monitored?

### Identity Register

| Component | Identity name | Object ID | Principal type | Environment | Owner |
| --- | --- | --- | --- | --- | --- |
| TBD | TBD | TBD | Managed identity / Service principal / User | TBD | TBD |

## 8. Network Connectivity and DNS

1. Where will clients run?
   - Azure Kubernetes Service
   - App Service
   - Azure Functions
   - Virtual machines
   - Container Apps
   - On-premises
   - Another cloud
   - Other
2. Which subscriptions, regions, VNets, and subnets contain the clients?
3. Is VNet peering already available?
4. Do on-premises clients have VPN or ExpressRoute connectivity to Azure?
5. Who owns the Private Endpoint subnet?
6. Who owns the `privatelink.redis.azure.net` Private DNS zone?
7. Is Private DNS linked to every required client VNet?
8. How will on-premises DNS queries be conditionally forwarded into Azure?
9. Is Azure DNS Private Resolver available?
10. Can firewalls and NSGs allow TCP port `10000`?
11. For `OSSCluster`, can the network allow the dynamic TCP port range `8500-8599` without pinning individual ports?
12. Are network flows and DNS resolution monitored?

### Client Network Register

| Client | Subscription | VNet/subnet or on-prem network | DNS path | Routing path | Firewall owner |
| --- | --- | --- | --- | --- | --- |
| TBD | TBD | TBD | TBD | TBD | TBD |

## 9. Redis Topology and Features

1. Does the client support Redis Cluster?
2. Which clustering policy is required?
   - `OSSCluster`
   - `EnterpriseCluster`
   - `NoCluster`
3. Is a non-clustered endpoint a confirmed application requirement?
4. Is the expected dataset compatible with `NoCluster` size limits?
5. Are Redis modules required?
6. Does RediSearch require `EnterpriseCluster` and `NoEviction` for this workload?
7. Are preview SKUs or preview features being requested?
8. Who accepts the operational risk of preview functionality?

**Response:**

TBD

## 10. Monitoring and Support

1. Which Log Analytics workspace should receive diagnostics?
2. Which Azure Monitor action group should receive alerts?
3. Who responds to memory, CPU, server-load, eviction, connection, and Resource Health alerts?
4. What are the warning and critical thresholds?
5. Does the application expose Redis dependency metrics?
6. Are authentication failures and connection-pool exhaustion logged?
7. Who will deploy and operate a synthetic Redis connectivity probe?
8. Will the probe test the complete path below?

```text
Private DNS resolution -> TCP connection -> TLS -> Entra token -> AUTH -> PING
```

9. What is the support and escalation process?
10. What service-level objective applies to this Redis dependency?

### Operational Ownership

| Responsibility | Owner | Support group | Escalation contact |
| --- | --- | --- | --- |
| Application | TBD | TBD | TBD |
| Azure Managed Redis | TBD | TBD | TBD |
| Network and Private Endpoint | TBD | TBD | TBD |
| Private DNS | TBD | TBD | TBD |
| Entra identities | TBD | TBD | TBD |
| Monitoring and alerting | TBD | TBD | TBD |
| Synthetic probe | TBD | TBD | TBD |

## 11. Security and Compliance

1. What data classifications apply to values stored in Redis?
2. Will secrets, credentials, payment data, or sensitive personal data be stored?
3. Is customer-managed-key encryption required?
4. If CMK is required, who owns the Key Vault, key rotation, permissions, and incident response?
5. What log-retention period is required?
6. Are diagnostic records allowed in the selected Log Analytics region?
7. Are Azure Policy exemptions required?
8. Which organizational tags are mandatory?

**Response:**

TBD

## 12. Change and Release Management

1. What maintenance windows are available?
2. Can the application tolerate connection termination during configuration changes?
3. How much notice is required for maintenance, scaling, or replacement?
4. Who reviews and approves destructive Terraform plans?
5. How will clustering-policy or Redis-module changes be tested?
6. Is blue/green Redis migration required for major changes?
7. Who validates the application after a Redis change?
8. What is the rollback or rehydration procedure?

**Response:**

TBD

## 13. Acceptance Tests

The consumer application team must demonstrate the following before production approval:

- [ ] The application resolves the Azure Managed Redis hostname to the expected private address.
- [ ] Public network access is unavailable.
- [ ] The client establishes an encrypted connection.
- [ ] The workload authenticates using its assigned Entra identity.
- [ ] Entra token refresh works without an application restart.
- [ ] Unauthorized identities cannot connect.
- [ ] The application reconnects after a Redis connection interruption.
- [ ] The application tolerates or handles Redis unavailability.
- [ ] The application behaves safely when Redis contains no data.
- [ ] Cache rehydration does not overwhelm the authoritative data source.
- [ ] Cluster discovery works, including dynamic shard endpoints where applicable.
- [ ] Monitoring receives connection events and platform metrics.
- [ ] Alerts reach the responsible support team.
- [ ] The synthetic probe verifies DNS, TLS, authentication, and `PING`.
- [ ] The operational runbook has been reviewed and exercised.

## 14. Architecture Decisions

| Decision | Selected value | Rationale | Owner | Approval reference |
| --- | --- | --- | --- | --- |
| Redis usage | TBD | TBD | TBD | TBD |
| Data-loss tolerance | TBD | TBD | TBD | TBD |
| SKU and capacity | TBD | TBD | TBD | TBD |
| Clustering policy | TBD | TBD | TBD | TBD |
| Eviction policy | TBD | TBD | TBD | TBD |
| High availability | TBD | TBD | TBD | TBD |
| Persistence | None / RDB / AOF | TBD | TBD | TBD |
| Redis modules | TBD | TBD | TBD | TBD |
| Authentication | Entra / approved access-key exception | TBD | TBD | TBD |
| Application identity | TBD | TBD | TBD | TBD |
| Private Endpoint subnet | TBD | TBD | TBD | TBD |
| Private DNS ownership | Policy-managed / module-managed | TBD | TBD | TBD |
| On-premises connectivity | TBD | TBD | TBD | TBD |
| Monitoring owner | TBD | TBD | TBD | TBD |
| Synthetic probe owner/runtime | TBD | TBD | TBD | TBD |
| Recovery approach | TBD | TBD | TBD | TBD |

## 15. Risks, Assumptions, and Open Questions

| Type | Description | Owner | Due date | Status |
| --- | --- | --- | --- | --- |
| Risk / Assumption / Question | TBD | TBD | TBD | Open |

## 16. Approval

| Role | Name | Decision | Date | Comments |
| --- | --- | --- | --- | --- |
| Application owner | TBD | Approve / Reject | TBD | TBD |
| Application technical lead | TBD | Approve / Reject | TBD | TBD |
| Platform owner | TBD | Approve / Reject | TBD | TBD |
| Network/DNS owner | TBD | Approve / Reject | TBD | TBD |
| Security representative | TBD | Approve / Reject | TBD | TBD |
| Operations owner | TBD | Approve / Reject | TBD | TBD |

## Closing Validation Question

> Please demonstrate how the application behaves when Redis is unreachable, when authentication expires, and when Redis becomes available again with no cached data.

**Evidence or test-result links:**

TBD
