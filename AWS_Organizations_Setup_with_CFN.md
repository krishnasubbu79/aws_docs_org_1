# AWS Organization Setup Approach

**(Management Account, Root, Org, OU, Accounts, Baseline — Manual vs CloudFormation)**

---

## 1. Purpose

This document provides a clear, audit-aligned approach for setting up a new AWS Organization, including:

* Management account initialization
* Root account security
* Organization and OU structure
* SCP guardrails
* Account creation
* Baseline security services

It also defines **which steps must be manual** and **which can be automated via CloudFormation**.

---

## 2. High-Level Execution Flow

```
Bootstrap → Org → Guardrails → Core Accounts → Baseline → Workload Accounts
```

Sequence is critical. Deviating from this order introduces security gaps.

---

## 3. Phase 0 — Management Account & Root Setup (Bootstrap)

### 🔧 Manual (Mandatory)

This phase establishes trust and cannot be automated.

### Steps:

1. Create AWS Management Account

   * Use dedicated root email alias (plus-addressing format: `aws-root+mgmt@company.com`)
   * Set strong password

2. Secure Root Account

   * Enable hardware MFA (minimum 2 devices)
   * Store password in secure vault
   * Implement split custody model

3. Enable AWS Organizations

   * Select **All Features**

4. Enable Trusted Access for CloudFormation StackSets

   * In the AWS Organizations console (Management Account), navigate to **Services** and enable **Trusted access** for **AWS CloudFormation StackSets**
   * This is a mandatory one-time action required before any service-managed StackSet can be deployed
   * It cannot be performed via CloudFormation — it must be done manually in the console or via the AWS CLI (`aws organizations enable-aws-service-access --service-principal stacksets.cloudformation.amazonaws.com`)
   * Must be completed before Phase 3

5. Enable IAM Identity Center

   * Create initial Operator access

---

### Notes:

* Root configuration is NOT automatable via CloudFormation
* This is a one-time foundational setup

---

## 4. Phase 1 — Organization Structure (OU + SCP)

### ⚙️ Automatable via CloudFormation

### Template 1: Organization Foundation

### Create:

* Organizational Units (OUs):

  * Vault-OU / Security (Tier-0)
  * Processing-OU / Infrastructure (Tier-1)
  * Restore-OU / Workloads (Tier-2)
  * Sandbox-OU (Optional)

* Service Control Policies (SCPs):

  * Deny public internet exposure
  * Deny IAM users and access keys
  * Deny disabling logging services
  * Deny unauthorized restore operations (future MPA integration)

* Attach SCPs to:

  * Root
  * Relevant OUs

---

### Notes:

* Root entity is implicit and not created via CloudFormation
* SCPs must be applied early to prevent exposure

---

## 5. Phase 2 — Core Accounts Creation

### ⚙️ Automatable via CloudFormation

### Template 2: Core Accounts

Use:

* `AWS::Organizations::Account`

### Create:

* Security Account
* Log Archive Account
* Shared Services Account

---

### Notes:

* Account creation is asynchronous
* CloudFormation creates accounts but does NOT fully configure them

---

## 5.5 Phase 2.5 — Delegated Admin Registration (Manual Prerequisite)

### 🔧 Manual (Mandatory — Must complete before Phase 3)

This phase must be completed after core accounts exist and before StackSets deploy the security baseline. Without this step, Security Hub, GuardDuty, and Config will enrol member accounts without a central administrator, breaking centralized visibility.

### Steps (performed in Management Account):

* **Security Hub**: Register Security Account as Delegated Administrator
  (Security Hub → Settings → Delegated Administrator)

* **GuardDuty**: Register Security Account as Delegated Administrator
  (GuardDuty → Settings → Delegated Administrator)

* **AWS Config**: Register Security Account as Aggregator Administrator

---

### Notes:

* These are console-level actions in the Management Account — not CloudFormation
* If deploying to multiple regions, each delegation must be repeated per region
* Failure to complete this step before Phase 3 will result in member accounts having no centralized security admin

---

## 6. Phase 3 — Baseline Security & Logging

### ⚙️ Automatable via CloudFormation StackSets (Mandatory)

### Deployment Method:

* CloudFormation StackSets (service-managed permissions)

---

### Critical Prerequisite — Log Archive S3 Bucket (Before StackSets Run)

The organization-wide CloudTrail trail (created by StackSet) writes logs to a centralized S3 bucket in the Log Archive account. This bucket **must exist before** the CloudTrail StackSet deploys, or the trail creation will fail.

**Required action:** Deploy a targeted CloudFormation stack directly to the Log Archive account (not a StackSet) to create the central logging bucket with:

* Object Lock enabled (WORM — Compliance mode)
* Versioning enabled
* Bucket policy granting `cloudtrail.amazonaws.com` write access, scoped using `aws:SourceOrgID` condition to prevent cross-org writes

This is a **targeted stack, not a StackSet**, because it is account-specific infrastructure that must only exist in Log Archive.

> Sequence: Log Archive S3 bucket → then CloudTrail StackSet → then all other baseline StackSets

---

### Baseline Components:

#### CloudTrail

* Organization-wide trail
* Logs centralized in Log Archive account
* All regions enabled

#### AWS Config

* Enabled in all accounts
* Aggregator in Security account

#### Security Hub

* Enabled organization-wide
* Delegated admin = Security account

#### GuardDuty

* Enabled organization-wide

#### Systems Manager (SSM)

* Session Manager enabled
* No SSH dependency

#### Logging Infrastructure

* Centralized S3 buckets
* Versioning and encryption enabled

---

### Critical Requirement:

> Baseline MUST auto-apply to all new accounts

No manual intervention allowed.

---

## 7. Phase 4 — Workload Account Creation

### ⚙️ Automatable via CloudFormation

### Template 3: Workload Accounts

Use:

* `AWS::Organizations::Account`

### Responsibilities:

* Create accounts
* Assign to appropriate OU
* Apply tags (environment, owner, purpose)

---

### Notes:

* Account creation is asynchronous and may require tracking/retries
* Baseline must apply immediately after creation

---

## 8. Phase 5 — IAM Identity Center Setup

### 🟡 Hybrid Approach

---

### 🔧 Initial Setup (Manual)

* Enable IAM Identity Center
* Create:

  * Operator user/group
  * Initial permission set

---

### ⚙️ Optional Automation (Later)

* `AWS::SSO::PermissionSet`
* `AWS::SSO::Assignment`

---

## 9. Phase 6 — Root Strategy (All Accounts)

### 🔧 Manual (Management Account vs Member Accounts)

**For the Management Account (Tier 1):**
* Enable hardware MFA (split custody)
* Store password securely in a vault
* Apply physical break-glass strategy

**For all Member Accounts (Tier 2):**
* Assign root email (handled via automated account creation)
* **Remove root credentials** via IAM Centralized Root Access (performed from the Management Account)
* No hardware MFA or local password required

---

### Notes:

CloudFormation CANNOT:

* Configure root user
* Enable MFA
* Rotate root credentials

---

## 10. Responsibility Summary

### 🔧 Manual (Critical Control Plane)

* Management account creation
* Management Account Root setup & Member Account Root removal
* IAM Identity Center initial configuration
* Break-glass implementation

---

### ⚙️ Automated (CloudFormation / StackSets)

* OU creation
* SCP creation and attachment
* Account creation
* Baseline security services
* Identity Center (post-bootstrap, optional)

---

## 11. Key Guardrails

* SCPs must be applied BEFORE large-scale account creation
* No account should exist without baseline security controls
* Root must never be used for operational tasks
* Logging must be centralized and immutable

---

## 12. Common Pitfalls (To Avoid)

* Attempting to automate root account configuration
* Delaying SCP implementation
* Manually configuring baseline services
* Creating accounts before guardrails are in place
* Inconsistent application of controls across accounts
* Leaving active root credentials on member accounts instead of using IAM Centralized Root Access

---

## 13. Final Execution Plan

1. Manual:

   * Create management account
   * Secure root
   * Enable Organizations and Identity Center

2. CloudFormation Template 1:

   * Create OUs and SCPs

3. CloudFormation Template 2:

   * Create core accounts (Security, Log Archive, Shared Services)

4. StackSets:

   * Deploy baseline security and logging services

5. CloudFormation Template 3:

   * Create workload accounts

6. Manual (Post-Account Creation):

   * Remove root credentials for new member accounts via IAM Centralized Root Access

7. Optional:

   * Automate Identity Center configuration

---

## 14. Key Takeaway

* Automation handles **structure and scalability**
* Manual controls handle **trust, identity, and root security**

Both are required for a secure, audit-ready AWS organization.

---
