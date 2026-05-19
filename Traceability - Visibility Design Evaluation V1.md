# Traceability & Visibility Evaluation — Async-First OIC Gen 3 Pipeline Tracking

---

## 1. The Traceability Challenge in Async-First Architecture

Before evaluating what we have, it is important to acknowledge what async introduces that sync does not:

```
SYNC WORLD                          		ASYNC WORLD
──────────────────────────────      ─────────────────────────────────────
Business flow writes directly  →    Business flow enqueues → dispatcher
to ATP. Record exists the           writes later. Record exists 30–60s
instant the stage completes.        after the stage completes.

Single thread of execution.    →    Two independent execution threads:
Trace follows one timeline.         (1) business flow timeline
                                    (2) dispatcher/persistence timeline

Failure = one place to look.  →    Failure could be in business logic,
                                    queue delivery, or dispatcher write.
                                    Must correlate across all three.
```

This means traceability in our design must be **multi-dimensional** — it must span the business flow, the queue, and the persistence layer — and it must be **stitchable** from any starting point (correlationId, businessKey, OIC instance ID, or timestamp range).

---

## 2. The Correlation Key Hierarchy

This is the foundational concept. Every piece of data in every layer carries one or more of these keys, enabling cross-layer joins at investigation time.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CORRELATION KEY HIERARCHY                                │
│                                                                             │
│  Level 1 — BUSINESS IDENTITY (human-meaningful, externally provided)        │
│  ─────────────────────────────────────────────────────────────────────────  │
│  businessKey = "PO-2026-00421"                                              │
│    │                                                                        │
│    │  What it is:   The business transaction identifier from the source     │
│    │                system (PO number, invoice ID, order ref, etc.)         │
│    │  Where it lives: OIC_PIPELINE_RUNS, OIC Queue message envelope,        │
│    │                  email alert body, OCI Logging entries                 │
│    │  Used for:     Human-initiated investigations ("trace PO-2026-00421")  │
│    │                Operations dashboard filtering                          │
│    │                Support ticket correlation                              │
│                                                                             │
│  Level 2 — PIPELINE IDENTITY (system-generated, pipeline-scoped)            │
│  ─────────────────────────────────────────────────────────────────────────  │
│  correlationId = "f47ac10b-58cc-4372-a567-0e02b2c3d479"  (UUID v4)          │
│    │                                                                        │
│    │  What it is:   The single key that ties ALL records across ALL tables  │
│    │                and ALL layers for one end-to-end pipeline execution    │
│    │  Where it lives: ALL queue messages, ALL ATP records, OCI Object       │
│    │                  Storage object keys, OCI Logging entries,             │
│    │                  email alerts, HTTP headers to child integrations      │
│    │  Used for:     Cross-layer record joins, full timeline reconstruction  │
│    │                Child integration correlation (same key, same run)      │
│                                                                             │
│  Level 3 — STAGE IDENTITY (system-generated, stage-scoped)                  │
│  ─────────────────────────────────────────────────────────────────────────  │
│  trackingId = "a1b2c3d4-..."  (UUID v4, one per stage attempt)              │
│    │                                                                        │
│    │  What it is:   Unique ID for a specific stage execution attempt        │
│    │  Where it lives: OIC_STAGE_TRACKING, OIC_ACTIVITY_LOG,                 │
│    │                  queue STAGE_UPDATE and ACTIVITY_LOG messages          │
│    │  Used for:     Pinpointing exactly which attempt of which stage        │
│    │                produced a specific log entry or payload snapshot       │
│                                                                             │
│  Level 4 — MESSAGE IDENTITY (queue-scoped, dispatcher-scoped)               │
│  ─────────────────────────────────────────────────────────────────────────  │
│  messageId = "c9d8e7f6-..."  (UUID v4, one per queue message)               │
│    │                                                                        │
│    │  What it is:   Unique ID of the queue message itself                   │
│    │  Where it lives: Queue message envelope, OCI_TRACKING_DLQ              │
│    │  Used for:     Dispatcher deduplication (idempotency on retry)         │
│    │                DLQ investigation — trace which enqueue event failed    │
│                                                                             │
│  Level 5 — OIC NATIVE IDENTITY (OIC-platform-scoped)                        │
│  ─────────────────────────────────────────────────────────────────────────  │
│  oicInstanceId = OIC native flow tracking ID                                │
│    │                                                                        │
│    │  What it is:   The OIC platform's own instance tracking identifier     │
│    │  Where it lives: OIC_PIPELINE_RUNS, OIC monitoring console             
│    │  Used for:     Bridge from ATP tracking records back to OIC native     │
│    │                monitoring console — access to OIC flow audit logs,     │
│    │                payload inspector, error details in OIC UI              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Traceability Coverage Evaluation by Layer

### 3.1 Layer-by-Layer Assessment

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                  TRACEABILITY COVERAGE MATRIX                                  │
├─────────────────────┬────────────────────────┬───────────────┬─────────────────┤
│ Layer               │ What Is Visible         │ Correlation   │ Gap / Risk     │
│                     │                         │ Keys Present  │                │
├─────────────────────┼────────────────────────┼───────────────┼─────────────────┤
│ Business            │ Stage start/end times   │ correlationId │ 30–60s lag     │
│ Integration Flow    │ Stage statuses          │ businessKey   │ before ATP     │
│ (OIC Runtime)       │ Error codes & messages  │ trackingId    │ reflects       │
│                     │ Payload snapshots       │ messageId     │ current state  │
│                     │ Non-mandatory skips     │ oicInstanceId │                │
├─────────────────────┼────────────────────────┼───────────────┼─────────────────┤
│ OCI Queue           │ Enqueue timestamps      │ correlationId │ Messages not    │
│ (oic-tracking-      │ Message types           │ messageId     │ human-browsable │
│  events,            │ Delivery attempts       │ integrationId │ natively —      │
│  oic-activity-logs) │ Visibility timeouts     │               │ need dispatcher │
│                     │ Queue depth (monitoring)│               │ to materialise  │
├─────────────────────┼────────────────────────┼───────────────┼─────────────────┤
│ ATP                 │ Full pipeline timeline  │ correlationId │ Eventual —      │
│ OIC_PIPELINE_RUNS   │ Business key            │ businessKey   │ written async   │
│ OIC_STAGE_TRACKING  │ Stage state machine     │ trackingId    │ by dispatcher   │
│ OIC_ACTIVITY_LOG    │ Duration per stage      │ stageId       │                 │
│                     │ Retry attempts          │ oicInstanceId │                 │
│                     │ Structured log stream   │               │                 │
├─────────────────────┼────────────────────────┼───────────────┼─────────────────┤
│ OCI Object Storage  │ Failure payload body    │ correlationId │ Only failure    │
│ (payload snapshots) │ Exact request at        │ stageId       │ stages. No      │
│                     │ failure moment          │ timestamp     │ success payload │
│                     │ Object URI in ATP       │               │ (by design)     │
├─────────────────────┼────────────────────────┼───────────────┼─────────────────┤
│ OCI Logging         │ Queue enqueue failures  │ correlationId │ Fallback only — │
│ (fallback)          │ Circuit breaker events  │ businessKey   │ unstructured    │
│                     │ Dispatcher DLQ writes   │               │ text, harder    │
│                     │ Tracking bypass events  │               │ to query        │
├─────────────────────┼────────────────────────┼───────────────┼─────────────────┤
│ OIC Native Monitor  │ OIC flow execution      │ oicInstanceId │ Not joined to   │
│ (OIC Console)       │ OIC payload inspector   │ businessKey   │ custom tracking │
│                     │ OIC error fault detail  │ (via tracking │ tables natively │
│                     │ OIC retry history       │  fields)      │ — manual pivot  │
├─────────────────────┼────────────────────────┼───────────────┼─────────────────┤
│ Email Alerts        │ correlationId           │ correlationId │ One-way push —  │
│                     │ businessKey             │ businessKey   │ no drill-down   │
│                     │ Failed stage detail     │ stageId       │ from email to   │
│                     │ Error code + message    │               │ dashboard yet   │
│                     │ Query API link          │               │ (addressable)   │
├─────────────────────┼────────────────────────┼───────────────┼─────────────────┤
│ DLQ                 │ Raw original message    │ correlationId │ Async lag +     │
│ OIC_TRACKING_DLQ    │ Failure reason          │ messageId     │ DLQ means some  │
│                     │ Attempt count           │ messageType   │ records may     │
│                     │ First/last failed time  │               │ never reach ATP │
│                     │ Resolution status       │               │ without manual  │
│                     │                         │               │ intervention    │
└─────────────────────┴────────────────────────┴───────────────┴─────────────────┘
```

---

### 3.2 Cross-Layer Correlation Map

This shows precisely how you stitch records across layers using the key hierarchy:

```
                        CROSS-LAYER CORRELATION MAP

  businessKey ──────────────────────────────────────────────────────────────────┐
  "PO-2026-00421"                                                               │
        │                                                                       │
        │ JOIN                                                                  │
        ▼                                                                       │
  OIC_PIPELINE_RUNS                                                             │
  ├── CORRELATION_ID  ◀─────────────────────────────────── PRIMARY PIVOT KEY    │
  ├── BUSINESS_KEY    ◀─────────────────────────────────── human entry point    │
  ├── OIC_INSTANCE_ID ──────────────────────────────────▶  OIC Console          │
  ├── OVERALL_STATUS                                                            │
  └── INITIATED_AT                                                              │
        │                                                                       │
        │ JOIN on CORRELATION_ID                                                │
        ▼                                                                       │
  OIC_STAGE_TRACKING                                                            │
  ├── TRACKING_ID    ◀──────────────────────────────────── stage-level pivot    │
  ├── STAGE_ID                                                                  │
  ├── STATUS                                                                    │
  ├── DURATION_MS                                                               │
  ├── ERROR_CODE / ERROR_MESSAGE                                                │
  └── PAYLOAD_SNAPSHOT_REF ────────────────────────────▶  OCI Object Storage    │
        │                                                       │               │
        │ JOIN on CORRELATION_ID + TRACKING_ID                  │               │
        ▼                                                       ▼               │
  OIC_ACTIVITY_LOG                               Failure Payload JSON           │
  ├── SEVERITY                                   (exact request at              │
  ├── ACTION                                      failure moment)               │
  ├── MESSAGE                                                                   │
  ├── DETAIL (JSON)                                                             │
  └── STACK_TRACE                                                               │
        │                                                                       │
        │ If DLQ entries exist                                                  │
        ▼                                                                       │
  OIC_TRACKING_DLQ                                                              │
  ├── ORIGINAL_MESSAGE_ID ──────────── links to queue message                   │
  ├── MESSAGE_TYPE                                                              │
  ├── RAW_PAYLOAD (full original message preserved)                             │
  └── FAILURE_REASON                                                            │
        │                                                                       │
        │ If circuit breaker fired                                              │
        ▼                                                                       │
  OCI Logging ◀──────────────────────────────────── correlationId in log text  │
  (unstructured fallback)                                                       │
        │                                                                       │
        └─────────────────────────────────────────────────────────────────────▶│
                                                                                │
  OIC Native Monitor ◀────────────────────── oicInstanceId from PIPELINE_RUNS ─┘
  (flow-level audit,
   OIC payload inspector)
```

---

## 4. Failure Tracing — End-to-End Worked Example

**Scenario:** A PO integration fails at the Enrichment stage (STAGE_002) because the MDM system returns a 503. The operations team receives an email alert and needs to fully investigate.

---

### 4.1 What Happened — The Failure Timeline

```
TIME         EVENT                                   LAYER
───────────────────────────────────────────────────────────────────────────
T+00:00.000  Business integration triggered           OIC Runtime
             (PO-2026-00421 arrives via REST)

T+00:00.005  JS: corrId = "f47ac10b-..."             OIC Runtime (local)
             (no network call — instant)

T+00:00.030  Enqueue PIPELINE_INIT message            OCI Queue
             messageId: "msg-init-001"               (oic-tracking-events)

T+00:00.055  Enqueue ACTIVITY_LOG "PIPELINE_STARTED"  OCI Queue
                                                     (oic-activity-logs)

T+00:00.080  STAGE_001 enters IN_PROGRESS             OIC Runtime
T+00:00.082  Enqueue STAGE_UPDATE IN_PROGRESS         OCI Queue
T+00:00.107  Enqueue ACTIVITY_LOG VALIDATION_STARTED  OCI Queue

T+00:01.200  STAGE_001 validation passes              OIC Runtime
T+00:01.202  Enqueue STAGE_UPDATE COMPLETED           OCI Queue
T+00:01.227  Enqueue ACTIVITY_LOG VALIDATION_PASSED   OCI Queue

T+00:01.250  STAGE_002 enters IN_PROGRESS             OIC Runtime
T+00:01.252  Enqueue STAGE_UPDATE IN_PROGRESS         OCI Queue
T+00:01.277  Enqueue ACTIVITY_LOG ENRICHMENT_STARTED  OCI Queue

T+00:01.300  MDM REST call initiated                  OIC Runtime → MDM

T+01:01.300  MDM call times out (60s timeout)         OIC Fault Handler
             ← 503 Service Unavailable

T+01:01.350  OIC Fault Handler: STAGE_002 executes    OIC Runtime
             Enqueue STAGE_UPDATE FAILED              OCI Queue
               { errorCode: "MDM_TIMEOUT",
                 errorMessage: "MDM 503 after 60s",
                 capturePayload: true,
                 payloadContent: { mdmRequest... } }

T+01:01.375  Enqueue ACTIVITY_LOG ERROR               OCI Queue
               { action: "MDM_CALL_FAILED",
                 detail: { httpStatus: 503,
                            supplierCode: "SUP-001",
                            mdmEndpoint: "/suppliers/lookup",
                            attemptDuration: 60003 } }

T+01:01.400  Enqueue PIPELINE_COMPLETE FAILED         OCI Queue
T+01:01.420  Business integration terminates          OIC Runtime
             (re-throws fault, returns 500 to caller)

── async lag ~30s ───────────────────────────────────────────────────────

T+01:30.000  DISPATCHER wakes (scheduled poll)        OIC Dispatcher

T+01:30.200  Dequeues all tracking-events messages     OIC Queue
             Processes in order:
               msg-init-001 → SP_INIT_PIPELINE ✅
               STAGE_001 IN_PROGRESS            ✅
               STAGE_001 COMPLETED              ✅
               STAGE_002 IN_PROGRESS            ✅
               STAGE_002 FAILED                 ←─ triggers:
                 ├── PUT payload to OCI Object Storage ✅
                 │     key: f47ac10b/STAGE_002/2026-05-15T100101_payload.json
                 ├── SP_UPDATE_STAGE FAILED      ✅
                 └── OIC Email Action → alert sent ✅
               PIPELINE_COMPLETE FAILED         ✅

T+01:30.800  Dequeues activity-log messages (batch)   OCI Queue
             SP_BATCH_WRITE_LOGS (6 entries)    ✅

T+01:31.000  ATP fully reflects pipeline state        ATP
             Operations team receives email alert
```

---

### 4.2 The Failure Alert Email — Entry Point for Investigation

```
From:    oic-alerts@yourcompany.com
To:      integration-ops@yourcompany.com
Subject: [OIC ALERT] PO to ERP Sync — STAGE FAILED — ENRICHMENT

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 INTEGRATION STAGE FAILURE — PROD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Integration:      PO to ERP Sync  (PO_TO_ERP_SYNC v1.2)
 Environment:      PROD
 ─────────────────────────────────────────────────────
 Correlation ID:   f47ac10b-58cc-4372-a567-0e02b2c3d479  ◀── KEY 1
 Business Key:     PO-2026-00421                          ◀── KEY 2
 OIC Instance ID:  oic-inst-7e3f9a21                      ◀── KEY 3
 ─────────────────────────────────────────────────────
 Failed Stage:     ENRICHMENT  (STAGE_002)
 Error Code:       MDM_TIMEOUT
 Error Message:    Supplier MDM returned 503 after 60s
 Failed At:        2026-05-15 10:01:01 UTC
 ─────────────────────────────────────────────────────
 🔍 Trace this pipeline:
    /tracking/v1/pipeline/f47ac10b-.../summary
    /tracking/v1/pipeline/f47ac10b-.../stages
    /tracking/v1/pipeline/f47ac10b-.../logs

 📦 Failure payload snapshot:
    oci://tracking-payloads/f47ac10b-.../STAGE_002/
    2026-05-15T100101_payload.json

 🖥  OIC Console (flow-level audit):
    https://oic.../monitoring/instances/oic-inst-7e3f9a21
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 4.3 Step-by-Step Trace Investigation

The investigator starts from the email and drills down using `correlationId` as the pivot at every step.

#### STEP 1 — Get Pipeline Summary

```sql
-- What happened at the pipeline level?
SELECT
    CORRELATION_ID,
    BUSINESS_KEY,
    INTEGRATION_ID,
    OVERALL_STATUS,
    INITIATED_AT,
    COMPLETED_AT,
    TOTAL_STAGES,
    COMPLETED_STAGES,
    FAILED_STAGE_ID,
    OIC_INSTANCE_ID
FROM OIC_PIPELINE_RUNS
WHERE CORRELATION_ID = 'f47ac10b-58cc-4372-a567-0e02b2c3d479';
```

```
Result:
CORRELATION_ID  : f47ac10b-58cc-4372-a567-0e02b2c3d479
BUSINESS_KEY    : PO-2026-00421
INTEGRATION_ID  : PO_TO_ERP_SYNC
OVERALL_STATUS  : FAILED
INITIATED_AT    : 2026-05-15 10:00:00.005
COMPLETED_AT    : 2026-05-15 10:01:01.400
TOTAL_STAGES    : 4
COMPLETED_STAGES: 1
FAILED_STAGE_ID : STAGE_002
OIC_INSTANCE_ID : oic-inst-7e3f9a21

Finding: Pipeline ran 61 seconds before failing at STAGE_002.
         Only 1 of 4 stages completed.
```

---

#### STEP 2 — Get Full Stage Timeline

```sql
-- What did each stage do?
SELECT
    STAGE_ORDER,
    STAGE_ID,
    STAGE_NAME,
    STATUS,
    IS_MANDATORY,
    ATTEMPT_NUMBER,
    STARTED_AT,
    COMPLETED_AT,
    DURATION_MS,
    ERROR_CODE,
    ERROR_MESSAGE,
    PAYLOAD_SNAPSHOT_REF
FROM OIC_STAGE_TRACKING
WHERE CORRELATION_ID = 'f47ac10b-58cc-4372-a567-0e02b2c3d479'
ORDER BY STAGE_ORDER, ATTEMPT_NUMBER;
```

```
┌───────┬──────────┬─────────────────┬───────────┬────────────┬────────────────┐
│ ORDER │ STAGE_ID │ STAGE_NAME      │ STATUS    │ ATTEMPT_#  │ DURATION_MS    │
├───────┼──────────┼─────────────────┼───────────┼────────────┼────────────────┤
│   1   │ STAGE_001│ INBOUND_VALID.. │ COMPLETED │     1      │    1,120       │
│   2   │ STAGE_002│ ENRICHMENT      │ FAILED    │     1      │   60,050       │  ← 60s timeout
│   3   │ STAGE_003│ ERP_SUBMISSION  │ PENDING   │     1      │    null        │  ← never reached
│   4   │ STAGE_004│ NOTIFICATION    │ PENDING   │     1      │    null        │  ← never reached
└───────┴──────────┴─────────────────┴───────────┴────────────┴────────────────┘

STAGE_002 detail:
  ERROR_CODE          : MDM_TIMEOUT
  ERROR_MESSAGE       : Supplier MDM returned 503 after 60s
  PAYLOAD_SNAPSHOT_REF: oci://tracking-payloads/f47ac10b-.../
                        STAGE_002/2026-05-15T100101_payload.json

Finding: STAGE_001 passed in 1.1s. STAGE_002 ran exactly 60s
         (hitting timeout threshold). STAGE_003 and STAGE_004
         were never reached. Payload snapshot captured at failure.
```

---

#### STEP 3 — Read the Full Activity Stream

```sql
-- What exactly happened, event by event?
SELECT
    LOG_TIMESTAMP,
    SEVERITY,
    STAGE_ID,
    ACTION,
    MESSAGE,
    DETAIL,
    STACK_TRACE
FROM OIC_ACTIVITY_LOG
WHERE CORRELATION_ID = 'f47ac10b-58cc-4372-a567-0e02b2c3d479'
ORDER BY LOG_TIMESTAMP;
```

```
TIME            SEV    STAGE      ACTION                  MESSAGE
──────────────  ─────  ─────────  ──────────────────────  ──────────────────────────────────────
10:00:00.055    INFO   —          PIPELINE_STARTED        Pipeline initiated for PO-2026-00421
10:00:00.082    INFO   STAGE_001  VALIDATION_STARTED      Validating inbound PO payload
10:00:01.202    INFO   STAGE_001  VALIDATION_PASSED       Schema and business rules passed
10:00:01.252    INFO   STAGE_002  ENRICHMENT_STARTED      Supplier lookup initiated for SUP-001
10:01:01.375    ERROR  STAGE_002  MDM_CALL_FAILED         Supplier MDM returned 503 after 60s
                                                          DETAIL: {
                                                            "httpStatus": 503,
                                                            "supplierCode": "SUP-001",
                                                            "mdmEndpoint": "/suppliers/lookup",
                                                            "attemptDuration": 60003,
                                                            "mdmHost": "mdm.internal:8080",
                                                            "requestId": "mdm-req-44821"
                                                          }
                                                          STACK_TRACE: oracle.cloud....
                                                            TimeoutException at MDMAdapter...

Finding: The MDM call was made for supplier SUP-001 against
         mdm.internal:8080/suppliers/lookup. It took exactly
         60,003ms and returned 503. The MDM request ID is
         mdm-req-44821 — usable to correlate on the MDM side.
```

---

#### STEP 4 — Retrieve the Failure Payload Snapshot

```
Object Storage URI:
oci://tracking-payloads/f47ac10b-.../STAGE_002/2026-05-15T100101_payload.json
```

```json
{
  "capturedAt":    "2026-05-15T10:01:01.350Z",
  "correlationId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "stageId":       "STAGE_002",
  "stageName":     "ENRICHMENT",
  "payloadType":   "OUTBOUND_REQUEST",
  "content": {
    "mdmRequest": {
      "endpoint":     "http://mdm.internal:8080/suppliers/lookup",
      "method":       "GET",
      "headers": {
        "Accept":          "application/json",
        "X-Correlation-ID":"f47ac10b-58cc-4372-a567-0e02b2c3d479",
        "X-Request-ID":    "mdm-req-44821"
      },
      "queryParams": {
        "supplierCode": "SUP-001",
        "fields":       "name,address,taxId,bankAccount"
      }
    },
    "mdmResponse": {
      "httpStatus":    503,
      "body":          "Service Unavailable",
      "responseTime":  60003
    }
  }
}
```

```
Finding: Exact MDM request is preserved. The request itself
         was well-formed. The 503 is a MDM availability issue,
         not a data or mapping problem. MDM request ID
         mdm-req-44821 can be used to query MDM-side logs.
         Root cause: MDM infrastructure, not OIC integration.
```

---

#### STEP 5 — Pivot to OIC Native Console

Using `oicInstanceId = oic-inst-7e3f9a21` from Step 1:

```
OIC Console → Monitoring → Integrations → Instances
  Search: oic-inst-7e3f9a21

Reveals:
  ├── Full OIC flow execution graph (visual)
  ├── OIC payload inspector at each mapper/invoke step
  ├── OIC native error fault detail (Java exception chain)
  └── OIC retry history (if retry policy was configured)

Cross-reference: OIC shows the MDM invoke step failed with
                 ConnectTimeoutException — consistent with ATP
                 activity log DETAIL showing 503 after 60,003ms.
                 Two independent evidence sources confirm same root cause.
```

---

#### STEP 6 — Check the DLQ (Was Any Tracking Data Lost?)

```sql
-- Were any tracking messages for this pipeline lost in the DLQ?
SELECT
    DLQ_ID,
    MESSAGE_TYPE,
    ORIGINAL_MESSAGE_ID,
    ATTEMPT_COUNT,
    FAILURE_REASON,
    FIRST_FAILED_AT,
    RESOLVED
FROM OIC_TRACKING_DLQ
WHERE CORRELATION_ID = 'f47ac10b-58cc-4372-a567-0e02b2c3d479'
AND   RESOLVED = 'N';
```

```
Result: 0 rows

Finding: No tracking messages were lost for this pipeline run.
         All queue messages were successfully processed by
         the dispatcher. ATP records are complete.
         (If rows existed here, it would explain gaps in the
          stage timeline or missing log entries.)
```

---

#### STEP 7 — Cross-Pipeline Analysis (Is This MDM Failure Widespread?)

```sql
-- Has MDM been failing for other pipelines too?
SELECT
    pr.INTEGRATION_ID,
    pr.BUSINESS_KEY,
    pr.CORRELATION_ID,
    pr.INITIATED_AT,
    st.ERROR_CODE,
    st.ERROR_MESSAGE
FROM   OIC_STAGE_TRACKING  st
JOIN   OIC_PIPELINE_RUNS   pr ON st.CORRELATION_ID = pr.CORRELATION_ID
WHERE  st.STAGE_ID     = 'STAGE_002'
AND    st.STATUS        = 'FAILED'
AND    st.ERROR_CODE    = 'MDM_TIMEOUT'
AND    st.COMPLETED_AT >= SYSTIMESTAMP - INTERVAL '2' HOUR
ORDER  BY st.COMPLETED_AT DESC;
```

```
Result: 14 rows in last 2 hours

INTEGRATION_ID    BUSINESS_KEY    INITIATED_AT              ERROR_CODE
────────────────  ──────────────  ──────────────────────    ──────────
PO_TO_ERP_SYNC    PO-2026-00421   2026-05-15 10:00:00       MDM_TIMEOUT
PO_TO_ERP_SYNC    PO-2026-00418   2026-05-15 09:47:12       MDM_TIMEOUT
PO_TO_ERP_SYNC    PO-2026-00415   2026-05-15 09:31:08       MDM_TIMEOUT
INVOICE_PROCESS   INV-2026-08821  2026-05-15 09:28:44       MDM_TIMEOUT
...

Finding: 14 pipeline runs across 2 integrations have hit
         MDM_TIMEOUT at STAGE_002 in the last 2 hours.
         This is a systemic MDM infrastructure issue,
         not an isolated PO-2026-00421 problem.
         Escalate to MDM infrastructure team immediately.
```

---

### 4.4 Complete Trace Reconstruction — Visual Timeline

```
BUSINESS TIMELINE (real execution time, OIC Runtime)
────────────────────────────────────────────────────────────────────────────
T+00:00  ●── PIPELINE_INIT ─────────────────────────────────────────────────
              corrId generated locally (JS UUID)
              PIPELINE_INIT enqueued → messageId: msg-init-001

T+00:00  ●── STAGE_001: IN_PROGRESS ─────────────────────────────────────
              Enqueued: STAGE_UPDATE(IN_PROGRESS), ACTIVITY_LOG(INFO)

T+01.20  ●── STAGE_001: COMPLETED ──────────────────────────────────────────
              Enqueued: STAGE_UPDATE(COMPLETED), ACTIVITY_LOG(INFO)

T+01.25  ●── STAGE_002: IN_PROGRESS ─────────────────────────────────────
              Enqueued: STAGE_UPDATE(IN_PROGRESS), ACTIVITY_LOG(INFO)
              → MDM REST call initiated

T+61.30  ✖── STAGE_002: FAILED ──────────────────────────────────────────
              MDM 503 timeout after 60s
              Enqueued: STAGE_UPDATE(FAILED + payload)
                        ACTIVITY_LOG(ERROR + detail + stackTrace)
                        PIPELINE_COMPLETE(FAILED)
              → Business integration terminates, returns 500

PERSISTENCE TIMELINE (async, ~30s lag, Dispatcher execution)
────────────────────────────────────────────────────────────────────────────
T+90:00  ◉── DISPATCHER POLL ────────────────────────────────────────────
              Dequeues all tracking-events messages
              Processes in chronological enqueue order:

              SP_INIT_PIPELINE         → OIC_PIPELINE_RUNS created
                                         4 × OIC_STAGE_TRACKING PENDING

              SP_UPDATE_STAGE          → STAGE_001 IN_PROGRESS
              SP_UPDATE_STAGE          → STAGE_001 COMPLETED
              SP_UPDATE_STAGE          → STAGE_002 IN_PROGRESS
              SP_UPDATE_STAGE FAILED   → OCI Object Storage PUT ✅
                                       → ATP STAGE_002 = FAILED ✅
                                       → Email alert sent ✅
              SP_COMPLETE_PIPELINE     → OIC_PIPELINE_RUNS FAILED ✅

              SP_BATCH_WRITE_LOGS      → 6 activity log entries ✅

              ATP NOW FULLY REFLECTS ACTUAL EXECUTION STATE
```

---

## 5. Traceability Strengths & Gaps — Honest Assessment

### ✅ Strengths

```
1. SINGLE PIVOT KEY ACROSS ALL LAYERS
   correlationId is present in every queue message, every ATP record,
   every Object Storage object key, every OCI Logging entry, and every
   email alert. One key to rule them all — no cross-referencing needed
   between different ID schemes.

2. BUSINESS-MEANINGFUL ENTRY POINT
   businessKey (PO number, invoice ID) means operations staff can
   initiate a trace from a support ticket ("customer says PO-2026-00421
   failed") without needing to know any system-level ID first.

3. INDEPENDENT EVIDENCE SOURCES
   ATP (structured) + OCI Console (OIC-native) + OCI Logging (fallback)
   + OCI Object Storage (payload) all hold correlated data independently.
   If one is unavailable, others still provide partial trace.

4. FAILURE PAYLOAD AT EXACT MOMENT OF FAILURE
   The payload snapshot captured in the fault handler contains the exact
   request/response at the moment of failure — not reconstructed later.
   This is the most valuable debugging artifact in complex integration failures.

5. CROSS-PIPELINE BLAST RADIUS ANALYSIS
   Because all integrations share the same tracking schema, a single
   query across OIC_STAGE_TRACKING instantly shows if a downstream
   system failure (like MDM) is affecting multiple pipelines — something
   impossible in per-integration logging approaches.

6. ORDERED ACTIVITY STREAM
   LOG_TIMESTAMP (from enqueue time, not write time) preserves the true
   order of events despite async write lag. WRITTEN_AT shows when the
   dispatcher persisted it — the difference quantifies async lag.
```

### ⚠️ Gaps and Mitigations

```
GAP 1: THE 30–60 SECOND VISIBILITY WINDOW
────────────────────────────────────────────────────────────────────
Problem:  During the async lag window, ATP shows no records for an
          in-flight pipeline. If someone queries immediately after
          trigger, they see nothing — potentially alarming.

Mitigation A: Add an "in-flight" indicator to the PIPELINE_QUERY_API:
  → Query OCI Queue depth alongside ATP
  → If ATP has no record but queue has messages for this corrId,
    return status: "IN_FLIGHT_PENDING_PERSISTENCE"

Mitigation B: Reduce dispatcher schedule to every 10–15 seconds
  for PIPELINE_INIT and STAGE_UPDATE (tracking-events queue)
  Keep 30s for activity logs (lower priority)

Mitigation C: Document the lag clearly in the operations runbook
  so investigators know to wait one dispatcher cycle before
  concluding that "no records exist"

────────────────────────────────────────────────────────────────────
GAP 2: DLQ CREATES PARTIAL TRACE RECORDS
────────────────────────────────────────────────────────────────────
Problem:  If PIPELINE_INIT is in the DLQ (dispatcher couldn't write it),
          all subsequent STAGE_UPDATE messages will fail FK constraint
          (no parent OIC_PIPELINE_RUNS record). The stage trace is lost.

Mitigation A: Dispatcher processes PIPELINE_INIT messages FIRST,
  in strict order, before processing any STAGE_UPDATE for same corrId.
  (Already designed — Phase 1 of dispatcher processes tracking-events
  in enqueue order)

Mitigation B: DLQ contains full RAW_PAYLOAD of every failed message.
  Manual reconciliation procedure in runbook: insert PIPELINE_RUNS
  record from DLQ raw payload, then reprocess remaining DLQ entries.

Mitigation C: DLQ monitoring alert (daily email if RESOLVED='N' entries
  exist older than 1 hour) ensures gaps don't go unnoticed.

────────────────────────────────────────────────────────────────────
GAP 3: OCI QUEUE MESSAGES NOT DIRECTLY HUMAN-BROWSABLE
────────────────────────────────────────────────────────────────────
Problem:  During the async window, messages sitting in the queue
          cannot be queried by correlationId. Queue is FIFO —
          you cannot filter by message content natively.

Mitigation: PIPELINE_QUERY_API includes a "queue depth" indicator
  showing total pending messages. For in-flight investigation,
  the OIC flow instance ID (visible in OIC console immediately)
  is the primary tool — ATP is for post-dispatcher analysis.

────────────────────────────────────────────────────────────────────
GAP 4: CHILD INTEGRATION STAGE ORDERING
────────────────────────────────────────────────────────────────────
Problem:  Child integrations enqueue their stage updates under the
          same correlationId but their STAGE_ORDER numbers may
          overlap with parent stages if not carefully designed.

Mitigation: Stage manifest design convention: parent stages use
  100-series IDs (STAGE_100, STAGE_200), child stages use
  sub-series (STAGE_110, STAGE_120) preserving sort order
  and visual hierarchy in the activity stream.
```

---

## 6. Unique Correlation Information Summary — Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│           CORRELATION KEYS — INVESTIGATOR QUICK REFERENCE                   │
├─────────────────────────┬───────────────────────────────────────────────────┤
│ Starting Point          │ First Query / Action                              │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ Email alert received    │ Copy correlationId from email body                │
│                         │ → PIPELINE_QUERY_API /summary                     │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ Support ticket with     │ SELECT * FROM OIC_PIPELINE_RUNS                   │
│ business reference      │ WHERE BUSINESS_KEY = 'PO-2026-00421'              │
│                         │ → Extract correlationId, proceed as above         │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ OIC Console alert       │ SELECT * FROM OIC_PIPELINE_RUNS                   │
│ (OIC native monitoring) │ WHERE OIC_INSTANCE_ID = 'oic-inst-7e3f9a21'       │
│                         │ → Extract correlationId, proceed                  │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ Know approx fail time   │ SELECT * FROM OIC_PIPELINE_RUNS                   │
│ and integration name    │ WHERE INTEGRATION_ID = 'PO_TO_ERP_SYNC'           │
│                         │ AND   OVERALL_STATUS = 'FAILED'                   │
│                         │ AND   INITIATED_AT BETWEEN t1 AND t2              │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ DLQ entry found         │ Use ORIGINAL_MESSAGE_ID from DLQ                  │
│                         │ Extract correlationId from RAW_PAYLOAD            │
│                         │ → Manual reconciliation procedure                 │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ External system         │ Match X-Correlation-ID header value from          │
│ log has OIC header      │ external system's own logs to correlationId       │
│                         │ → Query OIC_PIPELINE_RUNS on CORRELATION_ID       │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ Suspect systemic issue  │ SELECT * FROM OIC_STAGE_TRACKING                  │
│ (many failures)         │ WHERE STATUS='FAILED' AND ERROR_CODE='MDM_TIMEOUT'│
│                         │ AND COMPLETED_AT >= SYSTIMESTAMP - INTERVAL '2'   │
│                         │   HOUR → blast radius analysis                    │
└─────────────────────────┴───────────────────────────────────────────────────┘

CORRELATION KEY PROPAGATION RULE (never break this):
  Every outbound REST call, queue message, Object Storage object,
  OCI Logging entry, and email alert MUST carry correlationId.
  If it doesn't, the trace chain is broken at that point.
```

---

The traceability architecture is **strong** for all post-dispatcher scenarios and provides **five independent investigation entry points** (correlationId, businessKey, oicInstanceId, error code, timestamp range). The only genuine blind spot is the 30–60 second async window — which is a deliberate and acceptable trade-off, and is fully mitigated by the OIC native console providing immediate visibility during that window via `oicInstanceId`.