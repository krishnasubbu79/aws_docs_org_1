# CloudFormation StackSets & Networking Requirements

**(Airgapped AWS Organization — Execution Model & Dependencies)**

---

## 1. Purpose

This document clarifies:

* How **CloudFormation StackSets operate across accounts**
* Whether **additional infrastructure (IAM roles, networking)** is required
* How StackSets behave in an **airgapped AWS organization**

---

## 2. Key Questions Answered

1. Do StackSets require roles inside target accounts?
2. Does CloudFormation require VPC/network access to create resources?
3. What networking is required for baseline deployments?

---

## 3. StackSets Permission Model

## 3.1 Recommended Mode: Service-Managed Permissions

> This is the **recommended and audit-friendly approach**

---

### How it works

* AWS Organizations is integrated with CloudFormation
* Trusted access is enabled
* AWS automatically manages required IAM roles

---

### Automatically created roles

#### In Management Account:

* `AWSCloudFormationStackSetAdministrationRole`

#### In Target Accounts:

* `AWSCloudFormationStackSetExecutionRole`

---

### Key Benefits

* No manual role creation required
* Consistent across all accounts
* Reduced operational overhead
* Fully aligned with audit expectations

---

## 3.2 Alternative Mode: Self-Managed Permissions (Not Recommended)

Requires:

* Manual IAM role creation in every account
* Manual trust policy configuration
* Ongoing maintenance

---

### Risks

* Configuration drift
* Increased operational complexity
* Higher audit burden

---

## 4. StackSet Execution Flow

```id="c2zj8g"
Account Created → StackSet Triggered → Execution Role Available → Resources Deployed
```

---

### Key Points

* Execution roles are automatically provisioned
* StackSets can deploy immediately after account creation
* No manual setup required inside target accounts

---

## 5. CloudFormation Networking Model

## 5.1 Important Principle

> CloudFormation operates in the **AWS control plane**, not inside your VPC

---

### Implications

* CloudFormation does NOT require:

  * VPC
  * Subnets
  * Internet Gateway
  * VPC Endpoints

---

## 5.2 Resource-Level Dependency

While CloudFormation itself does not require networking:

> The **resources being created may require network connectivity**

---

## 6. Baseline Deployment (Phase 3)

### Services Included

* AWS CloudTrail
* AWS Config
* AWS Security Hub
* Amazon GuardDuty
* IAM resources

---

### Networking Requirement

✅ **No VPC or endpoints required**

---

### Reason

These are **control-plane services** managed by AWS and do not depend on VPC-level networking.

---

## 7. Workload & VPC-Based Deployments (Later Phases)

When deploying:

* EC2 instances
* Systems Manager (SSM)
* Application workloads
* VPC endpoints

---

### Networking Becomes Required

For airgapped environments, the following VPC endpoints are typically required:

* `ssm`
* `ec2`
* `sts`
* `logs` (CloudWatch Logs)
* `s3` (Gateway endpoint)

---

### Purpose

* Enable service communication without internet access
* Maintain isolation while allowing AWS service interaction

---

## 8. Recommended Phase Separation

### Phase 3 — Baseline (No Network Dependency)

* Deploy security and logging services
* Use StackSets
* No VPC required

---

### Phase 4+ — Networking & Workloads

* Create VPCs and subnets
* Configure endpoints
* Deploy workloads

---

### Key Rule

> Do NOT mix baseline security deployment with VPC/workload provisioning

---

## 9. Audit Considerations

### Key Assurance Statement

> Baseline security services operate independently of VPC networking and are deployed using AWS control-plane mechanisms.

---

### Evidence Points

* StackSets execution logs
* IAM roles created automatically
* CloudTrail logs confirming deployment
* No dependency on network infrastructure

---

## 10. Summary

### StackSets

* Require IAM roles → **automatically managed (service-managed mode)**
* Do NOT require VPC or networking

---

### CloudFormation

* Runs in AWS control plane
* Does NOT require network access

---

### Networking

* Required ONLY for:

  * VPC-based resources
  * Workloads

---

## 11. Key Takeaways

* Use **service-managed StackSets** for simplicity and audit compliance
* Baseline services can be deployed in **fully airgapped accounts without VPCs**
* Separate **security baseline deployment** from **network/workload provisioning**
* Ensure networking is designed only when required

---
