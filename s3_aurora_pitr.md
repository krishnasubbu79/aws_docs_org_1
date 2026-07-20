# Cross-Account Backup Copy Architecture: S3 & Aurora PITR vs Snapshots

**Scope:** How continuous (PITR) and periodic (snapshot) backups behave in the source account (Account A), how copy dynamics differ between S3 and Aurora, and why snapshot materialization in Account A is a hard prerequisite for any cross-account copy into the data bunker (Account B).

---

## 1. Backup types in Account A — what they actually are

### S3

**Continuous (PITR):** AWS Backup fully manages the backup data. It works as a full baseline plus a running change-log fed by S3 object-level events through EventBridge. This provides point-in-time restore up to a 35-day maximum. The backup data physically lives in AWS Backup storage, encrypted with the vault's KMS key.

**Periodic (snapshot):** Not an independent rescan of the bucket. For both S3 backup types, the first backup is full and every subsequent backup is **incremental at object level** (per AWS documentation) — periodic recovery points are materialized against the backup data already held for the bucket, and all of it is encrypted with the vault's KMS key.

**Consequence — the same-vault rule:** AWS mandates that continuous and periodic backups of an S3 bucket **must reside in the same backup vault**. The rule is documented; AWS does not document its internal rationale. A likely explanation is the shared incremental lineage combined with vault-specific encryption, access, and lock controls, but that explanation should be treated as an inference. This means vault lock, retention bounds, and deny policies on that vault govern both backup types together — the PITR points and the snapshots cannot be given different immutability postures. Additionally, continuous backup for a bucket may be configured in only **one** backup plan.

**Prerequisite:** S3 Versioning must be enabled on the bucket. All object versions present at backup start (including delete markers and versions pending lifecycle expiry) are included in the recovery point and billed.

### Aurora

**Continuous (PITR):** A thin management wrapper over the native RDS automated backup (storage-level backup plus transaction logs). The data lives in RDS-service-managed storage; the recovery point in the AWS Backup vault is effectively a pointer. Vault-level controls (access policy, vault lock) still govern it.

**Periodic (snapshot):** A discrete, self-contained cluster snapshot with **no incremental relationship** to the PITR transaction-log stream. Aurora treats snapshots taken through AWS Backup as **manual DB cluster snapshots**; they are always full backups capturing the cluster volume size at creation. Snapshot storage is **free without limit while the snapshot's age is within the automated backup (PITR) retention period** of an active cluster; only snapshots outside that window bill, at full size per GB-month. (Internal storage mechanics such as copy-on-write are not documented by AWS — only this billing behavior is.)

**Consequence — no vault constraint:** Since the two artifacts share nothing, Aurora snapshots can be placed in a **dedicated staging/transit vault**, separate from the PITR recovery points. This enables tight IAM scoping: the bunker-facing role's `ListRecoveryPointsByBackupVault` permission can be restricted to just that vault ARN. (For S3, the role unavoidably gets read visibility into the shared vault and must filter recovery points by type.)

---

## 2. Why snapshots must be created in Account A first

Two constraints combine to force this:

1. **Continuous recovery points are never copyable on demand.** AWS Backup rejects on-demand `StartCopyJob` against a continuous recovery point — for any resource type, from any principal. A PITR chain is a rolling change-log, not a discrete artifact; there is nothing copyable until AWS Backup materializes a snapshot.

2. **The only sanctioned path for copying "from" a continuous backup is a copy action embedded in the backup plan rule** — in which case AWS Backup snapshots the resource on schedule and copies that snapshot (the copy lands as a periodic recovery point with no PITR capability). This path is **excluded by our design constraint**: Account A's configuration must not contain the destination. A plan-embedded copy action stores Account B's vault ARN in A's config, making the bunker discoverable to anyone who can read A's backup plans.

Therefore the architecture is:

- Account A runs **new daily periodic backup plans** (S3 and Aurora) with **no copy action**. To an auditor of A's *AWS Backup configuration*, this is unremarkable: daily snapshots into local vaults. **Scope of the claim, stated precisely:** Account B is absent from A's AWS Backup configuration (no plans, copy actions, or vault policies name B). B nonetheless remains discoverable in A through: (a) the assumable role's **IAM trust policy**, which contains B's account ID or role ARN; (b) the Aurora **CMK key policy/grants** authorizing B; (c) **CloudTrail** `StartCopyJob` events carrying the destination vault ARN; and (d) transient **copy-job history**. An intermediary account in the trust chain can obscure B's identity in IAM if required; the KMS and CloudTrail exposure is structural and intentional (audit posture).
- Account B **pulls**: its orchestrator assumes a generically-named role in A → `ListRecoveryPointsByBackupVault` (filter latest `COMPLETED` periodic point per resource) → `StartCopyJob` with B's vault ARN as the destination parameter → poll `DescribeCopyJob`.
- B's destination vault ARN is absent from A's AWS Backup plans. B can still be identified from A's IAM trust policy and Aurora CMK policy/grants, as well as from CloudTrail and copy-job history; that audit visibility is deliberate.

Escape hatch: the role in A retains `StartBackupJob` so B can force an immediate snapshot of current state when the latest scheduled one (up to ~24h old) isn't fresh enough.

The continuous backups themselves never leave Account A. They serve exactly one purpose: local fine-grained PITR. Copying a *past* state cross-account requires the workaround of PITR-restoring to a temporary resource in A, snapshotting it, and copying that.

---

## 3. Copy dynamics — where S3 and Aurora diverge

### S3: incremental transfer, independent chain in B

- First copy of a bucket into B's vault is a **full seed**; every subsequent copy transfers and stores only changed data (incremental at object level, matching S3's backup model). Daily copy cost is proportional to churn, not bucket size.
- B's vault builds its **own self-contained incremental chain** under B's CMK. Recovery points in B never reference blocks in A — if A is fully compromised and wiped, B restores completely. This is the property that makes the bunker a bunker.
- Every recovery point in B is **logically full**: any single point restores the complete bucket state as of its creation. "Incremental" describes physical storage layout only.
- Deleting older points in B is safe — AWS Backup reference-counts the underlying backup data and retains whatever surviving recovery points still need. Short retention (7–14 days) works.
- **Reseed risk:** if copy failures persist long enough for retention to expire *every* recovery point for a bucket in B, the next successful copy is a full seed again. Alarm on copy-job failures in B's orchestrator.
- **Silent PITR failure mode (operational watch-item):** S3 continuous backup depends on the bucket-level **EventBridge notification setting**. If it is disabled, S3 stops publishing object events, the continuous recovery window silently stops advancing, and AWS Backup raises **no alert or job failure**; changes made during the gap are never retroactively captured (periodic snapshot jobs keep running unaffected). Deleting the AWS Backup–managed rule (`AwsBackupManagedRule-N`) also stops continuous backup; on the plan rule's next trigger, AWS Backup recreates the managed rule and starts a **new** continuous backup. Controls, in order of authority: (1) directly verify `GetBucketNotificationConfiguration` returns `EventBridgeConfiguration` (its absence = disabled), e.g. via a scheduled check; (2) verify the presence and enabled state of the AWS Backup–managed EventBridge rule; (3) poll the continuous recovery point and alarm if its status becomes `STOPPED` — AWS does not publish a native CloudWatch latest-restorable-time metric for S3, so any staleness probe must be custom; (4) use the AWS Config managed rule `s3-event-notifications-enabled` and a CloudTrail-based EventBridge rule on `PutBucketNotificationConfiguration` events, both of which AWS Backup documentation recommends for this scenario. Note also that `put-bucket-notification-configuration` is a replace (not merge) operation — existing SNS/SQS/Lambda targets must be re-included.

### Aurora: full destination storage copy every time

- Each destination snapshot is logically full and independently billed. That storage behavior is distinct from cross-Region transfer metering, which can be incremental after the first copy.
- **In-account (A):** Aurora treats AWS Backup–created snapshots as manual snapshots and provides **unlimited free storage for snapshots within the automated backup (PITR) retention period**. With the selected **3-day retention**, a daily snapshot remains inside the 35-day PITR window and therefore adds **no Aurora snapshot-storage charge in A**. Billing begins only if a snapshot outlives the PITR retention window — at which point it is billed at **full snapshot size** per GB-month (snapshots are full backups, not incremental). Caveat: if the cluster is deleted, surviving snapshots lose free-window coverage and immediately bill at full size.
- **Cross-account (A → B):** Every copy in B is stored and billed as a **full, independent snapshot**; repeated Aurora snapshot copies are not incremental for destination storage. Thus B's steady-state snapshot inventory is approximately **cluster volume size × copies retained**. Transfer metering is different: if the copy is cross-Region, AWS states that data-transfer charges are **incremental** and Aurora transfers the minimum changed data required to construct each full destination copy. Same-Region cross-account copies have no cross-Region transfer charge.
- **Hard prerequisite for this encrypted-bunker design:** the cluster must be encrypted with a **customer-managed key (CMK)**. Note the current default landscape: Aurora clusters created **on or after February 18, 2026** are encrypted by default with an **AWS-owned key (SSE-RDS)**; the `aws/rds` AWS-managed key is now a legacy option. **Neither** the AWS-owned default **nor** `aws/rds` can be shared cross-account — both block the copy outright. Clusters on either non-CMK key must be migrated (snapshot → copy with CMK → restore) before this architecture works for them. Legacy unencrypted clusters are a separate exception: their snapshot copies remain unencrypted and do not meet this design's encryption requirement.

### Summary table

| Dimension | S3 | Aurora |
|---|---|---|
| Continuous backup data location | AWS Backup–managed storage (vault CMK) | RDS-managed storage (vault holds pointer) |
| Snapshot vs PITR relationship | Same incremental chain | Independent artifacts |
| Vault placement | Snapshot **must** share vault with PITR | Snapshot can use dedicated staging vault |
| Snapshot cost in A | Incremental at object level; workload-dependent | Free within PITR retention window; full-size billing only beyond it |
| Cross-account copy | Incremental after first seed | Full destination snapshot every time; cross-Region transfer incremental after first seed |
| Chain in B | Independent, self-contained | N/A (discrete snapshots) |
| Retention lever in B | Safe short retention; watch reseed | Short retention directly caps storage cost |
| KMS requirement | Destination vault may use a B CMK or `aws/backup` | Source cluster CMK authorizes B's AWS Backup service-linked role; B vault CMK required |

---

## 4. Cost model (internal reference)

Rates below are region-specific — pull current numbers from the AWS Backup and Aurora pricing pages for our region. The model (what is billed, what is free, what the drivers are) is the durable part; the doc-verified anchor rate is Aurora backup storage at ~$0.021/GB-month (us-east-1 reference).

**Notation:** `V` = source data size (bucket size / cluster volume), `Δ` = daily changed data, `R_A` / `R_B` = retention days in Account A / Account B.

### S3 cost model

| Component | What is billed | Driver |
|---|---|---|
| Continuous (PITR) in A | AWS Backup warm storage: full baseline + object-level change history for the 35-day (max) window | `V` + churn × window. High-churn buckets grow this significantly |
| Periodic snapshots in A | Incremental at object level against data already in the vault | Changed objects/versions plus request, EventBridge, and per-object minimum-billing effects |
| Copy to B — first run | Full seed transferred and stored | `V` |
| Copy to B — steady state | Object-level incremental transfer + storage | Approximately `Δ` per day; B's chain total ≈ `V + (Δ × R_B)`, subject to retained object versions and per-object billing minimums |
| Cross-region component | Inter-region data transfer per GB, only if B's vault is in a different region | Same-region cross-account copies avoid this |
| Restore | Per-GB restore fee at AWS Backup rates | Restore-test scope and frequency |

Cost notes specific to S3:

- Backup storage is billed at **AWS Backup rates, not S3 storage rates** — do not estimate from S3 Standard pricing.
- The recovery point includes **all object versions present at backup start**, including noncurrent versions, delete markers, and objects pending lifecycle expiry — and all of it is billed. Version-lifecycle hygiene on the source bucket directly reduces backup cost.
- AWS Backup bills each backed-up object and delete marker with a 128-KiB minimum. Large populations of small objects or delete markers can therefore dominate storage cost.
- Include S3 `GET`, `LIST`, and `HEAD` requests, EventBridge events, possible S3 Inventory charges for very large buckets, and retrieval/access effects for S3 Standard-IA, One Zone-IA, Glacier Instant Retrieval, and Intelligent-Tiering.
- No cold-tier transition for S3 backups (warm only; S3 is the only resource with warm *tiering*, evaluate separately if long retention is ever added).
- **Reseed = cost spike:** if all recovery points for a bucket expire in B (sustained copy failures + short `R_B`), the next copy re-bills a full `V` seed. The copy-failure alarm is a cost control, not just an availability control.

### Aurora cost model

| Component | What is billed | Driver |
|---|---|---|
| Continuous (PITR) in A | Incremental change records for the retention window, **minus free tier equal to latest cluster volume size** | Write/change volume, not IOPS count. `TotalBackupStorageBilled` is the truth source |
| Daily snapshots in A (3-day retention) | **No Aurora snapshot-storage charge.** The snapshots remain inside the 35-day automated backup retention window | No snapshot-storage charge while snapshot age remains inside PITR retention and the cluster exists |
| Snapshot escaping the window | Full snapshot size per GB-month (snapshots are full backups, never incremental) | Only if retention settings drift — see guardrails |
| Copy to B | Each destination snapshot is billed at full size; no free snapshot window applies to copies | Storage in B ≈ `V × R_B`; for cross-Region copies, initial seed plus incremental changed-data transfer charges |
| Cluster deletion event | All surviving snapshots instantly lose free-window coverage and bill at full size until deleted | Decommissioning runbooks must include snapshot cleanup |

Worked shape (symbolic): monthly Aurora backup spend ≈ `[PITR billed usage in A] + [V × R_B × rate_B] + [incremental transfer, if cross-Region]` — where the first term is change-driven and often near zero for low-churn clusters, and the second term is the dominant, size-driven cost of the entire architecture.

### Cost guardrails and monitoring

- Keep snapshot-plan retention (3 days) far below continuous retention — the no-snapshot-storage-charge property depends on the snapshot age remaining inside the PITR retention window, and margin protects against someone shrinking PITR retention later. Do **not** "use the free window" by raising snapshot retention toward 35 days: at parity, any PITR retention reduction converts the whole snapshot inventory into full-size billed storage at once.
- CloudWatch (per Aurora cluster): `SnapshotStorageUsed` is the **drift alarm** — it counts only snapshots beyond the automated retention window, so it should read **zero** in steady state, and any value above zero means snapshots are escaping the free window. `TotalBackupStorageBilled` (= PITR billed usage + snapshot billed usage − free tier) is a **trend metric, not a drift alarm**: it also rises with ordinary write churn in the PITR window, so an increase alone does not indicate retention drift. Enabling the snapshot plan should not move either metric.
- Alarm on copy-job failures in B's orchestrator (S3 reseed risk; Aurora silent RPO erosion).
- Primary levers, in order of impact: Aurora `R_B` (linear on the dominant cost), Aurora copy frequency (daily vs. weekly if cross-account RPO allows), S3 source-bucket version lifecycle, and region colocation of B's vault (eliminates inter-region transfer).

## 5. One-time prerequisites and hardening

**Foundation:**

- Both accounts in the same AWS Organization; cross-account backup enabled in AWS Backup settings via the management account.
- Account B vault access policy: allow `backup:CopyIntoBackupVault` from Account A (principal or `aws:PrincipalOrgID` condition).
- **Vault Lock on B's vault** — immutability is *not* automatic. Recovery points in B are deletable by B-side principals until AWS Backup Vault Lock is configured; use **compliance mode** (with min/max retention bounds) for the data-bunker guarantee. Governance mode is bypassable by privileged principals and is insufficient for the ransomware threat model.

**KMS responsibility matrix:**

| Element | Key | Policy must authorize |
|---|---|---|
| Aurora cluster / source snapshot (A) | **CMK, mandatory for this encrypted design** (the AWS-owned SSE-RDS default and legacy `aws/rds` both block this cross-account flow) | B's AWS Backup service-linked role/account for the copy, including `kms:DescribeKey`, `kms:CreateGrant`, `kms:Decrypt`, `kms:Encrypt`, `kms:GenerateDataKey*`, and `kms:ReEncrypt*`. The source key is not needed for a later restore after the B-owned copy has completed. |
| S3 source objects (A) | Source SSE-KMS key (if used) | A's AWS Backup service role for `kms:Decrypt` — AWS Backup decrypts with the source key and **re-encrypts with the vault's key** (documented behavior; expect `kms:Decrypt` CloudTrail events on the source key during backups) |
| A's staging vaults | Vault CMK | A's AWS Backup service role for S3 and other fully managed recovery points. Aurora snapshots inherit the cluster CMK; the staging-vault key does not re-encrypt them. |
| B's destination vault | **CMK required for Aurora**; S3 alone may use a B CMK or `aws/backup` | B's `AWSServiceRoleForBackup`, which performs the destination-side copy. A is authorized separately by B's vault access policy, not by granting A use of B's CMK. |
| Restore in B | B destination recovery-point key, or another B key selected where the restore API permits | B's restore role and the applicable RDS service role |

**B-controlled orchestrator role assumed into A — scope tightly:**

- Caller actions: `backup:ListRecoveryPointsByBackupVault` (resource-scoped to the staging vault ARNs), `backup:StartCopyJob`, `backup:DescribeCopyJob`, optional `backup:StartBackupJob`, and `iam:PassRole`. The orchestrator does not need data-plane KMS permissions merely because it starts a job.
- `StartCopyJob`: constrain the source vault with `Resource`, constrain the destination with `backup:CopyTargets` / `backup:CopyTargetOrgPaths` conditions where supported, and add explicit deny statements for all other vaults.
- `StartBackupJob`: constrain the destination to the exact staging-vault ARN. Do not rely on resource tags attached to `StartBackupJob` as the sole source-resource boundary; enforce the allowed resource set in the passed AWS Backup service role and in B's orchestrator input allowlist.
- `iam:PassRole`: restrict to the exact AWS Backup service-role ARN, with `iam:PassedToService = backup.amazonaws.com`.
- Consider `backup:CopyFromBackupVault` deny statements in SCPs or vault access policies on all *other* vaults in A, so only the designated staging vaults can ever be a copy source.

**AWS Backup service role in A (passed to jobs):**

- Grant only the source S3/RDS permissions, source-key KMS permissions, and AWS Backup permissions the job needs. Include the required `backup:CopyFromBackupVault` and `backup:CopyIntoBackupVault` permissions for copy operations, scoped as tightly as the APIs and condition keys permit.
- Prefer a dedicated custom role over the broad default role so B cannot use the orchestrator path to back up or copy unrelated resources in A.

**AWS Backup service-linked role in B:**

- B's `AWSServiceRoleForBackup` performs the destination-side copy. It needs access to B's destination-vault CMK and, for the Aurora copy, authorization on A's source cluster CMK. B's vault access policy separately permits the cross-account copy into the destination vault.
- The role name, trust policy, source CMK policy/grants, and audit records expose the relationship to an administrator in A. That is intended. Hiding the destination from AWS Backup plan configuration is achievable; making the cross-account trust relationship invisible to Account A administrators is neither achievable nor desirable.

## 6. Net design

PITR stays local to A (recovery granularity). A's periodic plans produce copyable artifacts with a destination-blind configuration. B initiates all copies on its own schedule, holds all orchestration state, and accumulates snapshot-grade recovery points (restore-to-creation-time, no PITR) under its own keys — with immutability **enforced by Vault Lock in compliance mode**, not assumed. A's AWS Backup plans remain destination-blind, while IAM/KMS policies and audit records retain the deliberate cross-account visibility described above. S3 copies remain incremental after the first seed; every Aurora copy is a full-size destination snapshot even though cross-Region transfer after the initial seed is incremental.
