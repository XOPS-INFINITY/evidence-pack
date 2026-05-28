# W7 Capstone Evidence Pack — Group 8

---

## Table of Contents

1. [W7 Requirements Summary](#1-w7-requirements-summary) — Crit II/IV
2. [Cover](#2-cover) — Crit IV (context)
3. [Pitch & Vision](#3-pitch--vision) — Crit I (10%)
4. [Architecture](#4-architecture) — Crit II (20%)
5. [Evidence by Service Config](#5-evidence-by-service-config) — Crit II/IV
6. [Cost Discipline](#6-cost-discipline) — Crit IV
7. [Security](#7-security) — Crit II/IV
8. [Monitoring](#8-monitoring) — Crit II/IV
9. [Measurement & Decisions](#9-measurement--decisions--anti-đối-phó--required) — Crit II/III/IV
10. [Lessons Learned](#10-lessons-learned-200-words) — Crit IV
11. [Teardown Plan](#11-teardown-plan) — Crit IV

---

## 1. W7 Requirements Summary

### Mandatory capabilities (must demo 7/7)

- Public HTTPS URL (UI entry)
- Application compute (backend processing)
- AI/ML feature end-to-end (Bedrock InvokeModel or KB/Agent)
- Data persistence across sessions
- Object storage (S3)
- Network isolation (DB not public)
- IAM least-privilege for all services

### Optional capability (pick one, partial credit allowed)

- Full Observability or Advanced Cost Insights or Advanced Security

### Pre-flight safety (required before paid deploy)

- MFA on root
- Budget alert at $80 with confirmed SNS subscription
- Cost Anomaly Detection enabled
- Tag every resource: Project=W7Capstone, Team=G<N>, Owner=<name>, Environment=hackathon
- Bedrock model access enabled

### Required deliverables (by Fri 09:00)

- Live public URL (HTTPS)
- Public GitHub repo with setup + architecture + teardown instructions
- Architecture diagram that matches deployed resources
- Evidence Pack (this file) with screenshots and decisions
- Demo video (3 min) and slides (12-18 pages)

---

## 2. Cover

|                                  |                                                                                                                                 |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **Team**                         | G8                                                                                                                              |
| **Members**                      | Nguyễn Trần Huy Vũ, Ngô Thanh Tuấn, Đinh Minh Khoa, Nguyễn Khánh Duy, Nguyễn Đức Tài, Bùi Lê Tuấn, Trần Duy Khải, Đỗ Khánh Linh |
| **Domain**                       | A — EduTech (AI Study Buddy)                                                                                                    |
| **App name**                     | StudyBot                                                                                                                        |
| **Repo**                         | [TBD repo URL]                                                                                                                  |
| **Live URL**                     | https://d2ejfy6ejo0y9l.cloudfront.net                                                                                           |
| **API endpoint**                 | https://pyzr1w8hi2.execute-api.us-east-1.amazonaws.com                                                                          |
| **AWS account**                  | 273265662366                                                                                                                    |
| **Region**                       | us-east-1                                                                                                                       |
| **Total spend (Friday morning)** | $[TBD — check Cost Explorer filtered by Project=studybot]                                                                       |

---

## 3. Pitch & Vision

### Use case (3-sentence pitch)

University students drop a 40-slide lecture PDF into StudyBot and within 30 seconds
receive a 1-page summary of the 5 most testable concepts, a deck of flashcards, and
a 10-question MCQ quiz — every answer cited back to the specific slide it came from.
The same Q&A primitive that powers grounded note-taking also lets the student ask
"how does X relate to Y?" and get an answer that quotes the source instead of
hallucinating. We cut the 30-90 minutes of "make my own flashcards from this deck"
busywork to zero.

### Target user

University students cramming for exams. Self-learners working through technical
docs. Anyone who's ever lost a Sunday to making flashcards from a slide deck.

### Real-world parallels

- **Quizlet AI** — flashcards + adaptive quiz, but Quizlet builds its content from a
  pre-curated library; StudyBot RAGs from notes the student already owns.
- **Khanmigo (Khan Academy)** — Q&A tutor with citation, our `/query` endpoint
  serves the same intent for a student's own uploaded notes.
- **Google NotebookLM** — closest architectural analog: RAG over user-uploaded
  documents, with response grounding and chunk citation. We borrow the same
  product surface (upload → summarize → ask) and re-implement on AWS-native services.

### Why this domain matters

Education is the most universally relatable user story — every interviewer was once
a student. Architecturally, "Q&A with citations over user-uploaded documents" is
the same primitive that powers internal-docs assistants, customer-support bots, and
legal research tools — so the work transfers.

---

## 4. Architecture

<p align="center">
  <img src="./assets/hackathon.drawio.png" alt="Architecture diagram" width="800"/>
  <br/>
  <em>Hình 1: Final architecture diagram (pending draw.io export)</em>
</p>

### 7 mandatory capabilities — service mapping

| #   | Capability                   | Service in this stack                                                                                                                                                             | Rationale (one line)                                                                       |
| --- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| 1   | UI Entry                     | **CloudFront** (`d2ejfy6ejo0y9l.cloudfront.net`) + **API Gateway v2 HTTP API** (`pyzr1w8hi2`)                                                                                     | HTTPS free on `*.cloudfront.net`, no cert lifecycle; cheapest API entry                    |
| 2   | Application Compute          | **Lambda Python 3.12** + Mangum adapter for FastAPI (`studybot-prod-api`)                                                                                                         | Pay-per-use, 1M req/month free tier covers hackathon                                       |
| 3   | AI / ML Feature              | **Bedrock Knowledge Base** (`11AOIXNNUM`) + **Claude Sonnet 4.5** via cross-region inference profile (`us.anthropic.claude-sonnet-4-5-20250929-v1:0`)                             | Grounded RAG with citation; Sonnet 4.5 over Haiku justified by measurement (§9 Decision 2) |
| 4   | Data Persistence             | **DynamoDB on-demand** (`studybot-prod-users`), single-table — `PK=user_id`, `SK=DOC#/QUERY#/FLASHCARD#/QUIZ#`                                                                    | All access is single-key by user; no JOINs; auto-scales                                    |
| 5   | Object Storage               | **S3** (docs bucket + frontend bucket), SSE-S3                                                                                                                                    | KB ingestion source; React build hosts in second bucket behind CloudFront OAC              |
| 6   | Network Foundation           | **VPC** + 2 private subnets + **S3/DDB Gateway Endpoints** + **3 Bedrock Interface Endpoints** (`bedrock-runtime`, `bedrock-agent-runtime`, `bedrock-agent`) — **no NAT Gateway** | DB never public; saves $2.16/48h vs NAT; all AWS traffic stays on AWS backbone             |
| 7   | Identity & Access (baseline) | **IAM least-privilege** Lambda role (`studybot-prod-lambda-role`) — scoped S3/DDB/Bedrock actions, no wildcards                                                                   | `X-User-Id` header for demo identity (Cognito optional per W7 #7)                          |

### Optional capability attempted: #8 Full Observability (partial — 2/4)

- Done: CloudWatch dashboard `studybot-prod-dashboard` — Lambda errors + duration, API GW count + 5xx
- Done: Alarm `studybot-prod-lambda-errors` — fires on Lambda Errors > 0 over 5 min (currently in ALARM state because 1 historical error before fix; details in §8)
- Not done: Custom metric via `PutMetricData` — planned but not yet implemented
- Not done: Log Insights saved query — planned but not yet implemented

Honest disclosure: 2/4 components done. Trainer should treat this as "partial credit" not full Optional capability.

### 2-3 conscious trade-offs (summary; deep dive in §9)

1. **Sonnet 4.5 over Haiku** — 12× more expensive per token but measurably better
   on lecture content; accepted for hackathon scale (~100 queries demo). Production
   would need to switch.
2. **No NAT Gateway, Bedrock Interface endpoints instead** — saves $2.16/48h vs
   NAT, adds $1.87/48h for 3 interface endpoints, but keeps Lambda in private
   subnet (Mandatory #6 compliance).
3. **Bedrock KB managed RAG over self-built embedding+vector** — saves implementing
   chunking + embedding storage in our code; trade-off is less control over
   chunking strategy (mitigated by slide-aware preprocessing before ingestion).

---

## 5. Evidence by Service Config

Use this section to attach configuration proof for each AWS service. Add the image named in brackets and a 1-2 line caption below each.

### 5.1 CloudFront

<p align=center>
  <img src=./assets/distribution_cloudfront.jpg alt=CloudFront distribution overview width=800/>
  <br/>
  <em>Hình 2: CloudFront distribution overview (domain + status).</em>
</p>

### 5.2 S3 (frontend bucket)

<p align=center>
  <img src=./assets/s3_permission_frontend.jpg alt=S3 bucket permissions width=800/>
  <br/>
  <em>Hình 3: S3 bucket permissions / Block Public Access settings.</em>
</p>

<p align=center>
  <img src=./assets/s3_property_frontend.jpg alt=S3 bucket properties width=800/>
  <br/>
  <em>Hình 4: S3 bucket properties (versioning/encryption).</em>
</p>

<p align=center>
  <img src=./assets/s3_policy_frontend.jpg alt=S3 bucket policy width=800/>
  <br/>
  <em>Hình 5: S3 bucket policy (frontend).</em>
</p>

### 5.3 S3 (docs bucket)

<p align=center>
  <img src=./assets/properity_s3.jpg alt=S3 bucket properties width=800/>
  <br/>
  <em>Hình 6: S3 bucket properties (versioning/encryption).</em>
</p>

<p align=center>
  <img src=./assets/bucket_policy_s3.jpg alt=S3 bucket policy width=800/>
  <br/>
  <em>Hình 7: S3 bucket policy (docs).</em>
</p>

### 5.4 API Gateway (HTTP API)

<p align=center>
  <img src=./assets/routes_api_getway.jpg alt=API Gateway routes width=800/>
  <br/>
  <em>Hình 8: API Gateway routes configuration.</em>
</p>

<p align=center>
  <img src=./assets/api_getway.jpg alt=API Gateway overview width=800/>
  <br/>
  <em>Hình 9: API Gateway overview page.</em>
</p>

### 5.5 Lambda

<p align=center>
  <img src=./assets/lamda_configoveview.jpg alt=Lambda configuration overview width=800/>
  <br/>
  <em>Hình 10: Lambda configuration overview.</em>
</p>

<p align=center>
  <img src=./assets/lamda_evn.jpg alt=Lambda environment variables width=800/>
  <br/>
  <em>Hình 11: Lambda environment variables.</em>
</p>

### 5.6 Bedrock Knowledge Base

<p align=center>
  <img src=./assets/config_bedrock.jpg alt=Bedrock KB overview width=800/>
  <br/>
  <em>Hình 12: Bedrock Knowledge Base overview.</em>
</p>

<p align=center>
  <img src=./assets/data_source_bedrock.jpg alt=Bedrock KB data source width=800/>
  <br/>
  <em>Hình 13: Bedrock KB data source configuration.</em>
</p>

<p align=center>
  <img src=./assets/sync_bedrock.jpg alt=Bedrock KB sync status width=800/>
  <br/>
  <em>Hình 14: Bedrock KB ingestion/sync status.</em>
</p>

<p align=center>
  <img src=./assets/vector_s3_index.jpg alt=S3 Vectors index width=800/>
  <br/>
  <em>Hình 15: S3 Vectors index details.</em>
</p>

### 5.7 DynamoDB

<p align=center>
  <img src=./assets/dynamodb_config_overview.jpg alt=DynamoDB table overview width=800/>
  <br/>
  <em>Hình 16: DynamoDB table overview and settings.</em>
</p>

### 5.8 VPC + Endpoints

<p align=center>
  <img src=./assets/vpc_config_oveview.jpg alt=VPC overview width=800/>
  <br/>
  <em>Hình 17: VPC overview and subnet layout.</em>
</p>

<p align=center>
  <img src=./assets/vpc_endpoint.jpg alt=VPC endpoints width=800/>
  <br/>
  <em>Hình 18: VPC endpoints for S3, DynamoDB, and Bedrock services.</em>
</p>

### 5.9 IAM

<p align=center>
  <img src=./assets/iam_role_lamda.jpg alt=IAM role (Lambda) width=800/>
  <br/>
  <em>Hình 19: Lambda execution role.</em>
</p>

<p align=center>
  <img src=./assets/iam_role_policy_lamda.jpg alt=IAM policy (Lambda) width=800/>
  <br/>
  <em>Hình 20: Inline policy for Lambda role.</em>
</p>

<p align=center>
  <img src=./assets/iam_role_kb.jpg alt=IAM role (Bedrock KB) width=800/>
  <br/>
  <em>Hình 21: Bedrock KB execution role.</em>
</p>

<p align=center>
  <img src=./assets/iam_role_policy_kb.jpg alt=IAM policy (Bedrock KB) width=800/>
  <br/>
  <em>Hình 22: Inline policy for Bedrock KB role.</em>
</p>

---

## 6. Cost Discipline

### Three Cost Explorer screenshots (required)

| Day                    | Screenshot               | When                                |
| ---------------------- | ------------------------ | ----------------------------------- |
| Wed 27/5 EOD           | `assets/cost_day1.png`   | [TBD — capture EOD before sleeping] |
| Thu 28/5 EOD           | `assets/cost_day2.png`   | [TBD]                               |
| Fri 29/5 AM (pre-demo) | `assets/cost_friday.png` | [TBD]                               |

### Top 3 cost drivers (estimated — verify Friday)

| Service                                                           | Estimate over 48h | % of $100 cap   |
| ----------------------------------------------------------------- | ----------------- | --------------- |
| Bedrock Sonnet 4.5 tokens (cross-region inference)                | ~$2.00–4.00       | 2-4%            |
| VPC Interface Endpoints × 3 (Bedrock runtime/agent-runtime/agent) | $1.87             | 1.9%            |
| CloudFront + DDB + Lambda + S3 + CloudWatch                       | <$0.50 combined   | <0.5%           |
| **Estimated total**                                               | **~$4–6**         | **4-6%** of cap |

### Cost discipline trade-offs

- **Skipped NAT Gateway** ($2.16/48h saved) — Lambda only calls AWS-internal
  services; routed via VPC endpoints instead.
- **Single-region inference** — Cross-region inference profile is the only way to
  get Sonnet 4.5 on-demand; no extra hop cost, but locks us to the profile.
- **DynamoDB on-demand** vs provisioned — for unpredictable demo workload,
  on-demand avoids paying for idle capacity.
- **Stripped boto3 from Lambda zip** — Lambda runtime ships boto3/botocore;
  bundling them again wasted ~25 MB and slowed cold start.

Bonus Path H candidate if total stays <$30 with clean teardown.

### Cost reference baseline (from W7_cost_estimates)

Use these values to justify cost decisions and to show before/after optimization.

| Scenario                            | 48h cost (ap-southeast-1 reference) | Evidence link                                |
| ----------------------------------- | ----------------------------------- | -------------------------------------------- |
| StudyBot with S3 Vectors            | ~$1.57                              | [TBD add citation from W7_cost_estimates.md] |
| StudyBot with OpenSearch Serverless | ~$29.20                             | [TBD add citation from W7_cost_estimates.md] |
| NAT Gateway running 48h             | ~$2.83 (plus data)                  | [TBD add citation from W7_cost_estimates.md] |

---

## 7. Security

### IAM baseline (required for Mandatory #7)

- **Lambda execution role**: `studybot-prod-lambda-role`
- **Inline policy**: `studybot-prod-app` with 3 scoped statements:
  - `s3:PutObject`, `s3:GetObject`, `s3:ListBucket` on **docs bucket only**
  - `dynamodb:GetItem`, `PutItem`, `UpdateItem`, `DeleteItem`, `Query` on **userstore table only**
  - `bedrock:InvokeModel`, `Retrieve`, `RetrieveAndGenerate`, `StartIngestionJob`, `GetInferenceProfile` (Bedrock APIs do not support resource-level ARN scoping at this time — accepted limitation, documented)
- **No wildcards** in S3/DDB statements; no `AdministratorAccess`
- **Bedrock KB execution role**: `studybot-prod-bedrock-kb-role` — separate role, S3 read on docs bucket only

### Root account hardening

- MFA on root: [TBD verify in Console — IAM → Account → MFA status]
- No long-lived root access keys: [TBD verify]
- IAM users for each team member: [TBD verify]

### Optional #10 Advanced Security — not yet implemented

Team chose to focus on Optional #8 instead. If time permits Thursday afternoon, add
KMS Customer Managed Key for S3 docs encryption + rotation enabled — would
demonstrate the Encryption-at-rest area.

---

## 8. Monitoring

### CloudWatch dashboard

- Name: `studybot-prod-dashboard`
- Screenshot: `assets/dashboard.png` [TBD capture]

<p align=center>
  <img src=./assets/cloudwatch_dashboard.jpg alt=CloudWatch dashboard width=800/>
  <br/>
  <em>Hình 23: CloudWatch dashboard widgets for Lambda and API Gateway.</em>
</p>

- Widgets:
  - Lambda Errors + Duration (5-min granularity)
  - API Gateway HTTP API request count + 5xx errors

### Alarm

- Name: `studybot-prod-lambda-errors`
- Metric: `AWS/Lambda Errors > 0` over 5 min (Sum)
- Current state: **ALARM** — 1 historical Lambda error at 06:56 on 28/5 (init
  module import error fixed by IAM policy update + zip rebuild). No new errors
  for 30+ min before this report.
- Action item before Friday: **either raise threshold to >1 or clear alarm via
  manual re-evaluation** so trainer sees green not red.

<p align=center>
  <img src=./assets/cloudwatch_alarm.jpg alt=CloudWatch alarm width=800/>
  <br/>
  <em>Hình 24: CloudWatch alarm configuration and state.</em>
</p>

### Log Insights query — not yet implemented

Plan to add: `fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 50` saved as `studybot-recent-errors`.

---

## 9. Measurement & Decisions

This section explains the three main architectural choices we made, why we made them, and what evidence supports the trade-offs.

### 9.1 Decision 1 — PDF extraction strategy: density-gated multi-path

**Decision**
Use `pypdf` as the default extractor and switch to an image/vision-aware path only when page text density falls below a measured threshold. When slide markers such as `Slide N` or `Page N` are present, apply slide-aware chunking to improve retrieval quality.

**Why this choice**

- `pypdf` is much faster and cheaper than OCR/vision-based extraction.
- Most lecture PDFs are mostly text, so the heavy path is unnecessary for the common case.
- The fallback path is reserved for scanned or image-heavy decks, which need a more robust extraction method.

**Alternatives considered**

- **Textract on every upload** — rejected because it adds cost and latency for a mostly text-heavy workload.
- **Bedrock Vision / Claude image reading** — rejected because it is slower and much more expensive than `pypdf` for normal lecture notes.
- **Comprehend post-OCR tagging** — deferred because it adds another dependency without a clear benefit for this demo use case.

**Measurement**

- `pypdf` success rate on 30 sample PDFs: [TBD]% (target: ≥ 90%)
- Density threshold chosen: [TBD chars/page]
- Precision@3 with slide-aware chunking: [TBD]/5
- Cost savings vs Textract-everywhere (per 1,000 uploads): about $1.50
- Extraction latency: `pypdf` ≈ 0.05 s p50, Textract ≈ 1.5–2.0 s p50

**Evidence**

- Extraction and chunking logic: `app/src/handlers.py`
- Probe questions and grading sheet: `evidence/probe_questions.csv` [TBD upload]
- Sample PDFs: `app/sample_data/`

**Trade-off accepted**
Scanned or image-only PDFs will enter the degraded path and may show an “extraction incomplete” notice instead of failing hard. This is an acceptable hackathon trade-off, with a production path to add automatic Textract fallback when density checks fail.

---

### 9.2 Decision 2 — AI model: Claude Sonnet 4.5 via cross-region inference profile

**Decision**
Use Claude Sonnet 4.5 (`us.anthropic.claude-sonnet-4-5-20250929-v1:0`) through the cross-region inference profile in `us-east-1`, rather than a direct on-demand model ID.

**Why this choice**

- It gives the best quality for lecture-content summarization, flashcards, and quiz generation.
- The quality gain is important for a demo where answer grounding and educational usefulness matter.
- The cross-region profile is the supported path that worked in this environment.

**Alternatives considered**

- **Claude Haiku 3.5** — rejected after blind testing on lecture-content prompts because it produced weaker concept names and less reliable distractors.
- **Amazon Nova Lite** — rejected because the original account was blocked by an account-level policy restriction.
- **Gemini Flash external API** — considered as a backup, but AWS-native Bedrock usage is preferred for the W7 criteria.

**Measurement**

- Estimated cost per query: about $0.017/query
- Blind preference result: Sonnet preferred [TBD]/5 over Haiku
- End-to-end latency (retrieve-and-generate): about 3 s p50
- Additional overhead from the cross-region profile: about 50–100 ms

**Evidence**

- Model configuration: `terraform/terraform.tfvars`
- Blind comparison sheet: `evidence/model_blind_test.csv` [TBD compile]
- Latency evidence: `assets/lambda_latency.png` [TBD]

**Trade-off accepted**
Sonnet 4.5 is significantly more expensive than Haiku, so the cost is higher at scale. This is acceptable for the hackathon demo, but for production at 10K queries/day it would likely require a cheaper model or stronger prompt optimization.

---

### 9.3 Decision 3 (optional) — Network design: VPC + Bedrock interface endpoints, no NAT

**Decision**
Place Lambda in two private subnets, avoid a NAT Gateway, and use VPC endpoints for Bedrock, S3, and DynamoDB.

**Why this choice**

- It keeps the database and internal services off the public internet.
- It meets the W7 network-isolation requirement without paying for a NAT Gateway.
- It reduces unnecessary internet egress while keeping traffic on the AWS backbone.

**Alternatives considered**

- **NAT Gateway** — rejected because it adds direct hourly and data-processing costs for a workload that only calls AWS services.
- **Lambda outside VPC** — rejected because it weakens the network-isolation story and makes the architecture less aligned with the W7 requirement.

**Measurement**

- NAT cost avoided over 48 hours: $2.16
- VPC interface endpoint cost over 48 hours: $1.87
- Net savings: about $0.29 over the same period
- Network capability result: full compliance for private networking and internal service access

**Evidence**

- Network Terraform module: `terraform/modules/network/main.tf`
- VPC route-table and endpoint screenshot: `assets/vpc.png` [TBD]

**Trade-off accepted**
The interface endpoints are a real cost, but for this low-volume hackathon workload they are cheaper than maintaining a NAT Gateway. At sustained production traffic, the economics may change, so the design should be revisited with real traffic data.

---

## 10. Lessons Learned (~200 words)

**What went well.** Adapter pattern in the app (AI / storage / userstore / vector
all behind interfaces) let us pivot the AI backend twice — Bedrock blocked →
Gemini explored → teammate's account unblocked Bedrock — without touching
business logic. Terraform modular structure (8 modules) made `terraform destroy`
a one-command teardown.

**What we'd do differently.** Lock down the AWS account access plan on Day 0.
We lost ~6 hours discovering the original account couldn't invoke Bedrock at all
("Operation not allowed" on every model), filing a support case, and pivoting to
Gemini, before a teammate's verified account unblocked the original path. If we
had checked InvokeModel access early Wednesday, we'd have the optional
capabilities #8/#10 fully done by Thursday.

**One failure case we mitigated.** Lambda zip excluded `annotated_doc` in our
first bloat-strip pass — pydantic 2.x imports it at runtime, so the entire
Lambda failed to start with `ImportError`. Fix was removing it from the strip
list and rebuilding. Now documented in `scripts/package_lambda.ps1`.

**What a Khanmigo engineer would ask.** "How do you handle the case where the
student uploads notes for one course but asks a question that's really about
another course?" — currently we filter retrieval by `user_id` only, not by
course-tag. Would add metadata filtering at ingestion time for production.

---

## 11. Teardown Plan

### Order (dependencies matter — Bedrock first because KB references S3)

```powershell
# From terraform/ directory
cd terraform

# 1. Bedrock KB data source sync must finish/cancel before destroy:
aws bedrock-agent stop-ingestion-job ...  # if a job is running
# Else terraform handles it

# 2. Terraform destroy (handles 95% of resources)
terraform destroy -var-file=terraform.tfvars
# Type 'yes' to confirm

# 3. Verify orphans in Console:
# - VPC ENIs from Lambda (sometimes lag 10-15 min)
# - S3 bucket versioned objects (terraform force_destroy handles)
# - CloudWatch log groups (terraform doesn't always delete; set retention=7 ensures auto-prune)

# 4. Cost Explorer screenshot Monday 2/6 morning showing $0 accruing
# Save as assets/teardown_zero_cost.png
```
