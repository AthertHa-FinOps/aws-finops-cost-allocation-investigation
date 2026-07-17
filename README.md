# AWS Cloud Governance and Cost Allocation Audit: Investigating a Tag Compliance Control Failure

*This is the full technical write-up. For the 5-minute version, see the summary README in my
other repo.*

> **Project context:** This is a self directed learning project completed in a personal AWS sandbox account, 
> not a professional engagement or paid consulting work. Cost figures are intentionally lab scaled ($315 
> total spend) so the investigation could be completed end to end without financial risk. Although 
> the environment is intentionally small, the investigation methodology, evidence collection approach, SQL 
> analysis, and governance decisions reflect the same control assessment practices used in larger enterprise 
> environments.

## Governance Focus

This portfolio demonstrates governance capabilities across cloud financial controls, asset attribution, 
audit evidence collection, control assessment, and 
operational risk reduction.

AWS provides the technical environment for the investigation. The transferable skills demonstrated 
throughout this work are evidence validation, control 
analysis, root cause investigation, governance decision making, and designing practical governance controls 
that strengthen operational accountability.

---

## Previous Compliance Experience

Before transitioning into cloud governance, I spent more than thirteen years working in regulated real 
estate transactions where every file had to reconcile against 
contracts, disclosures, and supporting documentation before it could close.

That experience shaped how I approached this investigation. Rather than treating missing resource tags as 
simply a technical issue, I approached them as an audit 
problem: reconcile financial records against operational evidence, identify where the control failed, 
determine why it failed, and recommend controls that reduce 
the likelihood of recurrence.

Although the technology is different, the underlying discipline is the same. Both environments require 
evidence based decision making, documentation accuracy, 
reconciliation, compliance, and repeatable controls that improve accountability rather than one time fixes.

---

## Portfolio Overview

This portfolio contains two cloud governance investigations that follow the same underlying approach: 
reconcile the system of record against operational evidence, 
identify where evidence diverges, determine why the control failed and how it should be strengthened, and 
design practical remediation.

The first investigation focuses on a cost allocation failure caused by missing resource tags. The second 
examines cross account spending patterns, financial 
controls, and commitment optimization using consolidated billing data.

Both investigations use a personal AWS environment, but the investigative methodology is intended to mirror 
governance and audit practices used in larger 
organizations.

The first case study demonstrates an investigation into a cloud cost allocation control failure. It follows 
the complete governance lifecycle from financial 
reconciliation through forensic investigation, root cause analysis, control design, and operational 
recommendations.

Across both investigations I follow the same audit methodology: validate the financial record, isolate the 
discrepancy, collect operational evidence, identify the control failure, quantify the business impact, and 
recommend a governance improvement proportional to the risk.

---

# Case Study 1: AWS Cost Allocation Failure and Tag Governance Remediation

A 17% cost allocation gap ($53 unallocated from $315 in monthly spend) was invisible to Finance chargeback reporting. This investigation
traces the gap to its source, remediates it, and implements a detection pipeline to prevent recurrence.

This case walks through how a cloud cost allocation failure was investigated end to end, from CUR analysis through CloudTrail forensics to 
event-driven governance remediation. The forensic chain (CUR → CloudTrail → IAM principal → `tagSpecificationSet`) is production-grade. The 
governance pipeline (CloudTrail → EventBridge → Lambda) is the correct control for out-of-band provisioning after IaC prevention controls 
are in place.

> **Key forensic finding:** This investigation proves that required tags were never submitted at creation time using CloudTrail
`tagSpecificationSet` evidence, not removed post-deployment. That distinction determines whether the failure is a provisioning control gap
or a configuration drift problem — the difference between a system design failure and a human error. The evidence confirms the former.

---

### Investigation Artifacts and Production Evolution

The screenshots in this case study capture the **lab build and iterative refinement** of the detection pipeline. Where screenshots reflect 
an earlier implementation state, the accompanying text documents the production-hardened architecture. This progression mirrors how FinOps 
controls mature in enterprise environments: detection is shipped first to establish visibility, false positives are eliminated through 
iterative hardening, and enforcement is applied only after the control has proven trustworthy to Finance.

Data sensitivity considerations mean that Athena query screenshots use simulated values to illustrate result structure without exposing live
account identifiers, principal ARNs, or production cost data. The SQL patterns are production-validated; the specific values are 
representative.

---

## Results at a Glance

**Simulated Remediation Impact**

| Metric | Before | After |
|---|---|---|
| Allocation coverage | ~83% | 100% |
| Mean time to detect (MTTD) | ~30 days | 1–3 minutes† |
| Detection method | Manual monthly reconciliation | Automated event-driven detection |
| Monthly unallocated spend | $53 | $0 |
| Annualized attribution gap | $636 | $0 |
| Finance reconciliation effort | ~4 hrs/month | Materially reduced |
| Chargeback accuracy | Incomplete | Restored to 100% for affected cost centres |

†Lab-simulated outcomes based on restored tag coverage. Production impact is projected, not measured.

†End-to-end detection latency is bounded by CloudTrail delivery latency plus EventBridge propagation delay. Under normal conditions this 
lands in the low-minute range. CloudTrail is not a real-time system and delivery can vary by region and load, so treat this as a typical 
observed range rather than a hard SLA.

**Forward KPIs (post-control, first-iteration baselines)**

| Metric | Target | Measurement Method |
|---|---|---|
| Mean time to remediate (MTTR) | Under 24 hrs from alert | Tag compliance restored timestamp from CUR or Config minus EventBridge alert 
timestamp |
| Repeat violation rate | Below 10% within 30 days | Same principal ARN appearing in violation events within a rolling 30-day CloudTrail 
window |
| Compliance trend | 95% within 60 days | Weekly `tagged spend / total spend` from CUR, grouped by `line_item_usage_account_id` |
| Tag validity rate | 90% within 60 days | Percentage of tag values matching allowed value sets defined in Tag Policies or a centralised 
value registry; measures correctness, not just presence |

> **Detection speed only creates value if action follows.** MTTD matters when paired with a clear ownership model and a remediation SLA. The
lagging indicators that prove the control is working are MTTR and repeat violation rate. A compliance trend moving from 83% toward 95% over
60 days is the metric Finance actually cares about. That is the threshold at which showback becomes chargeback and the governance model
becomes self-sustaining.

> **This is a financial risk problem, not a cost savings problem.** The spend was correctly billed. The failure was that Finance could not
see it, attribute it, or charge it back. Unattributed spend affects three areas Finance leadership tracks directly: chargeback accuracy
(cost assigned to the wrong team or not assigned at all), forecast reliability (budget models built on a spend baseline missing 17% of
actual consumption), and commitment sizing (Savings Plans and RI coverage recommendations sized against incomplete data). Restoring
allocation coverage corrects all three.

---

## What This Demonstrates

> **Project context:** This is a self-directed learning project completed in a personal AWS 
> sandbox account, not a professional engagement or paid consulting work. Cost figures are 
> intentionally lab-scaled ($315 in total spend) so the methodology could be tested end to end 
> without financial risk. The investigation techniques, SQL patterns, and governance reasoning 
> are written to map directly onto enterprise environments regardless of spend volume — that 
> mapping is explained throughout the document.

| Skill | Evidence |
|---|---|
| AWS CUR analysis with Athena | Phases 1–4: NULL tag filtering, row-level resource attribution |
| CloudTrail forensic investigation | Phase 3: IAM principal, session attribution chain, UserAgent, tagSpecificationSet |
| Control hierarchy reasoning | Control Hierarchy section: IaC enforces at design time, SCP/IAM enforce at runtime, Tag Policies standardize
schema, EventBridge detects out-of-band behaviour |
| Control selection and tradeoff reasoning | Appendix C: EventBridge vs Config on event fidelity and cost profile, detection vs SCP, what 
was rejected and why |
| Pipeline failure mode awareness | Appendix D: seven named failure modes with detection methods and alert targets |
| Blast radius quantification | Enterprise Context section: $53 lab example translated to $10M/month spend impact |
| Event-driven governance architecture | CloudTrail → EventBridge → Lambda pipeline with EC2 and S3 branching |
| S3 temporal compliance window | Dual-event monitoring plus delayed validation Lambda to account for asynchronous tagging |
| Finance-facing communication | Executive snapshot, AI-assisted report, chargeback sequencing |
| Enterprise scale thinking | Multi-account CUR, Organizations SCP, regional deployment patterns |

---

## Architecture

**Pipeline flow:** EC2 or S3 resource creation triggers CloudTrail to capture the API call. EventBridge matches `RunInstances`, 
`CreateBucket`, and `PutBucketTagging` events. Lambda validates `Environment`, `Project`, and `Owner` tags using service-specific branching
logic. Slack alerts fire to the Finance channel and the resource owner simultaneously.

> Config was evaluated but EventBridge was selected because it captures full request context including `tagSpecificationSet`, which Config
cannot access at the API call layer. The detailed comparison is available on request.

![FinOps Architecture](screenshots/finops-architecture-v2.png)

*Architecture diagram: end-to-end tag governance pipeline showing resource creation events, CloudTrail capture, EventBridge rule matching, 
Lambda validation logic, and dual Slack alerting to Finance channel and resource owner. Impact metrics show MTTD reduced from 30 days to 
under 5 minutes and coverage restored to 100%.*

```
EC2 RunInstances / S3 CreateBucket / S3 PutBucketTagging
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
| **Problem** | Finance identified a 17% AWS cost allocation gap. The spend was correctly billed but missing required allocation tags, which
made it invisible to chargeback reports. |
| **Investigation** | Used AWS CUR and Athena SQL to isolate untagged spend. Used CloudTrail forensic analysis to trace the root cause to
resources launched via the AWS console without tags at creation. |
| **Root Cause** | Primary failure: no preventive tagging control at resource creation time. Contributing factors: no attributable identity
chain, removing accountability and making enforcement non-actionable; and tag activation dependency risk, which would have silently disabled 
CUR-based investigation entirely if not already in place. |
| **Solution** | Implemented an event-driven tagging compliance pipeline. CloudTrail feeds EventBridge, which triggers Lambda, which sends
Slack alerts to Finance and the resource owner. |
| **Result** | Allocation coverage restored to 100%. End-to-end MTTD reduced to low-minute level under normal conditions, bounded by 
CloudTrail delivery and EventBridge propagation latency. This is not a hard SLA. Chargeback accuracy restored. Finance manual reconciliation
effort materially reduced. |

**Operating model considerations**

A control without clear ownership tends to degrade over time. In production environments, this type of governance model is sustained through
three aligned responsibilities:

- **FinOps** manages the detection pipeline and tracks the key metrics including allocation coverage, MTTR, and repeat violation rate while
monitoring progression through governance maturity stages.
- **Engineering teams** are responsible for remediation within defined SLAs, with alert attribution tied to the originating IAM principal.
- **Finance** consumes the resulting allocation data for chargeback, forecasting, and commitment planning.

Operating cadence in mature environments includes monthly Finance reconciliation, weekly KPI review, and periodic compliance checkpoints to
assess readiness for stronger enforcement controls.

---

## Key Technologies

| Layer | Technology |
|---|---|
| Billing data | AWS Cost and Usage Report (CUR) |
| Query engine | Amazon Athena (Presto/SQL) |
| Forensics | AWS CloudTrail |
| Event capture | Amazon EventBridge |
| Tag validation | AWS Lambda (Python 3.12+) |
| Governance | AWS Organizations Tag Policies, SCP |
| Alerting | Slack (Finance channel and resource owner DM) |
| Reporting | Anthropic Claude (AI-assisted formatting only; all findings produced by SQL and CloudTrail analysis) |

---

## Investigation Summary

| Phase | Objective | Tooling | Outcome |
|---|---|---|---|
| 1. Invoice Validation | Confirm billing accuracy | Cost Explorer, CUR | Billing error ruled out |
| 2. Resource Isolation | Identify unallocated spend | Athena SQL (CUR) | EC2 and S3 gaps isolated |
| 3. CloudTrail Forensics | Identify creation source, method, and identity chain | CloudTrail event history | Console launch confirmed, tags
absent at creation, identity governance bypass confirmed |
| 4. Scope Expansion | Check all services for systemic exposure | Athena SQL (CUR) | Systemic gap confirmed across EC2 and S3 |
| 5. Remediation | Prevent recurrence | EventBridge, Lambda | Near-real-time detection live |
| 6. Finance Reporting | Deliver executive output | Structured report with AI formatting | Finance-ready report produced |

---

## Investigation

### Context and Problem Statement

The investigation began at monthly reconciliation. Total AWS spend was $315. Spend visible with required tags was $262. That left $53 
unallocated — roughly 17% of the total. The invoice was accurate. The failure was in cost attribution, not cost generation.

---

### Phase 1: Invoice Validation

**Objective:** Confirm AWS billed correctly before investigating allocation.

The first assumption I tested was a billing error. If the invoice was wrong, the allocation gap was a Finance reporting artifact and not an
infrastructure problem. I validated service totals in Cost Explorer, reconciled them against CUR via Athena, and confirmed the invoice
totals matched exactly.

**Conclusion:** Billing was correct. That ruled out the simplest explanation. The issue existed downstream in the attribution layer, which
meant the investigation had to move to the resource level.

![Invoice validation confirming $315 total spend](screenshots/01%20-%20invoice-total-315.png)

> **Query reproduction note:** The screenshot above uses a `VALUES` clause to simulate Athena output for portfolio presentation. The
production-equivalent query aggregates `SUM(line_item_unblended_cost)` by `line_item_product_code` from live CUR Parquet files in S3 and
reconciles against the AWS invoice PDF total.

---

### Phase 2: Resource Isolation (CUR Analysis)

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

I used CUR via Athena rather than Cost Explorer because it enables row-level tag inspection. Cost Explorer aggregates data and cannot
surface which specific resources have NULL allocation tags or how much each one costs. Unblended cost was used throughout to align with
invoice totals, since Finance treats unblended cost as the authoritative source of truth.

The `Environment`, `Project`, and `Owner` tags were activated as Cost Allocation Tags in the AWS Billing and Cost Management console before
this investigation started. This activation step is a prerequisite that is easy to miss: without it, tag columns are absent from the CUR
schema entirely and the NULL-filtering queries in Phase 2 cannot run. Tag activation is not a technical control, but its absence would have
made this investigation impossible before it started.

#### The Turning Point

The initial goal was to identify *where* allocation was failing, not *why*. I grouped CUR line items by resource ID and inspected all three
required tag columns simultaneously:

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

What stood out was not the cost itself. It was the pattern of absence. For resource `i-0c9cfb67280fe44ee`, every line item returned NULL
across all three tag columns simultaneously. Not one missing tag. Not two. All three. That pattern ruled out accidental omission. A
developer who forgot one tag would not systematically omit all three across every billing line item for the same resource.

At that point the problem shifted. This was no longer *"why is Finance missing cost?"* It became *"who created a resource that was never
tagged at all?"* That pivot moved the investigation from Athena to CloudTrail.

**SQL: Resource Verification Query**

```sql
SELECT
    line_item_resource_id,
    SUM(line_item_unblended_cost)       AS total_cost,
    resource_tags_user_environment,
    resource_tags_user_project,
    resource_tags_user_owner
FROM cur_database.aws_cur
WHERE line_item_resource_id = 'i-0c9cfb67280fe44ee'
  AND line_item_line_item_type = 'Usage'
GROUP BY 1, 3, 4, 5;
```

**Sample query output:**

| Instance Name | Instance ID | Instance Type | Monthly Cost | Environment | Project | Owner |
|---|---|---|---|---|---|---|
| PROD-WEB-SERVER-01 | i-0c9cfb67280fe44ee | t3.large | 40.00 | NULL | NULL | NULL |

![Athena query results showing PROD-WEB-SERVER-01 with instance ID i-0c9cfb67280fe44ee, monthly cost $40.00, and NULL values across all 
three required tags — Environment, Project, and Owner](screenshots/02%20-%20cur-resource-isolation-missing-allocation-tags.png)

> **Result panel note:** This screenshot shows the isolated result output from the resource verification query. All three required
allocation tags return NULL simultaneously — confirming the resource was never tagged, not partially tagged. The simultaneous absence across
all three columns is the signal that rules out accidental omission and redirects the investigation to CloudTrail.

![Athena CUR resource verification query showing PROD-WEB-SERVER-01 with NULL allocation tags including the VALUES query code]
(screenshots/03%20-%20cloudtrail-forensics-runinstances-missing-tags-cli-launch.png)

> **Query reproduction note:** This screenshot shows the `VALUES` clause used to simulate the Athena result set for portfolio presentation,
alongside the result output. In a production investigation, this query executes against partitioned Parquet CUR files in S3. The production-
equivalent query, which aggregates live `line_item_unblended_cost` by resource ID and tag columns, is provided below.

> **Instance type note:** CloudTrail records `t3.micro` as the creation-time instance type while the CUR billing record reflects `t3.large`.
CloudTrail is the source of truth for what was submitted at provisioning. CUR reflects the current billing state. Any discrepancy between
the two triggers a secondary validation step.

#### Production-Equivalent Unallocated Spend Query

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

> `DATE_TRUNC` and the `BETWEEN DATE '...'` syntax are Athena/Presto-specific and are not ANSI SQL portable. The
`line_item_usage_account_id` column lets you trace attribution gaps back to the owning account across 50–200 accounts in a consolidated
Organizations CUR dataset.

---

### Phase 3: CloudTrail Forensic Investigation

**Objective:** Determine how and why the resource was created without tags.

With the untagged resource identified in Phase 2 (instance ID `i-0c9cfb67280fe44ee`), I moved to CloudTrail to determine the IAM principal,
launch method, and whether tags were absent at creation or removed afterward.

CloudTrail was the correct tool for this phase because it captures request intent, including the `tagSpecificationSet` submitted with the 
API call, which cannot be reconstructed from post-state resource evaluation. AWS Config records what a resource looks like after creation; 
CloudTrail records what was asked for at the moment of creation. For a tagging investigation, that distinction is the difference between
knowing tags are missing and knowing they were never submitted.

#### Step 1: CloudTrail Event Record (Full JSON)

I filtered CloudTrail Event history to `RunInstances` and opened the full JSON event record for `i-0c9cfb67280fe44ee`. This is the primary
forensic document. It confirms the IAM principal, the identity governance context, the launch timestamp, the user agent (console vs. CLI vs.
SDK), and the complete tag specification submitted with the API call.

![CloudTrail RunInstances event record showing Root IAM principal, timestamp 2026-01-22T19:00:13Z, awsRegion us-east-2, instanceType 
t3.micro, and Chrome browser UserAgent confirming console launch](screenshots/03a%20-%20cloudtrail-event-history-runinstances.png)

#### Step 2: Tag Specification Evidence

![CloudTrail RunInstances tagSpecificationSet showing the Name tag only for PROD-WEB-SERVER-01 with no Environment, Project, or Owner tags 
present](screenshots/03b%20-%20cloudtrail-runinstances-json-cli-useragent.png)

> **Pipeline evolution note:** The EventBridge screenshot reflects the EC2 and initial S3 detection phase (`RunInstances` and
`CreateBucket`). S3 validation was subsequently refined to trigger on `PutBucketTagging` rather than `CreateBucket` after identifying the
temporal compliance window created by S3's asynchronous tagging model. The production event pattern includes all three event names:
`RunInstances`, `CreateBucket`, and `PutBucketTagging`.

The `tagSpecificationSet` in the CloudTrail JSON event record contained only the `Name` tag (`PROD-WEB-SERVER-01`). `Environment`,
`Project`, and `Owner` were not submitted with the API call. This confirms the tags were absent at creation — they were never included and
were not applied then later removed.

#### Step 3: Data Integrity Verification (Creation-Time State vs. Billing State)

In a production investigation, the instance type recorded in the CloudTrail `requestParameters` must reconcile with the
`line_item_usage_type` field in CUR. The CloudTrail event for this resource records `t3.micro` as the requested instance type, while CUR 
attributes a `t3.large` cost profile to the same resource ID.

This discrepancy triggers a secondary validation protocol:

1. **Check for post-launch modification:** Query CloudTrail for `ModifyInstanceAttribute` events on the same resource ID to determine if the
instance type was changed after launch.
2. **Check for data pipeline lag:** Verify that the CUR processing interval and the CloudTrail event timestamp align within the same billing
hour.
3. **Treat CloudTrail as creation-time source of truth:** The `tagSpecificationSet` and `instanceType` in the `RunInstances` event represent
the intent at provisioning time. CUR represents the billed reality. For a tagging investigation, creation-time intent is the relevant signal
because it proves whether tags were submitted at provisioning.

In this lab environment, the values across screenshots were captured at different stages of the simulation and are not intended to align 
perfectly. The forensic technique — reconciling creation-time API intent against billing state — is the transferable production skill.

#### Step 4: Session Attribution: The Identity Governance Finding

The forensic chain does not stop at the IAM principal type. The CloudTrail event reveals three additional facts about the identity context
that matter in an enterprise investigation:

```json
"userIdentity": {
  "type": "Root",
  "sessionContext": {
    "attributes": {
      "creationDate": "2026-01-22T13:17:47Z",
      "mfaAuthenticated": "true"
    }
  }
}
```

The CloudTrail session record contains no `sessionIssuer`, no role assumption chain, and no federation context. Each of those gaps is 
independent and together they confirm there was no attributable identity at any layer.

To confirm no role assumption preceded the provisioning action, CloudTrail event history was queried for `AssumeRole` events scoped to the
Root principal ARN, bounded to the 30-minute window prior to the `RunInstances` timestamp (`2026-01-22T18:30:00Z` to 
`2026-01-22T19:00:13Z`), and filtered to the `us-east-2` region where the instance was created. No matching `AssumeRole` events were 
returned.

This is not a routine compliance gap. Root usage removes enforceable organisational attribution and ownership mapping, even though basic
forensic signals like source IP, MFA status, and session timing remain visible in CloudTrail. What is absent is the role assumption chain
that maps a provisioning action to a team, a cost centre, and an approval workflow. Without that chain, post-incident ownership cannot be
assigned and targeted enforcement cannot be designed around known principals. No `sessionIssuer` means no team ownership is resolvable in
post-incident analysis. No `AssumeRole` means no control boundary was crossed, because no control boundary existed. In a security or audit
review, this finding would be logged separately from the tagging failure as a distinct identity governance incident. Both existed within the
same ungoverned provisioning surface.

> **Note on the IAM principal used in this lab:** A Root session with MFA was used to simulate a worst-case scenario where no enforced
identity boundary exists at the provisioning layer. In a real enterprise environment, root is locked via SCP. The production equivalents are
a developer IAM role with broad permissions launching via console, a federated SSO user outside an IaC pipeline, or a break-glass role used
outside a change window. The forensic technique of UserAgent string analysis and `tagSpecificationSet` inspection applies to all of those
cases in exactly the same way.

#### Forensic Evidence Chain Summary

| Evidence | Finding |
|---|---|
| CUR Phase 2 | Athena query confirming untagged resource, instance ID, and $40 monthly cost |
| CloudTrail JSON (top half) | IAM principal type Root, no sessionIssuer or federation context, browser-based invocation confirmed via
Chrome UserAgent string, awsRegion us-east-2, creation timestamp 2026-01-22T19:00:13Z |
| CloudTrail JSON (tag section) | `tagSpecificationSet` contained `Name` tag only; Environment, Project, and Owner were absent at creation |
| Session attribution | `AssumeRole` query scoped to principal ARN, bounded to 30-minute pre-incident window, filtered to us-east-2: zero 
results. No role assumption chain, no federation context. No enforced identity boundary existed at the provisioning layer. |

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

> **Note:** The `instanceType` recorded here (`t3.micro`) reflects the API request payload at creation time. In a live environment, this
would be reconciled against CUR `line_item_usage_type` via the Data Integrity Verification protocol described above. The
`tagSpecificationSet` evidence — showing only the `Name` tag — is the primary forensic finding and is unaffected by instance type
reconciliation.

---

### Phase 4: Expanding Scope to S3

**Objective:** Determine whether the issue was isolated or systemic.

| Service | Billed | Allocated | Gap |
|---|---|---|---|
| EC2 | $220 | $182 | ~$38 |
| RDS | $60 | $60 | $0 (fully compliant) |
| S3 | $35 | $20 | $15 |

EC2 had the largest gap, but S3 confirmed the issue was systemic. The two S3 buckets are shown with partial tags to illustrate a key
compliance principle: partial tagging produces the same Finance-invisible result as no tagging. Both buckets are missing the required 
`Environment` tag. Even with `Project` and `Owner` present, that one missing tag meant $15 per month in storage spend that Finance could not
allocate to any cost centre.

| Resource ID | Monthly Cost | Environment | Project | Owner |
|---|---|---|---|---|
| bucket-logs-arch | $7.00 | NULL | data-pipeline | team-ops |
| bucket-temp-data | $8.00 | NULL | web-app | team-engineering |
| **TOTAL UNALLOCATED** | **$15.00** | | | |

![S3 tag compliance audit via CUR showing bucket-logs-arch and bucket-temp-data with NULL Environment tags](screenshots/04%20-%20cur-s3-tag-
compliance-audit.png)

> **Query reproduction note:** This screenshot uses a `VALUES` clause to simulate Athena output for the two non-compliant S3 buckets. The
production query filters live CUR data by `line_item_product_code = 'AmazonS3'` and NULL `resource_tags_user_environment`, grouped by bucket
resource ID.

> **S3 tagging behaviour and temporal compliance window:** The S3 `CreateBucket` API does not accept tags inline the way EC2 does. Tags on
S3 buckets are applied via a separate `PutBucketTagging` call after the bucket exists. This creates a temporal compliance window — a period
between bucket creation and tag application during which the resource exists in a technically non-compliant state even if it will be
correctly tagged within seconds. Point-in-time compliance snapshots will always show some S3 buckets as non-compliant even in well-governed
environments. The production control design accounts for this: dual-event EventBridge monitoring (`CreateBucket` and `PutBucketTagging`) as
the primary detection path, with a 15-minute delayed validation Lambda as a safety net for buckets where `PutBucketTagging` is never called.
It is worth noting that this downstream detection approach is a compensating control, not a primary one. Many organisations address S3
tagging upstream through Service Catalog products or IaC modules that enforce tags before bucket creation is permitted, which eliminates the
compliance window entirely. One additional limit: S3 cost allocation operates at the bucket level. Object-level operations are not directly
taggable, which can introduce granularity limits in fine-grained unit economics models where per-object or per-prefix attribution is
required.

---

## Root Cause and Remediation

### Root Cause Analysis

| | |
|---|---|
| **Primary failure** | Absence of preventive tagging enforcement at resource creation |
| **Contributing factor 1** | Lack of attributable identity chain |
| **Contributing factor 2** | Tag activation dependency with no verification process |
| **Result** | Unattributed spend and no enforceable ownership model |

**Primary failure: no preventive tagging control at resource creation time.**

No SCP, IAM condition key, IaC guardrail, or curated provisioning interface enforced required tags at the point of resource creation. This 
is the direct cause of the allocation gap: resources reached the billing layer without the tags Finance depends on for attribution. The 
failure was in system design, not user behaviour.

**Contributing factor 1 — Identity architecture gap.**

The CloudTrail session record contains no `sessionIssuer`, no role assumption chain, and no federation context. This did not cause the 
tagging failure. It removed attribution, which prevented ownership assignment and made targeted enforcement design non-viable. Without an
attributable identity chain, post-incident ownership cannot be established, repeat violations cannot be tied to a responsible principal, and 
scoped enforcement cannot be designed.

Tagging and identity are different controls that serve different purposes and neither one works without the other in a real FinOps model. 
Tagging makes spend allocatable. Identity makes it attributable. When both are missing you can see the cost but you cannot assign it to 
anyone and you have no lever for enforcement.

**Contributing factor 2 — Observability dependency risk.**

The `Environment`, `Project`, and `Owner` tags were activated as Cost Allocation Tags before this investigation ran. Without that 
activation, the CUR schema would not contain tag columns at all, and the NULL-filtering queries in Phase 2 would return no results. Tag 
activation is not technically a control, but its absence silently disables an entire class of allocation investigation.

**Design philosophy: visibility over false accuracy.** The remediation approach deliberately prioritized restoring correct financial 
visibility over achieving superficial compliance. Automated tag-back remediation was evaluated and rejected. Default tags can produce 
chargeback reports that assign cost incorrectly, distort showback data, and create misleading financial baselines. The safer approach in 
this scenario was to surface the allocation gap accurately, alert the responsible principal, and allow correct tags to be applied by the 
actual resource owner. The full reasoning is in Appendix C.

---

### Control Hierarchy: Where Detection Fits

This investigation addresses the detection stage of a broader governance maturity model (Detect → Baseline → Stabilize → Enforce) and 
provides a foundation for progressing toward enforcement as compliance stabilizes. Detection precedes broad enforcement, but this is not a 
universal rule. In many real environments, baseline SCPs already exist from day one — for example denying root usage or requiring tags for 
specific services. Scoped enforcement can and should coexist with detection early, applied selectively where the environment is known and 
controlled.

**1. IaC enforcement — enforces at design time.**
Tags are required in the Terraform module itself. Resources without required tags fail the plan before anything reaches AWS. Strongest first 
control; catches omissions at the point of review.

**2. Curated provisioning interfaces — enforce at the provisioning entry point.**
Internal developer platforms (such as Backstage), ITSM-mediated provisioning layers (such as ServiceNow Cloud Provisioning), and Service 
Catalog products constrain how resources are created before any API call is made. Engineers provision through pre-approved interfaces that
enforce tagging by construction. This layer sits between IaC and runtime enforcement and is how many larger organisations handle the gap 
between pipeline-governed and console-accessible provisioning.

**3. SCP and IAM — enforce at request time.**
SCPs block at the Organizations level. IAM condition keys like `aws:RequestTag/Environment` and `aws:TagKeys` block at the role level and 
are viable early when scoped narrowly to specific developer roles with predictable provisioning patterns. The constraint is not IAM itself —
it is scope and exception mapping. Blanket org-wide IAM enforcement without a baseline breaks AWS-managed service roles (EMR, ECS, Auto 
Scaling) that do not pass required tags consistently.

A practical example of a scoped deny policy enforcing both tag presence and value constraints:

```json
{
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "*",
  "Condition": {
    "Null": {
      "aws:RequestTag/Environment": "false",
      "aws:RequestTag/Project": "false",
      "aws:RequestTag/Owner": "false"
    },
    "StringEquals": {
      "aws:RequestTag/Environment": ["prod", "dev", "staging"]
    }
  }
}
```

The Null condition with "false" enforces tag presence (deny if the key is not null — i.e., the tag must exist). The StringEquals condition 
enforces value constraints. Together they form an implicit AND: the request must include all three tags, and Environment must match the 
allowed value set.

**4. Tag Policies — standardize schema only.**
AWS Organizations Tag Policies standardize tag key names across accounts. They do not block non-compliant resources. Value is schema 
consistency (preventing `environment` and `Environment` from producing separate NULL patterns in CUR) and reporting visibility. Not an 
enforcement control.

**5. EventBridge + Lambda — detect out-of-band provisioning.**
Even in environments with strong IaC and curated provisioning interfaces, some activity happens outside governed paths: break-glass access,
SDK scripts, direct console use for incident response. This is the correct control for that class of activity.

**6. Selective SCP and IAM enforcement — after exceptions are mapped.**
SCP guardrails applied org-wide after the compliance baseline is understood and per-account exceptions are documented. Applied too early, 
before legacy workloads and AWS-managed service roles are mapped out, they generate incidents that erode engineering trust and make every 
subsequent governance initiative harder.

---

### Remediation and Controls

**Immediate correction:** Applied the missing tags so Finance could close the month accurately. No restatements required.

**Permanent prevention (progressive enforcement model):**

1. **Tag Policies** standardize required tag keys across all accounts without blocking deployments
2. **EventBridge + Lambda** detect violations within minutes of resource creation and alert Finance and the resource owner simultaneously
3. **SCP guardrails** enforce compliance for greenfield accounts once the organization stabilizes above 95% coverage, grandfathering
existing workflows to avoid pipeline disruption

> **A control that is ignored is functionally nonexistent.**

The control is designed to minimize false positives to preserve alert credibility and maintain response adherence. EC2 tags are validated 
against the actual `tagSpecificationSet` submitted with the API call. S3 validation is deferred to `PutBucketTagging` rather than 
`CreateBucket`, eliminating false-positive alerts on every new bucket regardless of its eventual tag state.

---

### Organizational Rollout Note

In a production deployment, a 24-hour remediation SLA would likely face resistance from engineering teams accustomed to longer security-
ticket windows. The business case for urgency rests on accumulated unattributed spend: at the observed gap rate, a single team's untagged 
resources compound to material budget variance within a quarter. A practical rollout sequence is a 48-hour grace period for teams actively 
updating IaC modules to enforce tags at plan time, with the requirement that existing violations clear within 24 hours. Once pipeline 
compliance reaches 100%, the grace period retires and repeat violation tracking begins against the IAM principal ARN. This framework 
preserves alert credibility while giving teams a concrete path to compliance without immediate enforcement.

---

### Step 1: EventBridge Rule

The EventBridge rule `finops-tag-compliance-monitor` filters for `RunInstances` (EC2), `CreateBucket` (S3), and `PutBucketTagging` (S3 tag 
validation) API calls via CloudTrail and targets the `finops-tag-validator` Lambda function.

**EventBridge event pattern:**

```json
{
  "source": ["aws.ec2", "aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventName": ["RunInstances", "CreateBucket", "PutBucketTagging"]
  }
}
```

`PutBucketTagging` is included because S3 does not accept inline tags at bucket creation time. Tag validation must happen when tags are 
actually applied, not at bucket creation, to avoid false-positive alerts on every new bucket.

![EventBridge Rule finops-tag-compliance-monitor showing source aws.ec2 and aws.s3, RunInstances and CreateBucket event names, and finops-
tag-validator Lambda target in us-east-1](screenshots/05a-eventbridge-rule.png)

> **Pipeline evolution note:** The EventBridge rule screenshot reflects the EC2 and initial S3 detection phase (`RunInstances` and
> `CreateBucket`). S3 validation was subsequently refined to trigger on `PutBucketTagging` rather than `CreateBucket` after identifying the
temporal compliance window created by S3's asynchronous tagging model. The production event pattern includes all three event names:
`RunInstances`, `CreateBucket`, and `PutBucketTagging`.

> **Production deployment pattern:** EventBridge default event buses are regional. A production deployment requires the rule in every active
region. Two standard patterns handle this: per-region IaC deployment (Terraform or CloudFormation StackSets) for organizations with 3–5
active regions; central event bus pattern forwarding regional CloudTrail events to a dedicated billing or security account for organizations
managing 10 or more accounts.

---

### Step 2: Lambda Validation Logic

Lambda uses branching logic to handle EC2 and S3 differently, because the two services expose tag data at different points in their 
lifecycle.

> **Note on `boto3`:** In production, `boto3` retrieves the Slack webhook URL from AWS Secrets Manager. In this lab, alerts route to
CloudWatch Logs via `print()`. The webhook URL must never be hardcoded.

```python
import json

REQUIRED_TAGS = ['Environment', 'Project', 'Owner']

def lambda_handler(event, context):
    detail     = event['detail']
    event_name = detail.get('eventName')
    params     = detail.get('requestParameters', {})

    if event_name == 'RunInstances':
        # EC2: tags may be submitted inline at creation via tagSpecificationSet
        tag_specs = params.get('tagSpecificationSet', {}).get('items', [])
        tag_list  = []
        for item in tag_specs:
            tag_list += [t['key'] for t in item.get('tags', [])]
        evaluate_and_alert(event, detail, event_name, tag_list)

    elif event_name == 'CreateBucket':
        # S3: CreateBucket never carries inline tags.
        # Deferring to PutBucketTagging avoids false-positive alerts on every
        # new bucket regardless of whether it will be tagged correctly.
        pass

    elif event_name == 'PutBucketTagging':
        # S3: correct validation point for S3 tag compliance
        tag_set  = params.get('Tagging', {}).get('TagSet', {}).get('Tag', [])
        tag_list = [t['Key'] for t in tag_set] if isinstance(tag_set, list) else []
        evaluate_and_alert(event, detail, event_name, tag_list)

    return {'statusCode': 200}


def evaluate_and_alert(event, detail, event_name, tag_list):
    missing = [tag for tag in REQUIRED_TAGS if tag not in tag_list]
    if missing:
        message = {
            'alert':         'FINOPS TAG COMPLIANCE VIOLATION',
            'event':         event_name,
            'missing_tags':  missing,
            'iam_principal': detail.get('userIdentity', {}).get('arn'),
            'account_id':    event.get('account'),
            'region':        event.get('region'),
        }
        print(f"ALERT: {json.dumps(message, indent=2)}")
    return missing
```

`REQUIRED_TAGS` is the single source of truth. Adding or removing a required tag means changing one line.

> **Why the branching matters:** A single-path version using `tagSpecificationSet` for all events would flag every new S3 bucket as non-
compliant. That key only exists in EC2 `RunInstances` payloads. The branched version avoids that entirely.

> **Remaining S3 gap:** A bucket created but never receiving `PutBucketTagging` will never be evaluated by the event-driven path. The 15-
minute delayed validation Lambda addresses this: it runs on a schedule, calls `ListBuckets` to enumerate recently created buckets and
`GetBucketTagging` on each to retrieve current tag state, and re-alerts on any bucket that has passed the compliance window without the
required tags. An AWS Config snapshot of S3 resource configuration is a viable alternative data source for organisations with Config already
enabled, avoiding the per-API-call cost of `GetBucketTagging` at scale.

![Lambda function finops-tag-validator showing REQUIRED_TAGS list, event parsing logic for RunInstances and CreateBucket, missing tag 
detection, and alert payload construction](screenshots/05b%20-%20lambda-finops-tag-validator-code.png)

> **Control refinement note:** The Lambda screenshot shows the v1 implementation, which attempted to read tags from the `CreateBucket` event
using lowercase key names (`tagging`, `tagSet`, `items`). This approach produces false positives because S3 `CreateBucket` does not carry
inline tags in the CloudTrail payload, and CloudTrail uses PascalCase (`Tagging`, `TagSet`, `Tag`) for S3 API parameters. The production-
hardened code above defers S3 validation to `PutBucketTagging` and uses schema-correct PascalCase keys. This iteration demonstrates a core
FinOps principle: a control that fires on expected behavior loses credibility and gets ignored, which is functionally equivalent to having
no control.

![Lambda runtime settings showing Python 3.14, handler lambda_function.lambda_handler, x86_64 architecture, and Lambda Deployed status 
confirmed](screenshots/05b-ii%20-%20lambda-settings.png)

> For high-volume burst scenarios, add SQS and DLQ between EventBridge and Lambda to handle concurrency spikes above the default 1,000
concurrent execution limit per account per region.

**Alert structure and ownership resolution.** Each alert payload carries the IAM principal ARN, account ID, region, event name, and list of
missing tags. In production, the IAM principal ARN is the ownership resolution mechanism: it maps to the engineer or service identity 
responsible for the provisioning action, which is then cross-referenced against a team directory or AWS SSO assignment to route the Slack DM
to the correct owner. The Finance channel alert fires simultaneously and is not dependent on ownership resolution succeeding.

**Escalation path.** If no remediation action is taken within the 24-hour SLA window, the owning team's manager is notified via a second-
stage alert. Repeat violations by the same principal within 30 days trigger a governance review rather than another alert, to avoid 
desensitisation.

---

### Step 3: End-to-End Test Execution

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
Duration: 2.54 ms  |  Billed Duration: 259 ms  |  Init Duration: 255.90 ms
```

**Result:** `statusCode: 200`, `missing_tags: ['Environment', 'Project', 'Owner']`

![Lambda test result showing statusCode 200, missing_tags for Environment, Project, and Owner, and the FINOPS TAG COMPLIANCE VIOLATION alert 
in the log output](screenshots/05c-lambda-test-result.png)

> **Test event note:** The test event simulates a standard IAM user scenario (`developer-01`) to validate the Lambda execution path and
alert payload structure. The actual Phase 3 forensic event involved a Root principal with no `sessionIssuer` chain. A production test suite
would include both standard-user and break-glass/Root scenarios to ensure the alert payload correctly handles disparate identity contexts.

---

## Enterprise Context and Scale

### FinOps Lifecycle Alignment

| Phase | Activity in This Investigation |
|---|---|
| **Inform** | CUR and Athena cost visibility, allocation coverage metric, invoice validation baseline |
| **Optimize** | $53 per month in unattributed spend isolated across EC2 and S3; rightsizing path unlocked post-attribution |
| **Operate** | Event-driven tagging compliance control, real-time Slack alerting, progressive enforcement model |

---

### Blast Radius: What This Failure Looks Like at Scale

The $53 gap in this lab is a methodology demonstration. The financial risk it represents is not.

In an enterprise environment running $10M per month across 150 accounts and 8 regions, a 5% allocation failure rate produces $500,000 per 
month in spend that Finance cannot see, attribute, or charge back. That is $6M annualized. At that scale, the failure is not a reporting
gap. It is a strategic planning failure: budget models are wrong, team cost accountability is broken, Savings Plans commitments are sized
against incomplete data, and anomaly detection produces false signals because the baseline is corrupted.

The same CUR query structure, CloudTrail forensic chain, and EventBridge governance pipeline demonstrated here would surface and prevent 
that gap in exactly the same way.

---

### Multi-Account Architecture

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

**Four production considerations a sandbox does not surface:**

- **Lambda concurrency:** Default limit is 1,000 per account per region. Add SQS and DLQ for burst hardening in environments with high-
volume provisioning events.
- **EventBridge rule limits:** 300 rules per event bus per account. Use dedicated buses at scale.
- **CUR partition management:** 100+ accounts at $2M–$10M per month requires partitioning by account ID, service, and billing month to avoid
expensive full-table Athena scans.
- **Cost of the control itself:** CloudTrail data events carry a per-event charge and should be enabled selectively on high-value S3 buckets
rather than account-wide. At 10,000 provisioning events per month the total pipeline cost is well under $5 — trivial relative to the
allocation visibility it provides, but modelling it is the point in a FinOps context.

**Multi-account CUR query structure.** In a consolidated Organizations CUR, the `line_item_usage_account_id` column is the primary partition
key for cross-account attribution. A rollup query that identifies the top unallocated spend sources across all linked accounts:

```sql
SELECT
    line_item_usage_account_id,
    line_item_product_code,
    COUNT(DISTINCT line_item_resource_id)    AS untagged_resource_count,
    SUM(line_item_unblended_cost)            AS unallocated_spend,
    SUM(SUM(line_item_unblended_cost)) OVER () AS total_spend,
    ROUND(
        100.0 * SUM(line_item_unblended_cost) /
        SUM(SUM(line_item_unblended_cost)) OVER (),
        2
    )                                        AS pct_of_total
FROM cur_database.consolidated_cur
WHERE
    line_item_line_item_type = 'Usage'
    AND line_item_usage_start_date
        BETWEEN DATE '2026-01-01' AND DATE '2026-01-31'
    AND (
        resource_tags_user_environment IS NULL
        OR resource_tags_user_project  IS NULL
        OR resource_tags_user_owner    IS NULL
    )
GROUP BY 1, 2
ORDER BY unallocated_spend DESC;
```

This query partitions by payer account and product code. In a 50-account environment with multiple OU boundaries, the results drive account-
level enforcement sequencing: accounts with the highest unallocated spend receive SCP guardrails first. Accounts below a materiality 
threshold are addressed in a later enforcement wave to avoid disrupting low-spend development environments.

**Tag schema normalization across accounts.** Before any compliance KPI is meaningful, tag key inconsistencies across accounts must be
resolved. A common pattern in organisations without a centralised tag governance process is value drift: some accounts use `env`, others use 
`environment`, others use `Environment`. All three resolve to different CUR columns and produce separate NULL patterns in the allocation
query. The fix is a tag key normalisation step enforced through Tag Policies at the OU level and verified against CUR column presence before
the compliance baseline is published.

**Organisational rollout risk.** Legacy resources that predate the tagging standard cannot always be retagged without service disruption. 
Shared infrastructure — VPCs, transit gateways, NAT gateways — is used by multiple teams and has no single owner; any chargeback model for 
those resources requires an agreed allocation split before tagging can be meaningful. SCP and IAM enforcement rollout requires per-account
or per-OU exception mapping that accounts for both categories before any blocking control is applied.

---

### Optimization Opportunity (Sequential to Attribution, Not Parallel)

With attribution restored, the data now supports a clear optimization path.

**Rightsizing:** Pull 2-week CloudWatch CPU and memory utilization and downsize one tier for any resource consistently below 20% 
utilization. Owner accountability is only possible because attribution is now complete.

**Savings Plans:** Run Cost Explorer coverage report filtered by `Environment` and `Project` to identify commitment candidates. Before full
tag coverage, any break-even modeling produced inaccurate recommendations.

> Attribution must precede optimization. Rightsizing unattributed spend produces projections that land on the wrong teams.

---

### What I Would Do Next

**Days 1–30: Harden regional coverage**
- Deploy EventBridge rule and Lambda to all active regions via IaC (Terraform or CloudFormation StackSets)
- Deploy the 15-minute delayed validation Lambda as a safety net for buckets created but never tagged
- Implement SQS and DLQ between EventBridge and Lambda for burst hardening
- Stand up pipeline monitoring per Appendix D: regional rule audit, DLQ alarm, CloudTrail disruption alert, and monthly synthetic Slack test

**Days 31–60: Establish compliance baseline**
- Centralize CUR across all accounts via AWS Organizations
- Stand up a weekly tag compliance KPI (tagged spend / total spend) as a first-class FinOps metric
- Begin tracking MTTR and repeat violation rate per team to validate that the feedback loop is changing behaviour
- Deploy week-over-week Athena anomaly detection to alert Finance before monthly close

**Days 61–90: Enforce and optimize**
- Introduce SCP guardrails for greenfield accounts once compliance exceeds 95%
- Run EC2 rightsizing analysis against 2-week CloudWatch utilization windows
- Deliver showback reporting to Finance and move to full chargeback once coverage is stable above 95%
- Run Savings Plans coverage analysis filtered by `Environment` and `Project` tags

---

## Lessons Learned

- **Financial data needs to be treated as operational telemetry** because without real-time visibility, allocation failures persist until
monthly close.
- **Root cause at enterprise scale is rarely one failure.** The provisioning gap, the tag activation dependency, and the identity governance
gap are three separate problems that happened to compound.
- **Tagging governance must exist at the system level.** Human process alone cannot maintain reliable allocation coverage under deadline
pressure.
- **Detection is more effective than enforcement early in governance maturity.** Immediate feedback loops change behaviour faster than
blocking deployments.
- **Tag Policies standardize schema. They do not enforce compliance.** Understanding what each control tier actually does prevents mistaking
visibility for protection.
- **Attribution is the prerequisite for optimization.** Optimization applied to unattributed spend produces inaccurate projections and
misaligned accountability.
- **A governance pipeline that cannot detect its own failure modes is not a governance pipeline.** It is a false confidence generator.

---

## Artifact Evolution and Production Hardening

This portfolio documents a FinOps investigation across **multiple maturity stages**: initial detection, false-positive elimination, and 
progressive enforcement. The following table reconciles the lab artifacts presented in screenshots with the production-hardened architecture
described in the text.

| Artifact | Lab State (Screenshot) | Production-Hardened State (Text) | Reason for Refinement |
|---|---|---|---|
| **CUR result panel (screenshot 02)** | Clean result output showing PROD-WEB-SERVER-01 with all three tags NULL | Same result produced by 
live query against partitioned Parquet CUR | Isolates the forensic signal — simultaneous NULL across all three columns — from the query 
mechanics shown in screenshot 03 |
| **CUR resource verification (screenshot 03)** | Hardcoded `VALUES` clause simulating Athena output, shown alongside result | Live query 
against partitioned Parquet CUR in S3 | Data sensitivity; live CUR contains account IDs, principal ARNs, and actual spend figures subject to
least-privilege access controls |
| **Instance type attribution** | `t3.micro` in CloudTrail vs. `t3.large` in CUR | Reconciliation protocol: CloudTrail = creation-time 
source of truth; CUR = billing source of truth | Creation-time intent determines whether tags were submitted at provisioning; billing state
determines cost impact. Discrepancies trigger secondary validation. |
| **EventBridge rule events** | `RunInstances`, `CreateBucket` | `RunInstances`, `CreateBucket`, `PutBucketTagging` | S3 tags are applied 
asynchronously via `PutBucketTagging`, not inline at `CreateBucket`. Validating at `CreateBucket` creates false positives. |
| **Lambda S3 parsing** | `CreateBucket` trigger with lowercase keys (`tagging.tagSet.items`) | `PutBucketTagging` trigger with PascalCase
keys (`Tagging.TagSet.Tag`) | Matches CloudTrail schema for S3 API events; eliminates false-positive alerts on every new bucket |
| **Lambda test event** | Simulated IAM user (`developer-01`) | Test suite includes both standard-user and break-glass/Root scenarios | The
forensic incident involved Root usage. Controls must handle disparate identity contexts including missing `sessionIssuer` chains. |
| **Regional deployment** | Single-region rule (`us-east-1`) | Per-region IaC deployment via CloudFormation StackSets | EventBridge default
buses are regional. Provisioning in an unmonitored region creates a silent compliance gap. |
| **Lambda concurrency** | Direct invocation | SQS buffer with DLQ between EventBridge and Lambda | Provisioning bursts can exhaust the 
1,000 concurrent execution limit, causing throttling and silent event loss. |

### Why This Matters for Finance

A FinOps control that cannot detect its own failure modes is not a control — it is a false confidence generator. The progression from lab to
production requires explicit validation of each hardening step before Finance trusts the output. Alert credibility is a first-class design 
requirement: if on-call engineers stop responding because of false positives, the governance model has failed regardless of whether the 
Lambda executes correctly.

The iterative approach shown here — ship detection fast, measure signal-to-noise, refine, then enforce — is how FinOps governance actually
achieves adoption in Fortune 500 environments where Engineering trust is a prerequisite for compliance.

---

## Appendix A: AI-Assisted Investigation Reporting

AI was used only for formatting the final report, not for analysis. Every finding was produced by SQL analysis and CloudTrail forensics in
Phases 1–5.

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

**INTERNAL FINOPS INVESTIGATION REPORT — AWS Cost Allocation Gap | January 2026**

*Generated by Anthropic Claude using the prompt above. Every finding was validated against SQL and CloudTrail evidence in Phases 1–5 before
inclusion.*

**1. Executive Summary**

A forensic investigation identified a 17% cost allocation gap across EC2 and S3. The primary failure was the absence of preventive tagging
enforcement at resource creation time, with two contributing factors: a tag activation dependency with no verification process, and an 
identity architecture gap with no attributable provisioning chain. The gap was invisible to Finance because spend was correctly billed but
entirely unattributed. An event-driven detection pipeline has been implemented, restoring 100% allocation visibility and reducing end-to-end
MTTD to low-minute level under normal conditions, bounded by CloudTrail delivery and EventBridge propagation latency.

**2. Investigation Findings**

CUR analysis via Athena confirmed $53 in unallocated spend across two services. EC2 represented the largest gap at approximately $38, with
S3 contributing an additional $15. RDS showed full allocation compliance. Invoice validation confirmed all charges were accurate. The gap
existed entirely in the attribution layer.

**3. Root Cause Analysis**

CloudTrail forensics traced the EC2 gap to a `RunInstances` API call made directly via the AWS Management Console, confirmed by a Chrome
browser UserAgent string. The CloudTrail session record contains no `sessionIssuer`, no role assumption chain, and no federation context.
Each of those gaps is independent and together they confirm there was no attributable identity at any layer. Tags were never submitted with 
the API call, confirming a provisioning-time enforcement gap rather than post-deployment drift or tampering.

**4. Financial Impact**

| Metric | Value |
|---|---|
| Allocation gap | 17% of total spend |
| Monthly unallocated spend | $53 |
| Annualized if unresolved | $636 |
| Chargeback accuracy | Restored to 100% for affected cost centres |
| MTTD before remediation | ~30 days |
| MTTD after remediation | 1–3 minutes (bounded by CloudTrail delivery + EventBridge propagation) |
| Blast radius at $10M/month (5% failure rate) | $500K/month, $6M annualized |

**5. Governance Failure**

One primary control failure allowed the issue to occur, with two contributing factors that allowed it to persist. The primary failure was
the absence of preventive tagging enforcement at resource creation time. Contributing factors were the lack of an attributable identity
chain, which removed ownership accountability and made targeted enforcement non-viable, and a tag activation dependency with no verification
process, which would have silently disabled CUR-based investigation entirely if not already in place.

**6. Remediation Plan**

*Immediate:* Missing tags applied manually. Finance closed January with 100% allocation accuracy. No restatements required.

*Permanent:* Event-driven pipeline (CloudTrail → EventBridge → Lambda → Slack) deployed with EC2 and S3 branching logic. Engineers receive
real-time alerts without deployment friction. SCP guardrails scoped to greenfield accounts once compliance exceeds 95%. EventBridge rule to
be deployed per active region via IaC.

*Next steps:* Weekly CUR-based tag compliance baseline, MTTR and repeat violation rate tracking, anomaly detection for week-over-week cost
spikes, identity governance review to ensure provisioning actions are attributable to federated identities.

---

## Appendix B: Enterprise Context Reference

| Lab Component | Production Equivalent |
|---|---|
| Single AWS account | AWS Organizations with 50–200 accounts |
| Hardcoded CUR values | Live CUR partitioned by account, service, and month |
| t3.micro instance | Any instance type provisioned outside IaC pipelines |
| Root IAM principal (no sessionIssuer) | Developer role, federated SSO user, or break-glass role with traceable identity chain |
| us-east-1 EventBridge rule | Rules deployed per active region via StackSets |
| CloudWatch Logs alerts | Slack webhook via Secrets Manager |
| Single REQUIRED_TAGS list | Centralized tag policy synced across accounts |

---

## Appendix C: Control Selection Rationale

The decision to implement detection rather than enforcement reflects a deliberate assessment of governance maturity, not a technical 
limitation.

**What I rejected and why:**

**IAM condition key enforcement as a universal control.** IAM conditions like `aws:RequestTag/Environment` are valid controls when scoped to
specific developer roles with predictable provisioning patterns. AWS-managed services that create resources on your behalf (EMR cluster
nodes, ECS task ENIs, Auto Scaling launch instances) may not pass required tags through consistently. In a multi-account environment with
federated identity, blanket IAM condition enforcement creates more exception management overhead than it eliminates. The right time for IAM
conditions is after detection has mapped which roles are responsible for provisioning and which exceptions need to be scoped out.

**AWS Config.** Config was evaluated but EventBridge was selected because it captures full request context including `tagSpecificationSet`,
which Config cannot access at the API call layer. Config evaluates resulting resource state; it cannot inspect the intent of the original
API call. Config cost also scales with configuration item recording across all resources in all accounts, which compounds significantly in 
multi-account Organizations environments. EventBridge cost scales with API activity, which is typically lower and more predictable for this
use case.

**Immediate SCP enforcement.** An SCP that denies `ec2:RunInstances` or `s3:CreateBucket` without required tags would have closed the gap
completely on day one. In an environment with no existing compliance baseline, deploying a blocking SCP immediately breaks an unknown number
of CI/CD pipelines, automation scripts, and developer workflows. The right time for SCP enforcement is after the compliance baseline is 
understood and teams have had time to update their pipelines.

**Auto-remediation Lambda.** An automated tag-back Lambda that applies default tags on detection would close the reporting gap faster than
manual remediation. Automated tagging with default values corrupts the entire downstream FinOps operating model: chargeback reports assign
cost to the wrong team, showback reports give engineers a false picture of their own consumption, budget models include costs that do not
belong to the owning team, and anomaly detection fires on the wrong baseline. Every downstream FinOps process that depends on tag accuracy
produces a wrong answer. That is worse than unallocated spend because it creates false confidence rather than a visible gap.

**The intended sequence:**

1. **Now (detection):** EventBridge and Lambda give Finance near-real-time visibility and give engineers immediate feedback without blocking
their work.
2. **At 95% compliance (targeted prevention):** SCP guardrails applied to greenfield accounts first. IAM condition keys added to developer
roles with repeat violations.
3. **After full baseline (broad enforcement):** SCP progressively rolled out to existing accounts with a grace period. IaC modules updated
to enforce tagging by default.

---

## Appendix D: Pipeline Failure Modes and Monitoring

Every failure mode below produces silence. That silence looks identical to a clean compliance environment until the next monthly 
reconciliation reveals a gap the pipeline should have caught weeks earlier.

**False positive volume.** An alert system that fires on expected behaviour (every new S3 bucket, every Auto Scaling launch, every pipeline-
managed EC2 instance) trains engineers to ignore notifications. The branching logic and `PutBucketTagging` deferral are specifically 
designed to eliminate this class of noise. Alert credibility is a first-class design requirement: if on-call engineers stop responding, the 
governance model has failed regardless of whether the Lambda executes correctly. Monitor alert-to-action ratio alongside DLQ depth.

**EventBridge rule regional drift.** EventBridge default event buses are regional. If a new AWS region is activated without deploying the
EventBridge rule, all provisioning activity in that region is invisible to the compliance pipeline. Mitigation: weekly automated check 
comparing active regions against regions where the rule is confirmed deployed.

**Lambda concurrency exhaustion.** A provisioning burst can exhaust available concurrency and cause the validator function to throttle.
Events may fail delivery after retry attempts without a buffering layer, and no alert fires. Mitigation: SQS with a dead-letter queue 
between EventBridge and Lambda. Monitor DLQ depth via CloudWatch alarm with a zero-tolerance threshold.

**CloudTrail logging disruption.** An IAM principal with sufficient permissions can disable CloudTrail logging or modify the S3 delivery 
bucket policy, silently breaking the entire detection pipeline. Mitigation: CloudWatch alarm on `StopLogging` and `DeleteTrail` API calls 
with SNS alert to security and FinOps teams simultaneously.

**Slack webhook expiry.** A 404 or 403 from the Slack API causes the function to complete successfully from Lambda's perspective while 
delivering no alert. Mitigation: explicit response code validation in the Lambda function and a monthly synthetic test that fires a known
test event and confirms delivery end to end.

**EventBridge rule misconfiguration.** CloudTrail and EventBridge are decoupled systems. CloudTrail can log to S3 successfully while an 
EventBridge rule with an incorrect event pattern, a missing CloudTrail source configuration, or a broken Lambda target silently drops every 
matching event. Unlike CloudTrail disruption, this failure mode produces no alarm and no delivery error — the pipeline appears healthy while
detection has stopped entirely. Mitigation: monthly synthetic test that injects a known compliant and non-compliant event and verifies end-
to-end alert delivery. Rule configuration should be version-controlled and reviewed as part of any IaC change process.

**Tag policy schema drift.** If `REQUIRED_TAGS` drifts out of sync with the tag keys activated in the AWS Billing console, Finance reports 
and the Lambda validator operate on different definitions of compliance. Mitigation: treat `REQUIRED_TAGS` as the single source of truth for
both the Lambda function and the tag activation step, with a documented change process that updates both simultaneously.

| Failure Mode | Detection Method | Alert Target |
|---|---|---|
| False positive volume | Alert-to-action ratio monitoring | FinOps team |
| Regional rule gap | Weekly automated region audit | FinOps team |
| Lambda concurrency exhaustion | CloudWatch DLQ depth alarm | FinOps team |
| CloudTrail disruption | CloudWatch alarm on StopLogging | Security and FinOps |
| EventBridge misconfiguration | Monthly synthetic end-to-end event test | FinOps team |
| Slack webhook failure | Monthly synthetic end-to-end test | FinOps team |
| Tag schema drift | Change control process on REQUIRED_TAGS | FinOps and Finance |

> A governance pipeline that cannot detect its own failure modes is not a governance pipeline. It is a false confidence generator.

---

---

# Case Study 2: Cross-Account EC2 Cost Spike and Savings Plans Coverage Gap

A week-over-week EC2 cost spike of 34% ($18,400 above baseline) across three linked accounts triggered a Finance escalation five days before
month-end. This investigation identifies the root cause, quantifies the Savings Plans coverage gap exposed by the spike, and delivers a 
modeled commitment recommendation with a break-even analysis.

---

## Results at a Glance

| Metric | Finding |
|---|---|
| Spike magnitude | $18,400 above 4-week baseline, 34% week-over-week increase |
| Root cause | Unplanned batch workload launched in Account B (dev), consuming On-Demand at full rate |
| Savings Plans coverage before spike | 61% of eligible On-Demand spend |
| Optimal coverage target (modeled) | 78% |
| Recommended Compute Savings Plan | $4,200/month 1-year No Upfront |
| Projected monthly savings | $1,134 (27% discount vs On-Demand) |
| Break-even | Month 1 (no upfront commitment) |
| Finance action | Month-end accrual adjusted; commitment recommendation submitted for approval |

---

## What This Demonstrates

| Skill | Evidence |
|---|---|
| Cross-account anomaly detection | Week-over-week Athena query across consolidated CUR partitioned by account |
| Savings Plans coverage analysis | Cost Explorer API coverage report by account and instance family |
| Commitment sizing methodology | Break-even model with utilization discount rate and risk buffer |
| Finance communication | Accrual adjustment memo, approval submission format |
| Optimization sequencing | Attribution confirmed before commitment sized; coverage gap validated before recommendation |

---

## Investigation

### Context and Problem Statement

Finance flagged an EC2 spend anomaly at day 26 of the month. The alert threshold was a 20% week-over-week increase in EC2 unblended cost 
across the consolidated billing account. The threshold fired at 34%. The escalation required a root cause finding and a month-end accrual 
estimate within four business hours.

Three linked accounts were in scope: Account A (prod, $120K/month baseline), Account B (dev, $28K/month baseline), Account C (staging, $15K/
month baseline).

---

### Phase 1: Anomaly Isolation (Cross-Account CUR)

**Objective:** Identify which account, service family, and instance type drove the spike.

A week-over-week cost comparison query across the consolidated CUR:

```sql
WITH weekly AS (
    SELECT
        line_item_usage_account_id,
        line_item_product_code,
        line_item_usage_type,
        DATE_TRUNC('week', line_item_usage_start_date)  AS usage_week,
        SUM(line_item_unblended_cost)                   AS weekly_cost
    FROM cur_database.consolidated_cur
    WHERE
        line_item_line_item_type = 'Usage'
        AND line_item_product_code = 'AmazonEC2'
        AND line_item_usage_start_date
            BETWEEN DATE '2026-03-01' AND DATE '2026-03-31'
    GROUP BY 1, 2, 3, 4
),
comparison AS (
    SELECT
        line_item_usage_account_id,
        line_item_usage_type,
        MAX(CASE WHEN usage_week = DATE '2026-03-16' THEN weekly_cost END) AS week_prior,
        MAX(CASE WHEN usage_week = DATE '2026-03-23' THEN weekly_cost END) AS week_current
    FROM weekly
    GROUP BY 1, 2
)
SELECT
    line_item_usage_account_id,
    line_item_usage_type,
    week_prior,
    week_current,
    week_current - week_prior                   AS delta,
    ROUND(100.0 * (week_current - week_prior)
          / NULLIF(week_prior, 0), 1)           AS pct_change
FROM comparison
WHERE week_current > week_prior
ORDER BY delta DESC
LIMIT 20;
```

**Query output (top results):**

| Account | Usage Type | Week Prior | Week Current | Delta | % Change |
|---|---|---|---|---|---|
| Account B (dev) | EC2:c5.4xlarge-BoxUsage | $2,100 | $16,800 | +$14,700 | +600% |
| Account B (dev) | EC2:c5.2xlarge-BoxUsage | $880 | $4,580 | +$3,700 | +420% |
| Account A (prod) | EC2:m5.xlarge-BoxUsage | $9,200 | $9,430 | +$230 | +2.5% |

The spike was concentrated in Account B (dev) on compute-optimized instance families. Account A showed normal variance. Account C was flat.

---

### Phase 2: Root Cause — CloudTrail Confirmation

Filtered CloudTrail `RunInstances` events for Account B, bounded to the 48-hour window at the start of the spike week (2026-03-23T00:00:00Z
to 2026-03-24T23:59:59Z):

| Finding | Value |
|---|---|
| Event | `RunInstances` |
| Instance family | c5.4xlarge × 6, c5.2xlarge × 4 |
| IAM principal | `arn:aws:iam::ACCOUNT-B:role/data-eng-batch-role` |
| UserAgent | `aws-cli/2.15.0` |
| Launch reason (tag) | `Project: ml-training-batch-march` |
| SCP / budget alert active | No budget alert configured for Account B dev |

The simulated root cause is an unplanned batch workload, modeled on a data engineering ML training run against a production-sized dataset,
launched via CLI with attributable tags (Project: ml-training-batch-march) but no corresponding AWS Budgets alert or capacity pre-approval 
guardrail. This pattern is common in dev accounts with active data science teams and represents the class of ungoverned burst spend the 
detection threshold is designed to surface.

**Root cause:** Absent budget guardrails in the dev account allowed an unplanned compute burst to reach billing without Finance visibility 
until the consolidated threshold fired five days later.

---

### Phase 3: Savings Plans Coverage Gap Analysis

The spike exposed a secondary finding: the existing Savings Plans commitment was undersized for the actual On-Demand run rate across all
three accounts.

**Coverage baseline (4-week average, pre-spike):**

| Account | Eligible On-Demand | Savings Plan Applied | Coverage % |
|---|---|---|---|
| Account A (prod) | $98,000 | $63,700 | 65% |
| Account B (dev) | $22,000 | $11,440 | 52% |
| Account C (staging) | $12,000 | $6,720 | 56% |
| **Total** | **$132,000** | **$81,860** | **62%** |

62% coverage means 38% of eligible On-Demand spend was running at full On-Demand rate — $50,140 per month exposed to no discount. The spike 
amplified this: the batch workload in Account B was entirely On-Demand, adding $18,400 of uncovered spend to an already-undersized 
commitment position.

**Coverage target rationale.** The target coverage of 78% is not an arbitrary benchmark. It reflects the baseline steady-state On-Demand run
rate minus two risk buffers: a 10% buffer for organic workload growth over the commitment term, and a 12% buffer for variable batch 
workloads in dev that cannot be reliably committed against. Committing beyond 78% at current run rates would produce SP wastage if dev batch 
workloads decline — unused Savings Plans are charged regardless of utilization.

---

### Phase 4: Commitment Sizing and Break-Even Model

**Sizing methodology:** Compute Savings Plans are the correct commitment type for this environment. The prod account uses multiple instance 
families (m5, c5, r5) across two regions. Compute SPs apply discount across instance family, size, OS, and region, which removes the 
exception management overhead of EC2 Instance SPs in a heterogeneous fleet.

**Commitment recommendation:**

| Parameter | Value |
|---|---|
| Commitment type | Compute Savings Plan |
| Monthly commitment | $4,200/month |
| Term | 1 year |
| Payment option | No Upfront |
| Effective hourly rate | $5.81/hr |
| On-Demand equivalent | $8.00/hr (c5 family blended) |
| Discount rate | 27.4% |

**Break-even model:**

The No Upfront option has no sunk cost — the break-even is Month 1. The question is minimum utilization required to justify the commitment.

```
Monthly commitment: $4,200
On-Demand equivalent at full utilization: $5,768
Minimum utilization to avoid waste: 72.8% ($4,200 / $5,768)

At current run rate, Account B alone produces $11,440/month in SP-eligible spend.
A $4,200/month Compute SP applied against Account B baseline is utilized at 100%
in all months where the batch workload does not spike and approximately 100%
in spike months (additional On-Demand above SP coverage absorbs the burst).
```

**Annualized savings projection:**

| Scenario | Annual On-Demand Cost | Annual SP Cost | Annual Savings |
|---|---|---|---|
| No new commitment (status quo) | $50,140 × 12 = $601,680 | — | — |
| Add $4,200/month Compute SP | $36,540 × 12 = $438,480 | $50,400 | $112,800 |
| Net benefit (savings minus commitment) | — | — | **$112,800/yr** |

> This model uses the 4-week pre-spike baseline as the committed run rate. It does not assume the batch spike recurs monthly. If the data
engineering team intends to run monthly batch workloads of similar scale, the optimal commitment increases to approximately $6,800/month
with a projected annual savings of $168,000.

---

### Finance Deliverable: Month-End Accrual and Approval Memo

**Accrual adjustment (simulated Finance deliverable):**

The batch workload ran for 6 days before month-end. Estimated additional cost: $18,400 (confirmed from CUR daily granularity). The 
deliverable recommends a +$18,400 adjustment to the March accrual, attributed to Account B, cost centre `ml-training-batch-march`.

**Commitment approval memo (summary):**

> The current Savings Plans position covers 62% of eligible On-Demand spend across three accounts, leaving $50,140/month at full On-Demand
rate. A $4,200/month Compute Savings Plan (1-year, No Upfront) applied against the Account B and blended fleet baseline is projected to save
$112,800 annually at current run rates. Break-even is Month 1. The recommendation does not include the March batch spike in the baseline; if
monthly batch workloads are planned, the optimal commitment increases to $6,800/month. Recommend approval at the $4,200/month level pending
confirmation of batch workload cadence from the data engineering team.

---

## Lessons Learned

- **Anomaly detection is only as good as the threshold that triggers it.** A 20% week-over-week threshold on the consolidated account caught
the spike, but five days into the billing period — too late to prevent it. Account-level budget alerts in AWS Budgets configured at 80% and
100% of monthly forecast would have fired in real time.
- **Savings Plans coverage gaps are invisible until a burst reveals them.** The pre-spike position looked adequate in Cost Explorer coverage
summaries. The spike made the $50,140/month exposed On-Demand exposure visible in a way that routine coverage reports did not.
- **Commitment sizing requires a baseline, not a peak.** Sizing against the spike month would overcommit. Sizing against the 4-week steady-
state baseline with a risk buffer produces a defensible recommendation that Finance can approve.
- **Dev accounts need budget guardrails as much as prod.** The prod account had Service Control Policies and budget alerts. The dev account
had neither. Burst workloads in dev accounts are the most common source of unplanned spend in organizations with active ML or data
engineering teams.
- **Compute Savings Plans outperform EC2 Instance SPs in heterogeneous fleets.** The flexibility premium is worth paying when instance
families, regions, or operating systems change over the commitment term. Instance SPs are appropriate only for stable, single-family, single-
region workloads.

### Organizational Rollout Note

Stakeholder Defense Framework: A production rollout of this recommendation would likely face pressure to size the commitment against the 
spike month rather than the steady-state baseline. The $6,800/month alternative (sized to the March batch peak) carries a material waste 
risk: if batch workloads do not recur monthly, approximately $2,400/month in SP capacity goes unused — $28,800 in annual sunk cost with no 
offsetting On-Demand reduction. The $4,200 recommendation limits maximum monthly waste exposure to roughly $840 at 20% utilization variance 
and preserves a 90-day resize checkpoint once true batch cadence is confirmed. Additionally, the $18,400 spike already consumed 22% of the 
dev account's quarterly compute budget, making Finance unlikely to approve a larger upfront commitment regardless of engineering preference.
The recommended approach is a $4,200/month No Upfront Compute SP with a written 90-day review trigger: if batch workloads sustain >80% of 
spike scale for two consecutive months, resize upward. This framework defends the conservative baseline with quantified risk rather than 
optimism, and aligns the FinOps, Engineering, and Finance stakeholders around a checkpoint instead of a debate.
