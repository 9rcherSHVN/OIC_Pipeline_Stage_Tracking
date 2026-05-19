OIC Gen 3 Pipeline Tracking — Refined Design & Feasibility Assessment

---

## Clarification Answers → Design Impacts

| Your Answer | Design Refinement |
|---|---|
| **OIC Gen 3** | Leverage OIC Gen 3's native REST Platform Services, improved JS engine, and Process Automation for richer capabilities |
| **ATP available** | ATP via DB Adapter (JDBC) is the primary store — ORDS adds the query/dashboard layer |
| **Synchronous child flows** | `correlationId` propagation is simpler — pass via HTTP header on every outbound invoke, extract at child entry |
| **Failure payload capture only** | Conditional payload snapshot to OCI Object Storage triggered only on `FAILED` status — keeps storage lean |
| **Single tenant** | Remove multi-tenant complexity — simplifies schema, config, and correlation |
| **Email alerts** | OIC Email action (natively available in Gen 3) for failure notifications — no external service needed |

---

## Part I — Refined Technical Design

### 1. Revised Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          OIC Gen 3 Instance                                 │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  Business Integration(s)                            │   │
│  │                                                                     │   │
│  │  [ENTRY]──▶[STAGE 1]──▶[STAGE 2]──▶[STAGE N]──▶[EXIT]             │   │
│  │      │         │            │            │          │               │   │
│  │      ▼         ▼            ▼            ▼          ▼               │   │
│  │  ┌──────────────────────────────────────────────────────────┐       │   │
│  │  │          Tracking API Calls (Synchronous REST)           │       │   │
│  │  │   INIT ──▶ STAGE_UPDATE ──▶ LOG_WRITE ──▶ STAGE_UPDATE  │       │   │
│  │  └─────────────────────┬────────────────────────────────────┘       │   │
│  └────────────────────────┼────────────────────────────────────────────┘   │
│                           │                                                │
│  ┌────────────────────────┼────────────────────────────────────────────┐   │
│  │         Tracking Framework Integrations (OIC Gen 3)                 │   │
│  │                        │                                            │   │
│  │   ┌────────────────────▼──────────────────────────────────────┐     │   │
│  │   │  ① PIPELINE_TRACKER_INIT   (App-Driven REST, sync)        │     │   │
│  │   │  ② STAGE_TRACKER_UPDATE    (App-Driven REST, sync)        │     │   │
│  │   │  ③ ACTIVITY_LOG_WRITE      (App-Driven REST, sync)        │     │   │
│  │   │  ④ PIPELINE_QUERY_API      (App-Driven REST, read-only)   │     │   │
│  │   └───────────────────────────────────────────────────────────┘     │   │
│  └───────────────────────────────────────────────────────────────── ─ ┬─┘  │
└─────────────────────────────────────────────────────────────────────  ┼────┘
                                                                        │
              ┌─────────────────────────────────────────────────────── ─┼──┐
              │              Oracle ATP Database                        │  │
              │                                                         ▼  │
              │  ┌─────────────────┐  ┌──────────────────┐  ┌────────────┐ │
              │  │OIC_PIPELINE_RUNS│  │OIC_STAGE_TRACKING│  │OIC_ACTIVITY│ │
              │  │                 │  │                  │  │   _LOG     │ │
              │  └─────────────────┘  └──────────────────┘  └────────────┘ │
              │                                                            │
              │  ┌───────────────────────────────────────────────────────┐ │
              │  │  Stored Procedures (batch inserts, state validation)  │ │
              │  └───────────────────────────────────────────────────────┘ │
              └────────────────────────────────────────────────────────────┘
                        │                              │
              ┌─────────▼──────────┐      ┌───────────▼─────────┐
              │  OCI Object Storage│      │    OIC Email Action  │
              │  (Failure Payload  │      │  (Failure Alerts)    │
              │   Snapshots)       │      │                      │
              └────────────────────┘      └─────────────────────┘
```

---

### 2. Refined Data Model — Single Tenant, Gen 3 Optimized

#### 2.1 Complete ATP Schema

```sql
-- ============================================================
-- OIC GEN 3 PIPELINE TRACKING SCHEMA
-- Single Tenant | ATP | Failure Payload Capture
-- ============================================================

-- Integration Configuration Registry
CREATE TABLE OIC_INTEGRATION_CONFIG (
    INTEGRATION_ID        VARCHAR2(200)  NOT NULL PRIMARY KEY,
    FLOW_DISPLAY_NAME     VARCHAR2(500),
    STAGE_MANIFEST        CLOB           NOT NULL,  -- JSON
    LOG_LEVEL             VARCHAR2(10)   DEFAULT 'INFO'
                              CHECK (LOG_LEVEL IN ('DEBUG','INFO','WARN','ERROR','AUDIT')),
    PAYLOAD_CAPTURE       VARCHAR2(10)   DEFAULT 'FAILURE'
                              CHECK (PAYLOAD_CAPTURE IN ('NONE','FAILURE','FULL')),
    PAYLOAD_BUCKET        VARCHAR2(200),            -- OCI Object Storage bucket name
    RETENTION_DAYS        NUMBER         DEFAULT 30,
    ALERT_ON_FAILURE      VARCHAR2(1)    DEFAULT 'Y',
    ALERT_RECIPIENTS      VARCHAR2(2000),           -- comma-separated emails
    ALERT_SUBJECT_PREFIX  VARCHAR2(200)  DEFAULT '[OIC ALERT]',
    TRACKING_ENABLED      VARCHAR2(1)    DEFAULT 'Y',  -- circuit breaker flag
    CREATED_AT            TIMESTAMP      DEFAULT SYSTIMESTAMP,
    UPDATED_AT            TIMESTAMP      DEFAULT SYSTIMESTAMP
);

-- Pipeline Run Registry
CREATE TABLE OIC_PIPELINE_RUNS (
    CORRELATION_ID        VARCHAR2(36)   NOT NULL PRIMARY KEY,
    INTEGRATION_ID        VARCHAR2(200)  NOT NULL,
    FLOW_DISPLAY_NAME     VARCHAR2(500),
    BUSINESS_KEY          VARCHAR2(500),
    INITIATED_BY          VARCHAR2(200),
    INITIATED_AT          TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    COMPLETED_AT          TIMESTAMP,
    OVERALL_STATUS        VARCHAR2(20)   DEFAULT 'IN_PROGRESS'
                              CHECK (OVERALL_STATUS IN
                                ('IN_PROGRESS','COMPLETED','FAILED',
                                 'PARTIALLY_COMPLETED','ABORTED')),
    TOTAL_STAGES          NUMBER         DEFAULT 0,
    COMPLETED_STAGES      NUMBER         DEFAULT 0,
    FAILED_STAGE_ID       VARCHAR2(50),
    OIC_INSTANCE_ID       VARCHAR2(500), -- OIC native tracking identifier
    CREATED_AT            TIMESTAMP      DEFAULT SYSTIMESTAMP,
    CONSTRAINT FK_RUN_CONFIG
        FOREIGN KEY (INTEGRATION_ID)
        REFERENCES OIC_INTEGRATION_CONFIG(INTEGRATION_ID)
);

-- Stage Tracking (State Machine)
CREATE TABLE OIC_STAGE_TRACKING (
    TRACKING_ID           VARCHAR2(36)   NOT NULL PRIMARY KEY,
    CORRELATION_ID        VARCHAR2(36)   NOT NULL,
    STAGE_ID              VARCHAR2(50)   NOT NULL,
    STAGE_NAME            VARCHAR2(200),
    STAGE_ORDER           NUMBER,
    STATUS                VARCHAR2(20)   NOT NULL DEFAULT 'PENDING'
                              CHECK (STATUS IN
                                ('PENDING','IN_PROGRESS','COMPLETED',
                                 'FAILED','SKIPPED')),
    IS_MANDATORY          VARCHAR2(1)    DEFAULT 'Y',
    STARTED_AT            TIMESTAMP,
    COMPLETED_AT          TIMESTAMP,
    DURATION_MS           NUMBER,
    ATTEMPT_NUMBER        NUMBER         DEFAULT 1,
    ERROR_CODE            VARCHAR2(100),
    ERROR_MESSAGE         VARCHAR2(4000),
    PAYLOAD_SNAPSHOT_REF  VARCHAR2(1000), -- OCI Object Storage URI (failures only)
    METADATA              CLOB,           -- JSON: extra context per stage
    CREATED_AT            TIMESTAMP      DEFAULT SYSTIMESTAMP,
    CONSTRAINT FK_STAGE_RUN
        FOREIGN KEY (CORRELATION_ID)
        REFERENCES OIC_PIPELINE_RUNS(CORRELATION_ID),
    CONSTRAINT UQ_STAGE_ATTEMPT
        UNIQUE (CORRELATION_ID, STAGE_ID, ATTEMPT_NUMBER)
);

-- Activity Stream Log
CREATE TABLE OIC_ACTIVITY_LOG (
    LOG_ID                VARCHAR2(36)   NOT NULL PRIMARY KEY,
    CORRELATION_ID        VARCHAR2(36)   NOT NULL,
    TRACKING_ID           VARCHAR2(36),
    STAGE_ID              VARCHAR2(50),
    LOG_TIMESTAMP         TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    SEVERITY              VARCHAR2(10)   NOT NULL
                              CHECK (SEVERITY IN
                                ('DEBUG','INFO','WARN','ERROR','AUDIT')),
    CATEGORY              VARCHAR2(20)
                              CHECK (CATEGORY IN
                                ('SYSTEM','BUSINESS','SECURITY','PERFORMANCE')),
    ACTOR                 VARCHAR2(200),
    ACTION                VARCHAR2(200),
    MESSAGE               VARCHAR2(4000),
    DETAIL                CLOB,           -- JSON: structured context
    STACK_TRACE           CLOB,           -- populated on ERROR severity only
    CREATED_AT            TIMESTAMP      DEFAULT SYSTIMESTAMP,
    CONSTRAINT FK_LOG_RUN
        FOREIGN KEY (CORRELATION_ID)
        REFERENCES OIC_PIPELINE_RUNS(CORRELATION_ID)
);

-- ============================================================
-- INDEXES
-- ============================================================
CREATE INDEX IDX_RUN_INTEG_STATUS ON OIC_PIPELINE_RUNS
    (INTEGRATION_ID, OVERALL_STATUS, INITIATED_AT DESC);
CREATE INDEX IDX_RUN_BKEY         ON OIC_PIPELINE_RUNS
    (BUSINESS_KEY, INITIATED_AT DESC);
CREATE INDEX IDX_RUN_STATUS_DATE  ON OIC_PIPELINE_RUNS
    (OVERALL_STATUS, INITIATED_AT DESC);

CREATE INDEX IDX_STG_CORR         ON OIC_STAGE_TRACKING
    (CORRELATION_ID, STAGE_ORDER);
CREATE INDEX IDX_STG_STATUS       ON OIC_STAGE_TRACKING
    (STATUS, CREATED_AT DESC);

CREATE INDEX IDX_LOG_CORR_TIME    ON OIC_ACTIVITY_LOG
    (CORRELATION_ID, LOG_TIMESTAMP);
CREATE INDEX IDX_LOG_SEV_TIME     ON OIC_ACTIVITY_LOG
    (SEVERITY, LOG_TIMESTAMP DESC);
CREATE INDEX IDX_LOG_STAGE        ON OIC_ACTIVITY_LOG
    (STAGE_ID, LOG_TIMESTAMP DESC);

-- ============================================================
-- DATA RETENTION: Partition by month (ATP supports auto-partition)
-- ============================================================
-- Recommended: Add interval partitioning on INITIATED_AT / LOG_TIMESTAMP
-- for automated partition pruning aligned to RETENTION_DAYS config
```

---

#### 2.2 Core Stored Procedures

Batch operations into stored procedures to minimize round-trips from OIC DB Adapter calls.

```sql
-- ============================================================
-- SP: Initialize Pipeline Run + All Stage Records
-- IMPORTANT: All Stage Records - INITIALIZATION.
-- ┌───────────┬──────────────┬─────────────────────┬─────────────┬────────────┬────────────────┐
-- │ **ORDER** │ **STAGE_ID** │** STAGE_NAME**      │ STATUS      │ ATTEMPT_#  │ DURATION_MS    │
-- ├───────────┼──────────────┼─────────────────────┼─────────────┼────────────┼────────────────┤
-- │   **1**   │ **STAGE_00**1│ **INBOUND_VALID..** │ _COMPLETED_ │     1      │    1,120       │
-- │   **2**   │ **STAGE_002**│ **ENRICHMENT**      │ _FAILED_    │     1      │   60,050       │  ← 60s timeout
-- │   **3**   │ **STAGE_003**│ **ERP_SUBMISSION**  │ _PENDING_   │     1      │    null        │  ← never reached
-- │   **4**   │ **STAGE_004**│ **NOTIFICATION **   │ _PENDING_   │     1      │    null        │  ← never reached
-- └───────────┴──────────────┴─────────────────────┴─────────────┴────────────┴────────────────┘
-- ============================================================
CREATE OR REPLACE PROCEDURE OIC_PKG.SP_INIT_PIPELINE (
    p_correlation_id     IN  VARCHAR2,
    p_integration_id     IN  VARCHAR2,
    p_business_key       IN  VARCHAR2,
    p_initiated_by       IN  VARCHAR2,
    p_oic_instance_id    IN  VARCHAR2,
    p_stage_manifest     IN  CLOB,      -- JSON stages array
    p_status             OUT VARCHAR2,
    p_error_msg          OUT VARCHAR2
) AS
    v_total_stages  NUMBER := 0;
    v_flow_name     VARCHAR2(500);
BEGIN
    -- Fetch display name from config
    SELECT FLOW_DISPLAY_NAME INTO v_flow_name
    FROM   OIC_INTEGRATION_CONFIG
    WHERE  INTEGRATION_ID = p_integration_id;

    -- Insert pipeline run record
    INSERT INTO OIC_PIPELINE_RUNS (
        CORRELATION_ID, INTEGRATION_ID, FLOW_DISPLAY_NAME,
        BUSINESS_KEY, INITIATED_BY, OIC_INSTANCE_ID,
        OVERALL_STATUS, INITIATED_AT
    ) VALUES (
        p_correlation_id, p_integration_id, v_flow_name,
        p_business_key, p_initiated_by, p_oic_instance_id,
        'IN_PROGRESS', SYSTIMESTAMP
    );

    -- Parse stage manifest JSON and bulk insert stage records
    -- (Uses Oracle JSON_TABLE for ATP JSON support)
    INSERT INTO OIC_STAGE_TRACKING (
        TRACKING_ID, CORRELATION_ID, STAGE_ID, STAGE_NAME,
        STAGE_ORDER, IS_MANDATORY, STATUS, CREATED_AT
    )
    SELECT
        SYS_GUID(),
        p_correlation_id,
        jt.stage_id,
        jt.stage_name,
        jt.stage_order,
        jt.is_mandatory,
        'PENDING',
        SYSTIMESTAMP
    FROM JSON_TABLE(p_stage_manifest, '$.stages[*]'
        COLUMNS (
            stage_id      VARCHAR2(50)  PATH '$.stageId',
            stage_name    VARCHAR2(200) PATH '$.stageName',
            stage_order   NUMBER        PATH '$.stageOrder',
            is_mandatory  VARCHAR2(1)   PATH '$.isMandatory'
        )
    ) jt;

    SELECT COUNT(*) INTO v_total_stages
    FROM   OIC_STAGE_TRACKING
    WHERE  CORRELATION_ID = p_correlation_id;

    UPDATE OIC_PIPELINE_RUNS
    SET    TOTAL_STAGES = v_total_stages
    WHERE  CORRELATION_ID = p_correlation_id;

    COMMIT;
    p_status    := 'SUCCESS';
    p_error_msg := NULL;

EXCEPTION WHEN OTHERS THEN
    ROLLBACK;
    p_status    := 'ERROR';
    p_error_msg := SQLERRM;
END SP_INIT_PIPELINE;


-- ============================================================
-- SP: Update Stage Status with State Machine Validation
-- IMPORTANT: All Stage Records - UPDATES.
-- ┌───────┬──────────┬─────────────────┬───────────────┬────────────────┬────────────────┐
-- │ ORDER │ STAGE_ID │ STAGE_NAME      │ **STATUS**    │ **ATTEMPT_#**  │ DURATION_MS    │
-- ├───────┼──────────┼─────────────────┼───────────────┼────────────────┼────────────────┤
-- │   1   │ STAGE_001│ INBOUND_VALID.. │ **COMPLETED** │    **1**       │    1,120       │
-- │   2   │ STAGE_002│ ENRICHMENT      │ **FAILED**    │    **1**       │   60,050       │  ← 60s timeout
-- │   3   │ STAGE_003│ ERP_SUBMISSION  │ **PENDING**   │    **1**       │    null        │  ← never reached
-- │   4   │ STAGE_004│ NOTIFICATION    │ **PENDING**   │    **1**       │    null        │  ← never reached
-- └───────┴──────────┴─────────────────┴───────────────┴────────────────┴────────────────┘
-- ============================================================
CREATE OR REPLACE PROCEDURE OIC_PKG.SP_UPDATE_STAGE (
    p_correlation_id     IN  VARCHAR2,
    p_stage_id           IN  VARCHAR2,
    p_new_status         IN  VARCHAR2,
    p_error_code         IN  VARCHAR2,
    p_error_message      IN  VARCHAR2,
    p_payload_ref        IN  VARCHAR2,  -- OCI Object Storage URI or NULL
    p_metadata           IN  CLOB,
    p_status             OUT VARCHAR2,
    p_error_msg          OUT VARCHAR2
) AS
    v_current_status  VARCHAR2(20);
    v_tracking_id     VARCHAR2(36);
    v_attempt         NUMBER;
    v_valid           NUMBER := 0;
BEGIN
    -- Fetch current state
    SELECT STATUS, TRACKING_ID, ATTEMPT_NUMBER
    INTO   v_current_status, v_tracking_id, v_attempt
    FROM   OIC_STAGE_TRACKING
    WHERE  CORRELATION_ID = p_correlation_id
    AND    STAGE_ID        = p_stage_id
    AND    ATTEMPT_NUMBER  = (
               SELECT MAX(ATTEMPT_NUMBER)
               FROM   OIC_STAGE_TRACKING
               WHERE  CORRELATION_ID = p_correlation_id
               AND    STAGE_ID        = p_stage_id
           );

    -- State machine validation
    -- Valid transitions matrix
    SELECT COUNT(*) INTO v_valid FROM DUAL WHERE
        (v_current_status = 'PENDING'      AND p_new_status = 'IN_PROGRESS') OR
        (v_current_status = 'IN_PROGRESS'  AND p_new_status = 'COMPLETED')   OR
        (v_current_status = 'IN_PROGRESS'  AND p_new_status = 'FAILED')      OR
        (v_current_status = 'IN_PROGRESS'  AND p_new_status = 'SKIPPED')     OR
        (v_current_status = 'FAILED'       AND p_new_status = 'IN_PROGRESS') OR -- retry
        (v_current_status = 'PENDING'      AND p_new_status = 'SKIPPED');

    IF v_valid = 0 THEN
        p_status    := 'INVALID_TRANSITION';
        p_error_msg := 'Transition from ' || v_current_status
                       || ' to ' || p_new_status || ' is not permitted';
        RETURN;
    END IF;

    -- On retry: insert new attempt record
    IF v_current_status = 'FAILED' AND p_new_status = 'IN_PROGRESS' THEN
        INSERT INTO OIC_STAGE_TRACKING (
            TRACKING_ID, CORRELATION_ID, STAGE_ID, STAGE_NAME,
            STAGE_ORDER, IS_MANDATORY, STATUS, ATTEMPT_NUMBER, CREATED_AT
        )
        SELECT SYS_GUID(), p_correlation_id, STAGE_ID, STAGE_NAME,
               STAGE_ORDER, IS_MANDATORY, 'IN_PROGRESS', v_attempt + 1, SYSTIMESTAMP
        FROM   OIC_STAGE_TRACKING
        WHERE  TRACKING_ID = v_tracking_id;
    ELSE
        -- Update existing record
        UPDATE OIC_STAGE_TRACKING
        SET    STATUS              = p_new_status,
               STARTED_AT         = CASE WHEN p_new_status = 'IN_PROGRESS'
                                         THEN SYSTIMESTAMP ELSE STARTED_AT END,
               COMPLETED_AT       = CASE WHEN p_new_status IN ('COMPLETED','FAILED','SKIPPED')
                                         THEN SYSTIMESTAMP ELSE NULL END,
               DURATION_MS        = CASE WHEN p_new_status IN ('COMPLETED','FAILED','SKIPPED')
                                         THEN EXTRACT(SECOND FROM
                                              (SYSTIMESTAMP - STARTED_AT)) * 1000
                                         ELSE NULL END,
               ERROR_CODE         = p_error_code,
               ERROR_MESSAGE      = p_error_message,
               PAYLOAD_SNAPSHOT_REF = p_payload_ref,
               METADATA           = p_metadata
        WHERE  TRACKING_ID = v_tracking_id;

        -- Update pipeline-level counters and status
        IF p_new_status = 'COMPLETED' THEN
            UPDATE OIC_PIPELINE_RUNS
            SET    COMPLETED_STAGES = COMPLETED_STAGES + 1
            WHERE  CORRELATION_ID   = p_correlation_id;
        ELSIF p_new_status = 'FAILED' THEN
            UPDATE OIC_PIPELINE_RUNS
            SET    FAILED_STAGE_ID  = p_stage_id,
                   OVERALL_STATUS   = 'FAILED',
                   COMPLETED_AT     = SYSTIMESTAMP
            WHERE  CORRELATION_ID   = p_correlation_id;
        END IF;
    END IF;

    COMMIT;
    p_status    := 'SUCCESS';
    p_error_msg := NULL;

EXCEPTION WHEN OTHERS THEN
    ROLLBACK;
    p_status    := 'ERROR';
    p_error_msg := SQLERRM;
END SP_UPDATE_STAGE;


-- ============================================================
-- SP: Write Activity Log Entry
-- IMPORTANT: INSERTS ONLY
-- ============================================================
CREATE OR REPLACE PROCEDURE OIC_PKG.SP_WRITE_ACTIVITY_LOG (
    p_log_id          IN  VARCHAR2,
    p_correlation_id  IN  VARCHAR2,
    p_tracking_id     IN  VARCHAR2,
    p_stage_id        IN  VARCHAR2,
    p_severity        IN  VARCHAR2,
    p_category        IN  VARCHAR2,
    p_actor           IN  VARCHAR2,
    p_action          IN  VARCHAR2,
    p_message         IN  VARCHAR2,
    p_detail          IN  CLOB,
    p_stack_trace     IN  CLOB,
    p_status          OUT VARCHAR2,
    p_error_msg       OUT VARCHAR2
) AS
BEGIN
    INSERT INTO OIC_ACTIVITY_LOG (
        LOG_ID, CORRELATION_ID, TRACKING_ID, STAGE_ID,
        LOG_TIMESTAMP, SEVERITY, CATEGORY, ACTOR, ACTION,
        MESSAGE, DETAIL, STACK_TRACE, CREATED_AT
    ) VALUES (
        p_log_id, p_correlation_id, p_tracking_id, p_stage_id,
        SYSTIMESTAMP, p_severity, p_category, p_actor, p_action,
        p_message, p_detail, p_stack_trace, SYSTIMESTAMP
    );
    COMMIT;
    p_status    := 'SUCCESS';
    p_error_msg := NULL;

EXCEPTION WHEN OTHERS THEN
    ROLLBACK;
    p_status    := 'ERROR';
    p_error_msg := SQLERRM;
END SP_WRITE_ACTIVITY_LOG;
```

---

### 3. Revised OIC Gen 3 Framework Integration Designs

#### 3.1 PIPELINE_TRACKER_INIT — Full Flow Design

```
REST Trigger: POST /tracking/v1/pipeline/init
Payload:
  {
    "integrationId":   "PO_TO_ERP_SYNC",
    "businessKey":     "PO-2026-00421",
    "initiatedBy":     "SCHEDULER | REST_CLIENT | OIC_SCHEDULE",
    "oicInstanceId":   "{$trackingVar}",   ← mapped from OIC native tracking field
    "metadata":        {}
  }

Flow Steps:
  ┌─[1] LOOKUP CONFIG───────────────────────────────────────┐
  │   DB Adapter SELECT:                                    │
  │   SELECT STAGE_MANIFEST, TRACKING_ENABLED               │
  │   FROM OIC_INTEGRATION_CONFIG                           │
  │   WHERE INTEGRATION_ID = req.integrationId              │
  └─────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─[2] CIRCUIT BREAKER CHECK───────────────────────────────┐
  │   IF TRACKING_ENABLED = 'N':                            │
  │       Return { correlationId: null, status: "BYPASSED" }│
  └─────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─[3] GENERATE CORRELATION ID─────────────────────────────┐
  │   JS Action:                                            │
  │   var corrId = java.util.UUID.randomUUID().toString();  │
  └─────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─[4] CALL SP_INIT_PIPELINE (DB Adapter: Stored Proc)─────┐
  │   IN:  corrId, integrationId, businessKey,              │
  │        initiatedBy, oicInstanceId, stageManifest        │
  │   OUT: p_status, p_error_msg                            │
  └─────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─[5] RETURN RESPONSE─────────────────────────────────────┐
  │   {                                                     │
  │     "correlationId": "{corrId}",                        │
  │     "status": "ACCEPTED",                               │
  │     "initiatedAt": "{timestamp}"                        │
  │   }                                                     │
  └─────────────────────────────────────────────────────────┘
```

---

#### 3.2 STAGE_TRACKER_UPDATE — Full Flow Design

```
REST Trigger: POST /tracking/v1/stage/update
Payload:
  {
    "correlationId":  "uuid",
    "stageId":        "STAGE_002",
    "status":         "FAILED",
    "errorCode":      "MDM_TIMEOUT",
    "errorMessage":   "Supplier MDM lookup timed out after 60s",
    "currentPayload": { ... },   ← only populated on FAILED if capture=FAILURE
    "metadata":       {}
  }

Flow Steps:
  ┌─[1] IF status=FAILED AND currentPayload present───────────────────┐
  │   → OCI Object Storage PUT:                                       │
  │     Bucket: {config.PAYLOAD_BUCKET}                               │
  │     Key: {correlationId}/{stageId}/{timestamp}_payload.json       │
  │     ← Returns: objectUri                                          │
  └───────────────────────────────────────────────────────────────────┘
        │ (objectUri or null)
        ▼
  ┌─[2] CALL SP_UPDATE_STAGE (DB Adapter: Stored Proc)────────────────┐
  │   IN: correlationId, stageId, newStatus, errorCode,               │
  │       errorMessage, objectUri, metadata                           │
  │   OUT: p_status (SUCCESS | INVALID_TRANSITION | ERROR)            │
  └───────────────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─[3] IF status=FAILED AND config.ALERT_ON_FAILURE='Y'──────────────┐
  │   OIC Gen 3 Email Action:                                         │
  │     To:      {config.ALERT_RECIPIENTS}                            │
  │     Subject: {config.ALERT_SUBJECT_PREFIX} {flowName} FAILED      │
  │     Body:    HTML template with correlationId, businessKey,       │
  │              stageId, errorCode, errorMessage, timestamp          │
  └───────────────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─[4] RETURN RESPONSE───────────────────────────────────────────────┐
  │   { "status": "SUCCESS | INVALID_TRANSITION | ERROR",             │
  │     "message": "..." }                                            │
  └───────────────────────────────────────────────────────────────────┘
```

---

#### 3.3 ACTIVITY_LOG_WRITE — Full Flow Design

```
REST Trigger: POST /tracking/v1/log/write
Payload:
  {
    "correlationId":  "uuid",
    "trackingId":     "uuid",
    "stageId":        "STAGE_002",
    "severity":       "ERROR",
    "category":       "BUSINESS",
    "actor":          "OIC_INTEGRATION",
    "action":         "SUPPLIER_LOOKUP_FAILED",
    "message":        "MDM returned 503 for supplier SUP-001",
    "detail":         { "httpStatus": 503, "supplierCode": "SUP-001" },
    "stackTrace":     "..."
  }

Flow Steps:
  ┌─[1] LOG LEVEL FILTER────────────────────────────────────┐
  │   JS: if (severity rank < configured LOG_LEVEL rank)    │
  │         return { status: "FILTERED" }                   │
  │   Rank: DEBUG=1, INFO=2, WARN=3, ERROR=4, AUDIT=5       │
  └─────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─[2] GENERATE LOG ID─────────────────────────────────────┐
  │   JS: var logId = java.util.UUID.randomUUID().toString()│
  └─────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─[3] CALL SP_WRITE_ACTIVITY_LOG (DB Adapter)─────────────┐
  │   IN: all fields from payload                           │
  │   OUT: p_status                                         │
  └─────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─[4] RETURN RESPONSE─────────────────────────────────────┐
  │   { "logId": "...", "status": "SUCCESS | FILTERED" }    │
  └─────────────────────────────────────────────────────────┘
```

---

### 4. Business Integration — Complete Canonical Pattern

This is the exact pattern developers copy and adapt:

```
Business Integration: PO_TO_ERP_SYNC
│
├─[ENTRY SCOPE]────────────────────────────────────────────────────────────
│  │
│  ├── [Assign] var_integrationId   = "PO_TO_ERP_SYNC"
│  ├── [Assign] var_businessKey     = trigger.payload.poNumber
│  ├── [Assign] var_trackingEnabled = true          ← default; overrideable
│  ├── [Assign] var_correlationId   = ""            ← populated by INIT call
│  │
│  ├── [REST Invoke] → PIPELINE_TRACKER_INIT
│  │     Header: Content-Type: application/json
│  │     Body:   { integrationId, businessKey, initiatedBy,
│  │               oicInstanceId: $tracking.instanceId }
│  │     ← Response: { correlationId, status }
│  │
│  ├── [Assign] var_correlationId = initResponse.correlationId
│  │
│  └── [Fault Handler: ENTRY]
│        └── Log to OCI Logging (tracker unavailable — continue anyway)
│            [Assign] var_trackingEnabled = false
│
├─[STAGE_001 SCOPE: INBOUND_VALIDATION]────────────────────────────────────
│  │
│  ├── [REST Invoke] → STAGE_TRACKER_UPDATE
│  │     { correlationId, stageId:"STAGE_001", status:"IN_PROGRESS" }
│  │
│  ├── [REST Invoke] → ACTIVITY_LOG_WRITE
│  │     { severity:"INFO", action:"VALIDATION_STARTED",
│  │       message:"Validating inbound PO payload" }
│  │
│  ├── [Try: Business Logic]
│  │     └── Schema validation, business rule checks
│  │         │
│  │         ├── [SUCCESS PATH]
│  │         │     ├── STAGE_TRACKER_UPDATE { status:"COMPLETED" }
│  │         │     └── ACTIVITY_LOG_WRITE { severity:"INFO",
│  │         │               action:"VALIDATION_PASSED" }
│  │         │
│  │         └── [FAILURE PATH → Fault Handler]
│  │               ├── STAGE_TRACKER_UPDATE
│  │               │     { status:"FAILED", errorCode:"VALIDATION_ERROR",
│  │               │       errorMessage: $fault.message,
│  │               │       currentPayload: $trigger.request }  ← captured
│  │               ├── ACTIVITY_LOG_WRITE
│  │               │     { severity:"ERROR", action:"VALIDATION_FAILED",
│  │               │       detail: { faultCode, faultMessage },
│  │               │       stackTrace: $fault.stackTrace }
│  │               └── [Re-throw fault] → pipeline terminates
│
├─[STAGE_002 SCOPE: ENRICHMENT]────────────────────────────────────────────
│  │  (isMandatory: true)
│  │
│  ├── STAGE_TRACKER_UPDATE { status:"IN_PROGRESS" }
│  ├── ACTIVITY_LOG_WRITE   { INFO: "Enrichment started" }
│  │
│  ├── [Try: Call MDM for supplier data]
│  │     │
│  │     └── [Fault Handler]
│  │           ├── STAGE_TRACKER_UPDATE
│  │           │     { status:"FAILED", errorCode:"MDM_ERROR",
│  │           │       currentPayload: $enrichmentRequest }  ← captured
│  │           ├── ACTIVITY_LOG_WRITE { severity:"ERROR", ... }
│  │           └── [Re-throw] → pipeline terminates
│  │
│  └── [SUCCESS] STAGE_TRACKER_UPDATE { status:"COMPLETED" }
│                ACTIVITY_LOG_WRITE   { INFO: "Enrichment completed" }
│
├─[STAGE_003 SCOPE: ERP_SUBMISSION]────────────────────────────────────────
│  │  (same pattern as STAGE_002)
│
├─[STAGE_004 SCOPE: NOTIFICATION]──────────────────────────────────────────
│  │  (isMandatory: false — swallow failures, mark SKIPPED)
│  │
│  ├── [Try: Send notification]
│  │
│  └── [Fault Handler — non-mandatory]
│        ├── STAGE_TRACKER_UPDATE { status:"SKIPPED" }
│        ├── ACTIVITY_LOG_WRITE   { severity:"WARN",
│        │         action:"NOTIFICATION_SKIPPED",
│        │         message:"Non-mandatory stage skipped due to error" }
│        └── [DO NOT re-throw — continue to exit]
│
└─[EXIT SCOPE]────────────────────────────────────────────────────────────
   │
   ├── [DB Adapter] UPDATE OIC_PIPELINE_RUNS
   │     SET OVERALL_STATUS='COMPLETED', COMPLETED_AT=SYSTIMESTAMP
   │     WHERE CORRELATION_ID = var_correlationId
   │
   └── ACTIVITY_LOG_WRITE { severity:"AUDIT",
             action:"PIPELINE_COMPLETED",
             message:"PO pipeline completed successfully" }
```

---

### 5. correlationId Propagation for Synchronous Child Integrations

Since all flows are synchronous in your context, propagation is straightforward:

```
Parent Integration
│
├── [Assign] var_correlationId = initResponse.correlationId
│
└── [Invoke Child Integration via REST]
      Request Headers:
        X-Correlation-ID: {var_correlationId}
        X-Business-Key:   {var_businessKey}
        X-Integration-ID: {var_integrationId}

Child Integration
│
├── [Entry] Extract from inbound header:
│     var_correlationId = $inbound.headers['X-Correlation-ID']
│
├── [IF correlationId present]
│     → Use it for all tracking calls (no new INIT call)
│     → Child stages appear under SAME pipeline run
│
└── [IF correlationId absent]
      → Call PIPELINE_TRACKER_INIT (child acting as standalone pipeline)
```

> **Design rule:** Child integrations **never call INIT** when invoked by a parent. They inherit the parent's `correlationId`, making all child stage tracking records appear as part of the same pipeline run — giving complete end-to-end visibility in one query.

---

## Part II — Feasibility & Practicality Assessment

This is the most critical part. Let me give you an honest, structured evaluation.

---

### ✅ What Is Genuinely Feasible and Works Well

| Area | Assessment |
|---|---|
| **ATP schema + stored procs** | Fully feasible. ATP handles this volume trivially. Stored procs minimize OIC-to-DB round trips — the right call. |
| **3 dedicated tracker integrations** | Feasible and clean. OIC Gen 3 supports this pattern well. Each has its own retry, fault isolation, and versioning. |
| **OIC Gen 3 DB Adapter (JDBC to ATP)** | Mature, well-supported. Stored procedure invocation is natively supported. |
| **correlationId threading (sync)** | Fully feasible in synchronous topology. Header propagation is reliable and easy to implement. |
| **OCI Object Storage for failure payloads** | Feasible. OIC Gen 3 REST adapter to OCI Object Storage API works reliably. Scope the payload to failure-only keeps it very lean. |
| **OIC native Email action** | Built-in OIC Gen 3 capability. Zero external dependency. Works for operational alerting. |
| **State machine validation in SP** | Feasible and the right place to enforce it — database-level, not OIC-flow-level. |
| **Stage manifest in config table** | Feasible. JSON_TABLE in ATP Oracle 19c+ handles this well. |

---

### ⚠️ Real Challenges to Plan For

#### Challenge 1: Latency Overhead Per Stage
Every stage boundary now has **2–3 synchronous REST calls** (update + log write). For a 4-stage pipeline this is **8–12 additional REST round-trips** within an already synchronous flow.

**Realistic latency estimate per tracker call (OIC REST → ATP via DB Adapter):**

```
PIPELINE_TRACKER_INIT:    ~150–300ms   (SP with JSON_TABLE bulk insert)
STAGE_TRACKER_UPDATE:     ~80–150ms    (SP update + index maintenance)
ACTIVITY_LOG_WRITE:       ~50–100ms    (SP single insert)

Per stage overhead:       ~180–400ms
4-stage pipeline total:  ~700ms–1.6s  added to business flow duration
```

**Mitigation strategies:**
- **Combine** `STAGE_TRACKER_UPDATE` + `ACTIVITY_LOG_WRITE` into a **single stored procedure call** that handles both in one DB round-trip — reducing calls from 3 to 2 per stage boundary
- Use **OIC Gen 3 Parallel Actions** for non-critical log writes where business flow doesn't depend on the log result
- Accept the overhead if pipelines are not latency-sensitive (batch/scheduled integrations tolerate this easily; real-time APIs may not)

#### Challenge 2: Fault Handler Complexity
OIC's fault handler model is **scope-based**, not try/catch per se. Developers must correctly scope each stage in its own `Scope` action and attach fault handlers at that level. If scoping is done incorrectly, `correlationId` variables may not be in scope inside fault handlers.

**Mitigation:**
- The template `.iar` file must enforce the correct scoping pattern
- Variable assignments for `correlationId` must be at the **global integration scope level**, not inside stage scopes

#### Challenge 3: Tracker Initialization Failure Handling
If `PIPELINE_TRACKER_INIT` fails (ATP unavailable, network issue), the business flow has no `correlationId`. Every downstream tracker call will then also fail.

**Mitigation (the circuit breaker):**
```
[INIT Fault Handler]
  ├── Log to OCI Logging: "Tracker unavailable"
  ├── [Assign] var_correlationId = "UNTRACKED-" + timestamp
  ├── [Assign] var_trackingEnabled = false
  └── [Continue business flow normally]

All tracker calls:
  [IF var_trackingEnabled = false] → skip tracker invocation entirely
```
This ensures tracking failures **never kill business flows** — a non-negotiable requirement.

#### Challenge 4: OIC Integration Count Governance
You are adding 3–4 new OIC integrations (the framework layer) to every OIC instance. OIC licensing is partly based on integration count (in some configurations). Verify this against your licensing model. The framework integrations are shared — so it's 3 total, not 3 per business integration.

#### Challenge 5: Developer Discipline
The pattern only works if developers **consistently** implement it. Without tooling enforcement (code reviews, template usage mandates), some integrations will have incomplete tracking.

**Mitigation:**
- Mandatory use of the `.iar` skeleton template
- Integration review checklist that verifies tracking calls are present
- Automated test: a test runner that calls the integration and verifies tracking records appear in ATP

---

### 📊 Overall Feasibility Verdict

```
┌─────────────────────────────────────────────────────────────────┐
│                   FEASIBILITY SCORECARD                         │
├──────────────────────────────────┬──────────┬───────────────────┤
│ Dimension                        │ Score    │ Notes             │
├──────────────────────────────────┼──────────┼───────────────────┤
│ Technical feasibility            │ ✅ HIGH  │ All components    │
│                                  │          │ are OIC-native    │
├──────────────────────────────────┼──────────┼───────────────────┤
│ Implementation complexity        │ ⚠️ MEDIUM│ Non-trivial but   │
│                                  │          │ well-bounded      │
├──────────────────────────────────┼──────────┼───────────────────┤
│ Runtime performance impact       │ ⚠️ MEDIUM│ ~700ms-1.6s added │
│                                  │          │ per pipeline run  │
├──────────────────────────────────┼──────────┼───────────────────┤
│ Maintenance overhead             │ ✅ LOW   │ Centralized in    │
│                                  │          │ 3 integrations    │
├──────────────────────────────────┼──────────┼───────────────────┤
│ Developer adoption ease          │ ✅ HIGH  │ Template + helpers │
│                                  │          │ make it plug-in   │
├──────────────────────────────────┼──────────┼───────────────────┤
│ Observability value delivered    │ ✅ HIGH  │ End-to-end audit  │
│                                  │          │ trail, debugging  │
├──────────────────────────────────┼──────────┼───────────────────┤
│ Resilience (tracking vs business)│ ✅ HIGH  │ Circuit breaker   │
│                                  │          │ fully protects    │
├──────────────────────────────────┼──────────┼───────────────────┤
│ Scalability                      │ ✅ HIGH  │ ATP scales; index │
│                                  │          │ + partitioning    │
├──────────────────────────────────┼──────────┼───────────────────┤
│ OVERALL                          │ ✅ HIGH  │ Recommended to    │
│                                  │          │ proceed           │
└──────────────────────────────────┴──────────┴───────────────────┘
```

---

### 🚀 Recommended Implementation Phasing

```
PHASE 1 — Foundation (Weeks 1–3)
  ├── ATP schema creation (tables, indexes, partitioning)
  ├── Stored procedures (SP_INIT, SP_UPDATE_STAGE, SP_WRITE_LOG)
  ├── OIC DB Adapter connection to ATP
  └── PIPELINE_TRACKER_INIT integration (basic, tested)

PHASE 2 — Core Framework (Weeks 4–6)
  ├── STAGE_TRACKER_UPDATE integration
  │     ├── State machine validation
  │     ├── OCI Object Storage failure payload capture
  │     └── Email alert on FAILED status
  ├── ACTIVITY_LOG_WRITE integration
  └── OIC Integration Config table + initial entries

PHASE 3 — Developer Enablement (Week 7)
  ├── Skeleton .iar template integration (4-stage sample)
  ├── JS helper functions library
  ├── Developer guide (1-page cheat sheet)
  └── Integration review checklist

PHASE 4 — Observability Layer (Week 8)
  ├── ORDS REST endpoints over ATP tracking tables
  ├── PIPELINE_QUERY_API (read-only query integration)
  └── Basic OAC / APEX dashboard over tracking tables

PHASE 5 — Pilot + Hardening (Weeks 9–10)
  ├── Pilot 2–3 real business integrations using framework
  ├── Performance benchmarking (measure actual latency overhead)
  ├── Circuit breaker testing (ATP unavailability scenarios)
  └── Retention job (purge data beyond RETENTION_DAYS)
```

---

### Bottom Line

This design is **technically sound, practically achievable on OIC Gen 3 + ATP, and delivers genuine operational value**. The main risk to manage is **latency overhead for any real-time/low-latency integrations** — which is mitigated by combining SP calls and selectively applying async log writes. For batch, scheduled, and business-event-driven integrations (the majority of OIC use cases), the overhead is entirely acceptable and the observability gain far outweighs the cost.
