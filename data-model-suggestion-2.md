# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Electronic Health Record (EHR) · Created: 2026-05-19

## Philosophy

This model treats the clinical record as an immutable append-only event log. Every action — recording a vital sign, updating a medication, adding a diagnosis, signing a note — is captured as a timestamped event in a single `clinical_event` store. The current state of any patient's record is derived by replaying events in chronological order. Read-optimized materialized views (the "query side" of CQRS) project the event stream into familiar relational shapes for clinical workflows, reporting, and API responses.

This architecture is inspired by event sourcing patterns used in financial systems, airline reservation systems, and increasingly in healthcare (see InfoQ's "Healthy Architectures" reference). The event store becomes the single source of truth, providing a complete, tamper-evident audit trail that satisfies HIPAA requirements by design rather than as an afterthought. Temporal queries ("what was this patient's medication list on March 15?") are trivially answered by replaying events up to that timestamp.

The trade-off is complexity: developers must think in terms of events and projections rather than direct CRUD on tables. Read model staleness (eventual consistency) must be managed. The event store can grow very large for active patients, and schema evolution of event payloads requires careful versioning.

**Best for:** Organizations that require complete audit trails, temporal queries, AI analytics on clinical change patterns, and regulatory environments where proving exactly what happened and when is a core requirement.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is preserved forever; HIPAA compliance is inherent
- (+) Temporal queries are trivial — reconstruct any patient's state at any point in time
- (+) AI/ML analytics on event streams can detect clinical patterns, deterioration signals, and documentation anomalies
- (+) Event replay enables safe schema migrations — rebuild read models from events without data loss
- (-) Higher implementation complexity — developers must design events, projections, and event handlers
- (-) Read models are eventually consistent — clinical workflows may need synchronous projection updates
- (-) Event store grows indefinitely — requires snapshot strategies for high-volume patients
- (-) Debugging requires event replay tooling rather than simple SQL inspection
- (-) More infrastructure: event bus, projection workers, snapshot management

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Read models (materialized views) expose data as FHIR R4 resources via the API layer; events carry FHIR resource references |
| US Core IG | Read model projections include all US Core must-support elements |
| W3C PROV-O | Event metadata (who, when, why, how) aligns with W3C Provenance Ontology for AI-generated content traceability |
| HIPAA Security Rule | The immutable event store IS the audit trail — 45 CFR 164.312(b) compliance is architectural |
| SNOMED CT / LOINC / ICD-10 / RxNorm | Coded values stored in event payloads using standard code systems |
| ISO 13606 / openEHR | The versioned-composition model in openEHR (where each commit creates a new version) is philosophically aligned with event sourcing |
| ATNA (IHE) | Event metadata captures audit-trail node authentication information per IHE ATNA profile |

---

## Event Store (Write Side)

### Clinical Event

```sql
CREATE TABLE clinical_event (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Aggregate identity
    aggregate_type      TEXT NOT NULL,                      -- 'Patient', 'Encounter', 'Condition', 'Observation', etc.
    aggregate_id        UUID NOT NULL,                      -- The entity this event belongs to
    -- Event metadata
    event_type          TEXT NOT NULL,                      -- e.g., 'PatientRegistered', 'VitalSignRecorded', 'MedicationPrescribed'
    event_version       INTEGER NOT NULL,                   -- Monotonically increasing per aggregate
    -- Tenant and actor
    organization_id     UUID NOT NULL,
    patient_id          UUID,                               -- NULL for org-level events
    actor_id            UUID NOT NULL,                      -- User who caused the event
    actor_type          TEXT NOT NULL CHECK (actor_type IN ('practitioner', 'system', 'patient', 'ai-agent')),
    -- Payload
    payload             JSONB NOT NULL,                     -- Full event data (see examples below)
    -- Provenance
    correlation_id      UUID,                               -- Links related events (e.g., encounter-start triggers multiple events)
    causation_id        UUID,                               -- The event that caused this event
    source_system       TEXT,                               -- 'ehr-ui', 'fhir-api', 'hl7-v2-bridge', 'ai-ambient', etc.
    ai_generated        BOOLEAN NOT NULL DEFAULT FALSE,
    ai_model            TEXT,
    ai_confidence       NUMERIC,
    -- Timestamps
    occurred_at         TIMESTAMPTZ NOT NULL,               -- When the clinical event actually happened
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(), -- When it was stored (server time)
    -- IP and session for HIPAA
    ip_address          INET,
    session_id          TEXT,
    -- Ordering guarantee
    global_sequence     BIGSERIAL NOT NULL UNIQUE
) PARTITION BY RANGE (recorded_at);

-- Monthly partitions
CREATE TABLE clinical_event_2026_01 PARTITION OF clinical_event
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Optimistic concurrency: unique per aggregate + version
CREATE UNIQUE INDEX idx_event_aggregate_version
    ON clinical_event(aggregate_type, aggregate_id, event_version);

CREATE INDEX idx_event_aggregate ON clinical_event(aggregate_type, aggregate_id);
CREATE INDEX idx_event_patient ON clinical_event(patient_id);
CREATE INDEX idx_event_org ON clinical_event(organization_id);
CREATE INDEX idx_event_type ON clinical_event(event_type);
CREATE INDEX idx_event_occurred ON clinical_event(occurred_at);
CREATE INDEX idx_event_correlation ON clinical_event(correlation_id);
CREATE INDEX idx_event_sequence ON clinical_event(global_sequence);
```

### Event Payload Examples

```sql
-- PatientRegistered
-- payload: {
--   "mrn": "MRN-2026-00042",
--   "firstName": "Jane",
--   "lastName": "Smith",
--   "dateOfBirth": "1985-03-15",
--   "gender": "female",
--   "birthSex": "F",
--   "race": "2106-3",
--   "ethnicity": "2186-5",
--   "address": {"line1": "123 Main St", "city": "Portland", "state": "OR", "postalCode": "97201"},
--   "phone": "503-555-0142"
-- }

-- VitalSignRecorded
-- payload: {
--   "encounterId": "uuid-of-encounter",
--   "code": "85354-9",
--   "codeSystem": "http://loinc.org",
--   "display": "Blood pressure panel",
--   "components": [
--     {"code": "8480-6", "display": "Systolic BP", "valueQuantity": 128, "unit": "mmHg"},
--     {"code": "8462-4", "display": "Diastolic BP", "valueQuantity": 82, "unit": "mmHg"}
--   ],
--   "effectiveDateTime": "2026-05-19T10:30:00Z"
-- }

-- ConditionDiagnosed
-- payload: {
--   "encounterId": "uuid-of-encounter",
--   "code": "44054006",
--   "codeSystem": "http://snomed.info/sct",
--   "display": "Type 2 diabetes mellitus",
--   "clinicalStatus": "active",
--   "verificationStatus": "confirmed",
--   "category": "encounter-diagnosis",
--   "onsetDateTime": "2026-05-19"
-- }

-- MedicationPrescribed
-- payload: {
--   "encounterId": "uuid-of-encounter",
--   "medicationCode": "860975",
--   "medicationSystem": "http://www.nlm.nih.gov/research/umls/rxnorm",
--   "medicationDisplay": "Metformin 500 MG Oral Tablet",
--   "dosageText": "500mg PO BID",
--   "route": "oral",
--   "frequency": "BID",
--   "refillsAllowed": 3,
--   "reasonCode": "E11.9",
--   "reasonDisplay": "Type 2 diabetes mellitus without complications"
-- }

-- NoteAuthored (AI-assisted)
-- payload: {
--   "encounterId": "uuid-of-encounter",
--   "noteType": "34117-2",
--   "title": "History and Physical",
--   "subjective": "Patient presents with polyuria and polydipsia...",
--   "objective": "VS: BP 128/82, HR 78, T 98.6F...",
--   "assessment": "New diagnosis of type 2 diabetes mellitus...",
--   "plan": "1. Start metformin 500mg BID...",
--   "aiDraftSource": "ambient-encounter-recording",
--   "clinicianEdited": true
-- }
```

### Event Snapshot (Performance Optimization)

```sql
CREATE TABLE event_snapshot (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type      TEXT NOT NULL,
    aggregate_id        UUID NOT NULL,
    snapshot_version    INTEGER NOT NULL,                   -- Event version at snapshot time
    state               JSONB NOT NULL,                    -- Full materialized state at this point
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_type, aggregate_id, snapshot_version)
);

CREATE INDEX idx_snapshot_aggregate ON event_snapshot(aggregate_type, aggregate_id);
```

---

## Read Models (Query Side — Materialized Projections)

### Patient Read Model

```sql
CREATE TABLE patient_view (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    mrn                 TEXT NOT NULL,
    first_name          TEXT NOT NULL,
    last_name           TEXT NOT NULL,
    middle_name         TEXT,
    date_of_birth       DATE NOT NULL,
    date_of_death       DATE,
    gender              TEXT NOT NULL,
    birth_sex           TEXT,
    race                TEXT,
    ethnicity           TEXT,
    preferred_language  TEXT,
    phone_mobile        TEXT,
    phone_home          TEXT,
    email               TEXT,
    address_line1       TEXT,
    city                TEXT,
    state               VARCHAR(2),
    postal_code         VARCHAR(10),
    country             VARCHAR(3) DEFAULT 'USA',
    active              BOOLEAN NOT NULL DEFAULT TRUE,
    -- Projection metadata
    last_event_version  INTEGER NOT NULL,
    last_projected_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, mrn)
);

CREATE INDEX idx_pv_org ON patient_view(organization_id);
CREATE INDEX idx_pv_name ON patient_view(last_name, first_name);
```

### Active Problem List View

```sql
CREATE TABLE problem_list_view (
    id                  UUID PRIMARY KEY,
    patient_id          UUID NOT NULL,
    organization_id     UUID NOT NULL,
    code_system         TEXT NOT NULL,
    code                TEXT NOT NULL,
    display             TEXT NOT NULL,
    clinical_status     TEXT NOT NULL,
    verification_status TEXT,
    category            TEXT NOT NULL,
    severity            TEXT,
    onset_datetime      TIMESTAMPTZ,
    recorded_date       DATE,
    recorded_by         UUID,
    last_event_version  INTEGER NOT NULL,
    last_projected_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plv_patient ON problem_list_view(patient_id);
CREATE INDEX idx_plv_status ON problem_list_view(clinical_status);
CREATE INDEX idx_plv_code ON problem_list_view(code_system, code);
```

### Current Medications View

```sql
CREATE TABLE medication_list_view (
    id                  UUID PRIMARY KEY,
    patient_id          UUID NOT NULL,
    organization_id     UUID NOT NULL,
    medication_code     TEXT NOT NULL,
    medication_system   TEXT NOT NULL,
    medication_display  TEXT NOT NULL,
    status              TEXT NOT NULL,
    dosage_text         TEXT,
    frequency           TEXT,
    route               TEXT,
    prescriber_id       UUID,
    prescribed_date     TIMESTAMPTZ,
    end_date            TIMESTAMPTZ,
    refills_remaining   INTEGER,
    is_controlled       BOOLEAN DEFAULT FALSE,
    last_event_version  INTEGER NOT NULL,
    last_projected_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mlv_patient ON medication_list_view(patient_id);
CREATE INDEX idx_mlv_status ON medication_list_view(status);
```

### Latest Vitals View

```sql
CREATE TABLE latest_vitals_view (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL,
    organization_id     UUID NOT NULL,
    encounter_id        UUID,
    loinc_code          TEXT NOT NULL,
    display             TEXT NOT NULL,
    value_quantity      NUMERIC,
    value_unit          TEXT,
    effective_datetime  TIMESTAMPTZ NOT NULL,
    interpretation      TEXT,
    recorded_by         UUID,
    last_event_version  INTEGER NOT NULL,
    last_projected_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (patient_id, loinc_code)                        -- Only latest value per vital type
);

CREATE INDEX idx_lvv_patient ON latest_vitals_view(patient_id);
```

### Encounter Timeline View

```sql
CREATE TABLE encounter_timeline_view (
    id                  UUID PRIMARY KEY,
    patient_id          UUID NOT NULL,
    organization_id     UUID NOT NULL,
    practitioner_id     UUID,
    practitioner_name   TEXT,
    status              TEXT NOT NULL,
    class_code          TEXT NOT NULL,
    type_display        TEXT,
    reason_display      TEXT,
    period_start        TIMESTAMPTZ NOT NULL,
    period_end          TIMESTAMPTZ,
    -- Denormalized counts for timeline display
    diagnosis_count     INTEGER DEFAULT 0,
    observation_count   INTEGER DEFAULT 0,
    medication_count    INTEGER DEFAULT 0,
    note_count          INTEGER DEFAULT 0,
    last_event_version  INTEGER NOT NULL,
    last_projected_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_etv_patient ON encounter_timeline_view(patient_id);
CREATE INDEX idx_etv_period ON encounter_timeline_view(period_start);
```

### Claims Summary View

```sql
CREATE TABLE claim_summary_view (
    id                  UUID PRIMARY KEY,
    patient_id          UUID NOT NULL,
    organization_id     UUID NOT NULL,
    encounter_id        UUID,
    claim_number        TEXT,
    claim_type          TEXT NOT NULL,
    status              TEXT NOT NULL,
    payer_name          TEXT,
    total_billed        NUMERIC(12,2),
    total_paid          NUMERIC(12,2),
    patient_responsibility NUMERIC(12,2),
    submitted_at        TIMESTAMPTZ,
    adjudicated_at      TIMESTAMPTZ,
    last_event_version  INTEGER NOT NULL,
    last_projected_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_csv_patient ON claim_summary_view(patient_id);
CREATE INDEX idx_csv_status ON claim_summary_view(status);
```

---

## Projection Infrastructure

### Projection Checkpoint (Tracks Projection Progress)

```sql
CREATE TABLE projection_checkpoint (
    projection_name     TEXT PRIMARY KEY,                   -- 'patient_view', 'problem_list_view', etc.
    last_global_sequence BIGINT NOT NULL DEFAULT 0,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status              TEXT NOT NULL DEFAULT 'running' CHECK (status IN ('running', 'paused', 'rebuilding', 'error')),
    error_message       TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Event Subscription (For External Consumers)

```sql
CREATE TABLE event_subscription (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_name     TEXT NOT NULL UNIQUE,               -- 'fhir-subscription-lab-results', 'ai-risk-model', etc.
    event_types         TEXT[] NOT NULL,                    -- Filter: which event types to deliver
    webhook_url         TEXT,
    last_global_sequence BIGINT NOT NULL DEFAULT 0,
    active              BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Temporal Query Examples

### Reconstruct Patient Problem List at a Specific Date

```sql
-- What conditions were active for patient X on 2026-03-15?
WITH condition_events AS (
    SELECT
        aggregate_id,
        event_type,
        payload,
        occurred_at,
        ROW_NUMBER() OVER (
            PARTITION BY aggregate_id
            ORDER BY event_version DESC
        ) AS rn
    FROM clinical_event
    WHERE patient_id = 'patient-uuid-here'
      AND aggregate_type = 'Condition'
      AND occurred_at <= '2026-03-15T23:59:59Z'
)
SELECT
    aggregate_id AS condition_id,
    payload->>'code' AS code,
    payload->>'display' AS display,
    payload->>'clinicalStatus' AS clinical_status,
    occurred_at
FROM condition_events
WHERE rn = 1
  AND payload->>'clinicalStatus' IN ('active', 'recurrence', 'relapse');
```

### AI Analytics: Detect Rapid Medication Changes

```sql
-- Find patients with 3+ medication changes in 7 days (potential adverse reaction pattern)
SELECT
    patient_id,
    COUNT(*) AS med_changes,
    MIN(occurred_at) AS first_change,
    MAX(occurred_at) AS last_change
FROM clinical_event
WHERE event_type IN ('MedicationPrescribed', 'MedicationStopped', 'MedicationDoseChanged')
  AND occurred_at >= now() - INTERVAL '7 days'
GROUP BY patient_id
HAVING COUNT(*) >= 3
ORDER BY med_changes DESC;
```

### Full Encounter Replay

```sql
-- Replay all events for a specific encounter in chronological order
SELECT
    event_type,
    actor_id,
    actor_type,
    ai_generated,
    payload,
    occurred_at,
    source_system
FROM clinical_event
WHERE payload->>'encounterId' = 'encounter-uuid-here'
   OR aggregate_id = 'encounter-uuid-here'
ORDER BY global_sequence;
```

---

## Reference Data (Shared with Read Side)

```sql
CREATE TABLE organization_ref (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    type            TEXT NOT NULL,
    npi             VARCHAR(10),
    active          BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE practitioner_ref (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organization_ref(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    npi             VARCHAR(10),
    specialty       TEXT,
    active          BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization_ref(id),
    practitioner_id UUID REFERENCES practitioner_ref(id),
    email           TEXT NOT NULL UNIQUE,
    password_hash   TEXT NOT NULL,
    role            TEXT NOT NULL,
    mfa_enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write) | 1 | clinical_event (partitioned monthly) |
| Snapshots | 1 | event_snapshot for replay optimization |
| Read Models (Projections) | 6 | patient_view, problem_list_view, medication_list_view, latest_vitals_view, encounter_timeline_view, claim_summary_view |
| Projection Infrastructure | 2 | projection_checkpoint, event_subscription |
| Reference Data | 3 | organization_ref, practitioner_ref, user_account |
| **Total** | **13** | Dramatically fewer tables than normalized model; complexity moves to event handlers and projections |

---

## Key Design Decisions

1. **Single event table as source of truth.** All clinical state changes flow through `clinical_event`. This is the only table that applications write to. Read models are rebuilt from events, making them disposable and rebuildable. If a read model's schema needs to change, it can be dropped and rebuilt from the event history.

2. **JSONB event payloads with typed metadata.** Event metadata (aggregate_type, event_type, actor, timestamps) are in typed columns for efficient indexing and querying. The clinical content itself is in a JSONB `payload` column, providing schema flexibility — new event types can be added without DDL changes.

3. **Optimistic concurrency via aggregate version.** The unique index on `(aggregate_type, aggregate_id, event_version)` prevents conflicting concurrent writes to the same entity. If two clinicians try to update the same patient record simultaneously, the second write will fail and must retry with the latest version.

4. **Monthly partitioning on recorded_at.** The event store is partitioned by month, enabling efficient time-range queries and archival. Partitions older than the HIPAA 6-year retention period can be moved to cold storage. For high-volume deployments, daily partitioning may be appropriate.

5. **AI provenance is first-class.** Every event records whether it was AI-generated, which model produced it, and the confidence score. This supports regulatory requirements for AI transparency and allows filtering or flagging AI-authored content in clinical review workflows.

6. **Read models track their projection state.** Each read model table includes `last_event_version` and `last_projected_at`, and the `projection_checkpoint` table tracks the global sequence position for each projector. This enables monitoring of projection lag and graceful rebuilds.

7. **Correlation and causation IDs for event chains.** When an encounter triggers multiple events (vitals, diagnoses, medications, notes), they share a `correlation_id`. The `causation_id` links cause-and-effect chains (e.g., a lab result triggers a clinical alert). This enables rich analytics on clinical workflows.

8. **Snapshots for performance.** Long-lived aggregates (patients with years of history) can accumulate thousands of events. The `event_snapshot` table stores periodic snapshots of materialized state, so replay starts from the latest snapshot rather than the beginning of time. Snapshots are created every N events (e.g., every 100).
