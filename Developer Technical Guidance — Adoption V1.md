## Developer Technical Guidance 
— Custom integration pipeline configuration registration
— Custom integration pipeline - stages - status configuration registration (instead of 'hardcoded' status)
— Passing Runtime Business Context into the Pipeline Stage Tracking Interface
— Passing custom json payload (business context specific) in javascript - FLEXIBILITY

---

## Clarifying Questions I Considered But Won't Block On

Before proceeding, here are assumptions I'm making — flag me if any are wrong:

1. **Business context varies per integration** — a PO sync carries different runtime fields than an invoice processor or a customer sync. The interface must accommodate all without being rigid.
2. **Developers own the business logic** — they know what context is meaningful at each stage; the framework should not prescribe what business fields to log, only how to pass them.
3. **The JS helper library** (from the final design) is the primary developer touchpoint — not raw queue message construction.
4. **"Business context"** means any runtime data the developer deems meaningful for tracing, debugging, or audit — field names, values, counts, codes, amounts, system references, etc.

---

## Part I — Interface Ease Evaluation

Before writing the how-to, developers and specialists need to know honestly what they are working with.

### 1.1 Current Interface — What Developers Actually Touch

```
┌─────────────────────────────────────────────────────────────────────────┐
│              CURRENT DEVELOPER TOUCHPOINTS — HONEST INVENTORY           │
│                                                                         │
│  Touchpoint 1: correlationId generation                                 │
│  ─────────────────────────────────────                                  │
│  WHERE:  Integration entry point, once                                  │
│  HOW:    JS Action → generateUUID()                                     │
│  EFFORT: ✅ Trivial — one line, provided by helper library             │
│                                                                         │
│  Touchpoint 2: Stage manifest declaration                               │
│  ────────────────────────────────────────                               │
│  WHERE:  OIC_INTEGRATION_CONFIG table (one-time setup per integration)  │
│  HOW:    SQL INSERT with JSON stage array                               │
│  EFFORT: ⚠️ Low-medium — done once, but requires DB access             │
│           and JSON authoring discipline                                 │
│                                                                         │
│  Touchpoint 3: Enqueue calls at stage boundaries                        │
│  ────────────────────────────────────────────────                       │
│  WHERE:  Before and after each stage scope in OIC flow                  │
│  HOW:    JS helper → REST invoke to OCI Queue                           │
│  EFFORT: ⚠️ Medium — repetitive boilerplate; fault handler             │
│           scoping must be correct                                       │
│                                                                         │
│  Touchpoint 4: Business context population                              │
│  ─────────────────────────────────────────                              │
│  WHERE:  Inside buildStageUpdateMessage() and buildLogMessage() calls   │
│  HOW:    Populate `metadata` (stage) and `detail` (log) JSON fields     │
│  EFFORT: ⚠️⚠️ MEDIUM-HIGH — THIS IS THE GAP.                           │
│           Currently freeform JSON. No structure. No guidance.           │
│           Developers don't know what to put here or how to build it.    │
│                                                                         │
│  Touchpoint 5: Fault handler wiring                                     │
│  ────────────────────────────────────                                   │
│  WHERE:  Every stage scope fault handler                                │
│          Global entry bootstrap fault handler                           │
│  HOW:    OIC scope → fault handler → enqueue FAILED messages            │
│  EFFORT: ⚠️ Medium — must be done correctly for every stage            │
│           Easy to forget or mis-scope                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Interface Ease Scorecard — Honest Assessment

```
┌──────────────────────────────────────────────────────────────────────────┐
│               DEVELOPER INTERFACE EASE SCORECARD                         │
├────────────────────────────────────┬────────────┬────────────────────────┤
│ Aspect                             │ Score      │ Verdict                │
├────────────────────────────────────┼────────────┼────────────────────────┤
│ correlationId generation           │ ✅ Easy    │ One JS call, done      │
├────────────────────────────────────┼────────────┼────────────────────────┤
│ Stage manifest setup               │ ✅ Easy    │ One-time SQL config    │
├────────────────────────────────────┼────────────┼────────────────────────┤
│ Enqueue call structure             │ ✅ Easy    │ JS helpers abstract it │
├────────────────────────────────────┼────────────┼────────────────────────┤
│ Business context population        │ ❌ Hard    │ No structure, no       │
│ (the `detail` / `metadata` fields) │            │ guidance, freeform     │
│                                    │            │ JSON — biggest gap     │
├────────────────────────────────────┼────────────┼────────────────────────┤
│ Fault handler wiring               │ ⚠️ Medium  │ Repetitive, error-     │
│                                    │            │ prone scoping rules    │
├────────────────────────────────────┼────────────┼────────────────────────┤
│ Variable scope management          │ ⚠️ Medium  │ corrId must stay in    │
│ (corrId across OIC scopes)         │            │ global scope — easy    │
│                                    │            │ to get wrong           │
├────────────────────────────────────┼────────────┼────────────────────────┤
│ First integration onboarding       │ ⚠️ Medium  │ Many steps; template   │
│                                    │            │ .iar reduces this      │
├────────────────────────────────────┼────────────┼────────────────────────┤
│ Subsequent integrations            │ ✅ Easy    │ Clone template, fill   │
│ (after first)                      │            │ business logic only    │
└──────────────────────────────────────────────────────────────────────────┘

PRIMARY GAP IDENTIFIED:
  The business context population interface is currently a black box.
  Developers see `detail: {}` and `metadata: {}` and have no guidance
  on structure, naming conventions, what to extract from OIC variables,
  or how to build the JSON inside a JS Action.

  This guidance document fixes that gap completely.
```

### 1.3 What "Business Context" Means Across Integration Types

Before prescribing how to pass it, define what it is:

```
┌───────────────────────────────────────────────────────────────────────┐
│              BUSINESS CONTEXT TAXONOMY                                │
│                                                                       │
│  Category 1: TRANSACTION IDENTIFIERS                                  │
│  ────────────────────────────────────                                 │
│  The IDs that uniquely identify the business transaction.             │
│  These are the most critical — always log these.                      │
│                                                                       │
│  Examples: poNumber, invoiceId, orderId, customerId,                  │
│            employeeId, shipmentRef, batchId, messageRef               │
│                                                                       │
│  Category 2: BUSINESS ENTITY ATTRIBUTES                               │
│  ───────────────────────────────────────                              │
│  Key attribute values of the entities being processed.                │
│  Log the ones that matter for investigation, not all of them.         │
│                                                                       │
│  Examples: supplierCode, supplierName, totalAmount, currency,         │
│            lineItemCount, customerTier, productSKU, quantity          │
│                                                                       │
│  Category 3: PROCESSING RESULTS                                       │
│  ─────────────────────────────────                                    │
│  What the stage produced — counts, codes, references.                 │
│                                                                       │
│  Examples: validationPassedCount, validationFailedCount,              │
│            erpTransactionId, mdmEnrichedFields, recordsProcessed,     │
│            transformationRuleApplied, matchScore                      │
│                                                                       │
│  Category 4: SYSTEM INTERACTION CONTEXT                               │
│  ──────────────────────────────────────                               │
│  What system was called, how it responded.                            │
│                                                                       │
│  Examples: targetSystem, httpStatus, externalRefId, responseTime,     │
│            endpointCalled, retryAttempt, soapFaultCode                │
│                                                                       │
│  Category 5: BUSINESS RULES & DECISIONS                               │
│  ──────────────────────────────────────                               │
│  What decision was made and why.                                      │
│                                                                       │
│  Examples: routingDecision, approvalRequired, thresholdBreached,      │
│            ruleId, conditionEvaluated, splitReason                    │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Part II — The Business Context Object Pattern

### 2.1 Introducing the BusinessContext Object

The fix for the freeform JSON gap is a **standardized BusinessContext object** that every developer populates and carries through their integration. It is structured but extensible — a fixed envelope with a flexible content section.

```javascript
// ============================================================
// BUSINESS CONTEXT OBJECT — Standard Structure
// Developers construct ONE of these per integration entry
// and enrich it at each stage
// ============================================================

var businessContext = {

    // ── TIER 1: Transaction Identifiers ─────────────────────
    // REQUIRED. At least one must be populated.
    // These are the primary keys for cross-referencing.
    transaction: {
        primaryId:    "",   // The main business ID (PO number, invoice ID)
        primaryType:  "",   // Human label ("PO_NUMBER", "INVOICE_ID")
        secondaryId:  "",   // Optional secondary ID (e.g. line item ref)
        sourceSystem: "",   // System that originated the transaction
        targetSystem: ""    // System being written to
    },

    // ── TIER 2: Business Entity Snapshot ────────────────────
    // OPTIONAL. Key attributes of the entity being processed.
    // Developers choose what is meaningful for their domain.
    entity: {
        // free JSON — developer defines fields relevant to their domain
        // e.g. { "supplierCode": "SUP-001", "totalAmount": 15000.00 }
    },

    // ── TIER 3: Stage Result ─────────────────────────────────
    // POPULATED PER STAGE. Updated at each stage completion.
    // Reset/enriched as pipeline progresses.
    stageResult: {
        // free JSON — what this stage produced
        // e.g. { "validationStatus": "PASSED", "rulesApplied": 3 }
    },

    // ── TIER 4: System Interaction ───────────────────────────
    // POPULATED WHEN external call is made.
    // Captures what was called and what came back.
    systemCall: {
        // free JSON — populated on REST/SOAP/DB invoke
        // e.g. { "endpoint": "...", "httpStatus": 200, "responseMs": 245 }
    }
};
```

### 2.2 Updated JS Helper Library — BusinessContext-Aware

The JS helpers are updated to accept a `businessContext` parameter directly. Developers no longer craft raw JSON for `detail` and `metadata`.

```javascript
// ============================================================
// UPDATED OIC GEN 3 TRACKING HELPER LIBRARY — v2.0
// BusinessContext-aware | Embed as JS Action at integration entry
// ============================================================

// ── UTILITY FUNCTIONS ────────────────────────────────────────

function generateUUID() {
    return java.util.UUID.randomUUID().toString();
}

function nowISO() {
    return new java.text.SimpleDateFormat(
        "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"
    ).format(new java.util.Date());
}

// Safely serialize any object to JSON string
// Handles null, undefined, and nested objects
function toJSON(obj) {
    if (obj === null || obj === undefined) return "{}";
    try {
        return JSON.stringify(obj);
    } catch(e) {
        return '{"_serializationError": "' + e.message + '"}';
    }
}

// Deep-merge two objects (for context enrichment at each stage)
function mergeContext(base, updates) {
    var result = JSON.parse(toJSON(base));
    var upd    = JSON.parse(toJSON(updates));
    for (var key in upd) {
        if (upd.hasOwnProperty(key)) {
            if (typeof upd[key] === 'object' && !Array.isArray(upd[key])
                && result[key]) {
                result[key] = mergeContext(result[key], upd[key]);
            } else {
                result[key] = upd[key];
            }
        }
    }
    return result;
}

// ── BUSINESS CONTEXT FACTORY ─────────────────────────────────

// Called ONCE at integration entry to initialise the context object
function initBusinessContext(primaryId, primaryType,
                             sourceSystem, targetSystem, entityFields) {
    return {
        transaction: {
            primaryId:    primaryId    || "",
            primaryType:  primaryType  || "",
            secondaryId:  "",
            sourceSystem: sourceSystem || "",
            targetSystem: targetSystem || ""
        },
        entity:      entityFields || {},
        stageResult: {},
        systemCall:  {}
    };
}

// Called at each stage to enrich context with new information
// Returns new enriched copy — does NOT mutate the original
function enrichContext(existingContext, stageResultFields,
                       systemCallFields, additionalEntityFields) {
    var enriched = JSON.parse(toJSON(existingContext));
    if (stageResultFields)      enriched.stageResult  = stageResultFields;
    if (systemCallFields)       enriched.systemCall   = systemCallFields;
    if (additionalEntityFields) {
        enriched.entity = mergeContext(
            enriched.entity, additionalEntityFields);
    }
    return enriched;
}

// ── QUEUE MESSAGE BUILDERS ────────────────────────────────────

function buildInitMessage(corrId, integrationId, businessKey,
                          initiatedBy, oicInstanceId,
                          stageManifest, businessContext) {
    return JSON.stringify({
        messageId:     generateUUID(),
        messageType:   "PIPELINE_INIT",
        schemaVersion: "1.0",
        correlationId: corrId,
        integrationId: integrationId,
        enqueuedAt:    nowISO(),
        payload: {
            businessKey:     businessKey,
            initiatedBy:     initiatedBy,
            oicInstanceId:   oicInstanceId,
            stageManifest:   stageManifest,
            businessContext: businessContext  // ← attached at init
        }
    });
}

function buildStageUpdateMessage(corrId, integrationId,
                                 stageId, newStatus,
                                 errorCode, errorMessage,
                                 capturePayload, payloadContent,
                                 businessContext) {
    return JSON.stringify({
        messageId:     generateUUID(),
        messageType:   "STAGE_UPDATE",
        schemaVersion: "1.0",
        correlationId: corrId,
        integrationId: integrationId,
        enqueuedAt:    nowISO(),
        payload: {
            stageId:         stageId,
            newStatus:       newStatus,
            errorCode:       errorCode     || null,
            errorMessage:    errorMessage  || null,
            capturePayload:  capturePayload || false,
            payloadContent:  capturePayload ? payloadContent : null,
            metadata:        businessContext  // ← stage-level context
        }
    });
}

function buildLogMessage(corrId, integrationId,
                         stageId, trackingId,
                         severity, category,
                         actor, action, message,
                         businessContext, stackTrace) {
    return JSON.stringify({
        messageId:     generateUUID(),
        messageType:   "ACTIVITY_LOG",
        schemaVersion: "1.0",
        correlationId: corrId,
        integrationId: integrationId,
        enqueuedAt:    nowISO(),
        payload: {
            logId:       generateUUID(),
            trackingId:  trackingId || null,
            stageId:     stageId    || null,
            timestamp:   nowISO(),
            severity:    severity,
            category:    category   || "SYSTEM",
            actor:       actor      || "OIC_INTEGRATION",
            action:      action,
            message:     message,
            detail:      businessContext,  // ← full context as log detail
            stackTrace:  stackTrace || null
        }
    });
}

function buildCompleteMessage(corrId, integrationId,
                              overallStatus, businessContext) {
    return JSON.stringify({
        messageId:     generateUUID(),
        messageType:   "PIPELINE_COMPLETE",
        schemaVersion: "1.0",
        correlationId: corrId,
        integrationId: integrationId,
        enqueuedAt:    nowISO(),
        payload: {
            overallStatus:   overallStatus,
            completedAt:     nowISO(),
            finalContext:    businessContext  // ← final state snapshot
        }
    });
}

// ── ENQUEUE HELPER ────────────────────────────────────────────
// Builds the OCI Queue API request body from a message string
function buildQueueRequestBody(messageJsonString) {
    return JSON.stringify({
        messages: [{
            content: java.util.Base64.getEncoder().encodeToString(
                messageJsonString.getBytes("UTF-8")
            )
        }]
    });
}
```

---

## Part III — How-To Developer Guide

### How-To Structure for Each Development Task

```
┌──────────────────────────────────────────────────────────────────────┐
│  This guide covers 6 development tasks in order:                       │
│                                                                      │
│  Task 1: One-time integration registration (stage manifest)          │
│  Task 2: Integration entry — bootstrap tracking + build context      │
│  Task 3: Stage entry — mark IN_PROGRESS + log with context           │
│  Task 4: Stage success — mark COMPLETED + enrich context             │
│  Task 5: Stage failure — mark FAILED + capture payload + context     │
│  Task 6: Pipeline exit — finalise + audit log                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

### TASK 1 — One-Time Integration Registration

**When:** Before your integration goes live. Done once per integration. Revisit when stage structure changes.

**Where:** ATP database via SQL Developer, SQLcl, or DB Admin tool.

**What to do:** IMPORTANT - INTEGRATION - PIPELINE - STAGES - STATUS STANDARDIZATION

```sql
-- ============================================================
-- STEP 1: Register your integration in OIC_INTEGRATION_CONFIG
-- Replace all values with your integration's specifics
-- ============================================================

INSERT INTO OIC_INTEGRATION_CONFIG (
    INTEGRATION_ID,
    FLOW_DISPLAY_NAME,
    STAGE_MANIFEST,
    LOG_LEVEL,
    PAYLOAD_CAPTURE,
    PAYLOAD_BUCKET,
    RETENTION_DAYS,
    ALERT_ON_FAILURE,
    ALERT_RECIPIENTS,
    ALERT_SUBJECT_PREFIX,
    TRACKING_ENABLED
) VALUES (
    'PO_TO_ERP_SYNC',                          -- must match your OIC integration name exactly
    'Purchase Order to ERP Sync',              -- human-readable display name
    '{
      "stages": [
        {
          "stageId":     "STAGE_001",
          "stageName":   "INBOUND_VALIDATION",
          "stageOrder":  1,
          "isMandatory": "Y",
          "description": "Validate inbound PO payload"
        },
        {
          "stageId":     "STAGE_002",
          "stageName":   "ENRICHMENT",
          "stageOrder":  2,
          "isMandatory": "Y",
          "description": "Enrich with supplier master data from MDM"
        },
        {
          "stageId":     "STAGE_003",
          "stageName":   "ERP_SUBMISSION",
          "stageOrder":  3,
          "isMandatory": "Y",
          "description": "Submit transformed PO to ERP"
        },
        {
          "stageId":     "STAGE_004",
          "stageName":   "NOTIFICATION",
          "stageOrder":  4,
          "isMandatory": "N",
          "description": "Send confirmation notification"
        }
      ]
    }',
    'INFO',                                    -- DEBUG | INFO | WARN | ERROR | AUDIT
    'FAILURE',                                 -- NONE | FAILURE | FULL
    'oic-tracking-payloads',                   -- OCI Object Storage bucket name
    30,                                        -- retain tracking data for 30 days
    'Y',                                       -- send email on failure
    'integration-ops@yourcompany.com,dev-lead@yourcompany.com',
    '[OIC ALERT] PO Sync',
    'Y'                                        -- tracking enabled (circuit breaker)
);
COMMIT;
```

**Stage manifest design rules:**

```
RULE 1: stageId must be unique within your integration.
        Convention: STAGE_001, STAGE_002 ... STAGE_NNN
        For child integration stages: STAGE_110, STAGE_120 (sub-series)
        For child integration stage - STATUS: IMPORTANT

RULE 2: stageOrder drives the visual timeline display.
        Use increments of 10 (10, 20, 30) to allow future stage insertion.

RULE 3: isMandatory = "Y" means pipeline fails if this stage fails.
        isMandatory = "N" means fault handler marks SKIPPED and continues.

RULE 4: Keep stageId stable across versions.
        Changing stageId breaks historical reporting continuity.
        Add new stages; don't rename existing ones.
```

---

### TASK 2 — Integration Entry: Bootstrap Tracking and Build Initial Business Context

**When:** The very first actions in your OIC integration flow, before any business logic.

**Where:** OIC flow — immediately after the trigger (REST, Schedule, File, etc.)

**How to implement — Step by step:**

#### Step 2a: Add the JS Helper Library Action

Add a **JS Action** named `JS_LOAD_HELPERS` as the first action in your flow:

```javascript
// ============================================================
// JS_LOAD_HELPERS — Paste the full helper library here
// (The complete library from Part II Section 2.2)
// This makes all helper functions available in this JS scope.
// Note: In OIC Gen 3, declare your working variables here too.
// ============================================================

// [paste full helper library]

// Initialize working variables (these carry values through the flow)
var corrId       = generateUUID();         // pipeline correlation key
var integId      = "PO_TO_ERP_SYNC";       // must match OIC_INTEGRATION_CONFIG
var trackingOn   = true;                   // circuit breaker flag
var bCtx         = {};                     // business context — built below
```

#### Step 2b: Extract Business Context from Trigger Payload

Add a **JS Action** named `JS_INIT_BUSINESS_CONTEXT`:

```javascript
// ============================================================
// JS_INIT_BUSINESS_CONTEXT
// Extract meaningful values from the trigger payload and build
// the initial BusinessContext object.
//
// HOW TO ACCESS OIC VARIABLES IN A JS ACTION:
//   Input variables to JS Action are declared in OIC's
//   JS Action "Input" tab. Map OIC flow variables to JS
//   input parameter names. Then reference them by name below.
// ============================================================

// --- EXAMPLE: REST-triggered PO integration ---
// JS Action inputs mapped from trigger payload:
//   jsPoNumber      ← $trigger.request.body.poNumber
//   jsSupplierCode  ← $trigger.request.body.supplierCode
//   jsTotalAmount   ← $trigger.request.body.totalAmount
//   jsCurrency      ← $trigger.request.body.currency
//   jsRequestedBy   ← $trigger.request.body.requestedBy
//   jsOicInstanceId ← $trackingVar  (OIC built-in tracking variable)

bCtx = initBusinessContext(
    jsPoNumber,          // primaryId    — the main business identifier
    "PO_NUMBER",         // primaryType  — human label for primaryId type
    "PROCUREMENT_PORTAL",// sourceSystem — where this transaction came from
    "ERP_CLOUD",         // targetSystem — where it is going
    {
        // ── Entity fields: key attributes of the PO ──────────
        // Include what matters for your domain investigation.
        // Don't log everything — log what you'd search for.
        supplierCode:   jsSupplierCode,
        totalAmount:    jsTotalAmount,
        currency:       jsCurrency,
        requestedBy:    jsRequestedBy,
        lineItemCount:  null            // populated later after parsing
    }
);

// Build and enqueue PIPELINE_INIT message
// The stageManifest must match what is in OIC_INTEGRATION_CONFIG.
// It is re-sent here so the dispatcher can initialise stage records.
var stageManifest = {
    stages: [
        { stageId:"STAGE_001", stageName:"INBOUND_VALIDATION",
          stageOrder:1, isMandatory:"Y" },
        { stageId:"STAGE_002", stageName:"ENRICHMENT",
          stageOrder:2, isMandatory:"Y" },
        { stageId:"STAGE_003", stageName:"ERP_SUBMISSION",
          stageOrder:3, isMandatory:"Y" },
        { stageId:"STAGE_004", stageName:"NOTIFICATION",
          stageOrder:4, isMandatory:"N" }
    ]
};

var initMsg = buildInitMessage(
    corrId,
    integId,
    jsPoNumber,           // businessKey — used for human search
    "REST_TRIGGER",       // how this pipeline was initiated
    jsOicInstanceId,      // OIC native instance ID — bridge to OIC console
    stageManifest,
    bCtx                  // initial business context snapshot
);

// Store serialised context for use in subsequent JS Actions
var bCtxJson    = toJSON(bCtx);
var initMsgBody = buildQueueRequestBody(initMsg);

// Build PIPELINE_STARTED log message
var startLogMsg = buildLogMessage(
    corrId, integId,
    null, null,                        // no stage yet
    "INFO", "SYSTEM",
    "OIC_INTEGRATION",
    "PIPELINE_STARTED",
    "Pipeline initiated for PO: " + jsPoNumber,
    bCtx,
    null
);
var startLogBody = buildQueueRequestBody(startLogMsg);
```

> **OIC Gen 3 JS Action — passing variables out:** Declare output parameters in the JS Action "Output" tab. Map JS variables (`corrId`, `bCtxJson`, `initMsgBody`, `startLogBody`, `trackingOn`) to OIC flow variables so they are accessible throughout the flow.

#### Step 2c: Map JS Outputs to OIC Flow Variables

In OIC Gen 3, after the JS Action, add an **Assign action** to promote JS output values to integration-scope variables:

```
OIC Assign Action: ASSIGN_TRACKING_VARS
  var_corrId        ← JS_INIT_BUSINESS_CONTEXT.corrId
  var_integId       ← JS_INIT_BUSINESS_CONTEXT.integId
  var_bCtxJson      ← JS_INIT_BUSINESS_CONTEXT.bCtxJson
  var_trackingOn    ← JS_INIT_BUSINESS_CONTEXT.trackingOn
```

> **Critical scoping rule:** `var_corrId` and `var_bCtxJson` MUST be declared at the **top-level integration scope** (not inside any Stage Scope). Every nested scope and fault handler inherits top-level variables. If declared inside a scope, they become invisible to peer and parent scopes.

#### Step 2d: Map OIC Business Identifier Fields

```
OIC Business Identifiers (Integration Settings → Tracking):
  Tracking Field 1: var_corrId      → label: "Correlation ID"
  Tracking Field 2: var_businessKey → label: "Business Key"

This makes corrId and businessKey visible in the OIC native
monitoring console — linking your custom tracking to OIC's
built-in observability with zero extra effort.
```

#### Step 2e: Enqueue PIPELINE_INIT and PIPELINE_STARTED log

Add two **REST Invoke actions** (fire-and-forget):

```
REST Invoke 1: ENQUEUE_PIPELINE_INIT
  Connection:  OCI_QUEUE_CONNECTION
  Method:      POST
  Path:        /20210924/queues/{oic-tracking-events-queue-id}/messages
  Body:        var_initMsgBody
  (Do not map response — fire-and-forget)

REST Invoke 2: ENQUEUE_PIPELINE_STARTED_LOG
  Connection:  OCI_QUEUE_CONNECTION
  Method:      POST
  Path:        /20210924/queues/{oic-activity-logs-queue-id}/messages
  Body:        var_startLogBody
  (Do not map response — fire-and-forget)
```

#### Step 2f: Entry Bootstrap Fault Handler

Wrap Steps 2a–2e in a **Scope action** named `SCOPE_TRACKING_BOOTSTRAP`. Add a fault handler to this scope:

```
Fault Handler: FH_TRACKING_BOOTSTRAP
  ├── [OCI Logging REST call]
  │     Message: "Tracker queue unavailable for corrId: "
  │              + var_corrId + " | Fault: " + $fault.message
  │     (This is the OCI Logging fallback — always available)
  │
  ├── [Assign]
  │     var_trackingOn ← false     ← circuit breaker engages
  │
  └── [DO NOT re-throw]
        Business flow continues regardless of tracking failure
```

---

### TASK 3 — Stage Entry: Mark IN_PROGRESS and Log with Context

**When:** At the start of each stage scope, before any business logic.

**Where:** Inside each OIC Scope action that represents a pipeline stage.

**How to implement:**

#### Step 3a: Add JS Action at Stage Entry

```javascript
// ============================================================
// JS_STAGE_001_START — placed at top of SCOPE_STAGE_001
// Pattern is identical for all stages — only IDs change
// ============================================================

// Restore business context from OIC flow variable
// (JS Actions don't share state — must deserialise each time)
var bCtx = JSON.parse(jsVarBCtxJson);    // mapped from var_bCtxJson

var stageId   = "STAGE_001";
var stageName = "INBOUND_VALIDATION";

// Build STAGE_UPDATE IN_PROGRESS message
var stageStartMsg = buildStageUpdateMessage(
    jsVarCorrId,     // mapped from var_corrId
    jsVarIntegId,    // mapped from var_integId
    stageId,
    "IN_PROGRESS",
    null, null,      // no error at stage start
    false, null,     // no payload capture at start
    bCtx             // current context snapshot
);

// Build ACTIVITY_LOG INFO message
var stageStartLog = buildLogMessage(
    jsVarCorrId,
    jsVarIntegId,
    stageId,
    null,            // trackingId — dispatcher will resolve from stageId
    "INFO",
    "SYSTEM",
    "OIC_INTEGRATION",
    "STAGE_STARTED",
    stageName + " started",
    bCtx,            // full context so log entry is self-contained
    null
);

var stageStartMsgBody = buildQueueRequestBody(stageStartMsg);
var stageStartLogBody = buildQueueRequestBody(stageStartLog);
```

#### Step 3b: Enqueue Both Messages

```
REST Invoke: ENQUEUE_STAGE_001_START
  Body: var_stageStartMsgBody → oic-tracking-events queue

REST Invoke: ENQUEUE_STAGE_001_START_LOG
  Body: var_stageStartLogBody → oic-activity-logs queue
```

---

### TASK 4 — Stage Success: Enrich Context and Mark COMPLETED

**When:** Immediately after the stage's business logic succeeds (before the scope closes).

**The key pattern here:** enrich `bCtx` with what this stage produced, then pass the enriched context forward to the next stage.

```javascript
// ============================================================
// JS_STAGE_001_SUCCESS — runs after validation logic succeeds
// ============================================================

var bCtx = JSON.parse(jsVarBCtxJson);    // restore current context

// ── ENRICH: Add what this stage discovered/produced ─────────
// This is the developer's primary job at each success point.
// Fields here depend entirely on your business domain.
var stageResult = {
    validationStatus:     "PASSED",
    schemaVersion:        "v2.1",
    businessRulesApplied: 3,
    lineItemCount:        jsLineItemCount,   // extracted during validation
    poType:               jsPoType           // STANDARD | BLANKET | EMERGENCY
};

// Enrich context: stageResult updated, entity enriched with new fields
var enrichedCtx = enrichContext(
    bCtx,
    stageResult,               // what this stage produced
    null,                      // no external system called in validation
    { lineItemCount: jsLineItemCount,  // add to entity section too
      poType:        jsPoType }        // now known, update entity snapshot
);

// Build STAGE_UPDATE COMPLETED with enriched context
var stageCompleteMsg = buildStageUpdateMessage(
    jsVarCorrId, jsVarIntegId,
    "STAGE_001", "COMPLETED",
    null, null, false, null,
    enrichedCtx
);

// Build ACTIVITY_LOG INFO — use enriched context
var stageCompleteLog = buildLogMessage(
    jsVarCorrId, jsVarIntegId,
    "STAGE_001", null,
    "INFO", "BUSINESS",
    "OIC_INTEGRATION",
    "VALIDATION_PASSED",
    "Inbound PO validated: " + jsLineItemCount + " line items, type: " + jsPoType,
    enrichedCtx,
    null
);

var stageCompleteMsgBody = buildQueueRequestBody(stageCompleteMsg);
var stageCompleteLogBody = buildQueueRequestBody(stageCompleteLog);

// ── CRITICAL: Persist enriched context back to OIC variable ─
// This carries the enriched context to subsequent stages
var bCtxJsonUpdated = toJSON(enrichedCtx);
```

```
Assign Action after JS_STAGE_001_SUCCESS:
  var_bCtxJson ← JS_STAGE_001_SUCCESS.bCtxJsonUpdated
```

> **The context enrichment chain:** Each stage enriches `bCtx` and writes the updated JSON back to `var_bCtxJson`. The next stage picks it up, adds its own fields, and passes it forward. By pipeline exit, `bCtx` contains a cumulative snapshot of everything meaningful that happened across all stages.

**Domain-specific enrichment examples for other stage types:**

```javascript
// ── ENRICHMENT STAGE (STAGE_002) — after MDM call succeeds ──
var stageResult = {
    enrichmentSource:    "MDM_V3",
    fieldsEnriched:      ["supplierName","taxId","bankAccount","address"],
    mdmResponseMs:       245,
    supplierStatus:      "ACTIVE"
};

var systemCall = {
    endpoint:    "http://mdm.internal:8080/suppliers/lookup",
    httpStatus:  200,
    responseMs:  245,
    mdmRequestId:"mdm-req-44820"
};

var additionalEntity = {
    supplierName:   jsMdmSupplierName,
    supplierStatus: jsMdmSupplierStatus,
    taxId:          jsMdmTaxId
};

var enrichedCtx = enrichContext(bCtx, stageResult,
                                systemCall, additionalEntity);

// ── ERP_SUBMISSION STAGE (STAGE_003) — after ERP write succeeds ──
var stageResult = {
    erpTransactionId:  jsErpTxnId,
    erpStatus:         "CREATED",
    erpModule:         "AP",
    erpResponseMs:     1240
};

var systemCall = {
    endpoint:    "/fscmRestApi/resources/purchaseOrders",
    httpStatus:  201,
    responseMs:  1240,
    erpTxnId:    jsErpTxnId
};

var enrichedCtx = enrichContext(bCtx, stageResult, systemCall, null);
```

---

### TASK 5 — Stage Failure: Capture Context and Mark FAILED

**When:** Inside a fault handler scope for a mandatory or non-mandatory stage.

**This is the most critical task** — every piece of context captured here is what the investigator will use to diagnose the failure.

```javascript
// ============================================================
// JS_STAGE_002_FAILED — runs inside fault handler of STAGE_002
//
// JS Action inputs mapped from fault context:
//   jsFaultMessage  ← $fault.message
//   jsFaultCode     ← $fault.code
//   jsFaultStack    ← $fault.stackTrace
//   jsMdmStatus     ← response HTTP status (if accessible before fault)
//   jsMdmRequest    ← the request body sent to MDM (captured before invoke)
//   jsMdmResponse   ← the raw response body (if any was returned)
// ============================================================

var bCtx = JSON.parse(jsVarBCtxJson);    // restore accumulated context

// ── Enrich with failure-specific context ─────────────────────
// Document WHAT was being attempted and WHAT went wrong.
var stageResult = {
    failurePoint:    "MDM_REST_INVOKE",
    attemptDuration: jsMdmDurationMs,
    retryAttempt:    1
};

var systemCall = {
    endpoint:       "http://mdm.internal:8080/suppliers/lookup",
    httpStatus:     503,
    responseMs:     jsMdmDurationMs,
    faultCode:      jsFaultCode,
    faultMessage:   jsFaultMessage,
    mdmRequestId:   jsMdmRequestId     // from request header — cross-reference MDM logs
};

var failureCtx = enrichContext(bCtx, stageResult, systemCall, null);

// ── PAYLOAD SNAPSHOT ─────────────────────────────────────────
// Capture the exact request that was sent to MDM.
// Only captured on FAILURE (as per PAYLOAD_CAPTURE = 'FAILURE' config).
// This is what gets stored in OCI Object Storage by the dispatcher.
var failurePayload = {
    captureReason:  "STAGE_FAILED",
    mdmRequest: {
        url:           "http://mdm.internal:8080/suppliers/lookup",
        method:        "GET",
        headers:       { "X-Correlation-ID": jsVarCorrId,
                         "X-Request-ID":     jsMdmRequestId },
        queryParams:   { supplierCode: bCtx.entity.supplierCode }
    },
    mdmResponse: {
        httpStatus:    503,
        body:          jsMdmResponse,
        responseMs:    jsMdmDurationMs
    }
};

// Build STAGE_UPDATE FAILED message (with payload for capture)
var stageFailedMsg = buildStageUpdateMessage(
    jsVarCorrId, jsVarIntegId,
    "STAGE_002", "FAILED",
    "MDM_TIMEOUT",
    jsFaultMessage,
    true,              // capturePayload: true — triggers Object Storage PUT
    failurePayload,    // exact payload at failure moment
    failureCtx
);

// Build ACTIVITY_LOG ERROR message
var stageFailedLog = buildLogMessage(
    jsVarCorrId, jsVarIntegId,
    "STAGE_002", null,
    "ERROR", "BUSINESS",
    "OIC_INTEGRATION",
    "MDM_CALL_FAILED",
    "MDM returned " + jsMdmStatus + " for supplier "
        + bCtx.entity.supplierCode
        + " after " + jsMdmDurationMs + "ms",
    failureCtx,
    jsFaultStack       // full stack trace on ERROR severity
);

// Build PIPELINE_COMPLETE FAILED message
var pipelineFailedMsg = buildCompleteMessage(
    jsVarCorrId, jsVarIntegId,
    "FAILED",
    failureCtx         // final context snapshot at time of failure
);

var stageFailedMsgBody  = buildQueueRequestBody(stageFailedMsg);
var stageFailedLogBody  = buildQueueRequestBody(stageFailedLog);
var pipelineFailedBody  = buildQueueRequestBody(pipelineFailedMsg);
```

**Fault handler structure in OIC (mandatory stage):**

```
[Fault Handler: FH_STAGE_002]
  │
  ├── [JS Action] JS_STAGE_002_FAILED
  │     (builds all three message bodies above)
  │
  ├── [IF var_trackingOn = true]
  │     ├── REST Invoke: ENQUEUE_STAGE002_FAILED
  │     │     Body: stageFailedMsgBody → oic-tracking-events
  │     │
  │     ├── REST Invoke: ENQUEUE_STAGE002_FAILED_LOG
  │     │     Body: stageFailedLogBody → oic-activity-logs
  │     │
  │     └── REST Invoke: ENQUEUE_PIPELINE_FAILED
  │           Body: pipelineFailedBody → oic-tracking-events
  │
  └── [Re-throw fault]
        → propagates to parent scope → pipeline terminates
        → caller receives error response
```

**Non-mandatory stage fault handler (STAGE_004 example):**

```javascript
// JS_STAGE_004_SKIPPED — fault handler for non-mandatory stage
// Build SKIPPED (not FAILED) update + WARN log
var skipMsg = buildStageUpdateMessage(
    jsVarCorrId, jsVarIntegId,
    "STAGE_004", "SKIPPED",       // ← SKIPPED, not FAILED
    "NOTIFICATION_ERROR",
    jsFaultMessage,
    false, null,                  // no payload capture for skipped
    bCtx
);

var skipLog = buildLogMessage(
    jsVarCorrId, jsVarIntegId,
    "STAGE_004", null,
    "WARN",                       // ← WARN, not ERROR
    "SYSTEM",
    "OIC_INTEGRATION",
    "STAGE_SKIPPED",
    "Non-mandatory notification stage skipped: " + jsFaultMessage,
    bCtx, null
);
// DO NOT build PIPELINE_COMPLETE FAILED — pipeline continues
// DO NOT re-throw fault — flow continues to exit
```

---

### TASK 6 — Pipeline Exit: Finalise and Write Audit Log

**When:** The last action in the integration flow, after all stages complete successfully.

```javascript
// ============================================================
// JS_PIPELINE_EXIT — runs at integration exit scope
// bCtx at this point contains the fully enriched context
// accumulated across all stages
// ============================================================

var bCtx = JSON.parse(jsVarBCtxJson);

// Build PIPELINE_COMPLETE COMPLETED message
var pipelineCompleteMsg = buildCompleteMessage(
    jsVarCorrId, jsVarIntegId,
    "COMPLETED",
    bCtx              // full enriched context — records what every stage produced
);

// Build AUDIT log — this is the official end-of-pipeline record
var pipelineCompleteLog = buildLogMessage(
    jsVarCorrId, jsVarIntegId,
    null, null,
    "AUDIT",          // AUDIT severity — permanent record, never filtered
    "BUSINESS",
    "OIC_INTEGRATION",
    "PIPELINE_COMPLETED",
    "PO pipeline completed successfully. ERP Txn: "
        + bCtx.stageResult.erpTransactionId,
    bCtx,             // full accumulated context in the audit record
    null
);

var pipelineCompleteMsgBody = buildQueueRequestBody(pipelineCompleteMsg);
var pipelineCompleteLogBody = buildQueueRequestBody(pipelineCompleteLog);
```

---

## Part IV — Complete Context Flow Through All Stages

This diagram shows exactly how `bCtx` evolves from entry to exit:

```
PIPELINE ENTRY
──────────────────────────────────────────────────────────────────
bCtx = {
  transaction: { primaryId:"PO-2026-00421", primaryType:"PO_NUMBER",
                 sourceSystem:"PROCUREMENT_PORTAL",
                 targetSystem:"ERP_CLOUD" },
  entity:      { supplierCode:"SUP-001", totalAmount:15000.00,
                 currency:"CAD", requestedBy:"john.doe",
                 lineItemCount:null, poType:null },
  stageResult: {},
  systemCall:  {}
}

↓ STAGE_001: INBOUND_VALIDATION (success)
──────────────────────────────────────────────────────────────────
bCtx = {
  transaction: { primaryId:"PO-2026-00421", ... },
  entity:      { supplierCode:"SUP-001", totalAmount:15000.00,
                 currency:"CAD", requestedBy:"john.doe",
                 lineItemCount:12,           ← NEW: discovered during parse
                 poType:"STANDARD" },        ← NEW: determined by rule
  stageResult: { validationStatus:"PASSED",
                 businessRulesApplied:3,
                 lineItemCount:12, poType:"STANDARD" },
  systemCall:  {}
}

↓ STAGE_002: ENRICHMENT (success)
──────────────────────────────────────────────────────────────────
bCtx = {
  transaction: { primaryId:"PO-2026-00421", ... },
  entity:      { supplierCode:"SUP-001", totalAmount:15000.00,
                 currency:"CAD", requestedBy:"john.doe",
                 lineItemCount:12, poType:"STANDARD",
                 supplierName:"Acme Corp Ltd",  ← NEW: from MDM
                 supplierStatus:"ACTIVE",        ← NEW: from MDM
                 taxId:"CA-TX-88821" },          ← NEW: from MDM
  stageResult: { enrichmentSource:"MDM_V3",
                 fieldsEnriched:["supplierName","taxId","bankAccount"],
                 supplierStatus:"ACTIVE" },
  systemCall:  { endpoint:"/suppliers/lookup", httpStatus:200,
                 responseMs:245, mdmRequestId:"mdm-req-44820" }
}

↓ STAGE_003: ERP_SUBMISSION (success)
──────────────────────────────────────────────────────────────────
bCtx = {
  transaction: { primaryId:"PO-2026-00421", ... },
  entity:      { supplierCode:"SUP-001", totalAmount:15000.00,
                 currency:"CAD", requestedBy:"john.doe",
                 lineItemCount:12, poType:"STANDARD",
                 supplierName:"Acme Corp Ltd",
                 supplierStatus:"ACTIVE", taxId:"CA-TX-88821" },
  stageResult: { erpTransactionId:"ERP-TXN-2026-99341",  ← NEW: from ERP
                 erpStatus:"CREATED", erpModule:"AP",
                 erpResponseMs:1240 },
  systemCall:  { endpoint:"/purchaseOrders", httpStatus:201,
                 responseMs:1240,
                 erpTxnId:"ERP-TXN-2026-99341" }
}

↓ PIPELINE EXIT — AUDIT LOG
──────────────────────────────────────────────────────────────────
FINAL bCtx written to PIPELINE_COMPLETE and AUDIT log entry:
  → Complete transaction record: PO-2026-00421 → ERP-TXN-2026-99341
  → Full entity snapshot with all enriched attributes
  → Stage-by-stage result chain
  → Last system interaction (ERP submission)

This final bCtx in the AUDIT log is a self-contained
record of the entire business transaction — queryable without
joining stage tracking records.
```

---

## Part V — Developer Decision Reference

### What to Put in Each Context Field — Decision Guide

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Ask yourself these questions at each stage to decide what to log:      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  transaction.primaryId / primaryType                                    │
│  ──────────────────────────────────                                     │
│  Q: What business ID identifies this transaction to a human?            │
│  A: PO number, invoice ID, order ref, employee ID, batch ID            │
│  ALWAYS populate this — it is the human investigation entry point       │
│                                                                         │
│  entity fields                                                          │
│  ────────────                                                           │
│  Q: If this pipeline fails, what entity data would I search for        │
│     to understand the problem?                                          │
│  A: Supplier code, amount, currency, customer tier, product SKU        │
│  Populate at entry, enrich as each stage discovers new attributes       │
│                                                                         │
│  stageResult fields                                                     │
│  ──────────────────                                                     │
│  Q: What did this stage produce or decide?                              │
│  A: Counts (records processed), codes (ERP txn ID), decisions          │
│     (routing choice, approval required), status (ACTIVE, PENDING)      │
│  Populate at stage SUCCESS and stage FAILED                             │
│                                                                         │
│  systemCall fields                                                      │
│  ─────────────────                                                      │
│  Q: What external system did I call and what did it return?             │
│  A: Endpoint, HTTP status, response time, external reference ID        │
│  Populate whenever a REST/SOAP/DB call is made within a stage          │
│  ALWAYS include the external system's own request/reference ID         │
│  (e.g. mdmRequestId) — this enables cross-system correlation           │
│                                                                         │
│  WHAT NOT TO LOG:                                                       │
│  ─────────────────                                                      │
│  ✗ Passwords, API keys, PII (name, DOB, SIN/SSN)                       │
│  ✗ Full response bodies (use payload snapshot for that)                 │
│  ✗ Binary data, base64 blobs                                            │
│  ✗ Fields with no investigation value ("processingFlag: true")         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Common Mistakes and How to Avoid Them

```
┌──────────────────────────────────────────────────────────────────────────┐
│  MISTAKE 1: Declaring var_corrId inside a stage scope                    │
│  ──────────────────────────────────────────────────────                  │
│  Problem:   OIC scope variables are not visible outside that scope.      │
│             Fault handlers in child scopes cannot see corrId.            │
│  Fix:       Declare ALL tracking variables at the top-level global       │
│             integration scope. Assign once, reference everywhere.        │
│                                                                          │
│  MISTAKE 2: Not updating var_bCtxJson after enrichment                   │
│  ────────────────────────────────────────────────────                    │
│  Problem:   Each JS Action starts fresh. If you don't write              │
│             bCtxJsonUpdated back to var_bCtxJson via Assign Action,      │
│             the next stage starts with the original unenriched context.  │
│  Fix:       Always follow JS_STAGE_N_SUCCESS with:                       │
│             Assign: var_bCtxJson ← JS output bCtxJsonUpdated            │
│                                                                          │
│  MISTAKE 3: Using OIC mapper instead of JS for context building          │
│  ───────────────────────────────────────────────────────────             │
│  Problem:   OIC mapper cannot build dynamic nested JSON objects          │
│             from multiple source variables. The result is fragile.       │
│  Fix:       Always build context objects in JS Actions using the         │
│             helper library. OIC mapper is for business payload           │
│             transformation only, not tracking payload construction.      │
│                                                                          │
│  MISTAKE 4: Logging too much in stageResult                              │
│  ─────────────────────────────────────────                               │
│  Problem:   Logging full response bodies in stageResult causes           │
│             ATP CLOB size issues and makes logs unreadable.              │
│  Fix:       Log field names, counts, codes, statuses, and IDs.          │
│             Log full payloads only via the payload capture mechanism     │
│             (capturePayload:true on FAILED stage updates).               │
│                                                                          │
│  MISTAKE 5: Forgetting PIPELINE_COMPLETE in fault handlers               │
│  ─────────────────────────────────────────────────────────               │
│  Problem:   If PIPELINE_COMPLETE is not enqueued in fault handlers,      │
│             OIC_PIPELINE_RUNS stays in IN_PROGRESS forever.             │
│  Fix:       Every mandatory stage fault handler that re-throws must      │
│             enqueue PIPELINE_COMPLETE with status FAILED before          │
│             the re-throw.                                                │
│                                                                          │
│  MISTAKE 6: Not checking var_trackingOn before enqueue calls             │
│  ─────────────────────────────────────────────────────────               │
│  Problem:   If bootstrap fault handler set trackingOn=false but         │
│             subsequent enqueue calls still fire, they will also fail,    │
│             generating noisy errors in OCI Logging.                      │
│  Fix:       Wrap every enqueue call in:                                  │
│             [IF var_trackingOn = true] → REST Invoke enqueue             │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Part VI — Updated Interface Ease Assessment

With the BusinessContext pattern and updated helpers:

```
┌──────────────────────────────────────────────────────────────────────────┐
│          UPDATED DEVELOPER INTERFACE EASE SCORECARD                      │
├───────────────────────────────────────┬──────────┬───────────────────────┤
│ Aspect                                │ Before   │ After                 │
├───────────────────────────────────────┼──────────┼───────────────────────┤
│ correlationId generation              │ ✅ Easy  │ ✅ Easy (unchanged)   │
├───────────────────────────────────────┼──────────┼───────────────────────┤
│ Business context population           │ ❌ Hard  │ ✅ Easy               │
│                                       │ (no      │ initBusinessContext()  │
│                                       │ guidance)│ + enrichContext()      │
│                                       │          │ make it structured     │
├───────────────────────────────────────┼──────────┼───────────────────────┤
│ Knowing what to log at each stage     │ ❌ None  │ ✅ Decision guide      │
│                                       │          │ + taxonomy provided    │
├───────────────────────────────────────┼──────────┼───────────────────────┤
│ Context accumulation across stages    │ ❌ No    │ ✅ enrichContext()     │
│                                       │ pattern  │ pattern + var write-  │
│                                       │          │ back rule             │
├───────────────────────────────────────┼──────────┼───────────────────────┤
│ Fault handler implementation          │ ⚠️ Medium│ ✅ Easier — helper    │
│                                       │          │ builds all 3 messages  │
│                                       │          │ in one JS call        │
├───────────────────────────────────────┼──────────┼───────────────────────┤
│ First integration onboarding          │ ⚠️ Medium│ ⚠️ Medium (unchanged  │
│                                       │          │ — inherent complexity) │
├───────────────────────────────────────┼──────────┼───────────────────────┤
│ Subsequent integrations               │ ✅ Easy  │ ✅ Easy — context      │
│                                       │          │ pattern is copy-paste  │
│                                       │          │ with field changes     │
├───────────────────────────────────────┼──────────┼───────────────────────┤
│ OVERALL                               │ ⚠️Medium │ ✅ EASY               │
│                                       │          │ Primary barrier        │
│                                       │          │ removed               │
└───────────────────────────────────────┴──────────┴───────────────────────┘
```

The business context population barrier — which was the largest friction point for developers — is now fully addressed by the `initBusinessContext()` / `enrichContext()` pattern, the four-tier taxonomy, and the decision guide. The interface is now genuinely adoptable by any OIC developer who can read a JS Action.