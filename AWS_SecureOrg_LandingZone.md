# AWS Secure Organization Landing Zone Design

**(Minimal, Airgapped, Audit-Ready Approach)**

---

## 1. Purpose

This document defines a **minimal, scalable, and audit-ready AWS landing zone** for a Secure AWS Organization.

The design prioritizes:

* Strong **control-plane governance**
* Clear **separation of responsibilities**
* **Airgapped-ready architecture**
* Delayed introduction of networking (only when required)

---

## 2. Design Philosophy

### Key Principle

> Landing zone design should focus on **control, governance, and isolation first**, not networking.

---

### Objectives

* Enforce strict separation between:

  * Control plane
  * Security plane
  * Data plane
* Minimize initial complexity
* Ensure audit compliance from day one
* Enable incremental evolution (networking, workloads)

---

## 3. Core Account Structure

A minimal but strong landing zone requires **4–5 accounts**.

---

## 3.1 Management Account (Control Plane)

### Purpose

* Organization management
* Billing and account lifecycle
* Identity and access control

---

### Responsibilities

* AWS Organizations management
* Service Control Policies (SCPs)
* IAM Identity Center (SSO)
* CloudFormation (Org + account provisioning)

---

### Must NOT Contain

* Application workloads
* Sensitive data
* Security tooling

---

## 3.2 Security Account (Security Control Plane)

### Purpose

> Centralized **security visibility and governance**

---

### Responsibilities

* AWS Security Hub (Org administrator)
* Amazon GuardDuty (Org administrator)
* AWS Config Aggregator
* Centralized findings and alerts
* (Future) MPA workflows

---

### Design Rationale

* Isolates security control plane
* Reduces blast radius in case of compromise

---

## 3.3 Log Archive Account (Audit Plane)

### Purpose

> Immutable, centralized logging

---

### Responsibilities

* S3 buckets for:

  * CloudTrail logs
  * AWS Config logs
* Object Lock (WORM enforcement)
* Long-term log retention

---

### Critical Controls

* No routine human access
* Strict bucket policies
* Immutable storage

---

## 3.4 Shared Services Account (Optional)

### Purpose

> Centralized infrastructure services (only when needed)

---

### Initial State

* May remain empty during early phases

---

### Future Responsibilities

* Transit Gateway (TGW)
* PrivateLink endpoints
* Shared tooling and services

---

### Design Principle

> Introduce only when networking or shared infrastructure becomes necessary

---

## 3.5 Workload Accounts (Data Plane)

### Purpose

> Host actual **data and workloads**

---

### Responsibilities

* Data storage (e.g., S3, databases)
* Compute (if required later)
* Backup vault usage

---

### Restrictions

* No security control services
* No organization-level configurations

---

## 4. Organizational Structure

```id="3f6w9m"
Root / Management Account
├── Vault-OU (Tier-0)
│   ├── Security Account
│   └── Log Archive Account
├── Processing-OU (Tier-1)
│   └── Shared Services Account (Optional)
└── Restore-OU (Tier-2)
    ├── Org-Prod-1 (Workload)
    └── Org-Prod-2 (Workload)
```

---

## 5. Control Plane vs Data Plane Separation

| Layer                     | Account Type      |
| ------------------------- | ----------------- |
| Control Plane             | Management        |
| Security Control          | Security          |
| Audit / Logging           | Log Archive       |
| Infrastructure (Optional) | Shared Services   |
| Data / Workloads          | Workload Accounts |

---

### Principle

> No mixing of responsibilities across layers

---

## 6. Baseline Services (All Accounts)

Deployed via CloudFormation StackSets:

* AWS CloudTrail (organization-wide)
* AWS Config
* Amazon GuardDuty (member accounts)
* AWS Security Hub (member accounts)
* AWS Systems Manager (SSM)

---

## 7. Account-Specific Responsibilities

### Management Account

* IAM Identity Center
* SCP management
* Organization lifecycle

---

### Security Account

* Security Hub (admin)
* GuardDuty (admin)
* Config aggregator

---

### Log Archive Account

* Central logging buckets
* Object Lock enforcement
* No compute resources

---

### Workload Accounts

* Data and application workloads
* Backup vault usage

---

## 8. Networking Strategy

### Key Principle

> Networking is introduced **only when required**

---

## 8.1 Phase 1 — No Networking

* No VPCs
* No subnets
* No Internet Gateway
* No endpoints

---

### Supported Services Without VPC

* AWS CloudTrail
* AWS Config
* AWS Security Hub
* Amazon GuardDuty

---

### Outcome

* Fully functional control plane
* No network exposure
* Simplified audit posture

---

## 8.2 Phase 2 — Introduce Networking (When Needed)

Networking is added only when:

* Compute workloads are required
* Private data ingestion is needed
* Systems Manager access is required

---

### Future Components

* VPC
* Private subnets
* VPC endpoints
* Transit Gateway (optional)

---

## 9. Key Design Principles

### 9.1 Control First, Network Later

* Establish governance before infrastructure
* Avoid premature VPC design

---

### 9.2 Separation of Duties

* Security, logging, and workloads are isolated
* Each account has a clearly defined role

---

### 9.3 Minimalism

* Avoid unnecessary accounts
* Avoid unused infrastructure

---

### 9.4 Audit Readiness

* Centralized logging
* Immutable storage
* Enforced guardrails

---

## 10. Common Pitfalls to Avoid

* Deploying workloads in the Management account
* Storing logs in the Security account
* Creating Shared Services prematurely
* Adding VPCs “just in case”
* Mixing control plane and data plane responsibilities

---

## 11. Minimal Recommended Setup

For initial deployment:

```id="g0x7ke"
1. Management Account
2. Security Account
3. Log Archive Account
4. Workload Account (Org)
```

---

### Optional (later)

* Shared Services Account

---

## 12. Future Extensions

* Multi-Party Approval (MPA) workflows
* Airgapped VPC design
* PrivateLink-based ingestion
* Advanced monitoring and alerting

---

## 13. Summary

This landing zone design:

* Starts with **strong governance and isolation**
* Avoids unnecessary early complexity
* Enables **secure, incremental growth**
* Aligns with **audit and airgap requirements**

---

## 14. Key Takeaway

> A strong landing zone is defined by **clear boundaries and control**, not by infrastructure volume.

---
