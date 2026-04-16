## Phase 0 — POC Definition and Guardrails (must do first)

### 0.1 Lock the POC “demo outcomes”

**What we want to demonstrate (POC = operational):**

- Upload audio to **RAW S3** (or via loader) → pipeline triggers
- **Lambda 1** starts **Transcribe** + writes **DynamoDB** status
- **EventBridge** receives Transcribe Completed/Failed → triggers **Lambda 2**
- **Lambda 2** verifies transcript exists, updates DDB, moves RAW to **Quarantine**, pushes delayed cleanup to **SQS**
- **Lambda 3** consumes SQS and performs **safe delete** (or skips if kill-switch off)
- **CloudWatch logs/alarms** show health and failures clearly

**Why:** This sets the success criteria so you don’t drift into “nice-to-have” work.  
**Done when:** You can walk through the above live with 1 success + 1 failure test.

---

### 0.2 Environment + naming/tagging + IaC structure

**What to do:**

- Decide env separation (single account with env prefixes OR separate accounts)
- Apply naming/tagging standard consistently
- Repo structure: reusable modules + env stacks (`modules/`, `envs/dev|test|qa|prod`)

**Why:** Prevents rework and makes the POC portable to higher envs.  
**Done when:** A single env deploy produces consistent names, tags, and outputs.

---

## Phase 1 — POC Core Flow (end-to-end working pipeline)

### 1.1 Provision S3 buckets/prefixes (RAW + Transcribed + Quarantine)

**What to do:**

- Create **RAW landing** bucket (ingestion target)
- Create **Transcribed** bucket (output target)
- Create **Quarantine** location (bucket or prefix)
- Enable encryption, access controls, lifecycle (basic retention)

**Why:** Storage boundaries and states are core to this flow.  
**Done when:** You can upload to RAW and confirm buckets/prefixes exist with correct policies.

---

### 1.2 Build the ingestion approach (POC-friendly)

**What to do (POC option):**

- For POC, you can **manually upload** to RAW with a standard key format
- If required, add a lightweight “File Loader” (script/DataSync/job)

**Why:** Ingestion consistency is needed for deterministic downstream processing.  
**Done when:** Audio lands in RAW using predictable object keys and metadata.

---

### 1.3 DynamoDB job tracking table (state model)

**What to do:**

- Create table to store:
    - `jobId` (or callId)
    - raw bucket/key
    - transcript bucket/key
    - status: `JOB_STARTED → VERIFIED → QUARANTINED → DELETED` (+ `FAILED`)
    - timestamps + error reason (optional)

**Why:** This is your “source of truth” for recovery and idempotency.  
**Done when:** Every file has a single job record and state transitions are visible.

---

### 1.4 Lambda 1 — Ingest trigger + Transcribe starter

**What to do:**

- Trigger: **S3 ObjectCreated** on RAW bucket
- Function responsibilities:
    - validate file/key
    - generate unique jobId
    - write DDB status = `JOB_STARTED`
    - start **Amazon Transcribe** job (with required PII settings)

**Why:** Starts the workflow reliably on new audio arrival.  
**Done when:** Upload → DDB record created + Transcribe job starts (visible in logs/console).

---

### 1.5 EventBridge — Route Transcribe completion/failure to Lambda 2

**What to do:**

- Create EventBridge rules matching Transcribe job state changes (Completed/Failed)
- Target: Lambda 2

**Why:** Makes processing event-driven (no polling).  
**Done when:** You see Lambda 2 invoked after Transcribe completes or fails.

---

### 1.6 Lambda 2 — Transcript verifier + Quarantine mover + cleanup enqueue

**What to do:**

- On **Completed**:
    - verify transcript exists in Transcribed bucket
    - update DDB: `VERIFIED`
    - move/copy RAW → Quarantine
    - update DDB: `QUARANTINED`
    - send cleanup message to SQS (delayed)
- On **Failed**:
    - update DDB: `FAILED`
    - do not delete/move RAW unless you explicitly want that behavior

**Why:** Prevents deletion until transcript is actually confirmed and job is safe.  
**Done when:** Successful job always results in transcript + quarantine + SQS message.

---

### 1.7 SQS Cleanup Buffer (delay queue)

**What to do:**

- Create SQS queue for cleanup messages
- Configure delay/safety window (short window is fine for POC)
- Include message payload with jobId + object locations

**Why:** Gives a buffer window and decouples cleanup from verification.  
**Done when:** Messages appear in SQS after Lambda 2, respecting delay.

---

### 1.8 Lambda 3 — Safe cleanup + final deleter

**What to do:**

- Trigger: SQS event source mapping
- Safety checks:
    1. confirm transcript still exists
    2. confirm DDB status = `QUARANTINED`
    3. check `deletion_enabled` kill-switch
    4. delete quarantine RAW (or skip)
    5. update DDB status = `DELETED`

**Why:** Prevents accidental destructive deletes and supports incident safety.  
**Done when:** Cleanup only happens when all checks pass, and status updates correctly.

---

### 1.9 SSM Parameter Store kill-switch (`deletion_enabled`)

**What to do:**

- Create SSM param `deletion_enabled` true/false
- Lambda 3 must respect it

**Why:** Operational control during incidents/testing.  
**Done when:** Setting false stops deletes but pipeline still runs to QUARANTINED.

---

### 1.10 Observability (minimum for operational POC)

**What to do:**

- CloudWatch log groups + retention
- Alarms (minimum):
    - Lambda errors (1/2/3)
    - SQS backlog (queue depth)
    - DLQ depth (if enabled)
- Basic dashboard (optional but helpful)

**Why:** “Operational” means you can detect failures quickly.  
**Done when:** You can point to alerts/logs for both success and failure cases.

---

## Phase 1.5 — POC Reliability Hardening (these prevent most POC failures)

### 1.11 IAM + permissions sanity pass (common blocker #1)

**What to do:**

- Least-privilege roles per Lambda
- Ensure access to S3/DDB/Transcribe/EventBridge/SQS/SSM (+ KMS if used)

**Why:** Most POCs break on missing permissions.  
**Done when:** No AccessDenied errors in CloudWatch across the full flow.

---

### 1.12 Idempotency and duplicate delivery handling (common blocker #2)

**What to do:**

- Lambda 1: don’t create duplicate jobs for same object (use DDB conditional writes)
- Lambda 3: safe deletes should be repeatable (no “double delete” failures)

**Why:** S3 and SQS can deliver duplicates; your pipeline must tolerate it.  
**Done when:** Re-sending the same event does not create duplicates or corrupt state.

---

### 1.13 Failure paths + DLQ (common blocker #3)

**What to do:**

- Add DLQs where appropriate (Lambda/SQS)
- Define re-drive procedure: how to retry safely after fixing root cause

**Why:** Operational POC requires controlled recovery.  
**Done when:** You can force a failure and show how it gets captured and re-driven.

---

### 1.14 Test plan (common blocker #4)

**What to do:**

- Happy path: valid audio file → DELETED (or QUARANTINED if kill-switch off)
- Failure path: bad format / forced Transcribe failure → FAILED state
- Cleanup safety test: transcript missing → Lambda 3 must not delete

**Why:** Proves correctness and safety.  
**Done when:** You can run these scenarios in front of the team.

---

## Phase 2 — Optional (can be deferred and still call it an operational POC)

### 2.1 CloudTrail data events for S3 object audit

**What to do:** Enable object-level audit for copy/delete traceability.  
**Why:** Strong audit trail; useful for compliance.  
**Defer ok:** Yes for POC.

---

### 2.2 ML model call analysis + analytics dashboard

**What to do:** Trigger analysis from transcript creation and publish insights.  
**Why:** Adds business value beyond transcription.  
**Defer ok:** Yes for POC (unless the POC specifically requires analytics).

---

### 2.3 Multi-env promotion and strict CI/CD gates

**What to do:** Approvals, staged deployments, artifact versioning, etc.  
**Why:** Production readiness.  
**Defer ok:** Yes for POC, but you should have at least validate/plan/apply flow.

---

## What you can say “POC is done” when…

You can confidently demo:

- **1 success case** end-to-end
- **1 failure case** with clean status + logs
- Kill-switch stopping deletion
- Clear answer to: _Where’s the audio? Where’s the transcript? What’s the current state? What happens on failure?_
