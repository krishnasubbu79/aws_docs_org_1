# Secure AWS Organization End-to-End Execution Flow

This document outlines the end-to-end flow for building a highly secure, audit-ready, and airgapped AWS environment. It perfectly balances human-enforced security with automated scalability.

---

## Phase 0: The Human Trust Foundation (Strictly Manual)
Before any automation happens, you establish the root of trust.
1. **Bootstrap:** Create the AWS Management Account using the standardized email pattern (`aws-root+mgmt@company.com`).
2. **Lockdown:** Secure the root user by generating a 32+ character password stored in a vault, and registering at least two Hardware MFA devices.
3. **Split Custody:** Split the MFA devices and password access between the Security and Platform teams.
4. **Enable Core Services:** Manually enable AWS Organizations (All Features) and IAM Identity Center (SSO) so that no one ever has to use the root account again for daily tasks.
5. **Enable StackSets Trusted Access:** In the AWS Organizations console (from the Management Account), enable **Trusted access for AWS CloudFormation StackSets**. This is a one-time action that allows service-managed StackSets to automatically provision execution roles in member accounts. It must be done before any StackSet is deployed and cannot be automated via CloudFormation itself.

---

## Phase 1: The Control Plane Architecture (Automated via CloudFormation)
Use Infrastructure as Code (IaC) to build the invisible boundaries.
1. **OUs:** Deploy your Organizational Units: `Vault-OU` (Tier 0), `Processing-OU` (Tier 1), and `Restore-OU` (Tier 2).
2. **SCPs:** Immediately apply Service Control Policies to the OUs. This includes custom SCPs that deny the root user in member accounts from turning off CloudTrail or Config, as well as SCPs denying public internet access.

---

## Phase 2: The Core Landing Zone (Automated via CloudFormation)
With the boundaries in place, create the dedicated security and logging accounts.
1. **Account Creation:** Deploy the **Security Account** (for GuardDuty/Security Hub centralization) and the **Log Archive Account** (for immutable CloudTrail/Config storage) into the `Vault-OU`.

---

## Phase 2.5: Delegated Admin Registration (Manual — Management Account)
This step must occur **after the Security Account exists** (Phase 2) and **before the security baseline StackSets deploy** (Phase 3). StackSets that deploy Security Hub and GuardDuty will enrol member accounts into those services — but if the delegated admin is not registered first, member accounts will enrol without a central administrator, breaking the consolidated findings model.

1. **Security Hub:** In the Management Account, navigate to Security Hub → Settings → Delegated Administrator. Register the **Security Account** as the delegated admin for the entire organization.
2. **GuardDuty:** In the Management Account, navigate to GuardDuty → Settings → Delegated Administrator. Register the **Security Account** as the delegated admin for the entire organization.
3. **AWS Config:** In the Management Account, register the **Security Account** as the Config Aggregator administrator. The aggregator itself will be created by the Phase 3 StackSet, but the delegation must exist first.

> **Note:** These are console-based actions done in the Management Account. They cannot be performed via CloudFormation at the org level. Each delegation must be done per-region if multi-region support is required.

---

## Phase 3: The Security Baseline (Automated via StackSets — NO Network)
This is where the "Control First, Network Later" philosophy applies.

**Critical prerequisite before StackSets run:**
Before the organization-wide CloudTrail trail can be created, the **Log Archive account must have an S3 bucket** with the correct cross-account bucket policy allowing the CloudTrail service principal to write from the organization. This bucket does not exist yet — it must be created via a targeted CloudFormation stack deployed specifically to the Log Archive account (not a StackSet, since this is account-specific). The bucket must have:
- Object Lock enabled (WORM enforcement)
- Versioning enabled
- A bucket policy explicitly granting `cloudtrail.amazonaws.com` write access scoped to the organization (`aws:SourceOrgID` condition)

Only once this bucket exists should the StackSet creating the organization-wide CloudTrail trail be deployed.

1. **Log Archive S3 Bucket:** Deploy a targeted CFN stack to the Log Archive account to create the central logging bucket with Object Lock, versioning, and the org-scoped CloudTrail bucket policy.
2. **Airgapped Deployment:** Using AWS CloudFormation StackSets (with Service-Managed permissions), automatically deploy AWS CloudTrail (org trail pointing to the Log Archive bucket), AWS Config, Security Hub, and GuardDuty across all current and future accounts.
3. **Zero Network Attack Surface:** Because this relies entirely on the AWS Control Plane, you achieve a fully audited, monitored environment without creating a single VPC, Subnet, or Internet Gateway.

---

## Phase 4: Workloads & Networking (Automated via CloudFormation)
Once the environment is locked down and heavily monitored, introduce data and networking.
1. **Workload Accounts:** Create the actual Secure AWS Organization accounts (e.g., `Org-Prod-1`) in the `Restore-OU`. The Phase 3 StackSets automatically apply the security baseline to them the moment they are created.
2. **Networking Introduction:** Because workloads require connectivity (like Systems Manager or S3 access), deploy isolated VPCs and VPC Endpoints (`ssm`, `logs`, `s3`) strictly for internal communication.

---

## Ongoing: The Break-Glass Governance (Operational)
If things go catastrophically wrong (e.g., SSO goes down), the automated flow stops, and the Break-Glass protocol kicks in:
1. **Approval & Assembly:** A ticket is approved, and the Security and Platform teams physically come together to assemble the password and Hardware MFAs.
2. **Execution:** The team logs in, performs the emergency fix under screen recording, and logs out.
3. **Rotation:** Before closing out the ticket, the team *manually* rotates the root password, satisfying the audit requirements.

---

## Summary
The flow moves from **Manual Trust** -> **Automated Boundaries** -> **Networkless Security** -> **Isolated Workloads**. It ensures that by the time a single byte of data is placed in the organization, the environment is already fully constrained, continuously monitored, and completely immune to rogue root account usage.