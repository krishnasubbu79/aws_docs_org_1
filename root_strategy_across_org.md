# AWS Root Account Security & Break-Glass Strategy (Audit-Ready)

**Applies to: All Accounts in Secure AWS Organization**

---

## 1. Purpose

This document defines the **mandatory, audit-compliant controls** for securing AWS root accounts and implementing a **break-glass access mechanism** across all accounts in the Secure AWS Organization.

The objectives are to:

* Eliminate unauthorized root usage
* Enforce **multi-party controlled emergency access**
* Ensure **full auditability and traceability**
* Minimize the root credential surface area across the organization

---

## 2. Two-Tier Root Model

This organization uses a **two-tier approach** to root account management, reflecting the architectural difference between the Management Account and all member accounts.

| Account Type | Root Credential State | Break-Glass Path |
|---|---|---|
| Management Account | Active (password + hardware MFA) | Split custody — physical assembly required |
| Member Accounts (all others) | Removed — centralized via org admin | IAM Centralized Root Access (privileged session from Management Account) |

### Rationale

* The Management Account is the root of trust for the entire organization. Its root credentials must be tightly controlled but must physically exist, as there is no higher authority to recover from.
* Member account root credentials introduce unnecessary operational overhead (hardware MFA per account, credential rotation per account) without adding security value when centralized root access is available. Removing them reduces the attack surface and simplifies governance.
* SCPs do not apply to the Management Account root user. Physical MFA and split custody are its only controls. SCPs do apply to root users in member accounts — but since credentials are removed, the point is moot.

---

## 3. Scope

* **Tier 1**: Management Account (active root credentials, split custody model)
* **Tier 2**: All member accounts — Security Account, Log Archive Account, Shared Services Account, all Workload Accounts (root credentials removed, centralized access model)

---

## 4. Root Email Requirements (All Accounts)

Regardless of tier, every account MUST use a dedicated, non-personal email address.

**Standard Format:**

```
aws-root+<account-name>@company.com
```

**Requirements:**

* Not mapped to any individual user
* Access restricted to authorized personnel only (security team)
* Not used for any other services or subscriptions
* MUST NOT depend on infrastructure hosted within the AWS Organization it protects (prevents circular dependency during an outage)
* Protected with MFA at the email provider level

Root emails for member accounts remain necessary even after root credentials are removed — AWS uses the email for account identity, billing notifications, and any future credential recovery if required.

---

## 5. Tier 1 — Management Account Root Controls

### 5.1 Credential Requirements

* **Password**: Minimum 32 characters, randomly generated, stored in a secure vault with restricted access. MUST NOT be fully known to any single individual.
* **MFA**: Hardware MFA devices only. Minimum 2 devices registered.

### 5.2 Split Custody Model

| Component | Custodian |
|---|---|
| Root password (vault access) | Security Team |
| MFA Device 1 | Security Team |
| MFA Device 2 | Platform / Infrastructure Team |

No single individual SHALL have simultaneous access to the full password and all MFA devices.

### 5.3 Allowed Break-Glass Scenarios

Root access is permitted ONLY under:

* Loss of IAM Identity Center (SSO) — no administrative path available
* Critical misconfiguration blocking all IAM access
* Organization-level recovery requiring management account root authority
* Approved compliance-driven emergency actions

### 5.4 Break-Glass Workflow — Management Account

#### Step 1 — Initiation

* Incident identified and formally documented
* Ticket created in approved system
* Justification recorded

#### Step 2 — Multi-Party Approval

Minimum required:

* Security Lead
* Platform / Infrastructure Lead
* (Optional) Compliance / Audit Lead

Approval MUST be documented and timestamped.

#### Step 3 — Credential Assembly

* Retrieve vault access and assemble the password
* Both MFA device custodians physically convene
* Confirm presence of all authorized personnel

#### Step 4 — Access Execution

* Log in as root user
* Perform ONLY the approved, documented action

**Mandatory controls:**

* Screen recording active for full session duration
* Actions strictly time-bound
* No unrelated changes permitted

#### Step 5 — Session Termination

* Log out immediately after task completion
* Verify no active sessions remain

#### Step 6 — Post-Access Rotation (MANDATORY)

* Rotate root password before logging out
* Re-register MFA devices if any device was compromised or replaced
* Update vault records

> AWS does not support programmatic root credential rotation. This MUST be performed manually in the AWS Management Console before the session ends.

#### Step 7 — Audit & Documentation

* Pull CloudTrail logs for the session
* Document: actions performed, session duration, personnel present
* Attach all evidence to the incident ticket

---

## 6. Tier 2 — Member Account Root Controls

### 6.1 Root Credential Removal (Mandatory Setup)

For all member accounts, root credentials MUST be removed using **IAM Centralized Root Access** (AWS Organizations feature). This is performed from the Management Account during initial account setup.

**Effect of removal:**

* Root password is deleted for the member account
* Root MFA devices are deregistered for the member account
* Root sign-in to the member account via the standard console login page is disabled
* All root-level actions in the member account can only be performed via a privileged session initiated from the Management Account

**How to enable:**

In the Management Account, navigate to IAM → Root access management → select the target account → Remove root credentials.

> This must be applied to every member account at provisioning time. It should be validated as part of account baseline checks.

### 6.2 Privileged Root Session — How It Works

When a root-only action is required in a member account, the org admin (authorized operator in the Management Account) can initiate a **temporary privileged root session** for that specific account.

* The session is created from the Management Account — no member account MFA is required
* AWS issues time-limited credentials scoped to the target member account
* The session is fully logged in CloudTrail under the event `sts:AssumeRoot` in the org trail
* The session expires automatically; there is no persistent root access

### 6.3 Allowed Break-Glass Scenarios (Member Accounts)

* Root-only configuration change that cannot be performed via IAM roles
* Account recovery where all IAM access has been lost
* Compliance-driven emergency action requiring root authority in a specific account

### 6.4 Break-Glass Workflow — Member Accounts (via Org Admin)

#### Step 1 — Initiation

* Incident identified; specific member account targeted
* Ticket created and justification documented

#### Step 2 — Multi-Party Approval

* Same approval requirements as Tier 1 (Security Lead + Platform Lead minimum)
* The Management Account operator who will initiate the session must be identified and logged

#### Step 3 — Privileged Session Initiation

* Authorized operator logs into the Management Account via IAM Identity Center (SSO) with appropriate permissions
* Navigate to IAM → Root access management → select the target member account
* Initiate a privileged root session — AWS returns temporary credentials

> No physical MFA assembly is required. The Management Account SSO session is the authentication gate. This makes the org admin credentials and MFA the effective control for member account root access.

#### Step 4 — Access Execution

* Use the privileged root session credentials to perform ONLY the approved action in the target account
* Screen recording active
* Actions time-bound and scoped to the approved action

#### Step 5 — Session Termination

* Privileged session expires automatically (AWS-enforced timeout)
* Confirm no active sessions remain in the target account

#### Step 6 — Post-Access Validation

* No password rotation required (member account has no root password)
* Confirm root credential removal is still in effect for the account

#### Step 7 — Audit & Documentation

* Review CloudTrail for `sts:AssumeRoot` events in the org trail — these capture the privileged session initiation and all actions performed
* Document: target account, actions performed, session duration, operator identity
* Attach evidence to the incident ticket

---

## 7. Preventive Controls (SCP Enforcement)

SCPs apply to root users in member accounts. Since member account root credentials are removed, SCPs serve as a defense-in-depth layer in the event of any future credential re-enablement.

*(Critical Note: SCPs do not apply to the root user of the Management Account. For the Management Account, physical MFA and split custody are the absolute final lines of defense.)*

### 7.1 Deny Actions (applied org-wide)

* Disabling AWS CloudTrail
* Disabling AWS Config
* Creating or attaching Internet Gateways
* Assigning public IP addresses
* Unauthorized AWS Backup restore operations
* IAM misuse (user creation, access keys)

---

## 8. Logging and Monitoring Requirements

### 8.1 CloudTrail

* Organization-wide trail MUST be enabled
* Logs MUST be centralized in the Log Archive account
* MUST cover all regions and include management events

### 8.2 Root Activity Monitoring

Alerts MUST be generated for the following events:

**Standard root console login (Management Account or any unexpected member account access):**

```
eventName = ConsoleLogin
AND userIdentity.type = Root
```

**Privileged root session initiation (member accounts via org admin):**

```
eventName = AssumeRoot
```

**Response for any root event:**

* Immediate notification to Security team
* Incident ticket created automatically
* Correlation check: does a matching approved ticket exist?

---

## 9. Account Provisioning Checklist

For every new member account, the following MUST be completed before the account is considered baseline-compliant:

- [ ] Root email assigned (plus-addressing format, non-personal)
- [ ] Root credentials removed via IAM Centralized Root Access (Management Account)
- [ ] Account enrolled in centralized root access management
- [ ] Verified: root console login is disabled for the account
- [ ] SCP guardrails confirmed active on the account's OU

For the Management Account (one-time):

- [ ] Root email assigned
- [ ] 32+ character password set and stored in vault (split access)
- [ ] Hardware MFA registered (minimum 2 devices, split custody)
- [ ] Break-glass procedure documented and communicated to custodians

---

## 10. Audit & Evidence Requirements

### Management Account

* Root MFA configuration evidence
* Vault access records (who has access, last reviewed)
* CloudTrail logs for all `ConsoleLogin` events where `userIdentity.type = Root`
* Break-glass approval records and post-access rotation evidence

### Member Accounts

* IAM Centralized Root Access enrollment confirmation (per account)
* CloudTrail `AssumeRoot` event log for any privileged sessions used
* Break-glass approval records linked to any `AssumeRoot` events

---

## 11. Operational Controls

### 11.1 Periodic Validation

* Conduct break-glass drills for both tiers (Management Account physical assembly, and member account privileged session flow)
* Validate Management Account MFA device availability and custodian availability
* Confirm root credential removal is still active for all member accounts (periodic audit)

### 11.2 Continuous Monitoring

* Alert on any `ConsoleLogin` where `userIdentity.type = Root`
* Alert on any `AssumeRoot` event — correlate against open approved tickets

---

## 12. Prohibited Practices

* Storing root credentials in shared documents or collaboration tools
* Using SMS-based MFA for any root account
* Re-enabling root credentials on member accounts without a formal change request
* Skipping post-access rotation after Management Account root use
* Using root (or privileged root sessions) for routine operational tasks
* Initiating a privileged root session without an approved ticket

---

## 13. Compliance Alignment

This policy supports:

* Multi-party authorization requirements
* Least privilege principles
* Audit traceability (both standard root login and `AssumeRoot` event paths are fully CloudTrail-logged)
* Attack surface minimization (root credentials removed from all but the Management Account)

---

## 14. Summary

| | Management Account | Member Accounts |
|---|---|---|
| Root credentials | Active (password + hardware MFA) | Removed |
| Break-glass path | Physical assembly (split custody) | Privileged session from Management Account |
| MFA requirement | Hardware MFA (2 devices, split custody) | None (managed via org admin SSO) |
| SCP applicability | No | Yes |
| CloudTrail event | `ConsoleLogin` (Root) | `sts:AssumeRoot` |
| Post-access rotation | Mandatory | Not applicable |

---

## 15. Future Enhancements

* Integration with automated approval workflows (e.g., Step Functions or ticketing system webhooks) to gate `AssumeRoot` session creation
* Automated alerting pipeline: `AssumeRoot` event → Security Hub finding → PagerDuty
* Periodic automated audit report: confirm root removal status for all member accounts

---
