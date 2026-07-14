# Summary

Adds org-level deny guardrails for AWS Backup vault protection. Destructive backup operations are restricted to `org-master-trust*` and `Admin`; Vault Lock configuration changes on the monthly transit vaults additionally permit `t0-*-monthly-transit-backup-chk-role-*`; recovery point lifecycle updates are restricted to the same three roles via a dedicated principal-scoped statement.

# Reason

Backup vaults and recovery points are the last line of defense against data loss. Without an org guardrail, any sufficiently privileged principal in a member account could delete vaults or recovery points, weaken vault access policies, or alter Vault Lock / lifecycle settings.

A dedicated statement for `backup:UpdateRecoveryPointLifecycle` is required because this action authorizes against recovery point ARNs (e.g. `arn:aws:ec2:*::snapshot/*`), not vault ARNs. Vault-scoped `resources`/`not_resources` silently fail to match it — the original draft would never have denied it on transit vaults and would have denied it for the exempt automation role everywhere.

# What has changed

- **Stmt 1 `DenyBackupVaultDestructiveOperations`** (unchanged): denies `DeleteBackupVault`, `Delete/PutBackupVaultAccessPolicy`, `DeleteRecoveryPoint`, `DisassociateRecoveryPoint` on `*`; exempts org-master-trust*, Admin.
- **Stmt 2 `DenyTransitVaultLockConfigChanges`**: denies `Put/DeleteBackupVaultLockConfiguration` on transit vault ARNs (primary + secondary region); exempts the three roles. `UpdateRecoveryPointLifecycle` removed (resource-type mismatch).
- **Stmt 3 `DenyNonTransitVaultLockConfigChanges`**: same two actions on all vaults except transit (`not_resources`); exempts org-master-trust*, Admin only.
- **Stmt 4 `DenyRecoveryPointLifecycleChanges`** (new): denies `UpdateRecoveryPointLifecycle` on `*`; exempts the three roles. Principal-scoped since vault-scoping is impossible for this action.

All exemptions use `ArnNotLikeIfExists` on `aws:PrincipalArn` (fail-closed when key is absent).

# Production Impact

- No impact to AWS Backup service-side operation (automated lifecycle transitions, backup/copy jobs) or to the exempt roles.
- All other principals lose these actions org-wide; non-exempt roles currently performing them will hit explicit-deny `AccessDenied`.
- Accepted tradeoff: the chk-role's lifecycle exemption is global across recovery points (cannot be vault-scoped in an SCP). Contained by the single-action scope plus the role's identity and trust policies. Vault Lock independently enforces retention on locked vaults — SCP controls *who*, Vault Lock controls *what*.
- `not_resources` in Deny requires the Sep-2025 SCP full-IAM-language support; older linters may falsely flag Stmt 3.

# Test Evidence

- [ ] `terraform validate` / `plan` clean.
- [x] Verified against AWS Service Authorization Reference: `Put/DeleteBackupVaultLockConfiguration` → `backupVault` resource type; `UpdateRecoveryPointLifecycle` → `recoveryPoint`.
- [ ] Canary OU, non-exempt role: all restricted actions → explicit deny (attach CLI/CloudTrail).
- [ ] Canary OU, chk-role: lifecycle update on transit recovery point succeeds; lock config on non-transit vault denied.
- [ ] Canary OU, Admin role: all actions permitted.
- [ ] IAM Access Analyzer policy validation: no errors/warnings.
