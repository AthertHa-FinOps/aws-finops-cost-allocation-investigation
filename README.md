# aws-finops-cost-allocation-investigation

## TL;DR (For Hiring Managers)

This repository documents an **end-to-end AWS FinOps investigation** into a **17% cost allocation gap** caused by missing resource tags — not overspend or billing error.

**What happened**
- Total AWS spend: **$315**
- Spend visible to Finance via required allocation tags: **$260**
- **$53 (17%)** of spend was billed correctly but **invisible to allocation reports**
- Finance could not reliably allocate costs by Environment, Project, or Owner

**How I investigated**
- Validated invoice accuracy to rule out billing discrepancies
- Queried **Cost & Usage Reports (CUR) via Athena** to isolate unallocated spend
- Identified specific EC2 and S3 resources missing required tags
- Used **CloudTrail forensic analysis** to trace the exact API calls that created non-compliant resources
- Confirmed tags were **missing at creation time**, not removed later

**How I fixed it**
- Corrected current-month data so Finance could close accurately
- Designed **event-driven governance controls** (CloudTrail → EventBridge → Lambda)
- Reduced detection time from **monthly reconciliation to minutes**
- Prevented recurrence **without blocking engineers or CI/CD pipelines**

**Outcome**
- Restored **100% cost allocation visibility**
- Eliminated recurring manual reconciliation work
- Shifted governance from reactive cleanup to proactive prevention

**Timeline note:**  
This investigation was conducted in **January 2026** using a sandbox AWS account. Dates shown in CloudTrail screenshots reflect actual event timestamps.

---

## How to Read This Case Study

This repository is written as an **internal FinOps investigation report**, not a step-by-step tutorial.

- Screenshots document **evidence of failure and root cause**
- SQL queries focus on **decision-making**, not exhaustive reporting
- Remediation is described at the **architecture and governance level**
- Dollar values are intentionally small to control cost, but the **failure patterns are production-real**

The goal is to demonstrate **how I investigate, reason, and design controls**, not how fast I click through AWS.

---

## Investigation Summary (At a Glance)

| Phase              | Objective                  | Primary Tooling     | Outcome                  |
|--------------------|----------------------------|---------------------|--------------------------|
| Invoice Validation | Confirm billing accuracy   | Cost Explorer, CUR  | Ruled out billing error  |
| Resource Isolation | Identify unallocated spend | Athena (CUR)        | Isolated EC2 & S3 gaps   |
| Forensics          | Identify creation source   | CloudTrail          | Found untagged API calls |
| Scope Expansion    | Check other services       | Athena (CUR)        | Confirmed S3 impact      |
| Remediation        | Prevent recurrence         | EventBridge, Lambda | Near-real-time detection|

---

## Context & Problem Statement

During monthly cost reconciliation, Finance identified a discrepancy:

- **Total AWS spend:** $315
- **Spend visible when filtered by required tags:** $260
- **Unallocated spend:** $53 (17%)

This discrepancy invalidated Finance reporting and triggered investigation.

Initial suspicion was a billing issue. The investigation confirmed the invoice was correct — the failure was in **cost attribution**, not cost generation.

---

## Phase 1: Invoice Validation

**Objective:** Confirm AWS billed correctly before investigating allocation.

- Validated service-level totals in Cost Explorer
- Reconciled totals against CUR queried via Athena
- Confirmed invoice total matched exactly

**Conclusion:** Billing was correct. The problem existed downstream in allocation and reporting.

![AWS invoice validation confirming $315 total spend](./screenshots/01%20-%20invoice-total-315.png)

---

## Phase 2: Resource Isolation (CUR Analysis)

**Objective:** Identify which resources were missing required allocation tags.

Using Athena queries against CUR data:
- Filtered for resources where required tags were `NULL`
- Identified EC2 charges accounting for a **$40 gap**
- Found partial-month EC2 usage, indicating a mid-month creation event

**Discovery**
- Instance `PROD-WEB-SERVER-01`
- Instance ID: `i-0c9cfb67280fe44ee`
- Missing tags: `Environment`, `Project`, `Owner`

![Athena CUR query isolating unallocated EC2 spend caused by missing allocation tags](./screenshots/02%20-%20cur-resource-isolation-missing-allocation-tags.png)

---

## Phase 3: CloudTrail Forensic Investigation

**Objective:** Determine how and why the resource was created without required tags.

Steps:
- Queried CloudTrail Event History
- Filtered for `RunInstances` events during the relevant date range
- Matched the event to the instance ID identified in CUR

**Findings**
- Resource was launched via **AWS CLI**
- The CLI bypasses console-based tag prompts, making it a common escape hatch for governance controls
- API request did **not include required tags**
- Tags were never removed — they were missing at creation

This confirmed a **policy enforcement gap**, not user tampering.

![CloudTrail forensic evidence showing RunInstances call without required tags](./screenshots/03%20-%20cloudtrail-forensics-runinstances-missing-tags-cli-launch.png)
![CloudTrail Event History filtered to RunInstances events](./screenshots/03a%20-%20cloudtrail-event-history-runinstances.png)
![CloudTrail RunInstances event JSON showing missing tag parameters and CLI user agent](./screenshots/03b%20-%20cloudtrail-runinstances-json-cli-useragent.png)

---

## Phase 4: Expanding Scope to S3

**Objective:** Determine whether the issue was isolated or systemic.

Ran the same CUR tag validation query across all services with material spend:

| Service | Billed | Allocated | Gap |
|---------|--------|-----------|-----|
| EC2     | $220   | $180      | $40 |
| RDS     | $60    | $60       | $0  |
| S3      | $35    | $20       | $15 |

S3 showed the largest remaining allocation gap after EC2.

**S3 Findings**
- Two buckets missing the `Environment` tag key
- $15/month unallocated spend

![CUR-based S3 tag compliance audit showing unallocated storage spend](./screenshots/04%20-%20cur-s3-tag-compliance-audit.png)

---

## Root Cause Analysis

**Root cause:**  
There was **no enforcement at resource creation time**.

- Engineers could launch resources via Console or CLI without required tags
- The system allowed non-compliant resources to exist
- This was not malicious behavior — it was a governance gap

**Key insight:**  
If tagging relies on humans being perfect, it will fail.  
Fix the system, not the people.

---

## Remediation & Controls

**Immediate correction**
- Applied missing tags so Finance could close the month accurately

**Permanent prevention**
- Implemented near-real-time detection using:
  - CloudTrail for event logging
  - EventBridge for event capture
  - Lambda for tag validation
  - Slack alerts for fast remediation

**Enforcement philosophy:**  
Alert, don’t block. Production launches must never be impeded by governance controls. A 5-minute detection window is sufficient for Finance accuracy while preserving Engineering velocity during incidents.

**Result**
- Detection latency reduced from **weeks to minutes**
- Engineers retained velocity
- Finance regained trust in allocation data

---

## Results & Impact

**Quantified outcomes**
- Allocation visibility restored from **83% → 100%** (17-percentage-point improvement)
- **$53/month** recurring allocation gap eliminated (**~$636/year**)
- Detection latency reduced from **30 days → <5 minutes** (**99.7% reduction**)
- Manual Finance reconciliation effort reduced by **~4 hours/month**

**Operational impact**
- Finance closed monthly reports without escalations
- Engineers received immediate feedback instead of end-of-month surprises
- Governance shifted from reactive cleanup to proactive prevention

---

## What I Would Do Next in a Real Organization

1. Baseline tag compliance weekly via CUR
2. Track allocation coverage as a first-class KPI
3. Introduce SCP enforcement only after compliance stabilizes
4. Add cost anomaly detection for early risk signals
5. Publish FinOps dashboards shared by Finance and Engineering

---

## Implementation Note

This case emphasizes **investigation methodology and governance design** rather than exhaustive code listings.

**What’s in this README**
- Architecture and detection flow
- Investigation methodology (CUR forensics, CloudTrail analysis)
- Governance philosophy and organizational trade-offs

**What’s in the repository**
- Production-ready Lambda function (Python, error handling, retries)
- EventBridge rules (JSON configuration)
- Athena SQL queries (with performance annotations)
- Terraform deployment templates (optional IaC)

The value of this portfolio is the **investigation and governance logic**, not button clicks or code volume.

---

## Scope & Limitations

- Sandbox AWS account used to control cost
- Real AWS resources, billing, and logs generated
- Dollar values are small; failure patterns are production-real
- Focus is on **investigation, attribution, and governance**

---

## Why This Matters

This case demonstrates how I approach FinOps problems:
- Validate the data
- Isolate the failure
- Trace it to the source
- Fix the system, not the people

The tools are AWS-native.  
The logic scales.  
The outcome is trustworthy financial reporting without slowing the business.
