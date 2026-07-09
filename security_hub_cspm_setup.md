# Security Hub CSPM Setup Guide — AWS Organization with Delegated Administrator

Scope: enable Security Hub CSPM org-wide as a findings aggregator for custom
AWS Config rules (CIS-derived). No Security Hub standards enabled, no CIS
conformance packs.

## Prerequisites

- AWS Organizations set up with all features enabled.
- AWS Config recorder enabled in every account/region in scope (done).
- AWS Config delegated administrator + aggregator configured (done).
- Decide a **home region** — use the same region as your Config aggregator.
- Admin access to the org management account (for step 1 only) and the
  security tooling account.

> Note: "Security Hub CSPM" is the renamed classic Security Hub. If the
> console prompts you to enable the newer unified "Security Hub", that is a
> separate product — this guide covers CSPM only.

## Step 1 — Designate the delegated administrator

Run from the **org management account**, in your home region. Use the same
account as your Config delegated admin (security tooling account).

```bash
aws securityhub enable-organization-admin-account \
  --admin-account-id <SECURITY_TOOLING_ACCOUNT_ID> \
  --region <HOME_REGION>
```

Verify:

```bash
aws securityhub list-organization-admin-accounts --region <HOME_REGION>
```

Console alternative: Security Hub CSPM → Settings → General →
"Delegated Administrator".

## Step 2 — Enable Security Hub CSPM with central configuration

Run from the **delegated administrator account**, in the home region.

1. Open the Security Hub CSPM console in the home region and choose
   **Enable Security Hub** (if prompted with standards pre-selected,
   deselect all — especially AWS Foundational Security Best Practices).
2. Go to **Settings → Configuration → Start central configuration**.
3. Confirm the **home region** and select **linked regions** (every region
   where your Config rules will run). Optionally enable "auto-link new
   regions".
4. Choose **central configuration** (delegated admin manages all accounts).

CLI equivalent for the finding aggregator (cross-region aggregation):

```bash
aws securityhub create-finding-aggregator \
  --region <HOME_REGION> \
  --region-linking-mode SPECIFIED_REGIONS \
  --regions <REGION_1> <REGION_2>
```

## Step 3 — Create a configuration policy with NO standards

This is the key step for your use case: Security Hub on everywhere,
zero standards enabled.

Console: Settings → Configuration → Policies → **Create policy** →
enable Security Hub, **deselect all standards**.

CLI:

```bash
aws securityhub create-configuration-policy \
  --region <HOME_REGION> \
  --name "org-baseline-no-standards" \
  --description "Security Hub enabled, no standards - custom Config rules only" \
  --configuration-policy '{
    "SecurityHub": {
      "ServiceEnabled": true,
      "EnabledStandardIdentifiers": [],
      "SecurityControlsConfiguration": {
        "DisabledSecurityControlIdentifiers": []
      }
    }
  }'
```

Associate the policy with the org root (or specific OUs):

```bash
aws securityhub start-configuration-policy-association \
  --region <HOME_REGION> \
  --configuration-policy-identifier <POLICY_ARN> \
  --target '{"RootId": "<r-xxxx>"}'
```

Check association status:

```bash
aws securityhub list-configuration-policy-associations --region <HOME_REGION>
```

Members show as `SUCCESS` once Security Hub is enabled in their
account/regions. New accounts joining the org inherit the policy
automatically.

## Step 4 — Confirm the AWS Config integration

The Config → Security Hub integration is enabled by default; no wiring
needed. Confirm it is listed:

```bash
aws securityhub list-enabled-products-for-import --region <HOME_REGION>
```

Look for `.../product/aws/config`.

How it works:

- AWS Config sends evaluation results for **managed and custom rules** to
  Security Hub via EventBridge, in the account/region where the rule runs.
- Findings arrive in AWS Security Finding Format (ASFF) with a
  `Compliance.Status` of PASSED / FAILED and update automatically on
  re-evaluation (archived when the resource or rule is deleted).
- Service-linked Config rules are excluded (not relevant — you have no
  standards or conformance packs).
- Findings flow member → delegated admin (org linking) and linked region →
  home region (aggregation).

## Step 5 — End-to-end validation

Before rolling out all custom rules:

1. Deploy one custom Config rule to a member account (StackSets or Config
   organization rule).
2. Create/modify a resource so it is non-compliant.
3. In the delegated admin account, home region, filter findings:
   Product name = `Config`, Compliance status = `FAILED`,
   Account = the member account.
4. Expect the finding within ~5 minutes of the rule evaluation.

```bash
aws securityhub get-findings \
  --region <HOME_REGION> \
  --filters '{
    "ProductName": [{"Value": "Config", "Comparison": "EQUALS"}],
    "ComplianceStatus": [{"Value": "FAILED", "Comparison": "EQUALS"}]
  }' \
  --max-items 10
```

## Step 6 — Operating model (consuming findings)

- **Security team**: work in the delegated admin account, home region.
  Triage via workflow status: `NEW` → `NOTIFIED` → `RESOLVED` /
  `SUPPRESSED` (accepted risk).
- **Account owners**: each member account sees its own findings locally —
  app teams self-serve without admin access.
- **Automation rules** (Settings → Automations): normalize severity per
  rule, auto-suppress known noise before humans see it.
- **EventBridge**: every finding emits a `Security Hub Findings - Imported`
  event in the admin account. Route CRITICAL/HIGH to Jira/ServiceNow/Slack,
  or trigger Lambda/SSM remediations.
- **SIEM / data lake** (optional): forward via partner integrations or
  Amazon Security Lake (ASFF → OCSF).

## Cost notes

- No standards enabled → no per-check charges.
- You pay for **finding ingestion events** only; first 10,000 per
  account/region/month are free.
- Config rule evaluation charges apply on the Config side as usual.

## Later: enabling the CIS standard

When ready, add `standards/cis-aws-foundations-benchmark/v/3.0.0` to
`EnabledStandardIdentifiers` in the configuration policy. Security Hub
deploys its own service-linked Config rules and runs checks itself — it
coexists with your custom CIS-derived rules without conflict.

## References

- Designating a delegated administrator:
  https://docs.aws.amazon.com/securityhub/latest/userguide/designate-orgs-admin-account.html
- Central configuration:
  https://docs.aws.amazon.com/securityhub/latest/userguide/start-central-configuration.html
- AWS Config integration:
  https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-internal-providers.html
- Security Hub best practices:
  https://aws.github.io/aws-security-services-best-practices/guides/security-hub/
