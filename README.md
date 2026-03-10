# AWS FinOps Cost Allocation Investigation

This project documents a full FinOps investigation into an AWS cost allocation failure and the governance architecture built to prevent recurrence. Using Cost and Usage Reports (CUR), Athena SQL analysis, and CloudTrail forensics, I traced a 17% allocation gap to resources created without required tags via direct console action. The result was an event-driven governance control (CloudTrail to EventBridge to Lambda) that restored 100% cost attribution visibility and reduced detection latency from monthly reconciliation to under five minutes.

## TL;DR (For Hiring Managers)

Investigated a 17% AWS cost allocation gap caused by missing resource tags. Not overspend. Not a billing error. A reporting visibility failure.

Used CUR and Athena to isolate unallocated spend and CloudTrail forensics to trace the root cause to resources created via direct console action without required tags.

Implemented an event-driven governance control (CloudTrail to EventBridge to Lambda) that restored 100% cost attribution visibility and reduced mean time to detect (MTTD) from approximately 30 days to under 5 minutes.

The environment is intentionally small to control cost, but this failure pattern scales linearly. In enterprise environments where monthly spend ranges from $2M to $10M across 50 to 200 accounts, the same attribution gap can hide hundreds of thousands of dollars from Finance reporting.

## Architecture at a Glance

```
AWS Resource Creation Event
        │
        ▼
  AWS CloudTrail          ←── Captures all API-level events
        │
        ▼
Amazon EventBridge        ←── Filters for RunInstances / CreateBucket
        │
        ▼
  AWS Lambda              ←── Validates Environment, Project, Owner tags
        │
   ┌────┴────┐
   ▼         ▼
Slack:    Slack:
Finance   Resource Owner
Channel   (direct alert)
```

## Key Technologies

| Layer | Technology |
|---|---|
| Billing data | AWS Cost and Usage Report (CUR) |
| Query engine | Amazon Athena |
| Forensics | AWS CloudTrail |
| Event capture | Amazon EventBridge |
| Tag validation | AWS Lambda (Python) |
| Governance | AWS Organizations Tag Policies, SCP |
| Alerting | Slack (Finance and resource owner channels) |
| Reporting | Anthropic Claude (AI-assisted) |

## What This Project Demonstrates

- AWS Cost and Usage Report (CUR) analysis with Athena
- Cloud billing investigation methodology
- Tag compliance analysis and gap identification
- CloudTrail forensic investigation
- FinOps governance design and remediation architecture
- Multi-account architecture design and scale considerations
- AI-assisted FinOps reporting

## What Happened

- Total AWS spend: $315
- Spend visible to Finance via required allocation tags: $262
- $53 (approximately 17%) was billed correctly but invisible to allocation reports
- Finance could not reliably allocate costs by Environment, Project, or Owner

## How I Investigated

- Validated invoice accuracy to rule out billing discrepancies
- Queried Cost and Usage Reports (CUR) via Athena to isolate unallocated spend
- Identified EC2 and S3 resources missing required tags
- Used CloudTrail forensic analysis tracing API calls that created non-compliant resources
- Confirmed tags were missing at creation time, not removed later
- Applied AI-assisted analysis to generate an executive-ready investigation report

## How I Fixed It

- Corrected current-month data so Finance could close accurately
- Implemented an event-driven tagging compliance control (CloudTrail to EventBridge to Lambda)
- Reduced MTTD from approximately 30 days to under 5 minutes
- Prevented long-lived untagged resources through immediate detection and notification

## Outcome

- Restored 100% cost allocation visibility
- Reduced recurring manual reconciliation work
- Shifted governance from reactive cleanup to proactive prevention

Timeline note: This investigation was conducted in January 2026 using a sandbox AWS account. CloudTrail timestamps reflect real event times.

## How to Read This Case Study

This repository is written in the style of an internal FinOps investigation report, not a step-by-step tutorial. Screenshots document evidence and root cause. SQL queries emphasize decision-making, not report completeness. Remediation focuses on governance and system design. Dollar values are intentionally small but the failure patterns are production-real.

## Investigation Summary

| Phase | Objective | Tooling | Outcome |
|---|---|---|---|
| Invoice Validation | Confirm billing accuracy | Cost Explorer, CUR | Billing ruled out |
| Resource Isolation | Identify unallocated spend | Athena (CUR) | EC2 and S3 gaps isolated |
| Forensics | Identify creation source | CloudTrail | Untagged API calls found |
| Scope Expansion | Check other services | Athena (CUR) | Systemic issue confirmed |
| Remediation | Prevent recurrence | EventBridge, Lambda | Near-real-time detection |
| AI-Assisted Reporting | Generate executive report | Anthropic Claude | Finance-ready report produced |

## Context and Problem Statement

During monthly reconciliation, Finance identified a discrepancy. Total AWS spend was $315. Spend visible with required tags was $262. Unallocated spend was $53, approximately 17% of total. The invoice was accurate. The failure was in cost attribution, not cost generation.

## Phase 1: Invoice Validation

Objective: Confirm AWS billed correctly before investigating allocation.

Validated service totals in Cost Explorer, reconciled totals against CUR via Athena, and confirmed invoice totals matched exactly.

Conclusion: Billing was correct. The issue existed downstream.

![Invoice validation confirming $315 total spend](screenshots/01%20-%20invoice-total-315.png)

## Phase 2: Resource Isolation (CUR Analysis)

Objective: Identify resources missing required allocation tags.

Data flow:

```
AWS Account (Sandbox)
        │
        ▼
Cost & Usage Report (CUR)
Delivered daily to S3 billing bucket
        │
        ▼
AWS S3 — Billing Data Bucket
Raw CUR parquet files
        │
        ▼
AWS Athena
SQL queries against CUR schema
        │
        ▼
Unallocated Spend Detection
NULL tag filtering → resource isolation
        │
        ▼
Allocation Gap Confirmed
$53 (~17%) invisible to Finance reporting
```

Using Athena queries against CUR data with unblended costs to match the invoice source of truth, I filtered for records where required tags were NULL. EC2 charges accounted for approximately $38 to $40 in unallocated spend. Partial-month usage indicated a mid-month creation event.

SQL — Unallocated Spend Detection:

```sql
SELECT
    line_item_usage_account_id,
    line_item_product_code,
    line_item_resource_id,
    DATE_TRUNC('month', line_item_usage_start_date)  AS billing_month,
    SUM(line_item_unblended_cost)                    AS total_cost,
    resource_tags_user_environment,
    resource_tags_user_project,
    resource_tags_user_owner
FROM cur_table
WHERE
    line_item_line_item_type = 'Usage'
    AND line_item_usage_start_date
        BETWEEN DATE '2026-01-01' AND DATE '2026-01-31'
    AND (
        resource_tags_user_environment IS NULL
        OR resource_tags_user_project  IS NULL
        OR resource_tags_user_owner    IS NULL
    )
GROUP BY 1, 2, 3, 4, 6, 7, 8
ORDER BY total_cost DESC;
```

Unblended cost aligns with invoice totals, which Finance treats as the authoritative source of truth. Account ID and time filtering support multi-account environments and month-over-month comparison. Cost Explorer cannot surface row-level NULL tag data. CUR via Athena is required for this analysis.

Note on tag visibility: The required tags (Environment, Project, Owner) were activated as Cost Allocation Tags in the AWS Billing console before this investigation began. Tags must be activated in Billing under Cost Allocation Tags before they appear as columns in CUR datasets. This prerequisite is commonly missed in new AWS environments.

Discovery

- Instance: PROD-WEB-SERVER-01
- Missing tags: Environment, Project, Owner

![CUR query isolating unallocated EC2 spend](screenshots/02%20-%20cur-resource-isolation-missing-allocation-tags.png)

Note: Query shown is a representative example of the analysis methodology. The screenshot below reflects the actual Athena results output from the investigation.

## Phase 3: CloudTrail Forensic Investigation

Objective: Determine how and why the resource was created without tags.

Resource launched directly via the AWS Management Console. Console launch bypassed any IaC pipeline or automated workflow, and the API request did not include the required tags. Tags were never removed. They were absent at creation. The CloudTrail event identified the IAM principal as a Root session with MFA, confirming this was a direct console action rather than an automated CI/CD pipeline role. That ruled out infrastructure-as-code drift as a contributing factor.

This confirmed a governance enforcement gap, not user tampering.

![CloudTrail event history filtered to RunInstances](screenshots/03a%20-%20cloudtrail-event-history-runinstances.png)

![RunInstances event JSON showing missing tags](screenshots/03b%20-%20cloudtrail-runinstances-json-cli-useragent.png)

Note: JSON is truncated for readability. No required tags were present in the request.

## Phase 4: Expanding Scope to S3

Objective: Determine whether the issue was isolated or systemic.

| Service | Billed | Allocated | Gap |
|---|---|---|---|
| EC2 | $220 | $182 | ~$38 |
| RDS | $60 | $60 | $0 |
| S3 | $35 | $20 | $15 |

Approximately $38 from EC2 plus $15 from S3 totals $53 in unallocated spend. Minor variance is attributable to partial-day usage and CUR line-item granularity. All figures reconcile to invoice-level totals.

While EC2 represented the largest gap, S3 confirmed the issue was systemic. Two buckets were missing the required Environment tag. Project and Owner tags were present but incomplete allocation meant $15 per month in invisible storage spend. S3 storage charges appear as TimedStorage-ByteHrs line items in CUR, which was confirmed during query filtering to ensure complete service coverage.

![S3 tag compliance audit via CUR](screenshots/04%20-%20cur-s3-tag-compliance-audit.png)

## Root Cause Analysis

Root cause: No enforcement at resource creation time.

If tagging depends on humans being perfect, it will fail. This was not a user error. It was a system design gap that required a system-level response.

Key design decisions:

CUR and Athena were used instead of Cost Explorer because they enable row-level tag inspection that Cost Explorer cannot surface. EventBridge and Lambda were chosen over AWS Config because they provide near-real-time detection (seconds vs minutes-to-hours for Config rules), lower operational cost (no Config recorder charges for every resource state change), and simpler event-driven architecture for a creation-time enforcement pattern. Alerting was prioritized over blocking to preserve engineering velocity, with SCP enforcement planned only after compliance stabilizes above 95%.

These decisions reflect a bias toward fast feedback, low friction, and defensible financial reporting.

## Remediation and Controls

Immediate correction: Applied missing tags so Finance could close the month accurately.

Permanent prevention:

```
┌──────────────────────────────────────────────────────────┐
│              AWS Resource Creation Event                 │
│         (EC2 RunInstances, S3 CreateBucket, etc.)        │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│                     AWS CloudTrail                       │
│              Captures all API-level events               │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│                  Amazon EventBridge                      │
│         Rule filters for resource creation events        │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│                   AWS Lambda                             │
│     Validates required tags: Environment, Project,       │
│     Owner — flags missing or non-compliant resources     │
└──────────────┬──────────────────────┬────────────────────┘
               │                      │
               ▼                      ▼
     ┌──────────────────┐    ┌───────────────────────┐
     │  Slack: Finance  │    │  Slack: Resource Owner│
     │  Alert Channel   │    │  Direct Notification  │
     └──────────────────┘    └───────────────────────┘
```

Enforcement philosophy: Alert before block.

Non-compliant resources are surfaced immediately without breaking CI/CD pipelines or developer workflows. This preserves engineering velocity while maintaining financial accuracy.

The progressive enforcement model works in three layers. AWS Organizations Tag Policies standardize required tag keys across all accounts without blocking deployments. EventBridge and Lambda detection surfaces non-compliant resources within minutes of creation, alerting Finance and the resource owner simultaneously. SCP enforcement is introduced for greenfield accounts only once compliance stabilizes above 95%, grandfathering existing workflows to avoid pipeline disruption.

This is how real platform teams deploy governance: standardize first, detect second, enforce third.

### Step 1: EventBridge Rule

The EventBridge rule finops-tag-compliance-monitor was configured on the default event bus to filter for RunInstances (EC2) and CreateBucket (S3) API calls via CloudTrail, targeting the finops-tag-validator Lambda function. Status: Enabled.

![EventBridge Rule](screenshots/05a-eventbridge-rule.png)

### Step 2: Lambda Validation Logic

Lambda inspects the resource tags included in the API event payload and validates that Environment, Project, and Owner exist and are non-empty. The function handles both EC2 (RunInstances) and S3 (CreateBucket) creation events. Non-compliant resources trigger an alert containing the event name, missing tags, IAM principal, account ID, and region, giving both Finance and the resource owner everything needed to remediate immediately.

Full source: [`lambda/tag_validator.py`](lambda/tag_validator.py)

Key implementation notes:
- `REQUIRED_TAGS` list is the single source of truth. Add or remove tags in one place
- Slack webhook URL stored in Lambda environment variable, backed by AWS Secrets Manager in production. Never hardcoded
- S3 buckets alert immediately at creation since tags are applied separately. No grace period
- Design choice: alert fires without blocking the deployment, preserving CI/CD velocity

![Lambda function finops-tag-validator](screenshots/05b%20-%20lambda-finops-tag-validator-code.png)

![Lambda runtime settings](screenshots/05b-ii%20-%20lambda-settings.png)

### Step 3: End-to-End Test Execution

The function was tested with a simulated untagged RunInstances event. All three required tags were correctly identified as missing. The FINOPS TAG COMPLIANCE VIOLATION alert fired with the expected IAM principal, account ID, and region. Execution time was 2.54ms.

![Test Result](screenshots/05c-lambda-test-result.png)

This aligns with progressive FinOps governance, where guardrails precede enforcement.

## Results and Impact

| Metric | Before | After |
|---|---|---|
| Allocation coverage | ~82.5% | 100% |
| Tag compliance rate | ~82.5% | 100% |
| MTTD (mean time to detect) | ~30 days | Under 5 minutes |
| Manual reconciliation | ~4 hours/month | Reduced significantly |
| Finance escalations | Required monthly | Zero |

The approximately $636 per year figure represents spend that was correctly billed but invisible to allocation reports. This is a reporting integrity metric, not a cost savings figure. The goal was attribution accuracy, not spend reduction.

Assumptions: Manual reconciliation time of approximately 4 hours per month is based on Finance validating tag completeness, exporting reports, and coordinating remediation. Detection latency of approximately 30 days reflects monthly close cycles prior to automation. Allocation percentages use unblended CUR data to align with invoice totals.

## Scaling to a Real Organization

In organizations with 10 to 500 or more AWS accounts, tag compliance gaps compound across every account simultaneously. Centralize CUR via AWS Organizations into a single S3 bucket, run the same Athena queries across the consolidated dataset, and deploy the EventBridge to Lambda pipeline once via SCP at the Organizations root. Every account inherits the governance control automatically.

### What Breaks at Scale

Three failure points that the sandbox did not surface:

**Lambda concurrency limits.** In extreme burst scenarios (hundreds of simultaneous RunInstances events during large infrastructure provisioning) Lambda can throttle. SQS queuing between EventBridge and Lambda with a Dead Letter Queue is an optional hardening step for high-volume deployments. For most organizations, EventBridge to Lambda handles the load without buffering.

**EventBridge rule limits.** AWS enforces 300 rules per event bus per account. At scale, use a dedicated event bus per account forwarding to a central compliance bus.

**CUR partition management.** A consolidated CUR dataset across 100+ accounts at $2M to $10M/month requires partitioning by account ID, service, and billing month. Without it, full-table scans become expensive and slow.

**Slack alert implementation.** Lambda posts to Slack via an incoming webhook. The webhook URL is stored as a Lambda environment variable backed by AWS Secrets Manager. Never hardcoded. In production, retry logic handles transient Slack API failures. See [`lambda/tag_validator.py`](lambda/tag_validator.py) for the full implementation.

These failure points define the engineering work to harden the architecture for production. They do not change the core design.

## Multi-Account Architecture Extension

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Organizations Root                       │
│         SCP deployed here — all accounts inherit control        │
└────────────────────────┬────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
   │  Account A  │ │  Account B  │ │  Account C  │
   │  (Prod)     │ │  (Dev)      │ │  (Staging)  │
   └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
          │               │               │
          ▼               ▼               ▼
   ┌─────────────────────────────────────────────┐
   │            CloudTrail (All Accounts)        │
   │     Centralized logging via Organizations   │
   └─────────────────────┬───────────────────────┘
                         │
                         ▼
   ┌─────────────────────────────────────────────┐
   │         Central S3 — CUR + CloudTrail       │
   │   Partitioned by account_id / month         │
   └──────────────┬──────────────────────────────┘
                  │
        ┌─────────┴──────────┐
        ▼                    ▼
┌──────────────┐    ┌──────────────────────────┐
│ Athena SQL   │    │  EventBridge + Lambda     │
│ Consolidated │    │  Deployed via SCP —       │
│ CUR queries  │    │  inherited automatically  │
│ by account   │    └────────────┬─────────────┘
│ and BU       │                 ▼
└──────────────┘      ┌──────────────────────┐
                      │  Slack: Finance +    │
                      │  Resource Owner      │
                      └──────────────────────┘
```

Key decisions: Centralized CUR eliminates per-account configuration. Organizations CloudTrail ensures no account is missed in forensic investigation. SCP at root means every new account inherits the control automatically. The same Phase 2 Athena SQL runs unchanged. The `line_item_usage_account_id` column surfaces which account owns each gap.

## FinOps Lifecycle Alignment

| Phase | Activity in This Investigation |
|---|---|
| Inform | CUR and Athena cost visibility, allocation coverage metric, invoice validation baseline |
| Optimize | Identification of $53 per month in unattributed spend, systemic gap confirmed across EC2 and S3 |
| Operate | Event-driven tagging compliance control, real-time Slack alerting, progressive enforcement model |

The investigation started in Inform (what is the data telling us?), moved through Optimize (what is the gap and root cause?), and landed in Operate (what system prevents recurrence?). Optimization without Inform produces inaccurate savings models. Operate without Optimize produces governance controls targeting the wrong problem.

## What I Would Do Next

**Governance maturation**
- Centralize CUR across all accounts via AWS Organizations
- Establish weekly tag compliance baseline with allocation coverage tracked as a KPI
- Introduce SCP enforcement for greenfield accounts once compliance exceeds 95%
- Build a shared Finance and Engineering dashboard with real-time allocation coverage

**Cost optimization (once attribution is reliable)**

Attribution accuracy is the prerequisite for optimization. Unattributed spend cannot be rightsized, committed, or charged back responsibly. Ownership metadata must exist first.

- **EC2 rightsizing:** Cost Explorer recommendations become actionable once resources are tagged by owner and workload
- **Savings Plans / RI analysis:** Commitment modeling against untagged spend produces incorrect coverage recommendations. Full attribution enables accurate break-even analysis
- **Anomaly detection:** Week-over-week Athena queries against the 4-week rolling average by service and owner, alerting Finance before monthly close
- **Chargeback transition:** Showback first, chargeback once tag coverage exceeds 95% and Finance and Engineering align on exception handling

## Lessons Learned

Financial data must be treated as operational telemetry. Without real-time visibility, allocation failures persist until monthly close. Cost data needs the same alerting discipline as infrastructure monitoring.

Tagging governance must exist at the system level. Human process alone cannot maintain reliable allocation coverage. Engineers under deadline pressure will skip tags. The system must catch what humans miss, automatically and immediately.

Detection is more effective than enforcement early in governance maturity. Immediate feedback loops change tagging behavior without introducing deployment friction. Blocking deployments too early creates shadow IT and pipeline workarounds. Earn compliance before enforcing it.

Attribution is the prerequisite for optimization. Rightsizing recommendations, Savings Plans modeling, and chargeback frameworks all rely on reliable ownership metadata. Optimization applied to unattributed spend produces inaccurate projections and misaligned accountability.

## Skills Demonstrated

| Skill Area | Detail |
|---|---|
| Cloud FinOps | Cost attribution and financial reconciliation |
| AWS Billing Data Engineering | CUR analysis using Athena SQL |
| Cloud Governance | Event-driven tag compliance architecture |
| Cloud Forensics | CloudTrail investigation and IAM principal tracing |
| FinOps Operations | Allocation coverage metrics and MTTD tracking |
| Multi-Account Architecture | Organizations-level governance design and scale considerations |
| Technical Communication | Finance-ready reporting from infrastructure findings |
| AI-Assisted Reporting | Structured prompt engineering for executive output |

## Scope Rationale

This investigation scoped deliberately to attribution and governance, not optimization. Cost optimization recommendations are only defensible once ownership and attribution are reliable. Recommending rightsizing or Reserved Instance purchases against unallocated spend produces inaccurate savings projections and incorrect team-level accountability. Attribution accuracy is the prerequisite. Optimization follows.

## Why This Matters

Most allocation problems are not caught because Finance lacks the tooling to see them in real time and Engineering lacks the incentive to tag consistently. This investigation showed that governance design matters more than enforcement. Surfacing problems immediately changes behavior faster than blocking deployments.

The AI-assisted reporting step closes the final gap: translating investigation findings into executive-ready output at the speed the business requires, without replacing the analytical work that produced those findings.

<details>
<summary><strong>Appendix: AI-Assisted Investigation Reporting</strong> (click to expand)</summary>

AI was used to accelerate report generation after the investigation was complete, not to perform the investigation. Every finding was produced by SQL analysis and CloudTrail forensics in Phases 1 through 5. AI converted validated findings into executive narrative format.

Workflow:

```
SQL investigation output  →  Structured AI prompt  →  AI-generated report  →  Human validation  →  Finance-ready output
```

Prompt used:

```
You are a senior FinOps analyst preparing an internal investigation
report for a Finance and Engineering audience.

Produce a structured financial investigation report with these sections:
1. Executive Summary (3 sentences maximum)
2. Investigation Findings (what the data shows)
3. Root Cause Analysis (why it happened)
4. Financial Impact (percentage-based, with dollar context)
5. Governance Failure (what control was missing)
6. Remediation Plan (immediate fix and permanent prevention)

Investigation data:
- Total AWS spend: $315 | Unallocated: $53 (17%)
- Affected: EC2 ($38 gap), S3 ($15 gap), RDS ($0 gap)
- Root cause: Console-launched resources missing required tags at creation, no enforcement existed at provisioning time
- Detection: Athena SQL on CUR + CloudTrail RunInstances forensics
- Fix: CloudTrail to EventBridge to Lambda to Slack pipeline
- Outcome: Detection latency 30 days to under 5 minutes
```

AI-generated report:

> Generated by Anthropic Claude using the prompt above. Every finding was validated against SQL and CloudTrail evidence in Phases 1 through 5 before inclusion.

**INTERNAL FINOPS INVESTIGATION REPORT**
**AWS Cost Allocation Gap | January 2026**

**1. Executive Summary**

A forensic investigation identified a 17% cost allocation gap across EC2 and S3, spend that was correctly billed but invisible to Finance reporting due to missing resource tags at provisioning time. The failure was a governance design gap, not user error. No enforcement mechanism existed to require tags at resource creation. An event-driven detection pipeline has been implemented, restoring 100% allocation visibility and reducing MTTD from 30 days to under 5 minutes.

**2. Investigation Findings**

CUR analysis via Athena confirmed $53 in unallocated spend across two services. EC2 represented the largest gap at approximately $38, with S3 contributing an additional $15. RDS showed full allocation compliance. Invoice validation confirmed all charges were accurate. The gap existed entirely in the attribution layer.

**3. Root Cause Analysis**

CloudTrail forensics traced the EC2 gap to a RunInstances API call made directly via the AWS Management Console. The console provides no tag enforcement at resource creation time. No SCP or tag policy existed to reject untagged resource requests. Tags were never applied, not removed, confirming a provisioning-time enforcement gap rather than post-deployment drift or tampering.

**4. Financial Impact**

| Metric | Value |
|---|---|
| Allocation gap | 17% of total spend |
| Monthly unallocated spend | $53 |
| Annualized if unresolved | ~$636 |
| Manual reconciliation reduced | ~4 hours/month |
| MTTD before remediation | ~30 days |
| MTTD after remediation | Under 5 minutes |

In enterprise environments where monthly spend exceeds $2M to $10M across 50 to 200 accounts, the same allocation failure can hide hundreds of thousands of dollars in unattributed cost, making accurate chargeback, showback, and budget forecasting impossible.

**5. Governance Failure**

Three compounding control gaps allowed the issue to persist. There was no SCP or tag policy requiring tags before provisioning approval. There was no real-time monitoring to surface non-compliant resources at creation. There was no Finance visibility into allocation coverage as a tracked KPI. The absence of all three meant the gap could recur indefinitely, caught only during monthly close.

**6. Remediation Plan**

Immediate: Missing tags applied manually. Finance closed January with 100% allocation accuracy. No restatements required.

Permanent: Event-driven pipeline (CloudTrail to EventBridge to Lambda to Slack) deployed. Engineers receive real-time alerts without deployment friction. SCP enforcement scoped to greenfield accounts once compliance exceeds 95%.

Next steps: Weekly CUR-based tag compliance baseline, allocation coverage tracked as a first-class FinOps KPI, anomaly detection for week-over-week cost spikes.

All figures validated against SQL and CloudTrail findings in Phases 1 through 5.

> Analysis note: Unblended costs were used to align directly with AWS invoice totals, which Finance treats as the authoritative source of truth. The architecture is AWS-native and the investigation methodology applies regardless of spend scale.

</details>
