# AWS FinOps Cost Allocation Investigation

## TL;DR (For Hiring Managers)

This repository documents an **end-to-end AWS FinOps investigation** into a **17% cost allocation gap** caused by missing resource tags — not overspend or billing error.

---

## What Happened

- Total AWS spend: **$315**
- Spend visible to Finance via required allocation tags: **$260**
- **$53 (~17%)** of spend was billed correctly but **invisible to allocation reports**
- Finance could not reliably allocate costs by Environment, Project, or Owner

---

## How I Investigated

- Validated invoice accuracy to rule out billing discrepancies
- Queried **Cost & Usage Reports (CUR) via Athena** to isolate unallocated spend
- Identified EC2 and S3 resources missing required tags
- Used **CloudTrail forensic analysis** to trace the API calls that created non-compliant resources
- Confirmed tags were **missing at creation time**, not removed later

---

## How I Fixed It

- Corrected current-month data so Finance could close accurately
- Designed **event-driven governance controls** (CloudTrail → EventBridge → Lambda)
- Reduced detection time from **monthly reconciliation to minutes**
- Prevented recurrence **without blocking engineers or CI/CD pipelines**

---

## Outcome

- Restored **100% cost allocation visibility**
- Eliminated recurring manual reconciliation work
- Shifted governance from reactive cleanup to proactive prevention

**Timeline note:**  
This investigation was conducted in **January 2026** using a sandbox AWS account. CloudTrail timestamps reflect real event times.

---

## How to Read This Case Study

This repository is written in the style of an **internal FinOps investigation report**, not a step-by-step tutorial.

- Screenshots document **evidence and root cause**
- SQL queries emphasize **decision-making**, not report completeness
- Remediation focuses on **governance and system design**
- Dollar values are intentionally small, but **failure patterns are production-real**

---

## Investigation Summary (At a Glance)

| Phase              | Objective                  | Tooling              | Outcome                  |
|--------------------|----------------------------|----------------------|--------------------------|
| Invoice Validation | Confirm billing accuracy   | Cost Explorer, CUR   | Billing ruled out        |
| Resource Isolation | Identify unallocated spend | Athena (CUR)         | EC2 & S3 gaps isolated   |
| Forensics          | Identify creation source   | CloudTrail           | Untagged API calls found |
| Scope Expansion    | Check other services       | Athena (CUR)         | Systemic issue confirmed |
| Remediation        | Prevent recurrence         | EventBridge, Lambda  | Near-real-time detection |

---

## Context & Problem Statement

During monthly reconciliation, Finance identified a discrepancy:

- **Total AWS spend:** $315  
- **Spend visible with required tags:** $260  
- **Unallocated spend:** $53 (~17%)

The invoice was accurate. The failure was in **cost attribution**, not cost generation.

---

## Phase 1: Invoice Validation

**Objective:** Confirm AWS billed correctly before investigating allocation.

- Validated service totals in Cost Explorer
- Reconciled totals against CUR via Athena
- Confirmed invoice totals matched exactly

**Conclusion:** Billing was correct. The issue existed downstream.

![Invoice validation confirming $315 total spend](./screenshots/01%20-%20invoice-total-315.png)

---

## Phase 2: Resource Isolation (CUR Analysis)

**Objective:** Identify resources missing required allocation tags.

Using Athena queries against CUR data (**unblended costs** to match invoice source of truth):

- Filtered for records where required tags were `NULL`
- Identified EC2 charges accounting for **$40** in unallocated spend
- Partial-month usage indicated a mid-month creation event

**Discovery**
- **Instance:** `PROD-WEB-SERVER-01`
- **Instance ID:** `i-0c9cfb67280fe44ee`
- **Missing tags:**
  - `Environment`
  - `Project`
  - `Owner`

![CUR query isolating unallocated EC2 spend](./screenshots/02%20-%20cur-resource-isolation-missing-allocation-tags.png)

This query filters for records where required allocation tags are NULL, isolating spend that cannot be attributed in Finance reports.

---

## Phase 3: CloudTrail Forensic Investigation

**Objective:** Determine how and why the resource was created without tags.

**Findings**
- Resource launched via **AWS CLI**
- CLI bypasses console-based tag prompts
- API request **did not include required tags**
- Tags were never removed — they were missing at creation

This confirmed a **governance enforcement gap**, not user tampering.

![RunInstances call without required tags](./screenshots/03%20-%20cloudtrail-forensics-runinstances-missing-tags-cli-launch.png)

![CloudTrail event history filtered to RunInstances](./screenshots/03a%20-%20cloudtrail-event-history-runinstances.png)

![RunInstances event JSON showing missing tags](./screenshots/03b%20-%20cloudtrail-runinstances-json-cli-useragent.png)

*Note: JSON is truncated for readability. No required tags were present in the request.*

---

## Phase 4: Expanding Scope to S3

**Objective:** Determine whether the issue was isolated or systemic.

CUR validation across services:

| Service | Billed | Allocated | Gap |
|---------|--------|-----------|-----|
| EC2     | $220   | $182      | $40 |
| RDS     | $60    | $60       | $0  |
| S3      | $35    | $20       | $15 |

While EC2 represented the largest gap, S3 confirmed the issue was **systemic**.

**S3 Findings**
- Two buckets missing the required `Environment` tag
- Other tags present but insufficient for allocation
- **$15/month** unallocated storage spend

![S3 tag compliance audit via CUR](./screenshots/04%20-%20cur-s3-tag-compliance-audit.png)

Although Project and Owner tags were present, the missing Environment tag prevented Finance from allocating S3 costs, resulting in $15/month of invisible spend.

---

## Root Cause Analysis

**Root cause:**  
No enforcement at resource creation time.

If tagging depends on humans being perfect, it will fail.  
**Fix the system, not the people.**

---

## Remediation & Controls

### Immediate Correction
- Applied missing tags so Finance could close the month accurately

### Permanent Prevention
- CloudTrail for event logging
- EventBridge for event capture
- Lambda for tag validation
- Slack alerts for fast remediation

---

## Architecture Overview

```mermaid
flowchart TD
    A[CloudTrail] --> B[EventBridge Rule]
    B --> C[Lambda: Tag Validation]
    C --> D[Slack Notification]
```

### Enforcement Philosophy

**Alert, don’t block.**

This approach preserves engineering velocity while maintaining financial accuracy.  
Non-compliant resources are surfaced immediately without breaking CI/CD pipelines or developer workflows.

This aligns with progressive FinOps governance, where guardrails precede enforcement.

---

## Results and Impact

### Financial Reporting Integrity

- Allocation visibility restored from **~82.5% to 100%**
- Finance closed monthly reports without escalation

### Quantified Outcomes

- **$53/month** recurring allocation gap eliminated (**~$636/year**)
- Detection latency reduced from **~30 days to under 5 minutes**
- Manual Finance reconciliation reduced by **~4 hours per month**

### Operational Impact

- Finance closed monthly reports without escalation
- Engineers received immediate, actionable feedback
- Governance shifted from reactive cleanup to proactive prevention

---

## What I Would Do Next in a Real Organization

- Baseline tag compliance weekly using CUR
- Track allocation coverage as a first-class FinOps KPI
- Introduce SCP enforcement for **new (greenfield) accounts only** once compliance stabilizes
- Add cost anomaly detection for early risk signals
- Publish shared FinOps dashboards for Finance and Engineering

---

## Implementation Note

This case emphasizes **investigation methodology and governance design** rather than code volume.

---

## What’s in This Repository

- Production-ready Lambda (Python, retries, error handling)
- EventBridge rules (JSON)
- Athena SQL queries with performance notes
- Optional Terraform deployment templates

The value of this repository is the **reasoning and governance design**, not button clicks.

---

## Scope and Limitations

- Sandbox AWS account used to control cost
- Real AWS resources, billing data, and logs generated
- Dollar values are intentionally small; failure patterns are production-real
- Focus is on investigation, attribution, and governance design

---

## Why This Matters

This case demonstrates how I approach FinOps problems:

- Validate the data
- Isolate the failure
- Trace it to the source
- Fix the system, not the people

**Analysis note:**  
Unblended costs were used to align directly with AWS invoice totals, which Finance treats as the authoritative source of truth.

The tools are AWS-native.  

The logic scales.  
The outcome is trustworthy financial reporting without slowing the business.

This is the same investigative and governance approach I apply regardless of spend scale.
