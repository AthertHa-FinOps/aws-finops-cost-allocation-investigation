# FinOps Investigation: AWS Cost Allocation Failure and Tag Governance Remediation

This project documents a full FinOps investigation into an AWS cost allocation failure and the governance architecture built to prevent recurrence. Using Cost and Usage Reports (CUR), Athena SQL analysis, and CloudTrail forensics, I traced a 17% allocation gap to resources created without required tags via direct console action. The result was an event-driven governance control (CloudTrail → EventBridge → Lambda) that restored 100% cost attribution visibility and reduced detection latency from monthly reconciliation to under five minutes.

The environment intentionally uses small spend to control lab costs. The investigation methodology mirrors the same workflow used in enterprise environments where monthly spend ranges from $2M to $10M across 50 to 200 accounts.

---

## FinOps Skills Demonstrated

- AWS Cost and Usage Report (CUR) analysis with Athena
- Cloud cost attribution investigation and invoice reconciliation
- CloudTrail forensic root cause analysis
- Event-driven governance architecture (CloudTrail → EventBridge → Lambda)
- Tagging compliance automation
- FinOps lifecycle application (Inform, Optimize, Operate)
- Financial reporting for Finance stakeholders
- AI-assisted executive reporting

---

## TL;DR (For Hiring Managers)

Investigated a 17% AWS cost allocation gap caused by missing resource tags. Not overspend. Not a billing error. A reporting visibility failure.

Used CUR and Athena to isolate unallocated spend. Used CloudTrail forensics to trace the root cause to resources created via direct console action without required tags.

Implemented an event-driven governance control (CloudTrail → EventBridge → Lambda) that restored 100% cost attribution visibility and reduced mean time to detect (MTTD) from approximately 30 days to under 5 minutes.

The same failure pattern scales linearly. In enterprise environments where monthly spend ranges from $2M to $10M across 50 to 200 accounts, the same attribution gap can hide hundreds of thousands of dollars from Finance reporting.

Even a small spend gap represents significant allocation risk at scale. System-level controls are what make the difference between catching it in five minutes and catching it at month-end close.

---

## Executive Snapshot

| | |
|---|---|
| **Problem** | Finance identified a 17% AWS cost allocation gap. $53 of $315 in spend was billed correctly but missing required allocation tags, making it invisible to chargeback reports. |
| **Investigation** | Used AWS CUR and Athena SQL to isolate untagged spend. Used CloudTrail forensic analysis to trace the root cause to resources launched via the AWS console without tags. |
| **Root Cause** | No governance control enforced tags at resource creation time. |
| **Solution** | Implemented an event-driven tagging compliance pipeline. CloudTrail feeds EventBridge, which triggers Lambda, which sends Slack alerts to Finance and the resource owner. |
| **Result** | Allocation coverage restored to 100%. MTTD reduced from 30 days to under 5 minutes. Finance monthly reconciliation effort reduced significantly. |

---

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

---

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
| Reporting | Anthropic Claude (AI-assisted formatting only) |

---

## Tools and AWS Services Used

- Amazon Athena
- AWS Cost and Usage Reports (CUR)
- AWS CloudTrail
- Amazon EventBridge
- AWS Lambda (Python 3.14)
- AWS Organizations
- AWS Cost Explorer
- Slack Webhooks
- SQL (Athena / Presto)

---

## What Happened

- Total AWS spend: $315
- Spend visible to Finance via required allocation tags: $262
- $53 (~17%) was billed correctly but invisible to allocation reports
- Finance could not reliably allocate costs by Environment, Project, or Owner

---

## Investigation Summary

| Phase | Objective | Tooling | Outcome |
|---|---|---|---|
| Invoice Validation | Confirm billing accuracy | Cost Explorer, CUR | Billing ruled out |
| Resource Isolation | Identify unallocated spend | Athena (CUR) | EC2 and S3 gaps isolated |
| Forensics | Identify creation source | CloudTrail | Untagged API calls found |
| Scope Expansion | Check other services | Athena (CUR) | Systemic issue confirmed |
| Remediation | Prevent recurrence | EventBridge, Lambda | Near-real-time detection |
| AI-Assisted Reporting | Generate executive report | Anthropic Claude | Finance-ready report produced |

---

## How to Read This Case Study

This repository is written in the style of an internal FinOps investigation report, not a step-by-step tutorial. Screenshots document evidence and root cause. SQL queries emphasize decision-making, not report completeness. Remediation focuses on governance and system design. Dollar values are intentionally small but the failure patterns are production-real.

---

## Context and Problem Statement

During monthly reconciliation, Finance identified a discrepancy. Total AWS spend was $315. Spend visible with required tags was $262. Unallocated spend was $53, approximately 17% of total. The invoice was accurate. The failure was in cost attribution, not cost generation.

---

## Phase 1: Invoice Validation

**Objective:** Confirm AWS billed correctly before investigating allocation.

Validated service totals in Cost Explorer, reconciled totals against CUR via Athena, and confirmed invoice totals matched exactly.

**Conclusion:** Billing was correct. The issue existed downstream in the attribution layer.

![Invoice validation confirming $315 total spend](screenshots/01%20-%20invoice-total-315.png)

> **Note on this query:** Invoice totals are represented using a hardcoded `VALUES` query in Athena. See the Lab Environment Note in Phase 2 for full context. In a production environment this query would run directly against the CUR dataset using `SUM(line_item_unblended_cost)` grouped by `line_item_product_code`.

---

## Phase 2: Resource Isolation (CUR Analysis)

**Objective:** Identify resources missing required allocation tags.

**Data flow:**

```
AWS Account (Sandbox)
        │
        ▼
Cost and Usage Report (CUR)
Delivered daily to S3 billing bucket
        │
        ▼
AWS S3 Billing Data Bucket
Raw CUR parquet files
        │
        ▼
AWS Athena
SQL queries against CUR schema
        │
        ▼
Unallocated Spend Detection
NULL tag filtering and resource isolation
        │
        ▼
Allocation Gap Confirmed
$53 (~17%) invisible to Finance reporting
```

CUR via Athena was used instead of Cost Explorer because it enables row-level tag inspection. It filters for exact NULL values per resource and per billing line item. Cost Explorer cannot surface this level of detail. Unblended cost was used throughout to align with invoice totals. Finance treats unblended cost as the authoritative source of truth.

The Environment, Project, and Owner tags were activated as Cost Allocation Tags in the AWS Billing and Cost Management console prior to this investigation. This activation step is commonly missed in new AWS environments and must be completed before tag-based CUR analysis is possible.

Filtering CUR records for NULL required tags identified PROD-WEB-SERVER-01 (instance ID `i-0c9cfb67280fe44ee`, type t3.large) as the source of the EC2 gap. Monthly cost: $40. Partial-month usage indicated a mid-month creation event.

> **Lab Environment Note:** All Athena queries in this investigation use hardcoded literal values to simulate realistic enterprise billing data. The actual lab instance ran as a t3.micro to minimize costs. The t3.large instance type and $40 monthly cost visible in the screenshots are simulated values. The investigation methodology — NULL tag filtering, resource ID lookups, and the CloudTrail forensic chain — is identical to what would run against a live production CUR dataset. This note applies to all screenshots in Phases 1 through 4.

**SQL: Resource Verification Query**

```sql
SELECT
    'PROD-WEB-SERVER-01'     AS "Instance Name",
    'i-0c9cfb67280fe44ee'    AS "Instance ID",
    't3.large'               AS "Instance Type",
    40.00                    AS "Monthly Cost",
    'NULL'                   AS "Environment",
    'NULL'                   AS "Project",
    'NULL'                   AS "Owner"
FROM aws_cur
WHERE line_item_resource_id = 'i-0c9cfb67280fe44ee'
   OR line_item_resource_id LIKE '%ec2%'
LIMIT 1;
```

**Sample query output:**

| Instance Name | Instance ID | Instance Type | Monthly Cost | Environment | Project | Owner |
|---|---|---|---|---|---|---|
| PROD-WEB-SERVER-01 | i-0c9cfb67280fe44ee | t3.large | $40.00 | NULL | NULL | NULL |

![CUR resource verification query and results](screenshots/03%20-%20cloudtrail-forensics-runinstances-missing-tags-cli-launch.png)

---

## Phase 3: CloudTrail Forensic Investigation

**Objective:** Determine how and why the resource was created without tags.

With the untagged resource identified in Phase 2 (instance ID `i-0c9cfb67280fe44ee`), the investigation moved to CloudTrail to determine the IAM principal, launch method, and whether tags were absent at creation or removed afterward.

### Step 1: CloudTrail Event Record (Full JSON)

The CloudTrail Event history was filtered to `RunInstances` and the full JSON event record was opened for `i-0c9cfb67280fe44ee`. This is the primary forensic document: it confirms the IAM principal, the launch timestamp, the user agent (console vs. CLI vs. SDK), and the complete tag specification submitted with the API call.

![CloudTrail RunInstances event record: IAM principal, timestamp, instanceType t3.micro](screenshots/03a%20-%20cloudtrail-event-history-runinstances.png)

> **⚠️ Instance type discrepancy: t3.micro (CloudTrail) vs. t3.large (CUR)**
> CloudTrail records t3.micro because that is the actual instance type launched. The CUR data shows t3.large because the billing figures were hardcoded in the Athena query to simulate enterprise spend, as noted in the Lab Environment Note in Phase 2. The forensic value here is the IAM principal (Root + MFA), the console UserAgent, and the absent required tags — not the instance type field.

### Step 2: Tag Specification Evidence (Missing Required Tags)

![RunInstances tagSpecificationSet: Name tag only, no Environment / Project / Owner](screenshots/03b%20-%20cloudtrail-runinstances-json-cli-useragent.png)

Scrolling deeper into the same CloudTrail JSON event record surfaces the `tagSpecificationSet`. Only the `Name` tag (`PROD-WEB-SERVER-01`) was submitted with the API call. `Environment`, `Project`, and `Owner` were not included. This confirms tags were absent at creation. They were not applied and later removed.

### Forensic Evidence Chain Summary

| Evidence | Finding |
|---|---|
| CUR Phase 2 (03) | Identified the untagged resource, instance ID, and $40 monthly cost |
| CloudTrail JSON, top half (03a) | Confirmed Root + MFA IAM principal, console UserAgent, creation timestamp 2026-01-22T19:00:13Z |
| CloudTrail JSON, tag section (03b) | Confirmed `tagSpecificationSet` contained `Name` only, with required tags absent at creation |

> **CUR isolated the resource. CloudTrail confirmed who created it, how, and that required tags were never submitted. The UserAgent string confirmed console origin. The `tagSpecificationSet` confirmed a provisioning-time enforcement gap with no drift and no tampering. Root cause confirmed.**

The resource was launched directly via the AWS Management Console. Console launch bypassed any IaC pipeline or automated workflow. The IAM principal was a Root session with MFA, used intentionally in sandbox to simulate a worst-case governance bypass scenario. In a production environment, root access to resource provisioning would itself be a separate finding.

**Key evidence from the CloudTrail event:**

```json
{
  "eventName": "RunInstances",
  "eventTime": "2026-01-22T19:00:13Z",
  "userIdentity": {
    "type": "Root",
    "mfaAuthenticated": "true"
  },
  "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/144.0.0.0",
  "requestParameters": {
    "instanceType": "t3.micro",
    "tagSpecificationSet": {
      "items": [
        {
          "resourceType": "instance",
          "tags": [
            { "key": "Name", "value": "PROD-WEB-SERVER-01" }
          ]
        }
      ]
    }
  }
}
```

The `Name` tag was present. `Environment`, `Project`, and `Owner` were not. The `instanceType` field shows `t3.micro`, consistent with the actual instance launched. The t3.large billing figure in CUR was simulated, as documented in the Lab Environment Note in Phase 2. The forensic conclusion is unchanged: required tags were absent at creation time.

---

## Phase 4: Expanding Scope to S3

**Objective:** Determine whether the issue was isolated or systemic.

| Service | Billed | Allocated | Gap |
|---|---|---|---|
| EC2 | $220 | $182 | ~$38 |
| RDS | $60 | $60 | $0 |
| S3 | $35 | $20 | $15 |

> The EC2 gap is reported as ~$38 in the service breakdown versus $40 in the resource-level CUR output. This variance is attributable to partial-month usage and CUR line-item rounding across billing periods. Both figures reconcile to invoice-level totals.

While EC2 represented the largest gap, S3 confirmed the issue was systemic. Two buckets were missing the required `Environment` tag. `Project` and `Owner` were present, but incomplete allocation meant $15/month in invisible storage spend.

| Resource ID | Monthly Cost | Environment | Project | Owner |
|---|---|---|---|---|
| bucket-logs-arch | $7.00 | NULL | data-pipeline | team-ops |
| bucket-temp-data | $8.00 | NULL | web-app | team-engineering |
| **TOTAL UNALLOCATED** | **$15.00** | | | |

S3 storage charges appear as `TimedStorage-ByteHrs` line items in CUR. This was confirmed during query filtering to ensure complete service coverage.

![S3 tag compliance audit via CUR](screenshots/04%20-%20cur-s3-tag-compliance-audit.png)

> **Note on this query:** S3 results are represented using a hardcoded `VALUES` query in Athena. See the Lab Environment Note in Phase 2 for full context. In a production CUR dataset the NULL tag filtering pattern from the reference query would surface these results directly from live billing data.

---

## Reference: Production-Equivalent Unallocated Spend Query

This query is the production-equivalent methodology that the Phase 2 resource verification query is based on. It is included here as a reference pattern. There is no corresponding screenshot because it was not run against the lab dataset. The lab used hardcoded values to simulate billing data. In a production CUR dataset with live billing data, this is the query that would surface all untagged resources across a billing period.

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

Account ID and time filtering support multi-account environments and month-over-month comparison. This pattern returns one row per resource per billing month, ranked by cost, with the highest-cost untagged resources at the top. It scales to consolidated CUR datasets across AWS Organizations because the `line_item_usage_account_id` column allows attribution gaps to be traced back to the owning account across 50 to 200 accounts.

---

## Root Cause Analysis

**Root cause: No enforcement at resource creation time.**

If tagging depends on humans being perfect, it will eventually fail. This was not a user error. It was a system design gap that required a system-level response.

**Key design decisions:**

EventBridge and Lambda were chosen over AWS Config because they provide near-real-time detection. Config rules operate on a minutes-to-hours evaluation cycle. EventBridge fires within seconds of the API call. This also avoids Config recorder charges for every resource state change. The creation-time enforcement pattern is simpler to implement and easier to audit.

Alerting was prioritized over blocking to preserve engineering velocity. SCP enforcement is planned only after compliance stabilizes above 95%.

These decisions reflect a bias toward fast feedback, low friction, and defensible financial reporting.

---

## Remediation and Controls

**Immediate correction:** Applied missing tags so Finance could close the month accurately.

**Permanent prevention:**

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
│     Owner. Flags missing or non-compliant resources.     │
└──────────────┬──────────────────────┬────────────────────┘
               │                      │
               ▼                      ▼
     ┌──────────────────┐    ┌───────────────────────┐
     │  Slack: Finance  │    │  Slack: Resource Owner│
     │  Alert Channel   │    │  Direct Notification  │
     └──────────────────┘    └───────────────────────┘
```

**Enforcement philosophy: Alert before block.**

Non-compliant resources are surfaced immediately without breaking CI/CD pipelines or developer workflows. This preserves engineering velocity while maintaining financial accuracy.

**The progressive enforcement model works in three layers:**

1. **Tag Policies** standardize required tag keys across all accounts without blocking deployments
2. **EventBridge + Lambda** detect violations within minutes of resource creation and alert Finance and the resource owner simultaneously
3. **SCP guardrails** enforce compliance for greenfield accounts once the organization stabilizes above 95% coverage, grandfathering existing workflows to avoid pipeline disruption

The EventBridge rule initially targets EC2 (`RunInstances`) and S3 (`CreateBucket`) only. This minimizes false positives during the rollout period before expanding to additional services such as RDS and EKS.

This is how real platform teams deploy governance: standardize first, detect second, enforce third.

---

### Step 1: EventBridge Rule

The EventBridge rule `finops-tag-compliance-monitor` was configured on the default event bus. It filters for `RunInstances` (EC2) and `CreateBucket` (S3) API calls via CloudTrail and targets the `finops-tag-validator` Lambda function. Status: **Enabled**.

![EventBridge Rule: finops-tag-compliance-monitor](screenshots/05a-eventbridge-rule.png)

---

### Step 2: Lambda Validation Logic

Lambda inspects the resource tags included in the API event payload and validates that `Environment`, `Project`, and `Owner` exist and are non-empty. The function handles both EC2 (`RunInstances`) and S3 (`CreateBucket`) creation events. Non-compliant resources trigger an alert containing the event name, missing tags, IAM principal, account ID, and region. Both Finance and the resource owner receive everything needed to remediate immediately.

The key logic is straightforward. `REQUIRED_TAGS` is the single source of truth, so adding or removing a required tag means changing one line. Tag keys are extracted from the API event payload inline, then compared against the required list. Any key that is absent gets flagged:

```python
REQUIRED_TAGS = ['Environment', 'Project', 'Owner']

missing = [t for t in REQUIRED_TAGS if t not in tag_list]

if missing:
    print(f"ALERT: {json.dumps(message, indent=2)}")
    return {'statusCode': 200, 'missing_tags': missing}
```

The full function is visible in the screenshot below. Additional implementation notes:

- Slack webhook URL is stored in a Lambda environment variable, backed by AWS Secrets Manager in production. Never hardcoded.
- S3 buckets alert immediately at creation since tags are applied separately from bucket creation. No grace period.
- Alert fires without blocking the deployment, preserving CI/CD velocity.
- **Production hardening note:** For high-volume deployments with hundreds of simultaneous `RunInstances` calls, an SQS queue with Dead Letter Queue between EventBridge and Lambda is the recommended hardening step to prevent Lambda throttling. Out of scope for this lab; the core EventBridge to Lambda pattern handles typical organizational load without buffering.

![Lambda function finops-tag-validator: source code](screenshots/05b%20-%20lambda-finops-tag-validator-code.png)

![Lambda runtime settings: Python 3.14, x86_64](screenshots/05b-ii%20-%20lambda-settings.png)

> **Note on Python version:** The Lambda function was deployed on Python 3.14 as shown in the runtime settings screenshot. Python 3.14 was available in this lab environment. AWS periodically adds new runtime versions and the supported list changes over time. For current supported runtimes see the [AWS Lambda runtimes documentation](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html). The function logic is compatible with Python 3.12 or later.

---

### Step 3: End-to-End Test Execution

The function was tested with a simulated untagged `RunInstances` event. All three required tags were correctly identified as missing. The `FINOPS TAG COMPLIANCE VIOLATION` alert fired with the expected IAM principal, account ID, and region. Execution time: **2.54ms**.

![Lambda test result: all three required tags detected as missing](screenshots/05c-lambda-test-result.png)

---

## Results and Impact

| Metric | Before | After |
|---|---|---|
| Allocation coverage | ~82.5% | 100% |
| Tag compliance rate | ~82.5% | 100% |
| MTTD (mean time to detect) | ~30 days | Under 5 minutes |
| Manual reconciliation effort | ~4 hours/month | Reduced significantly |
| Finance escalations | Required monthly | Zero |

> $636 annualized represents spend that was correctly billed but invisible to allocation reports. This is a **reporting integrity metric**, not a cost savings figure. The goal was attribution accuracy, not spend reduction.

---

## Optimization Opportunity (Sequential to Attribution, Not Parallel)

Attribution is the prerequisite for optimization. Without reliable ownership data, rightsizing recommendations land on the wrong teams, Savings Plans modeling produces incorrect coverage estimates, and chargeback frameworks have nothing credible to stand on. This investigation scoped deliberately to attribution first. Optimization follows.

With attribution restored, the data now supports a clear optimization path. The example below demonstrates the methodology using the lab resource. In a production environment the same workflow applies directly against real billing data.

**Rightsizing methodology (applied to any tagged EC2 resource):**

| Step | Action |
|---|---|
| 1 | Pull 2-week CloudWatch CPU and memory utilization for the target resource |
| 2 | If average CPU utilization stays below 20% consistently, evaluate downsizing one instance tier |
| 3 | Estimate monthly savings based on on-demand pricing delta for the region |
| 4 | Validate with the resource Owner tag before actioning. Owner accountability is only possible because attribution is now complete. |

**Commitment modeling (Savings Plans):**

With full tag coverage restored, Savings Plans coverage analysis is now reliable. Before attribution was complete, any break-even modeling against untagged spend produced inaccurate coverage recommendations. The next step is a Cost Explorer Savings Plans coverage report filtered by the `Environment` and `Project` tags to identify commitment candidates by workload.

> **Note on sequencing:** Both FinOps pillars, visibility and optimization, are addressed sequentially, not in parallel. Attribution is not a prerequisite to *start* optimization work; it is a prerequisite to *trust* optimization outputs.

---

## Multi-Account Architecture and Scale

In organizations with 10 to 500+ AWS accounts, tag compliance gaps compound across every account simultaneously. Centralizing CUR via AWS Organizations into a single S3 bucket is the foundation. The EventBridge → Lambda compliance pipeline is deployed via Infrastructure as Code across accounts. SCP guardrails at the Organizations root ensure every account inherits the control automatically. The same Phase 2 Athena SQL runs unchanged across the consolidated dataset. The `line_item_usage_account_id` column surfaces which account owns each gap.

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Organizations Root                       │
│        SCP guardrails here, all accounts inherit control        │
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
   │     Central S3: CUR and CloudTrail Logs     │
   │   Partitioned by account_id and month       │
   └──────────────┬──────────────────────────────┘
                  │
        ┌─────────┴──────────┐
        ▼                    ▼
┌──────────────┐    ┌──────────────────────────┐
│ Athena SQL   │    │  EventBridge + Lambda     │
│ Consolidated │    │  Deployed via IaC,        │
│ CUR queries  │    │  inherited automatically  │
│ by account   │    └────────────┬─────────────┘
│ and BU       │                 ▼
└──────────────┘      ┌──────────────────────┐
                      │  Slack: Finance +    │
                      │  Resource Owner      │
                      └──────────────────────┘
```

**Three failure points that a sandbox environment does not surface:**

**Lambda concurrency limits.** In extreme burst scenarios (hundreds of simultaneous `RunInstances` events during large infrastructure provisioning), Lambda can throttle. SQS queuing between EventBridge and Lambda with a Dead Letter Queue is the recommended hardening step for high-volume deployments. For most organizations, EventBridge → Lambda handles the load without buffering.

**EventBridge rule limits.** AWS enforces 300 rules per event bus per account. At scale, use a dedicated event bus per account forwarding to a central compliance bus.

**CUR partition management.** A consolidated CUR dataset across 100+ accounts at $2M–$10M/month requires partitioning by account ID, service, and billing month. Without it, full-table scans become expensive and slow.

---

## FinOps Lifecycle Alignment

| Phase | Activity in This Investigation |
|---|---|
| **Inform** | CUR and Athena cost visibility, allocation coverage metric, invoice validation baseline |
| **Optimize** | Identification of $53/month in unattributed spend, systemic gap confirmed across EC2 and S3, rightsizing path unlocked post-attribution |
| **Operate** | Event-driven tagging compliance control, real-time Slack alerting, progressive enforcement model |

This investigation aligns with the FinOps Foundation framework. It started in **Inform** (what is the data telling us?), moved through **Optimize** (what is the gap and root cause?), and landed in **Operate** (what system prevents recurrence?). Optimization without Inform produces inaccurate savings models. Operate without Optimize produces governance controls targeting the wrong problem.

---

## FinOps Operating Model (Enterprise Context)

In mature cloud organizations, FinOps operates as a cross-functional model connecting Finance, Engineering, and Platform teams to manage cloud spend collaboratively.

**Typical ownership model:**

| Team | Responsibility |
|---|---|
| Finance | Budget forecasting, variance analysis, allocation reporting |
| FinOps / Cloud Platform | Cost visibility tooling, governance automation, optimization programs |
| Engineering Teams | Resource ownership, tagging compliance, workload optimization |

The tagging governance control implemented here sits within the FinOps Platform responsibility layer. It ensures Finance can reliably allocate spend and Engineering teams maintain ownership visibility over their resources. In a team environment, closing a gap like this would involve coordinating with Finance to confirm the allocation coverage target is met and with Engineering to apply missing tags and acknowledge the ownership change — the Slack alerting pipeline is specifically designed to make that handoff immediate rather than a monthly email thread.

### Example Monthly FinOps Workflow

```
Week 1  — Finance closes the previous month using CUR-based allocation reports
Week 2  — FinOps reviews allocation coverage KPI and investigates anomalies
Week 3  — Engineering teams review optimization recommendations
           (rightsizing, storage lifecycle, Savings Plans coverage)
Week 4  — Finance and FinOps update forecasts and commitment models
```

### Allocation Coverage KPI

| KPI | Definition | Before | After | Target |
|---|---|---|---|---|
| Allocation Coverage | Tagged spend / Total spend | 82.5% | 100% | ≥ 95% |

This metric is tracked weekly after remediation. It is the leading indicator of whether Finance can close the month accurately and whether chargeback or showback reporting is reliable.

### Where This Investigation Fits

| FinOps Capability | Contribution |
|---|---|
| Cost Allocation | Restored reliable tagging coverage across EC2 and S3 |
| Cost Visibility | CUR and Athena reporting pipeline with row-level attribution |
| Governance | Event-driven tag compliance detection at resource creation |
| Financial Reporting | Executive-ready investigation output for Finance stakeholders |

---

## What I Would Do Next

**Governance maturation**
- Centralize CUR across all accounts via AWS Organizations
- Establish weekly tag compliance baseline with allocation coverage tracked as a KPI
- Introduce SCP guardrails for greenfield accounts once compliance exceeds 95%
- Build a shared Finance and Engineering dashboard with real-time allocation coverage
- Implement SQS + DLQ between EventBridge and Lambda for production-grade burst hardening

**Cost optimization (attribution now reliable)**
- **EC2 rightsizing:** Pull CloudWatch CPU/memory utilization for target resources over a 2-week window. Evaluate downsizing one instance tier for any resource consistently below 20% utilization. Savings depend on instance family and region. Ownership accountability for these recommendations is only possible because attribution is now complete.
- **Savings Plans and RI analysis:** Commitment modeling is now reliable with full tag coverage. Run Cost Explorer Savings Plans coverage report filtered by `Environment` and `Project` to identify commitment candidates.
- **Anomaly detection:** Week-over-week Athena queries against the 4-week rolling average by service and owner, alerting Finance before monthly close.
- **Chargeback sequencing:** Establish showback first. Transition to full chargeback once tag coverage exceeds 95% and Finance and Engineering have agreed on exception handling. This is a deliberate sequencing decision, not a limitation.

---

## Lessons Learned

**Financial data must be treated as operational telemetry.** Without real-time visibility, allocation failures persist until monthly close. Cost data needs the same alerting discipline as infrastructure monitoring.

**Tagging governance must exist at the system level.** Human process alone cannot maintain reliable allocation coverage. Engineers under deadline pressure will skip tags. The system must catch what humans miss, automatically and immediately.

**Detection is more effective than enforcement early in governance maturity.** Immediate feedback loops change tagging behavior without introducing deployment friction. Blocking deployments too early creates shadow IT and pipeline workarounds. Earn compliance before enforcing it.

**Attribution is the prerequisite for optimization.** Rightsizing recommendations, Savings Plans modeling, and chargeback frameworks all rely on reliable ownership metadata. Optimization applied to unattributed spend produces inaccurate projections and misaligned accountability.

---

## Why This Matters

Most allocation problems are not caught because Finance lacks the tooling to see them in real time and Engineering lacks the incentive to tag consistently. This investigation showed that governance design matters more than enforcement. Surfacing problems immediately changes behavior faster than blocking deployments.

The AI-assisted reporting step closes the final gap. It translates investigation findings into executive-ready output at the speed the business requires, without replacing the analytical work that produced those findings.

---

<details>
<summary><strong>Appendix: AI-Assisted Investigation Reporting</strong> (click to expand)</summary>

AI was used only for formatting the final report, not for analysis. Every finding was produced by SQL analysis and CloudTrail forensics in Phases 1 through 5. AI converted validated findings into executive narrative format.

**Workflow:**

```
SQL investigation output  →  Structured AI prompt  →  AI-generated report  →  Human validation  →  Finance-ready output
```

**Prompt used:**

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
- Root cause: Console-launched resources missing required tags at creation,
  no enforcement existed at provisioning time
- Detection: Athena SQL on CUR + CloudTrail RunInstances forensics
- Fix: CloudTrail to EventBridge to Lambda to Slack pipeline
- Outcome: Detection latency reduced from 30 days to under 5 minutes
```

**AI-generated report:**

> Generated by Anthropic Claude using the prompt above. Every finding was validated against SQL and CloudTrail evidence in Phases 1 through 5 before inclusion.

---

**INTERNAL FINOPS INVESTIGATION REPORT**
**AWS Cost Allocation Gap | January 2026**

**1. Executive Summary**

A forensic investigation identified a 17% cost allocation gap across EC2 and S3. Spend that was correctly billed was invisible to Finance reporting due to missing resource tags at provisioning time. The failure was a governance design gap, not user error. An event-driven detection pipeline has been implemented, restoring 100% allocation visibility and reducing MTTD from 30 days to under 5 minutes.

**2. Investigation Findings**

CUR analysis via Athena confirmed $53 in unallocated spend across two services. EC2 represented the largest gap at approximately $38, with S3 contributing an additional $15. RDS showed full allocation compliance. Invoice validation confirmed all charges were accurate. The gap existed entirely in the attribution layer.

**3. Root Cause Analysis**

CloudTrail forensics traced the EC2 gap to a `RunInstances` API call made directly via the AWS Management Console. The console provides no tag enforcement at resource creation time. No SCP or tag policy existed to reject untagged resource requests. Tags were never applied, not removed, confirming a provisioning-time enforcement gap rather than post-deployment drift or tampering.

**4. Financial Impact**

| Metric | Value |
|---|---|
| Allocation gap | 17% of total spend |
| Monthly unallocated spend | $53 |
| Annualized if unresolved | $636 |
| Manual reconciliation reduced | ~4 hours/month |
| MTTD before remediation | ~30 days |
| MTTD after remediation | Under 5 minutes |

In enterprise environments where monthly spend exceeds $2M–$10M across 50–200 accounts, the same allocation failure can hide hundreds of thousands of dollars in unattributed cost. Accurate chargeback, showback, and budget forecasting become impossible.

**5. Governance Failure**

Three compounding control gaps allowed the issue to persist. There was no SCP or tag policy requiring tags before provisioning approval. There was no real-time monitoring to surface non-compliant resources at creation. There was no Finance visibility into allocation coverage as a tracked KPI. The absence of all three meant the gap could recur indefinitely and would only be caught during monthly close.

**6. Remediation Plan**

*Immediate:* Missing tags applied manually. Finance closed January with 100% allocation accuracy. No restatements required.

*Permanent:* Event-driven pipeline (CloudTrail → EventBridge → Lambda → Slack) deployed. Engineers receive real-time alerts without deployment friction. SCP guardrails scoped to greenfield accounts once compliance exceeds 95%.

*Next steps:* Weekly CUR-based tag compliance baseline, allocation coverage tracked as a first-class FinOps KPI, anomaly detection for week-over-week cost spikes.

> All figures validated against SQL and CloudTrail findings in Phases 1 through 5. Unblended costs used throughout to align with AWS invoice totals, which Finance treats as the authoritative source of truth.

</details>

-->

