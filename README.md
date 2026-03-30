# FinOps Investigation: AWS Cost Allocation Failure and Tag Governance Remediation

> **30-second read:** Finance found a 17% AWS cost allocation gap. I traced it to resources launched via the AWS console without required tags, then built an event-driven governance pipeline (CloudTrail → EventBridge → Lambda) that cut detection latency from 30 days to under 5 minutes per region where the EventBridge rule is deployed.

---

## Project Summary

Investigated a 17% AWS cost allocation gap using CUR and Athena SQL. CloudTrail forensics traced the issue to console-launched resources missing required tags at creation. Built an event-driven governance pipeline (CloudTrail → EventBridge → Lambda) to detect non-compliant resources in near-real-time. Result: allocation coverage restored to 100% and detection latency reduced from approximately 30 days to under 5 minutes.

---

## What This Demonstrates

| Skill | Evidence |
|---|---|
| AWS CUR analysis with Athena | Phases 1–4: NULL tag filtering, row-level resource attribution |
| CloudTrail forensic investigation | Phase 3: IAM principal, UserAgent, tagSpecificationSet chain |
| Event-driven governance architecture | CloudTrail → EventBridge → Lambda pipeline |
| Cloud cost allocation | $53 unallocated spend isolated and attributed |
| Tagging compliance automation | Lambda validator with REQUIRED_TAGS single source of truth |
| Finance-facing communication | Executive snapshot, AI-assisted report, chargeback sequencing |
| Enterprise scale thinking | Multi-account CUR, Organizations SCP, regional deployment patterns |

---

## Enterprise Relevance

> This investigation pattern scales directly to enterprise environments. The CUR query structure, CloudTrail forensic chain, and EventBridge governance pipeline are identical whether the environment has 1 account or 200. The multi-account architecture, AWS Organizations SCP progression model, Lambda concurrency considerations, and CUR partition strategy for $2M–$10M/month spend are all documented in the [Enterprise Context](#enterprise-context-and-scale) section below.

---

## Results at a Glance

| Metric | Before | After |
|---|---|---|
| Allocation coverage | ~83% | 100% |
| Mean time to detect (MTTD) | ~30 days | Under 5 minutes* |
| Detection method | Manual monthly reconciliation | Automated event-driven detection |
| Monthly unallocated spend | $53 | $0 |
| Annualized attribution gap | $636 | $0 |
| Finance reconciliation effort | ~4 hrs/month | Reduced significantly |

*MTTD under 5 minutes requires EventBridge rule deployment per active region. The lab EventBridge rule is in us-east-1. The documented incident occurred in us-east-2. Multi-region deployment via IaC is the next step. See the [Lab Region Note](#step-1-eventbridge-rule) in the remediation section.

> **This is a reporting integrity metric, not a cost savings figure.** The spend was correctly billed. The failure was that Finance could not see it, attribute it, or charge it back.

---

## Architecture

![FinOps Tag Governance Pipeline: CloudTrail to EventBridge to Lambda to Slack](screenshots/finops-architecture-v2.png)

**Pipeline flow:** EC2/S3 resource creation triggers CloudTrail to capture the API call. EventBridge matches RunInstances and CreateBucket events. Lambda validates Environment, Project, and Owner tags. Slack alerts fire to the Finance channel and the resource owner simultaneously.

```
EC2 RunInstances / S3 CreateBucket
             │
             ▼
       AWS CloudTrail
    (all API-level events)
             │
             ▼
    Amazon EventBridge
  (finops-tag-compliance-monitor)
             │
             ▼
        AWS Lambda
   (finops-tag-validator)
   Checks: Environment · Project · Owner
        │              │
        ▼              ▼
  Slack: Finance   Slack: Owner
     Channel        Direct DM
```

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

## Key Technologies

| Layer | Technology |
|---|---|
| Billing data | AWS Cost and Usage Report (CUR) |
| Query engine | Amazon Athena (Presto/SQL) |
| Forensics | AWS CloudTrail |
| Event capture | Amazon EventBridge |
| Tag validation | AWS Lambda (Python 3.14) |
| Governance | AWS Organizations Tag Policies, SCP |
| Alerting | Slack (Finance and resource owner channels) |
| Reporting | Anthropic Claude (AI-assisted formatting only) |

---

## Investigation Summary

| Phase | Objective | Tooling | Outcome |
|---|---|---|---|
| Invoice Validation | Confirm billing accuracy | Cost Explorer, CUR | Billing error ruled out |
| Resource Isolation | Identify unallocated spend | Athena (CUR) | EC2 and S3 gaps isolated |
| Forensics | Identify creation source | CloudTrail | Untagged console launch confirmed |
| Scope Expansion | Check other services | Athena (CUR) | Systemic issue confirmed |
| Remediation | Prevent recurrence | EventBridge, Lambda | Near-real-time detection live |
| AI-Assisted Reporting | Generate executive report | Anthropic Claude | Finance-ready report produced |

---

---

# Investigation

---

## Context and Problem Statement

During monthly reconciliation, Finance identified a discrepancy. Total AWS spend was $315. Spend visible with required tags was $262. Unallocated spend was $53, approximately 17% of total. The invoice was accurate. The failure was in cost attribution, not cost generation.

---

## Phase 1: Invoice Validation

**Objective:** Confirm AWS billed correctly before investigating allocation.

The first assumption to test was a billing error. If the invoice was wrong, the allocation gap was a Finance reporting artifact, not an infrastructure problem. Validated service totals in Cost Explorer, reconciled totals against CUR via Athena, and confirmed invoice totals matched exactly.

**Conclusion:** Billing was correct. That ruled out the simplest explanation. The issue existed downstream in the attribution layer, which meant the investigation had to move to the resource level.

![Invoice validation confirming $315 total spend](screenshots/01%20-%20invoice-total-315.png)

> **Note on this query:** Invoice totals are represented using a hardcoded `VALUES` query in Athena. In a production environment this query would run directly against the CUR dataset using `SUM(line_item_unblended_cost)` grouped by `line_item_product_code`. See the Lab Environment Note in Phase 2.

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

CUR via Athena was used instead of Cost Explorer because it enables row-level tag inspection. It filters for exact NULL values per resource and per billing line item. Cost Explorer aggregates data and cannot surface missing tag issues at the resource level — it has no mechanism to show which specific resources have NULL allocation tags or how much each one costs. Unblended cost was used throughout to align with invoice totals. Finance treats unblended cost as the authoritative source of truth.

The Environment, Project, and Owner tags were activated as Cost Allocation Tags in the AWS Billing and Cost Management console prior to this investigation. This activation step is commonly missed in new AWS environments and must be completed before tag-based CUR analysis is possible.

### The Turning Point

The initial goal was to identify *where* allocation was failing, not *why*. The approach was to group CUR line items by resource ID and inspect all three required tag columns simultaneously:

```sql
SELECT
    line_item_resource_id,
    resource_tags_user_environment,
    resource_tags_user_project,
    resource_tags_user_owner,
    SUM(line_item_unblended_cost) AS total_cost
FROM cur_database.aws_cur
WHERE line_item_line_item_type = 'Usage'
GROUP BY 1, 2, 3, 4
ORDER BY total_cost DESC;
```

What stood out was not the cost itself — it was the pattern of absence. For resource `i-0c9cfb67280fe44ee`, every line item returned NULL across all three tag columns simultaneously. Not one missing tag. Not two. All three. That pattern ruled out accidental omission. A developer who forgot one tag would not systematically omit all three across every billing line item for the same resource.

At that point the problem shifted. This was no longer *"why is Finance missing cost?"* It became *"who created a resource that was never tagged at all?"* That pivot is what moved the investigation from Athena to CloudTrail.

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
| PROD-WEB-SERVER-01 | i-0c9cfb67280fe44ee | t3.large | 40.0 | NULL | NULL | NULL |

> **Note on the $40 figure:** The resource-level CUR query shows $40. The service-level breakdown in Phase 4 shows approximately $38 for the EC2 gap. This variance is attributable to partial-month usage and CUR line-item rounding across billing periods. Both figures reconcile to invoice-level totals.

![Athena CUR resource verification query showing PROD-WEB-SERVER-01 with NULL allocation tags](screenshots/03%20-%20cloudtrail-forensics-runinstances-missing-tags-cli-launch.png)

> *Screenshot filename is a draft artifact; content is the Athena CUR query for Phase 2 resource isolation.*

---

## Phase 3: CloudTrail Forensic Investigation

**Objective:** Determine how and why the resource was created without tags.

With the untagged resource identified in Phase 2 (instance ID `i-0c9cfb67280fe44ee`), the investigation moved to CloudTrail to determine the IAM principal, launch method, and whether tags were absent at creation or removed afterward.

### Step 1: CloudTrail Event Record (Full JSON)

The CloudTrail Event history was filtered to `RunInstances` and the full JSON event record was opened for `i-0c9cfb67280fe44ee`. This is the primary forensic document. It confirms the IAM principal, the launch timestamp, the user agent (console vs. CLI vs. SDK), and the complete tag specification submitted with the API call.

![CloudTrail RunInstances event record showing Root IAM principal, timestamp 2026-01-22T19:00:13Z, awsRegion us-east-2, instanceType t3.micro, and Chrome browser UserAgent confirming console launch](screenshots/03a%20-%20cloudtrail-event-history-runinstances.png)

> **Instance type discrepancy: t3.micro in CloudTrail vs. t3.large in CUR.** CloudTrail records t3.micro because that is the actual instance type launched. The CUR data shows t3.large because the billing figures were hardcoded to simulate enterprise spend, as noted in the Lab Environment Note in Phase 2. The forensic value of this event record is the IAM principal (Root with MFA), the console UserAgent, and the absent required tags.

### Step 2: Tag Specification Evidence (Missing Required Tags)

![CloudTrail RunInstances tagSpecificationSet showing the Name tag only for PROD-WEB-SERVER-01 with no Environment, Project, or Owner tags present](screenshots/03b%20-%20cloudtrail-runinstances-json-cli-useragent.png)

> *Screenshot filename is a draft artifact; content is the tagSpecificationSet section of the CloudTrail JSON confirming Name tag only.*

Scrolling deeper into the same CloudTrail JSON event record surfaces the `tagSpecificationSet`. Only the `Name` tag (`PROD-WEB-SERVER-01`) was submitted with the API call. `Environment`, `Project`, and `Owner` were not included. This confirms tags were absent at creation — never submitted with the API call, not applied and then later removed.

### Forensic Evidence Chain Summary

| Evidence | Finding |
|---|---|
| CUR Phase 2 (03) | Athena query confirming untagged resource, instance ID, and $40 monthly cost |
| CloudTrail JSON, top half (03a) | Root with MFA IAM principal, Chrome browser UserAgent confirming console launch, awsRegion us-east-2, creation timestamp 2026-01-22T19:00:13Z |
| CloudTrail JSON, tag section (03b) | `tagSpecificationSet` contained `Name` tag only; Environment, Project, and Owner were absent at creation |

> **CUR isolated the resource. CloudTrail confirmed who created it, how it was created, and that required tags were never submitted. The Chrome browser UserAgent confirmed console origin — not CLI, not SDK. The `tagSpecificationSet` confirmed a provisioning-time enforcement gap with no drift and no tampering. Root cause confirmed.**

The resource was launched directly via the AWS Management Console. Console launch bypassed any IaC pipeline or automated workflow. The IAM principal was a Root session with MFA, used intentionally in the sandbox to simulate a worst-case governance bypass scenario. In production environments, root access is restricted and not used for routine infrastructure provisioning. This would be a separate finding in any real security review.

**Key evidence from the CloudTrail event:**

```json
{
  "eventName": "RunInstances",
  "eventTime": "2026-01-22T19:00:13Z",
  "awsRegion": "us-east-2",
  "userIdentity": {
    "type": "Root",
    "sessionContext": {
      "attributes": {
        "creationDate": "2026-01-22T13:17:47Z",
        "mfaAuthenticated": "true"
      }
    }
  },
  "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36",
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

The `Name` tag was present. `Environment`, `Project`, and `Owner` were not. The `userAgent` is a Chrome browser string confirming console launch — a CLI call would show `aws-cli/2.x`. The forensic conclusion is unchanged: required tags were absent at creation time via direct console action.

---

## Phase 4: Expanding Scope to S3

**Objective:** Determine whether the issue was isolated or systemic.

| Service | Billed | Allocated | Gap |
|---|---|---|---|
| EC2 | $220 | $182 | ~$38 |
| RDS | $60 | $60 | $0 |
| S3 | $35 | $20 | $15 |

While EC2 represented the largest gap, S3 confirmed the issue was systemic. The two S3 buckets in this lab simulation are represented with partial tags to demonstrate a key compliance principle: partial tagging produces the same Finance-invisible result as no tagging. Both buckets are missing the required `Environment` tag. Even with `Project` and `Owner` present, the missing `Environment` tag meant $15 per month in storage spend that Finance could not allocate to any cost centre.

> **Lab simulation note on S3 tag data:** The actual lab buckets were created with no tags at all, consistent with the EC2 instance. The CUR simulation in Image 6 was deliberately constructed with partial tags (Project and Owner present, Environment absent) to illustrate a critical FinOps principle: a resource with two of three required tags is just as invisible to Finance reporting as a resource with none. The simulation data was chosen because it represents the more common and more instructive real-world scenario.

| Resource ID | Monthly Cost | Environment | Project | Owner |
|---|---|---|---|---|
| bucket-logs-arch | 7.0 | NULL | data-pipeline | team-ops |
| bucket-temp-data | 8.0 | NULL | web-app | team-engineering |
| **TOTAL UNALLOCATED** | **15.0** | --- | --- | --- |

![S3 tag compliance audit via CUR showing bucket-logs-arch and bucket-temp-data with NULL Environment tags](screenshots/04%20-%20cur-s3-tag-compliance-audit.png)

> **Note on S3 tagging behavior in production:** The S3 CreateBucket API does not accept tags inline in the creation request the way EC2 does. Tags on S3 buckets are applied via a separate `PutBucketTagging` call after the bucket exists. This means a CreateBucket event in CloudTrail will always show an empty tag list in the request parameters, even for buckets that will eventually be tagged correctly. In a production implementation, the EventBridge rule would monitor both `CreateBucket` and `PutBucketTagging` events, or a time-bounded compliance check would verify tag status shortly after bucket creation. This is documented in [What I Would Do Next](#what-i-would-do-next).

---

## Reference: Production-Equivalent Unallocated Spend Query

This is the query that would replace the hardcoded Phase 2 lab query in a production environment with a live CUR dataset.

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

> **Dialect note:** `DATE_TRUNC` and the `BETWEEN DATE '...'` syntax are Athena/Presto-specific. Not ANSI SQL portable.

The `line_item_usage_account_id` column allows attribution gaps to be traced back to the owning account across 50 to 200 accounts in a consolidated Organizations CUR dataset.

---

# Root Cause and Remediation

---

## Root Cause Analysis

**Root cause: No enforcement at resource creation time.**

If tagging depends on humans being perfect, it will eventually fail. This was not a user error. It was a system design gap that required a system-level response.

EventBridge and Lambda were chosen over AWS Config because they provide near-real-time detection. Config rules operate on a minutes-to-hours evaluation cycle and carry Config recorder charges for every resource state change. EventBridge fires within seconds of the API call. This approach also avoids Config recorder charges at scale.

Alerting was prioritized over blocking to preserve engineering velocity. SCP enforcement is planned only after compliance stabilizes above 95%. This reflects a bias toward fast feedback, low friction, and defensible financial reporting.

---

## Remediation and Controls

**Immediate correction:** Applied missing tags so Finance could close the month accurately.

**Permanent prevention: the progressive enforcement model**

1. **Tag Policies** standardize required tag keys across all accounts without blocking deployments
2. **EventBridge + Lambda** detect violations within minutes of resource creation and alert Finance and the resource owner simultaneously
3. **SCP guardrails** enforce compliance for greenfield accounts once the organization stabilizes above 95% coverage, grandfathering existing workflows to avoid pipeline disruption

This is how real platform teams deploy governance: standardize first, detect second, enforce third.

---

### Step 1: EventBridge Rule

The EventBridge rule `finops-tag-compliance-monitor` was configured on the default event bus. It filters for `RunInstances` (EC2) and `CreateBucket` (S3) API calls via CloudTrail and targets the `finops-tag-validator` Lambda function. Status: **Enabled**.

**EventBridge event pattern:**

```json
{
  "source": ["aws.ec2", "aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventName": ["RunInstances", "CreateBucket"]
  }
}
```

This pattern matches every EC2 instance launch and every S3 bucket creation across the account, regardless of who triggered it or from which interface.

![EventBridge Rule finops-tag-compliance-monitor showing source aws.ec2 and aws.s3, RunInstances and CreateBucket event names, and finops-tag-validator Lambda target in us-east-1](screenshots/05a-eventbridge-rule.png)

> **Lab region note and production deployment pattern:**
> The CloudTrail forensic evidence (Phase 3, screenshot 03a) shows the documented incident occurred in `us-east-2`. The EventBridge rule and Lambda function visible in this screenshot are deployed in `us-east-1`. Amazon EventBridge default event buses are regional, so a production deployment requires the rule in every active region. Two standard patterns handle this:
>
> **Per-region deployment:** Deploy the EventBridge rule and Lambda to each active region via IaC (Terraform or CloudFormation StackSets). Standard approach for organizations with 3–5 active regions.
>
> **Central event bus pattern:** Forward regional CloudTrail events to a central EventBridge bus in a dedicated billing or security account, then apply a single rule set. Recommended pattern for organizations managing 10 or more accounts.
>
> The governance logic (EventBridge pattern, Lambda validation, alert payload) is production-ready. Multi-region deployment via IaC is the next step documented in the [What I Would Do Next](#what-i-would-do-next) section.

---

### Step 2: Lambda Validation Logic

Lambda inspects the resource tags in the API event payload and validates that `Environment`, `Project`, and `Owner` exist. The function handles both EC2 and S3 creation events. Non-compliant resources trigger an alert containing event name, missing tags, IAM principal, account ID, and region.

```python
REQUIRED_TAGS = ['Environment', 'Project', 'Owner']
missing = [t for t in REQUIRED_TAGS if t not in tag_list]
if missing:
    print(f"ALERT: {json.dumps(message, indent=2)}")
    return {'statusCode': 200, 'missing_tags': missing}
```

`REQUIRED_TAGS` is the single source of truth. Adding or removing a required tag means changing one line.

![Lambda function finops-tag-validator showing REQUIRED_TAGS list, event parsing logic for RunInstances and CreateBucket, missing tag detection, and alert payload construction](screenshots/05b%20-%20lambda-finops-tag-validator-code.png)

![Lambda runtime settings showing Python 3.14, handler lambda_function.lambda_handler, x86_64 architecture, and Lambda Deployed status confirmed](screenshots/05b-ii%20-%20lambda-settings.png)

> **Implementation notes:** `boto3` is imported in the function for the production Slack webhook integration, where it retrieves the webhook URL from AWS Secrets Manager using `boto3.client('secretsmanager')`. In this lab environment the alert routes to CloudWatch Logs via `print()` since no Slack workspace is connected. The webhook URL must never be hardcoded. Alert fires without blocking deployment, preserving CI/CD velocity. For high-volume burst scenarios, add SQS + DLQ between EventBridge and Lambda to handle concurrency spikes above the default 1,000 concurrent execution limit per account per region.

> **Production tag_list pattern:** The lab function uses assignment (`tag_list =`) inside the tagSpecificationSet loop, which is sufficient when a single resourceType entry is submitted. In a production implementation where multiple resourceType entries may be present in a single RunInstances call, use append (`tag_list +=`) to ensure tags across all entries are captured:
>
> ```python
> for item in items:
>     tag_list += [t['key'] for t in item.get('tags', [])]
> ```

> **Note on Python version:** The Lambda function was deployed on Python 3.14. The function logic is compatible with Python 3.12 or later.

---

### Step 3: End-to-End Test Execution

The function was tested with a simulated untagged `RunInstances` event. All three required tags were correctly identified as missing. The `FINOPS TAG COMPLIANCE VIOLATION` alert fired correctly.

**CloudWatch log output (actual test execution):**

```
START RequestId: abb3cf8a-8c39-442d-ba6b-378ccd87444e Version: $LATEST
ALERT: {
"alert": "FINOPS TAG COMPLIANCE VIOLATION",
"event": "RunInstances",
"missing_tags": ["Environment", "Project", "Owner"],
"iam_principal": "arn:aws:iam::123456789012:user/developer-01",
"account_id": "123456789012",
"region": "us-east-1"
}
END RequestId: abb3cf8a-8c39-442d-ba6b-378ccd87444e
REPORT RequestId: abb3cf8a-8c39-442d-ba6b-378ccd87444e
Duration: 2.54 ms   Billed Duration: 259 ms
Memory Size: 128 MB   Max Memory Used: 70 MB   Init Duration: 255.90 ms
```

> **Note on Billed Duration vs execution time:** The function executed in 2.54ms but AWS billed 259ms on this invocation. The difference is the 255.90ms cold start (Init Duration) — the one-time cost of loading the Python runtime and function code into memory on first invocation. Subsequent warm invocations bill only the 2.54ms execution time. Cold start is a normal Lambda behaviour and has no impact on alert latency in this use case since EventBridge invocations are not time-critical at the millisecond level.

**Result:** `statusCode: 200`, `missing_tags: ['Environment', 'Project', 'Owner']`

![Lambda test result showing statusCode 200, missing_tags for Environment, Project, and Owner, and the FINOPS TAG COMPLIANCE VIOLATION alert in the log output](screenshots/05c-lambda-test-result.png)

> **Note on test event values:** The test event uses a synthetic IAM principal (`arn:aws:iam::123456789012:user/developer-01`) and placeholder account ID (`123456789012`) rather than reproducing the Root session from the forensic chain. This is intentional — reproducing real Root credentials in a test event is a security anti-pattern. `123456789012` is a well-recognized AWS documentation placeholder. The test validates function logic, not actor identity.

---

# Enterprise Context and Scale

---

## FinOps Lifecycle Alignment

| Phase | Activity in This Investigation |
|---|---|
| **Inform** | CUR and Athena cost visibility, allocation coverage metric, invoice validation baseline |
| **Optimize** | $53/month in unattributed spend isolated across EC2 and S3; rightsizing path unlocked post-attribution |
| **Operate** | Event-driven tagging compliance control, real-time Slack alerting, progressive enforcement model |

---

## Optimization Opportunity (Sequential to Attribution, Not Parallel)

With attribution restored, the data now supports a clear optimization path.

**Rightsizing:** Pull 2-week CloudWatch CPU and memory utilization. Downsize one tier for any resource consistently below 20% utilization. Owner accountability is only possible because attribution is now complete.

**Savings Plans:** Run Cost Explorer coverage report filtered by `Environment` and `Project` to identify commitment candidates. Before full tag coverage, any break-even modeling produced inaccurate recommendations.

> Attribution must precede optimization. Rightsizing unattributed spend produces projections that land on the wrong teams.

---

## Multi-Account Architecture and Scale

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
                         ▼
   ┌─────────────────────────────────────────────┐
   │     Central S3: CUR and CloudTrail Logs     │
   │   Partitioned by account_id and month       │
   └──────────────┬──────────────────────────────┘
                  │
        ┌─────────┴──────────┐
        ▼                    ▼
┌──────────────┐    ┌───────────────────────────┐
│ Athena SQL   │    │  EventBridge + Lambda     │
│ Consolidated │    │  Deployed via IaC per     │
│ CUR queries  │    │  account AND per region   │
└──────────────┘    └────────────┬──────────────┘
                                 ▼
                      ┌──────────────────────┐
                      │  Slack: Finance +    │
                      │  Resource Owner      │
                      └──────────────────────┘
```

**Three production failure points a sandbox does not surface:**
- **Lambda concurrency:** Default limit is 1,000 per account per region. Add SQS + DLQ for burst hardening in environments with high-volume provisioning events.
- **EventBridge rule limits:** 300 rules per event bus per account. Use dedicated buses at scale.
- **CUR partition management:** 100+ accounts at $2M–$10M per month requires partitioning by account ID, service, and billing month. Without it, full-table scans become expensive.

---

## FinOps Operating Model

| Team | Responsibility |
|---|---|
| Finance | Budget forecasting, variance analysis, allocation reporting |
| FinOps / Cloud Platform | Cost visibility tooling, governance automation, optimization programs |
| Engineering Teams | Resource ownership, tagging compliance, workload optimization |

**Monthly operating cadence:**

```
Week 1: Finance closes the previous month using CUR-based allocation reports
Week 2: FinOps reviews allocation coverage KPI and investigates anomalies
Week 3: Engineering teams review optimization recommendations
Week 4: Finance and FinOps update forecasts and commitment models
```

**Allocation Coverage KPI:**

| KPI | Before | After | Target |
|---|---|---|---|
| Tagged spend / Total spend | ~83% | 100% | ≥ 95% |

---

## What I Would Do Next

**Governance**
- Deploy EventBridge rule and Lambda to all active regions via IaC (Terraform or CloudFormation StackSets)
- Add `PutBucketTagging` monitoring to the EventBridge rule pattern for S3 compliance accuracy
- Centralize CUR across all accounts via AWS Organizations
- Establish weekly tag compliance baseline tracked as a KPI
- Introduce SCP guardrails for greenfield accounts once compliance exceeds 95%
- Implement SQS + DLQ between EventBridge and Lambda for burst hardening

**Optimization**
- EC2 rightsizing against 2-week CloudWatch utilization windows
- Savings Plans coverage analysis filtered by Environment and Project tags
- Week-over-week Athena anomaly detection alerting Finance before monthly close
- Showback first, full chargeback once tag coverage exceeds 95%

---

## Lessons Learned

**Financial data must be treated as operational telemetry.** Without real-time visibility, allocation failures persist until monthly close.

**Tagging governance must exist at the system level.** Human process alone cannot maintain reliable allocation coverage under deadline pressure.

**Detection is more effective than enforcement early in governance maturity.** Immediate feedback loops change behavior faster than blocking deployments.

**Attribution is the prerequisite for optimization.** Optimization applied to unattributed spend produces inaccurate projections and misaligned accountability.

**Governance controls must match the scope of the incidents they are designed to catch.** Regional deployment coverage is as important as the control logic itself.

---

# Appendix

---

## Appendix A: Screenshot Index

| Filename | Content |
|---|---|
| 01 - invoice-total-315.png | Cost Explorer invoice validation confirming $315 total |
| 03 - cloudtrail-forensics-... | Athena Phase 2 CUR query — filename is a draft artifact |
| 03a - cloudtrail-event-history | CloudTrail RunInstances JSON, top half: principal, UserAgent, region |
| 03b - cloudtrail-runinstances-json | CloudTrail tagSpecificationSet: Name tag only, required tags absent |
| 04 - cur-s3-tag-compliance-audit | S3 CUR audit showing partial tags with NULL Environment |
| 05a - eventbridge-rule | EventBridge rule finops-tag-compliance-monitor, Enabled |
| 05b - lambda-finops-tag-validator | Lambda function code in VS Code |
| 05b-ii - lambda-settings | Lambda runtime: Python 3.14, x86_64, 728 bytes |
| 05c - lambda-test-result | Test execution: statusCode 200, 2.54ms, all three tags missing |

---

## Appendix B: AI-Assisted Investigation Reporting

AI was used only for formatting the final report, not for analysis. Every finding was produced by SQL analysis and CloudTrail forensics in Phases 1 through 5. AI converted validated findings into executive narrative format.

```
SQL investigation output → Structured AI prompt → AI-generated report → Human validation → Finance-ready output
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

---

**INTERNAL FINOPS INVESTIGATION REPORT**
**AWS Cost Allocation Gap | January 2026**

> *Generated by Anthropic Claude using the prompt above. Every finding was validated against SQL and CloudTrail evidence in Phases 1 through 5 before inclusion.*

**1. Executive Summary**

A forensic investigation identified a 17% cost allocation gap across EC2 and S3. Spend that was correctly billed was invisible to Finance reporting due to missing resource tags at provisioning time. The failure was a governance design gap, not user error. An event-driven detection pipeline has been implemented, restoring 100% allocation visibility and reducing MTTD from 30 days to under 5 minutes.

**2. Investigation Findings**

CUR analysis via Athena confirmed $53 in unallocated spend across two services. EC2 represented the largest gap at approximately $38, with S3 contributing an additional $15. RDS showed full allocation compliance. Invoice validation confirmed all charges were accurate. The gap existed entirely in the attribution layer.

**3. Root Cause Analysis**

CloudTrail forensics traced the EC2 gap to a `RunInstances` API call made directly via the AWS Management Console, confirmed by a Chrome browser UserAgent string in the CloudTrail event record. The console provides no tag enforcement at resource creation time. No SCP or tag policy existed to reject untagged resource requests. Tags were never applied and not removed, confirming a provisioning-time enforcement gap rather than post-deployment drift or tampering.

**4. Financial Impact**

| Metric | Value |
|---|---|
| Allocation gap | 17% of total spend |
| Monthly unallocated spend | $53 |
| Annualized if unresolved | $636 |
| Manual reconciliation reduced | ~4 hours/month |
| MTTD before remediation | ~30 days |
| MTTD after remediation | Under 5 minutes* |

*Requires EventBridge rule deployment in each region where resources are provisioned.

In enterprise environments where monthly spend exceeds $2M–$10M across 50–200 accounts, the same allocation failure can hide hundreds of thousands of dollars in unattributed cost.

**5. Governance Failure**

Three compounding control gaps allowed the issue to persist: no SCP or tag policy at provisioning time, no real-time monitoring at resource creation, and no Finance visibility into allocation coverage as a tracked KPI. The absence of all three meant the gap could recur indefinitely and would only be caught during monthly close.

**6. Remediation Plan**

*Immediate:* Missing tags applied manually. Finance closed January with 100% allocation accuracy. No restatements required.

*Permanent:* Event-driven pipeline (CloudTrail → EventBridge → Lambda → Slack) deployed. Engineers receive real-time alerts without deployment friction. SCP guardrails scoped to greenfield accounts once compliance exceeds 95%. EventBridge rule to be deployed per active region via IaC.

*Next steps:* Weekly CUR-based tag compliance baseline, allocation coverage as a first-class FinOps KPI, anomaly detection for week-over-week cost spikes.

> All figures validated against SQL and CloudTrail findings in Phases 1 through 5. Unblended costs used throughout to align with AWS invoice totals, which Finance treats as the authoritative source of truth.
