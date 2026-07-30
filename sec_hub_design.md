**ORGANIZATION SECURITY DESIGN**

# **AWS Security Hub Organization Design**

## *Phase 1 Config-only milestone and Phase 2 unified security target*

**Status:** Draft for design review

**Owner:** Security team / Cloud platform

**Author:** Krishnan
**ORGANIZATION SECURITY DESIGN**

# **AWS Security Hub Organization Design**

## *Phase 1 Config-only milestone and Phase 2 unified security target*

**Status:** Draft for design review

**Owner:** Security team / Cloud platform

**Author:** Krishnan

**Version:** 0.2

**Date:** 22 July 2026

**Organization:** New AWS Organization; initial approved footprint of eight Regions

| Design decision: Phase 1 retains customer-managed AWS Config and uses Security Hub CSPM only as the organization-wide finding aggregation layer for AWS Config managed and custom rule evaluations. Phase 2 introduces the new AWS Security Hub and broader security services only after explicit readiness and cost gates. |
| :---- |

| Phase 1 \- committed milestone | Phase 2 \- target state |
| :---- | :---- |
| AWS Config recorders and rules remain. Security Hub CSPM is enabled with no standards. AWS Config is the only accepted finding source. | New AWS Security Hub, FSBP, GuardDuty, Inspector, Macie, exposure analytics, OCSF workflows and operational alerting. |

This document supersedes design draft v0.1 for implementation planning. AWS service behavior and references were validated on 22 July 2026\.

# **1\. Executive decision**

The organization will implement security finding aggregation in two controlled stages. Phase 1 solves the immediate requirement: preserve a small set of centrally governed AWS Config rules and make their evaluation results visible across all accounts and approved Regions from a dedicated security tooling account.

Phase 1 will not enable Security Hub CSPM standards, Security Hub CSPM controls, GuardDuty, Inspector, Macie, new Security Hub Essentials, exposure analytics, automated remediation, ticketing or SIEM export. The only accepted finding producer is AWS Config.

Phase 2 is the architectural target, not a launch dependency. It will add the new AWS Security Hub and selected AWS security capabilities after Phase 1 is stable, the operating model is proven and the forecast cost is approved.

## **1.1 Outcomes**

* A single delegated administrator account for AWS Config and Security Hub CSPM operations.  
* A consolidated home-Region view of AWS Config managed and custom rule evaluations from organization accounts and linked Regions.  
* Automatic coverage for newly created accounts without manual per-account Security Hub configuration.  
* No Security Hub-generated standards findings and no non-Config product findings during Phase 1\.  
* A Phase 1 foundation that can be extended to the new Security Hub without replacing the Config investment.

## **1.2 Fixed and open decisions**

| ID | Decision | Status |
| :---: | :---- | :---: |
| D-01 | Retain customer-managed AWS Config recorders and existing rules. | Fixed |
| D-02 | Use Security Hub CSPM, not new Security Hub Essentials, for Phase 1\. | Fixed |
| D-03 | Disable all Security Hub CSPM standards and accept only AWS Config findings. | Fixed |
| D-04 | Use the security tooling account as delegated administrator and home-Region operator. | Fixed |
| D-05 | Enable broader security signals and the new Security Hub in Phase 2 only. | Fixed |
| D-06 | Confirm the eight Region identifiers and home Region. | Open before build |
| D-07 | Approve the Config rule catalog, recording scope, rule owners and remediation targets. | Open before build |

## **1.3 Design principles**

* Least capability first: enable only what Phase 1 requires.  
* Organization policy over per-account configuration: make coverage reproducible and inherited.  
* Same account and Region model across phases: avoid delegated administrator or home-Region migration later.  
* Evidence before expansion: require coverage, latency, cost and operating evidence before Phase 2\.  
* Source allowlisting: treat unexpected finding products as configuration drift.

# **2\. Scope, assumptions and terminology**

## **2.1 Phase 1 scope**

| In scope | Explicitly out of scope |
| :---- | :---- |
| AWS Organizations trusted access and delegated administration for Config and Security Hub CSPM. | New AWS Security Hub Essentials and OCSF exposure analytics. |
| Customer-managed Config recorders, delivery channels, aggregator and managed/custom rules. | FSBP, CIS, NIST, PCI DSS and other CSPM standards or controls. |
| AWS Config to Security Hub CSPM service integration and finding aggregation. | GuardDuty, Inspector, Macie and third-party finding ingestion. |
| Identity Center access, Config-only saved views and operational coverage checks. | EventBridge alerting, ticketing, SOAR, Security Lake and SIEM export. |
| Existing and new organization accounts in approved Regions, including management-account coverage. | Automatic remediation and resource-level exception automation. |

## **2.2 Assumptions**

* The organization uses AWS Organizations without AWS Control Tower.  
* The initial footprint contains eight approved AWS Regions; the exact list is a deployment input.  
* The security tooling account is in a Security OU and is not the organization management account.  
* A log archive account or central S3 bucket is available for AWS Config delivery artifacts.  
* The Config rules and any custom Lambda functions can be deployed through version-controlled infrastructure as code.  
* Phase 1 analysts need central visibility but not automated alerting or remediation at launch.

## **2.3 Service terminology**

| Term | Meaning in this design |
| :---- | :---- |
| **AWS Security Hub CSPM** | The service that accepts AWS Config managed and custom rule evaluations, stores ASFF findings and provides account/Region aggregation in Phase 1\. |
| **New AWS Security Hub** | The later unified security product that uses OCSF, correlates signals and generates exposure findings. Deferred to Phase 2\. |
| **Customer-managed recorder** | The AWS Config recorder deployed and governed by this organization. It remains required for the organization's Config rules and evidence needs. |
| **Home Region** | The Region where the delegated administrator creates the finding aggregator and the security team views consolidated findings. |
| **Linked Region** | An approved Region whose Security Hub CSPM findings are replicated to the home Region. |

# **3\. Organization and account architecture**

The management account retains organization ownership and performs trusted-access and delegated-administrator registration. The security tooling account performs day-to-day Config and Security Hub CSPM administration. Member accounts run the customer-managed Config recorder and rules and become Security Hub CSPM members. The log archive account stores Config delivery artifacts but does not act as the findings administrator.

| Account / scope | Phase 1 responsibility | Security requirement |
| :---- | :---- | :---- |
| Organization management account | Enable trusted access; designate delegated administrators; participate as a monitored account. | Do not host security operations. Enable Config and CSPM coverage manually where automatic organization policy does not apply. |
| Security tooling account | AWS Config and Security Hub CSPM delegated administrator; Config aggregator; CSPM home-Region finding aggregator; analyst access. | Restricted administration, break-glass access, MFA and CloudTrail monitoring. |
| Log archive account | Own central Config delivery bucket and long-term configuration evidence. | Bucket policy allows delivery but restricts read/delete; retention and KMS policy are centrally governed. |
| Member workload accounts | Run Config recorder, delivery channel and rules; send Config evaluations to local-Region CSPM. | No manual finding setup; organization and StackSet policies govern coverage. |
| Sandbox accounts | Receive the same Phase 1 security coverage unless an approved OU policy says otherwise. | No silent exclusion from Config or CSPM aggregation. |

## **3.1 Delegated administration**

* Security Hub CSPM: the management account designates the security tooling account as the organization administrator.  
* AWS Config: designate the same security tooling account if it will manage organization rules, conformance packs or organization aggregators.  
* Phase 2 services: reserve the same account for new Security Hub, GuardDuty, Inspector and Macie unless a later design decision documents an exception.

## **3.2 Region model**

The home Region and eight approved Regions must be finalized before deployment. AWS Config evaluations are Regional and are delivered to Security Hub CSPM in the Region where the evaluation occurs. Cross-Region aggregation does not enable either service; both Config and CSPM must be enabled in each Region that is expected to contribute findings.

| Region rule: Every Region allowed by organization Region-deny controls must either be included in the security coverage matrix or have an explicit, approved exception. |
| :---- |

# **4\. Phase 1 detailed design**

![Phase 1 architecture: AWS Config rule evaluations flow through the AWS-managed EventBridge integration into Security Hub CSPM and aggregate to the security tooling account home Region.][image1]

***Figure 1 \- Phase 1 Config-only finding flow***

## **4.1 AWS Config recording**

* Deploy one customer-managed configuration recorder per account per approved Region.  
* Record all resource types required by the approved Config rule catalog. Add broader inventory types only where there is a separate evidence or governance requirement.  
* Record IAM global resource types only in the home Region to avoid duplicate recording and cost.  
* Use continuous recording for rules that require timely finding changes. Daily recording is acceptable only where the rule owner accepts up to 24 hours of evaluation delay.  
* Deliver configuration history and snapshots to the central S3 destination using the AWS Config service-linked role and approved KMS/bucket policies.  
* Create an organization Config aggregator in the security tooling account for Config-native inventory and compliance investigation. This is separate from the Security Hub CSPM finding aggregator.

## **4.2 Rule deployment and catalog**

Every retained rule must exist in a version-controlled rule catalog. The catalog is the source for recording scope, operational ownership and Phase 1 acceptance tests.

| Required rule field | Purpose |
| :---- | :---- |
| **Rule identifier and version** | Stable deployment and finding correlation. |
| **Managed / custom implementation** | Determines organization rule, conformance pack, Lambda and StackSet dependencies. |
| **Resource types and trigger** | Determines recorder scope and expected evaluation latency. |
| **Parameters** | Prevents inconsistent per-account rule behavior. |
| **Business severity and owner** | Drives triage order, escalation and remediation accountability. |
| **Expected compliant / noncompliant test** | Provides repeatable acceptance and regression testing. |
| **Exception mechanism** | Defines how justified exceptions are recorded without disabling organization-wide coverage. |

Prefer organization Config rules or organization conformance packs when they support the required rule. Use service-managed StackSets for recorders, delivery channels, custom-rule prerequisites and other account/Region resources that cannot be expressed as organization Config resources.

## **4.3 Security Hub CSPM organization configuration**

* Enable Security Hub CSPM in the delegated administrator, all member accounts and all approved Regions.  
* Create one custom central configuration policy associated with the organization root.  
* Set Security Hub CSPM to enabled but configure zero security standards and zero CSPM controls.  
* Do not use the recommended CSPM policy because it enables FSBP. When using API or CLI setup, explicitly disable default standards.  
* Create the CSPM finding aggregator in the selected home Region and link the remaining approved Regions.  
* Manually enable and add the organization management account as a member because it is not automatically enabled through normal member-account onboarding.

## **4.4 Integration allowlist**

After Security Hub CSPM is enabled, AWS service integrations can begin accepting findings automatically when the other service is present. Phase 1 therefore uses an explicit product allowlist rather than assuming that unused services will remain silent.

| Finding product | Phase 1 state | Control |
| :---- | :---: | :---- |
| AWS Config | Enabled | Required integration; verify status is Accepting findings in every approved Region. |
| AWS Health | Disabled | Explicitly stop import. |
| IAM Access Analyzer | Disabled | Explicitly stop import if enabled in the account. |
| GuardDuty | Disabled | Service and finding import deferred to Phase 2\. |
| Inspector | Disabled | Service and finding import deferred to Phase 2\. |
| Macie | Disabled | Service and finding import deferred to Phase 2\. |
| All other AWS / partner products | Disabled | No product import without an approved design change. |

# **5\. Config finding lifecycle and data behavior**

## **5.1 Native integration behavior**

AWS Config uses an AWS-managed EventBridge service integration to deliver managed and custom rule evaluations to Security Hub CSPM. CSPM transforms the evaluations into AWS Security Finding Format findings and enriches them with resource data on a best-effort basis.

| Behavior | Design implication |
| :---- | :---- |
| **Regional delivery** | The Config rule and Security Hub CSPM must be enabled in the same Region. The finding aggregator then replicates the finding to the home Region. |
| **No historical backfill** | Only evaluations performed after CSPM is enabled are sent. Run a fresh rule evaluation after integration activation. |
| **Usually visible within five minutes** | Use a 15-minute Phase 1 acceptance threshold for a completed Config evaluation under normal conditions. |
| **Updates on compliance change** | Do not expect periodic duplicate updates when compliance has not changed. |
| **Best-effort EventBridge delivery** | AWS retries unsuccessful deliveries for up to 24 hours or 185 attempts. Coverage monitoring must detect persistent gaps. |
| **Service-linked Config rules excluded** | Phase 1 findings come from customer-managed Config rules; CSPM service-linked control rules are not part of this design. |
| **Resource deletion does not archive Config findings** | Analysts must understand stale-resource behavior; use Config history and workflow status during investigation. |
| **Active finding aging** | CSPM deletes a Config finding 90 days after its most recent update. CSPM is not the long-term evidence store. |

## **5.2 Analyst view**

Create a saved finding view in the home Region that focuses the operational queue on current Config noncompliance. The view should filter by product AWS Config, failed/noncompliant status, active record state and workflow status New or Notified, and expose account, Region, rule name, resource type and resource identifier.

The rule catalog's business severity and owner remain authoritative because the integration's provider severity may not represent organization-specific risk. Resource owners remediate in the member account; Config re-evaluation supplies the compliance change to CSPM.

## **5.3 Access model**

| Role | Minimum capability |
| :---- | :---- |
| **Security analyst** | Read Config and CSPM findings across members; update workflow status and notes; no standards, integration or organization administration. |
| **Security platform administrator** | Manage delegated administration, central policy, integrations, aggregation and automation configuration. |
| **Cloud platform engineer** | Deploy and remediate Config recorder/rule infrastructure; read central coverage state. |
| **Engineering lead** | Read findings for owned accounts/resources; no suppression or global configuration rights. |

# **6\. Phase 1 implementation roadmap**

| Step | Work | Exit criterion |
| :---: | :---- | :---- |
| 1\. Foundation | Create/confirm Security OU, security tooling account and log archive destination. Enable trusted access and delegated administration. | Delegated administrators are visible; access and bucket policies are approved. |
| 2\. Config platform | Deploy recorders, delivery channels, aggregator and the approved rule catalog across existing accounts and eight Regions. | Recording and delivery are healthy; every rule reports an evaluation in each applicable account/Region. |
| 3\. CSPM base | Select home Region, link approved Regions, create root central policy with CSPM enabled and all standards disabled. | All member accounts are associated; no standard subscriptions or CSPM control findings exist. |
| 4\. Integration control | Verify AWS Config import and explicitly disable every other finding product. | Integration inventory matches the Config-only allowlist in every Region. |
| 5\. Finding validation | Trigger fresh Config evaluations and exercise compliant-to-noncompliant-to-compliant test cases. | Expected findings appear centrally and update within the accepted latency. |
| 6\. Operations | Deploy Identity Center permission sets, saved Config view, runbook, ownership map and cost/coverage monitoring. | Security and platform teams complete an operating walkthrough. |
| 7\. New-account test | Create or use a test account to validate StackSet auto-deployment and CSPM root-policy inheritance. | The account becomes fully covered without manual member-account configuration. |

## **6.1 Infrastructure as code boundaries**

* Management-account stack: trusted access, delegated administrator registration and organization policy prerequisites.  
* Security-tooling stack: Config aggregator, CSPM finding aggregator, central policy, integrations, IAM and monitoring.  
* Organization StackSet: Config recorder, delivery channel and any rule/runtime prerequisites in each account and Region.  
* Rule package: organization rules/conformance packs or StackSet-managed rules, parameters and test fixtures.  
* Environment inputs: organization ID, administrator account ID, log bucket/KMS ARNs, home Region, linked Regions, OU targets and rule catalog version.

## **6.2 Rollback**

Rollback must not destroy Config evidence. If Phase 1 activation fails, stop additional CSPM product imports, disassociate the affected CSPM configuration policy if necessary, and retain customer-managed Config recorders, delivery channels, rules and S3 history. Do not delete the central Config delivery bucket or Config aggregator as part of a CSPM rollback.

# **7\. Phase 1 acceptance and operations**

## **7.1 Milestone acceptance criteria**

| Area | Acceptance condition |
| :---- | :---- |
| **Coverage** | 100% of active organization accounts, including the management and security tooling accounts, are covered in all approved Regions. |
| **Config health** | Recorders are recording, delivery channels are healthy and required resource types are in scope. |
| **Rule health** | Every approved rule has a recent evaluation in each applicable account/Region and no unexplained deployment drift. |
| **CSPM configuration** | CSPM is enabled organization-wide with no standards subscriptions and no CSPM controls enabled. |
| **Source isolation** | AWS Config is the only product accepting findings. No GuardDuty, Inspector, Macie, Health, Access Analyzer or partner findings appear. |
| **Finding delivery** | A fresh noncompliant evaluation is visible in the member Region and home Region within 15 minutes under normal operation. |
| **Finding update** | Remediation and re-evaluation update the central finding as designed. |
| **New account** | A test account receives Config and CSPM coverage without manual per-account Security Hub configuration. |
| **Access** | Analyst, platform administrator and read-only roles pass least-privilege access tests. |
| **Cost** | A 30-day forecast and budget alarm are approved before Phase 1 is declared steady state. |

## **7.2 Operational cadence**

| Cadence | Activity |
| :---- | :---- |
| **Daily** | Review new failed Config findings for rules classified critical or high in the rule catalog; investigate delivery or recorder health alarms. |
| **Weekly** | Review all open Config findings, aging, rule evaluation freshness and account/Region coverage. |
| **Monthly** | Review exceptions, StackSet drift, product allowlist, rule catalog changes, costs and new AWS Regions/accounts. |
| **Quarterly** | Test management-account coverage, new-account onboarding and one end-to-end noncompliance/remediation scenario. |

## **7.3 Cost controls**

* Model customer-managed Config configuration-item and rule-evaluation volume by account and Region.  
* Record only the resource types required by the rule catalog unless broader Config history is explicitly required.  
* Review continuous versus daily recording per resource type against finding latency requirements.  
* Track Security Hub CSPM finding-ingestion volume; no CSPM security-check charge should arise from standards because standards remain disabled.  
* Set budgets and anomaly alerts for AWS Config, Security Hub CSPM, S3 and KMS before organization-wide deployment.

# **8\. Phase 2 target architecture**

![Phase 2 target: AWS Config, Security Hub CSPM, GuardDuty, Inspector and Macie findings feed the new AWS Security Hub for OCSF correlation, exposure analytics, alerting and export.][image2]

***Figure 2 \- Phase 2 unified security target***

| Phase 2 gate: Phase 2 is not automatically activated by completion of Phase 1\. It requires a separate design approval covering capabilities, Regions, pricing, operating ownership, alert routing and exception handling. |
| :---- |

## **8.1 Target capabilities**

| Capability | Target use | Phase 1 foundation reused |
| :---- | :---- | :---- |
| New AWS Security Hub | OCSF findings, resource inventory, risk correlation and exposures. | Same security tooling account, account hierarchy and home/linked Region decision. |
| Security Hub CSPM \+ FSBP | AWS best-practice controls and compliance scores. | Existing CSPM organization membership and central policy mechanism. |
| GuardDuty | Threat detection and selected protection plans. | Same delegated administrator, Region matrix and coverage monitoring pattern. |
| Inspector | EC2, ECR and Lambda vulnerability management. | Organization policy and resource ownership model. |
| Macie | S3 inventory, policy findings and controlled sensitive-data discovery. | Account/Region coverage matrix and central security ownership. |
| EventBridge operations | Critical exposure/threat alerts, durable tickets and later SOAR. | Phase 1 workflow ownership and Identity Center roles. |
| Security Lake / SIEM | Longer-term OCSF retention and investigation. | Phase 1 evidence, source inventory and log archive account. |

## **8.2 Recommended Phase 2 enablement order**

1. Use the Security Hub Cost Estimator and approve the selected Essentials/additional capabilities and Region footprint.  
2. Enable the new Security Hub delegated administrator and organization policy using the existing security tooling account.  
3. Enable FSBP through a revised CSPM central policy; keep the existing Config integration and rule catalog unchanged.  
4. Enable Inspector scan types and GuardDuty protection plans with documented feature-specific cost and runtime decisions.  
5. Enable Macie inventory and controlled automated discovery only after S3 scope and exclusions are approved.  
6. Validate OCSF finding normalization and exposure generation before making exposures the primary analyst queue.  
7. Deploy EventBridge V2 filtering, deduplication, retry/DLQ and ticket ownership for critical exposures and selected threats.  
8. Add Security Lake or SIEM export only when retention and investigation requirements justify it.

## **8.3 Phase 2 decisions that must not be implicit**

* GuardDuty protection plans by workload type and Region.  
* Inspector EC2 scan mode, ECR rescan duration, Lambda standard scanning and optional Lambda code scanning.  
* Macie account/bucket scope, automated discovery exclusions and cost ceiling.  
* FSBP control tuning policy, exception authority and treatment of newly released controls.  
* New Security Hub plan/additional capabilities, including network scanning where applicable.  
* Critical alert destination, acknowledgement SLA, escalation, deduplication and durable ticket ownership.

# **9\. Risks and controls**

| Risk | Impact | Required control |
| :---- | :---- | :---- |
| Default standards enabled accidentally | CSPM creates unapproved findings and security-check costs. | Use a custom central policy and explicit no-default-standards setting; test for zero standard subscriptions. |
| Automatic non-Config integrations | Phase 1 receives Health, Access Analyzer or other findings. | Enforce and monitor the Config-only product allowlist in every Region. |
| No finding backfill | Existing Config noncompliance is absent after CSPM activation. | Trigger fresh rule evaluations after integration enablement and validate each catalog rule. |
| Regional coverage gap | Config evaluates a resource but CSPM is disabled in that Region, so no central finding appears. | Maintain one approved Region matrix and validate both services plus aggregation. |
| Duplicate global-resource recording | Unnecessary Config cost and duplicate evidence. | Record IAM global resource types in the home Region only. |
| Stale findings after resource deletion | Analysts may act on resources that no longer exist. | Consult Config history and resource inventory; use workflow status and periodic aging review. |
| Management account omitted | The most privileged account has no central Config finding coverage. | Enable Config and CSPM manually and include it in coverage tests. |
| Phase 2 scope creep | Essentials or other capabilities create unplanned cost and findings. | Require a separate Phase 2 design approval and organization policy change. |
| Rule owner not defined | Noncompliance accumulates without remediation accountability. | Do not admit a rule to the catalog without an owner, severity, test and remediation target. |

## **9.1 Security and audit controls**

* All management changes are made through version-controlled infrastructure as code or approved administrative runbooks.  
* CloudTrail organization logging and protected central retention are prerequisites for production operation, even though the logging design is maintained separately.  
* Delegated administrator and integration changes generate reviewable CloudTrail activity and configuration drift alerts.  
* KMS, S3 and IAM policies prevent member accounts from altering or deleting centralized Config evidence.  
* Security Hub workflow suppression is restricted to authorized roles and does not replace the formal exception register.

## **9.2 Open items before Phase 1 approval**

| Open item | Owner | Due before |
| :---- | :---: | :---: |
| Confirm home Region and exact eight-Region list. | Security / Cloud platform | Implementation step 2 |
| Approve Config rule catalog and required resource types. | Security control owners | Implementation step 2 |
| Confirm Config delivery bucket, KMS key and retention. | Logging platform | Implementation step 2 |
| Approve continuous/daily recording decisions. | Security / FinOps | Implementation step 2 |
| Define analyst workflow and per-rule remediation targets. | Security operations | Implementation step 6 |
| Approve Phase 1 budget and anomaly thresholds. | FinOps / Security | Steady state |

# **10\. References**

AWS documentation and pricing behavior were checked on 22 July 2026\. Service behavior can change; implementation code should pin intended settings explicitly and the design should be revalidated before Phase 2\.

9. [AWS service integrations with Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-internal-providers.html)  
10. [Enabling Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-settingup.html)  
11. [Understanding integrations in Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-findings-providers.html)  
12. [Enabling and configuring AWS Config for Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-setup-prereqs.html)  
13. [Understanding central configuration in Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/central-configuration-intro.html)  
14. [Understanding cross-Region aggregation](https://docs.aws.amazon.com/securityhub/latest/userguide/security-hub-region-aggregation.html)  
15. [Enabling the new AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-v2-enable.html)  
16. [Managing Security Hub organization configuration](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-v2-da-policy.html)  
17. [Security Hub and OCSF](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-ocsf.html)  
18. [AWS Security Hub pricing](https://aws.amazon.com/security-hub/pricing/)  
19. [AWS Security Hub CSPM pricing](https://aws.amazon.com/security-hub/cspm/pricing/)  
20. [AWS Config pricing](https://aws.amazon.com/config/pricing/)

# **Appendix A. Phase boundary summary**

| Capability | Phase 1 | Phase 2 target |
| :---- | :---- | :---- |
| Customer-managed AWS Config | Enabled; authoritative rule engine and evidence source | Retained |
| Security Hub CSPM | Enabled; no standards; Config import only | Enabled; FSBP and approved controls |
| New AWS Security Hub | Disabled | Enabled with approved plan/capabilities |
| GuardDuty | Disabled | Selected protection plans |
| Inspector | Disabled | Selected EC2/ECR/Lambda scan types |
| Macie | Disabled | Controlled S3 coverage and discovery |
| Finding format | ASFF in Security Hub CSPM | OCSF in new Security Hub, with CSPM integration |
| Primary queue | Config-only CSPM saved view | Exposure and threat queues |
| Alerting | Not enabled by default | EventBridge V2 to durable ticket/notification targets |
| Long-term analytics | Config S3 evidence; no SIEM export | Security Lake/SIEM when approved |

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>
**Version:** 0.2

**Date:** 22 July 2026

**Organization:** New AWS Organization; initial approved footprint of eight Regions

| Design decision: Phase 1 retains customer-managed AWS Config and uses Security Hub CSPM only as the organization-wide finding aggregation layer for AWS Config managed and custom rule evaluations. Phase 2 introduces the new AWS Security Hub and broader security services only after explicit readiness and cost gates. |
| :---- |

| Phase 1 \- committed milestone | Phase 2 \- target state |
| :---- | :---- |
| AWS Config recorders and rules remain. Security Hub CSPM is enabled with no standards. AWS Config is the only accepted finding source. | New AWS Security Hub, FSBP, GuardDuty, Inspector, Macie, exposure analytics, OCSF workflows and operational alerting. |

This document supersedes design draft v0.1 for implementation planning. AWS service behavior and references were validated on 22 July 2026\.

# **1\. Executive decision**

The organization will implement security finding aggregation in two controlled stages. Phase 1 solves the immediate requirement: preserve a small set of centrally governed AWS Config rules and make their evaluation results visible across all accounts and approved Regions from a dedicated security tooling account.

Phase 1 will not enable Security Hub CSPM standards, Security Hub CSPM controls, GuardDuty, Inspector, Macie, new Security Hub Essentials, exposure analytics, automated remediation, ticketing or SIEM export. The only accepted finding producer is AWS Config.

Phase 2 is the architectural target, not a launch dependency. It will add the new AWS Security Hub and selected AWS security capabilities after Phase 1 is stable, the operating model is proven and the forecast cost is approved.

## **1.1 Outcomes**

* A single delegated administrator account for AWS Config and Security Hub CSPM operations.  
* A consolidated home-Region view of AWS Config managed and custom rule evaluations from organization accounts and linked Regions.  
* Automatic coverage for newly created accounts without manual per-account Security Hub configuration.  
* No Security Hub-generated standards findings and no non-Config product findings during Phase 1\.  
* A Phase 1 foundation that can be extended to the new Security Hub without replacing the Config investment.

## **1.2 Fixed and open decisions**

| ID | Decision | Status |
| :---: | :---- | :---: |
| D-01 | Retain customer-managed AWS Config recorders and existing rules. | Fixed |
| D-02 | Use Security Hub CSPM, not new Security Hub Essentials, for Phase 1\. | Fixed |
| D-03 | Disable all Security Hub CSPM standards and accept only AWS Config findings. | Fixed |
| D-04 | Use the security tooling account as delegated administrator and home-Region operator. | Fixed |
| D-05 | Enable broader security signals and the new Security Hub in Phase 2 only. | Fixed |
| D-06 | Confirm the eight Region identifiers and home Region. | Open before build |
| D-07 | Approve the Config rule catalog, recording scope, rule owners and remediation targets. | Open before build |

## **1.3 Design principles**

* Least capability first: enable only what Phase 1 requires.  
* Organization policy over per-account configuration: make coverage reproducible and inherited.  
* Same account and Region model across phases: avoid delegated administrator or home-Region migration later.  
* Evidence before expansion: require coverage, latency, cost and operating evidence before Phase 2\.  
* Source allowlisting: treat unexpected finding products as configuration drift.

# **2\. Scope, assumptions and terminology**

## **2.1 Phase 1 scope**

| In scope | Explicitly out of scope |
| :---- | :---- |
| AWS Organizations trusted access and delegated administration for Config and Security Hub CSPM. | New AWS Security Hub Essentials and OCSF exposure analytics. |
| Customer-managed Config recorders, delivery channels, aggregator and managed/custom rules. | FSBP, CIS, NIST, PCI DSS and other CSPM standards or controls. |
| AWS Config to Security Hub CSPM service integration and finding aggregation. | GuardDuty, Inspector, Macie and third-party finding ingestion. |
| Identity Center access, Config-only saved views and operational coverage checks. | EventBridge alerting, ticketing, SOAR, Security Lake and SIEM export. |
| Existing and new organization accounts in approved Regions, including management-account coverage. | Automatic remediation and resource-level exception automation. |

## **2.2 Assumptions**

* The organization uses AWS Organizations without AWS Control Tower.  
* The initial footprint contains eight approved AWS Regions; the exact list is a deployment input.  
* The security tooling account is in a Security OU and is not the organization management account.  
* A log archive account or central S3 bucket is available for AWS Config delivery artifacts.  
* The Config rules and any custom Lambda functions can be deployed through version-controlled infrastructure as code.  
* Phase 1 analysts need central visibility but not automated alerting or remediation at launch.

## **2.3 Service terminology**

| Term | Meaning in this design |
| :---- | :---- |
| **AWS Security Hub CSPM** | The service that accepts AWS Config managed and custom rule evaluations, stores ASFF findings and provides account/Region aggregation in Phase 1\. |
| **New AWS Security Hub** | The later unified security product that uses OCSF, correlates signals and generates exposure findings. Deferred to Phase 2\. |
| **Customer-managed recorder** | The AWS Config recorder deployed and governed by this organization. It remains required for the organization's Config rules and evidence needs. |
| **Home Region** | The Region where the delegated administrator creates the finding aggregator and the security team views consolidated findings. |
| **Linked Region** | An approved Region whose Security Hub CSPM findings are replicated to the home Region. |

# **3\. Organization and account architecture**

The management account retains organization ownership and performs trusted-access and delegated-administrator registration. The security tooling account performs day-to-day Config and Security Hub CSPM administration. Member accounts run the customer-managed Config recorder and rules and become Security Hub CSPM members. The log archive account stores Config delivery artifacts but does not act as the findings administrator.

| Account / scope | Phase 1 responsibility | Security requirement |
| :---- | :---- | :---- |
| Organization management account | Enable trusted access; designate delegated administrators; participate as a monitored account. | Do not host security operations. Enable Config and CSPM coverage manually where automatic organization policy does not apply. |
| Security tooling account | AWS Config and Security Hub CSPM delegated administrator; Config aggregator; CSPM home-Region finding aggregator; analyst access. | Restricted administration, break-glass access, MFA and CloudTrail monitoring. |
| Log archive account | Own central Config delivery bucket and long-term configuration evidence. | Bucket policy allows delivery but restricts read/delete; retention and KMS policy are centrally governed. |
| Member workload accounts | Run Config recorder, delivery channel and rules; send Config evaluations to local-Region CSPM. | No manual finding setup; organization and StackSet policies govern coverage. |
| Sandbox accounts | Receive the same Phase 1 security coverage unless an approved OU policy says otherwise. | No silent exclusion from Config or CSPM aggregation. |

## **3.1 Delegated administration**

* Security Hub CSPM: the management account designates the security tooling account as the organization administrator.  
* AWS Config: designate the same security tooling account if it will manage organization rules, conformance packs or organization aggregators.  
* Phase 2 services: reserve the same account for new Security Hub, GuardDuty, Inspector and Macie unless a later design decision documents an exception.

## **3.2 Region model**

The home Region and eight approved Regions must be finalized before deployment. AWS Config evaluations are Regional and are delivered to Security Hub CSPM in the Region where the evaluation occurs. Cross-Region aggregation does not enable either service; both Config and CSPM must be enabled in each Region that is expected to contribute findings.

| Region rule: Every Region allowed by organization Region-deny controls must either be included in the security coverage matrix or have an explicit, approved exception. |
| :---- |

# **4\. Phase 1 detailed design**

![Phase 1 architecture: AWS Config rule evaluations flow through the AWS-managed EventBridge integration into Security Hub CSPM and aggregate to the security tooling account home Region.][image1]

***Figure 1 \- Phase 1 Config-only finding flow***

## **4.1 AWS Config recording**

* Deploy one customer-managed configuration recorder per account per approved Region.  
* Record all resource types required by the approved Config rule catalog. Add broader inventory types only where there is a separate evidence or governance requirement.  
* Record IAM global resource types only in the home Region to avoid duplicate recording and cost.  
* Use continuous recording for rules that require timely finding changes. Daily recording is acceptable only where the rule owner accepts up to 24 hours of evaluation delay.  
* Deliver configuration history and snapshots to the central S3 destination using the AWS Config service-linked role and approved KMS/bucket policies.  
* Create an organization Config aggregator in the security tooling account for Config-native inventory and compliance investigation. This is separate from the Security Hub CSPM finding aggregator.

## **4.2 Rule deployment and catalog**

Every retained rule must exist in a version-controlled rule catalog. The catalog is the source for recording scope, operational ownership and Phase 1 acceptance tests.

| Required rule field | Purpose |
| :---- | :---- |
| **Rule identifier and version** | Stable deployment and finding correlation. |
| **Managed / custom implementation** | Determines organization rule, conformance pack, Lambda and StackSet dependencies. |
| **Resource types and trigger** | Determines recorder scope and expected evaluation latency. |
| **Parameters** | Prevents inconsistent per-account rule behavior. |
| **Business severity and owner** | Drives triage order, escalation and remediation accountability. |
| **Expected compliant / noncompliant test** | Provides repeatable acceptance and regression testing. |
| **Exception mechanism** | Defines how justified exceptions are recorded without disabling organization-wide coverage. |

Prefer organization Config rules or organization conformance packs when they support the required rule. Use service-managed StackSets for recorders, delivery channels, custom-rule prerequisites and other account/Region resources that cannot be expressed as organization Config resources.

## **4.3 Security Hub CSPM organization configuration**

* Enable Security Hub CSPM in the delegated administrator, all member accounts and all approved Regions.  
* Create one custom central configuration policy associated with the organization root.  
* Set Security Hub CSPM to enabled but configure zero security standards and zero CSPM controls.  
* Do not use the recommended CSPM policy because it enables FSBP. When using API or CLI setup, explicitly disable default standards.  
* Create the CSPM finding aggregator in the selected home Region and link the remaining approved Regions.  
* Manually enable and add the organization management account as a member because it is not automatically enabled through normal member-account onboarding.

## **4.4 Integration allowlist**

After Security Hub CSPM is enabled, AWS service integrations can begin accepting findings automatically when the other service is present. Phase 1 therefore uses an explicit product allowlist rather than assuming that unused services will remain silent.

| Finding product | Phase 1 state | Control |
| :---- | :---: | :---- |
| AWS Config | Enabled | Required integration; verify status is Accepting findings in every approved Region. |
| AWS Health | Disabled | Explicitly stop import. |
| IAM Access Analyzer | Disabled | Explicitly stop import if enabled in the account. |
| GuardDuty | Disabled | Service and finding import deferred to Phase 2\. |
| Inspector | Disabled | Service and finding import deferred to Phase 2\. |
| Macie | Disabled | Service and finding import deferred to Phase 2\. |
| All other AWS / partner products | Disabled | No product import without an approved design change. |

# **5\. Config finding lifecycle and data behavior**

## **5.1 Native integration behavior**

AWS Config uses an AWS-managed EventBridge service integration to deliver managed and custom rule evaluations to Security Hub CSPM. CSPM transforms the evaluations into AWS Security Finding Format findings and enriches them with resource data on a best-effort basis.

| Behavior | Design implication |
| :---- | :---- |
| **Regional delivery** | The Config rule and Security Hub CSPM must be enabled in the same Region. The finding aggregator then replicates the finding to the home Region. |
| **No historical backfill** | Only evaluations performed after CSPM is enabled are sent. Run a fresh rule evaluation after integration activation. |
| **Usually visible within five minutes** | Use a 15-minute Phase 1 acceptance threshold for a completed Config evaluation under normal conditions. |
| **Updates on compliance change** | Do not expect periodic duplicate updates when compliance has not changed. |
| **Best-effort EventBridge delivery** | AWS retries unsuccessful deliveries for up to 24 hours or 185 attempts. Coverage monitoring must detect persistent gaps. |
| **Service-linked Config rules excluded** | Phase 1 findings come from customer-managed Config rules; CSPM service-linked control rules are not part of this design. |
| **Resource deletion does not archive Config findings** | Analysts must understand stale-resource behavior; use Config history and workflow status during investigation. |
| **Active finding aging** | CSPM deletes a Config finding 90 days after its most recent update. CSPM is not the long-term evidence store. |

## **5.2 Analyst view**

Create a saved finding view in the home Region that focuses the operational queue on current Config noncompliance. The view should filter by product AWS Config, failed/noncompliant status, active record state and workflow status New or Notified, and expose account, Region, rule name, resource type and resource identifier.

The rule catalog's business severity and owner remain authoritative because the integration's provider severity may not represent organization-specific risk. Resource owners remediate in the member account; Config re-evaluation supplies the compliance change to CSPM.

## **5.3 Access model**

| Role | Minimum capability |
| :---- | :---- |
| **Security analyst** | Read Config and CSPM findings across members; update workflow status and notes; no standards, integration or organization administration. |
| **Security platform administrator** | Manage delegated administration, central policy, integrations, aggregation and automation configuration. |
| **Cloud platform engineer** | Deploy and remediate Config recorder/rule infrastructure; read central coverage state. |
| **Engineering lead** | Read findings for owned accounts/resources; no suppression or global configuration rights. |

# **6\. Phase 1 implementation roadmap**

| Step | Work | Exit criterion |
| :---: | :---- | :---- |
| 1\. Foundation | Create/confirm Security OU, security tooling account and log archive destination. Enable trusted access and delegated administration. | Delegated administrators are visible; access and bucket policies are approved. |
| 2\. Config platform | Deploy recorders, delivery channels, aggregator and the approved rule catalog across existing accounts and eight Regions. | Recording and delivery are healthy; every rule reports an evaluation in each applicable account/Region. |
| 3\. CSPM base | Select home Region, link approved Regions, create root central policy with CSPM enabled and all standards disabled. | All member accounts are associated; no standard subscriptions or CSPM control findings exist. |
| 4\. Integration control | Verify AWS Config import and explicitly disable every other finding product. | Integration inventory matches the Config-only allowlist in every Region. |
| 5\. Finding validation | Trigger fresh Config evaluations and exercise compliant-to-noncompliant-to-compliant test cases. | Expected findings appear centrally and update within the accepted latency. |
| 6\. Operations | Deploy Identity Center permission sets, saved Config view, runbook, ownership map and cost/coverage monitoring. | Security and platform teams complete an operating walkthrough. |
| 7\. New-account test | Create or use a test account to validate StackSet auto-deployment and CSPM root-policy inheritance. | The account becomes fully covered without manual member-account configuration. |

## **6.1 Infrastructure as code boundaries**

* Management-account stack: trusted access, delegated administrator registration and organization policy prerequisites.  
* Security-tooling stack: Config aggregator, CSPM finding aggregator, central policy, integrations, IAM and monitoring.  
* Organization StackSet: Config recorder, delivery channel and any rule/runtime prerequisites in each account and Region.  
* Rule package: organization rules/conformance packs or StackSet-managed rules, parameters and test fixtures.  
* Environment inputs: organization ID, administrator account ID, log bucket/KMS ARNs, home Region, linked Regions, OU targets and rule catalog version.

## **6.2 Rollback**

Rollback must not destroy Config evidence. If Phase 1 activation fails, stop additional CSPM product imports, disassociate the affected CSPM configuration policy if necessary, and retain customer-managed Config recorders, delivery channels, rules and S3 history. Do not delete the central Config delivery bucket or Config aggregator as part of a CSPM rollback.

# **7\. Phase 1 acceptance and operations**

## **7.1 Milestone acceptance criteria**

| Area | Acceptance condition |
| :---- | :---- |
| **Coverage** | 100% of active organization accounts, including the management and security tooling accounts, are covered in all approved Regions. |
| **Config health** | Recorders are recording, delivery channels are healthy and required resource types are in scope. |
| **Rule health** | Every approved rule has a recent evaluation in each applicable account/Region and no unexplained deployment drift. |
| **CSPM configuration** | CSPM is enabled organization-wide with no standards subscriptions and no CSPM controls enabled. |
| **Source isolation** | AWS Config is the only product accepting findings. No GuardDuty, Inspector, Macie, Health, Access Analyzer or partner findings appear. |
| **Finding delivery** | A fresh noncompliant evaluation is visible in the member Region and home Region within 15 minutes under normal operation. |
| **Finding update** | Remediation and re-evaluation update the central finding as designed. |
| **New account** | A test account receives Config and CSPM coverage without manual per-account Security Hub configuration. |
| **Access** | Analyst, platform administrator and read-only roles pass least-privilege access tests. |
| **Cost** | A 30-day forecast and budget alarm are approved before Phase 1 is declared steady state. |

## **7.2 Operational cadence**

| Cadence | Activity |
| :---- | :---- |
| **Daily** | Review new failed Config findings for rules classified critical or high in the rule catalog; investigate delivery or recorder health alarms. |
| **Weekly** | Review all open Config findings, aging, rule evaluation freshness and account/Region coverage. |
| **Monthly** | Review exceptions, StackSet drift, product allowlist, rule catalog changes, costs and new AWS Regions/accounts. |
| **Quarterly** | Test management-account coverage, new-account onboarding and one end-to-end noncompliance/remediation scenario. |

## **7.3 Cost controls**

* Model customer-managed Config configuration-item and rule-evaluation volume by account and Region.  
* Record only the resource types required by the rule catalog unless broader Config history is explicitly required.  
* Review continuous versus daily recording per resource type against finding latency requirements.  
* Track Security Hub CSPM finding-ingestion volume; no CSPM security-check charge should arise from standards because standards remain disabled.  
* Set budgets and anomaly alerts for AWS Config, Security Hub CSPM, S3 and KMS before organization-wide deployment.

# **8\. Phase 2 target architecture**

![Phase 2 target: AWS Config, Security Hub CSPM, GuardDuty, Inspector and Macie findings feed the new AWS Security Hub for OCSF correlation, exposure analytics, alerting and export.][image2]

***Figure 2 \- Phase 2 unified security target***

| Phase 2 gate: Phase 2 is not automatically activated by completion of Phase 1\. It requires a separate design approval covering capabilities, Regions, pricing, operating ownership, alert routing and exception handling. |
| :---- |

## **8.1 Target capabilities**

| Capability | Target use | Phase 1 foundation reused |
| :---- | :---- | :---- |
| New AWS Security Hub | OCSF findings, resource inventory, risk correlation and exposures. | Same security tooling account, account hierarchy and home/linked Region decision. |
| Security Hub CSPM \+ FSBP | AWS best-practice controls and compliance scores. | Existing CSPM organization membership and central policy mechanism. |
| GuardDuty | Threat detection and selected protection plans. | Same delegated administrator, Region matrix and coverage monitoring pattern. |
| Inspector | EC2, ECR and Lambda vulnerability management. | Organization policy and resource ownership model. |
| Macie | S3 inventory, policy findings and controlled sensitive-data discovery. | Account/Region coverage matrix and central security ownership. |
| EventBridge operations | Critical exposure/threat alerts, durable tickets and later SOAR. | Phase 1 workflow ownership and Identity Center roles. |
| Security Lake / SIEM | Longer-term OCSF retention and investigation. | Phase 1 evidence, source inventory and log archive account. |

## **8.2 Recommended Phase 2 enablement order**

1. Use the Security Hub Cost Estimator and approve the selected Essentials/additional capabilities and Region footprint.  
2. Enable the new Security Hub delegated administrator and organization policy using the existing security tooling account.  
3. Enable FSBP through a revised CSPM central policy; keep the existing Config integration and rule catalog unchanged.  
4. Enable Inspector scan types and GuardDuty protection plans with documented feature-specific cost and runtime decisions.  
5. Enable Macie inventory and controlled automated discovery only after S3 scope and exclusions are approved.  
6. Validate OCSF finding normalization and exposure generation before making exposures the primary analyst queue.  
7. Deploy EventBridge V2 filtering, deduplication, retry/DLQ and ticket ownership for critical exposures and selected threats.  
8. Add Security Lake or SIEM export only when retention and investigation requirements justify it.

## **8.3 Phase 2 decisions that must not be implicit**

* GuardDuty protection plans by workload type and Region.  
* Inspector EC2 scan mode, ECR rescan duration, Lambda standard scanning and optional Lambda code scanning.  
* Macie account/bucket scope, automated discovery exclusions and cost ceiling.  
* FSBP control tuning policy, exception authority and treatment of newly released controls.  
* New Security Hub plan/additional capabilities, including network scanning where applicable.  
* Critical alert destination, acknowledgement SLA, escalation, deduplication and durable ticket ownership.

# **9\. Risks and controls**

| Risk | Impact | Required control |
| :---- | :---- | :---- |
| Default standards enabled accidentally | CSPM creates unapproved findings and security-check costs. | Use a custom central policy and explicit no-default-standards setting; test for zero standard subscriptions. |
| Automatic non-Config integrations | Phase 1 receives Health, Access Analyzer or other findings. | Enforce and monitor the Config-only product allowlist in every Region. |
| No finding backfill | Existing Config noncompliance is absent after CSPM activation. | Trigger fresh rule evaluations after integration enablement and validate each catalog rule. |
| Regional coverage gap | Config evaluates a resource but CSPM is disabled in that Region, so no central finding appears. | Maintain one approved Region matrix and validate both services plus aggregation. |
| Duplicate global-resource recording | Unnecessary Config cost and duplicate evidence. | Record IAM global resource types in the home Region only. |
| Stale findings after resource deletion | Analysts may act on resources that no longer exist. | Consult Config history and resource inventory; use workflow status and periodic aging review. |
| Management account omitted | The most privileged account has no central Config finding coverage. | Enable Config and CSPM manually and include it in coverage tests. |
| Phase 2 scope creep | Essentials or other capabilities create unplanned cost and findings. | Require a separate Phase 2 design approval and organization policy change. |
| Rule owner not defined | Noncompliance accumulates without remediation accountability. | Do not admit a rule to the catalog without an owner, severity, test and remediation target. |

## **9.1 Security and audit controls**

* All management changes are made through version-controlled infrastructure as code or approved administrative runbooks.  
* CloudTrail organization logging and protected central retention are prerequisites for production operation, even though the logging design is maintained separately.  
* Delegated administrator and integration changes generate reviewable CloudTrail activity and configuration drift alerts.  
* KMS, S3 and IAM policies prevent member accounts from altering or deleting centralized Config evidence.  
* Security Hub workflow suppression is restricted to authorized roles and does not replace the formal exception register.

## **9.2 Open items before Phase 1 approval**

| Open item | Owner | Due before |
| :---- | :---: | :---: |
| Confirm home Region and exact eight-Region list. | Security / Cloud platform | Implementation step 2 |
| Approve Config rule catalog and required resource types. | Security control owners | Implementation step 2 |
| Confirm Config delivery bucket, KMS key and retention. | Logging platform | Implementation step 2 |
| Approve continuous/daily recording decisions. | Security / FinOps | Implementation step 2 |
| Define analyst workflow and per-rule remediation targets. | Security operations | Implementation step 6 |
| Approve Phase 1 budget and anomaly thresholds. | FinOps / Security | Steady state |

# **10\. References**

AWS documentation and pricing behavior were checked on 22 July 2026\. Service behavior can change; implementation code should pin intended settings explicitly and the design should be revalidated before Phase 2\.

9. [AWS service integrations with Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-internal-providers.html)  
10. [Enabling Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-settingup.html)  
11. [Understanding integrations in Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-findings-providers.html)  
12. [Enabling and configuring AWS Config for Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-setup-prereqs.html)  
13. [Understanding central configuration in Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/central-configuration-intro.html)  
14. [Understanding cross-Region aggregation](https://docs.aws.amazon.com/securityhub/latest/userguide/security-hub-region-aggregation.html)  
15. [Enabling the new AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-v2-enable.html)  
16. [Managing Security Hub organization configuration](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-v2-da-policy.html)  
17. [Security Hub and OCSF](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-ocsf.html)  
18. [AWS Security Hub pricing](https://aws.amazon.com/security-hub/pricing/)  
19. [AWS Security Hub CSPM pricing](https://aws.amazon.com/security-hub/cspm/pricing/)  
20. [AWS Config pricing](https://aws.amazon.com/config/pricing/)

# **Appendix A. Phase boundary summary**

| Capability | Phase 1 | Phase 2 target |
| :---- | :---- | :---- |
| Customer-managed AWS Config | Enabled; authoritative rule engine and evidence source | Retained |
| Security Hub CSPM | Enabled; no standards; Config import only | Enabled; FSBP and approved controls |
| New AWS Security Hub | Disabled | Enabled with approved plan/capabilities |
| GuardDuty | Disabled | Selected protection plans |
| Inspector | Disabled | Selected EC2/ECR/Lambda scan types |
| Macie | Disabled | Controlled S3 coverage and discovery |
| Finding format | ASFF in Security Hub CSPM | OCSF in new Security Hub, with CSPM integration |
| Primary queue | Config-only CSPM saved view | Exposure and threat queues |
| Alerting | Not enabled by default | EventBridge V2 to durable ticket/notification targets |
| Long-term analytics | Config S3 evidence; no SIEM export | Security Lake/SIEM when approved |

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>
