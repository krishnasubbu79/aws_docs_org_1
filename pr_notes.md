# Summary

Adds deny guardrails for AWS Backup vault protection across all vaults, with scoped exemptions for break-glass and automation roles. The policy enforces three tiers of control: destructive backup operations are restricted to `org-master-trust*` and `Admin` only; Vault Lock configuration changes on the monthly transit vaults additionally permit the `t0-*-monthly-transit-backup-chk-role-*` automation role; and recovery point lifecycle updates are restricted to the same three roles via a dedicated principal-scoped statement.

# Reason

Backup vaults and their recovery points are the last line of defense against data loss and ransomware-style destruction. Without an org-level guardrail, any principal with sufficient identity-policy permissions in a member account could delete vaults, remove recovery points, weaken vault access policies, or alter Vault Lock / retention lifecycle settings. This change centralizes those controls so they cannot be bypassed from within member accounts.

A dedicated statement for `backup:UpdateRecoveryPointLifecycle` is required because this action authorizes against **recovery point ARNs** (e.g. `arn:aws:ec2:*::snapshot/*`, `arn:aws:backup:*:*:recovery-point:*`), not vault ARNs. Vault-ARN-scoped `resources` / `not_resources` blocks silently fail to match this action — the original draft would have (a) never denied it on transit vaults and (b) denied it for the exempt automation role everywhere. Splitting it into a principal-scoped deny fixes both.

# What has changed

- **Statement 1 — `DenyBackupVaultDestructiveOperations`** (unchanged logic): denies `DeleteBackupVault`, `DeleteBackupVaultAccessPolicy`, `PutBackupVaultAccessPolicy`, `DeleteRecoveryPoint`, `DisassociateRecoveryPoint` on `*` for all principals except `org-master-trust*` and `Admin`.
- **Statement 2 — `DenyTransitVaultLockConfigChanges`**: denies `PutBackupVaultLockConfiguration` / `DeleteBackupVaultLockConfiguration` on the monthly transit vault ARNs (primary + secondary region), exempting `org-master-trust*`, `Admin`, and `t0-*-monthly-transit-backup-chk-role-*`. `UpdateRecoveryPointLifecycle` **removed** from this statement (resource-type mismatch, see Reason).
- **Statement 3 — `DenyNonTransitVaultLockConfigChanges`**: same two Vault Lock actions denied on all vaults **except** the transit vaults (`not_resources`), exempting only `org-master-trust*` and `Admin`. `UpdateRecoveryPointLifecycle` removed here as well.
- **Statement 4 — `DenyRecoveryPointLifecycleChanges`** (new): denies `backup:UpdateRecoveryPointLifecycle` on `*`, exempting `org-master-trust*`, `Admin`, and the transit chk-role. Principal-scoped because vault-scoping is not possible for this action in the policy language.

All exemptions use `ArnNotLikeIfExists` on `aws:PrincipalArn`, so the deny also applies when the key is absent (fail-closed).

# Production Impact

- **No impact** to AWS Backup service-side operation: automated lifecycle transitions and backup/copy jobs run service-side and are not principal API calls; service-linked roles are not affected by SCPs.
- **No impact** to the exempt roles' current workflows.
- **All other principals** lose the ability to: delete vaults/recovery points, modify vault access policies, change Vault Lock configuration, and update recovery point lifecycles — org-wide. Any team currently performing these actions with a non-exempt role will start receiving explicit-deny `AccessDenied` errors.
- **Known accepted tradeoff:** the chk-role's `UpdateRecoveryPointLifecycle` exemption is global across recovery points (cannot be vault-scoped in an SCP). Blast radius is contained by (a) the exemption covering only this single action, (b) the role's identity policy, and (c) its trust policy. Vault Lock (compliance mode) independently enforces min/max retention on locked vaults regardless of this exemption — SCP controls *who*, Vault Lock controls *what is permissible on locked vaults*.
- `not_resources` in Deny statements requires the September 2025 SCP full-IAM-language support; older internal SCP linters may falsely flag statement 3.

# Test Evidence

- [ ] `terraform validate` / `terraform plan` clean — no unintended resource changes beyond the policy document.
- [ ] Design verified against AWS Service Authorization Reference: `Put/DeleteBackupVaultLockConfiguration` authorize on the `backupVault` resource type (vault-ARN scoping valid); `UpdateRecoveryPointLifecycle` authorizes on the `recoveryPoint` resource type (confirmed via AWS's own vault-policy examples using `arn:aws:ec2:*:*:snapshot/*`).
- [ ] Canary OU test — non-exempt role: `DeleteRecoveryPoint`, `PutBackupVaultAccessPolicy`, `PutBackupVaultLockConfiguration` (transit + non-transit vault), `UpdateRecoveryPointLifecycle` → all return explicit deny. *(attach CLI output / CloudTrail events)*
- [ ] Canary OU test — chk-role: `UpdateRecoveryPointLifecycle` on a transit vault recovery point succeeds; `PutBackupVaultLockConfiguration` on a non-transit vault denied. *(attach evidence)*
- [ ] Canary OU test — Admin role: all actions permitted (SCP exemption confirmed). *(attach evidence)*
- [ ] IAM Policy Simulator / Access Analyzer policy validation: no errors or security warnings.
