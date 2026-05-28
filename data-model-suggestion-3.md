# Data Model Suggestion 3: Hybrid Relational + JSONB (FHIR-Native)

> Project: Electronic Health Record (EHR) · Created: 2026-05-19

## Philosophy

This model follows the approach pioneered by Medplum and Fhirbase: store FHIR R4 resources as complete JSONB documents in PostgreSQL, with extracted relational columns for the most frequently queried search parameters. Each FHIR resource type gets its own table, but the primary data lives in a `content` JSONB column containing the full FHIR JSON representation. Typed columns alongside the JSONB hold indexed search parameters (patient reference, date, status, code) for fast SQL queries.

This is the architecture used by production FHIR servers today. Medplum's CDR, Fhirbase, and Health Samurai's Aidbox all use this pattern with PostgreSQL. It provides native FHIR compliance — the FHIR API simply reads/writes the JSONB column without transformation. Multi-specialty, multi-jurisdiction, and multi-region flexibility is inherent because new FHIR extensions and profiles add keys to the JSON document without requiring DDL changes.

The trade-off is that complex analytical queries across multiple resource types require JSONB path expressions or GIN-indexed containment queries, which are less intuitive than simple SQL joins on typed columns. Reporting and business intelligence tools may struggle with JSONB data without an intermediate ETL layer.

**Best for:** Rapid FHIR-first development, API-first platforms, digital health builders who need to support arbitrary FHIR profiles and extensions without schema migration, and teams who want to follow the Medplum/Fhirbase proven production pattern.

**Trade-offs:**
- (+) Native FHIR compliance — store and serve FHIR resources without transformation
- (+) Schema-flexible — new FHIR extensions, profiles, and resource types need no DDL changes
- (+) Fastest path to ONC certification — US Core conformance is structural, not schematic
- (+) Full resource versioning (vread, history) is trivial with a version table
- (+) Multi-specialty and multi-jurisdiction flexibility without nullable column explosion
- (-) Complex cross-resource analytics require JSONB path expressions or GIN indexes
- (-) BI and reporting tools often struggle with JSONB — may need a separate analytics warehouse
- (-) Disk usage higher than normalized model (JSONB includes field names with every document)
- (-) No referential integrity enforcement between FHIR resources at the database level (application-layer responsibility)
- (-) Developers must understand both SQL and FHIR JSON structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | The JSONB `content` column stores verbatim FHIR R4 JSON resources — no mapping layer needed |
| US Core IG (STU8) | US Core profiles are stored directly as FHIR JSON with `meta.profile` references |
| SMART on FHIR | OAuth scopes map directly to resource type tables; search parameters are indexed columns |
| FHIR Bulk Data ($export) | NDJSON export is a direct `SELECT content FROM <resource_type>` dump |
| SNOMED CT / LOINC / ICD-10 / RxNorm | Code systems stored within FHIR CodeableConcept structures in the JSONB content |
| HL7 FHIR Search | Extracted search parameter columns enable FHIR search API without full-document scanning |
| FHIR History / Versioning | `resource_version` table stores prior versions for `_history` and `vread` operations |
| HIPAA Security Rule | Audit log captures FHIR interactions per resource type |

---

## Core Resource Tables

### Patient

```sql
CREATE TABLE "Patient" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,                        -- Full FHIR Patient resource JSON
    -- Extracted search parameters (indexed for FHIR search)
    organization_id UUID,                                  -- Tenant (from managingOrganization)
    family          TEXT,                                   -- name[0].family
    given           TEXT[],                                 -- name[0].given
    birthdate       DATE,                                  -- birthDate
    gender          TEXT,                                   -- gender
    identifier      TEXT[],                                -- identifier[].value
    phone           TEXT[],                                -- telecom[].value where system=phone
    email           TEXT[],                                -- telecom[].value where system=email
    address_city    TEXT,                                   -- address[0].city
    address_state   TEXT,                                   -- address[0].state
    address_postalcode TEXT,                                -- address[0].postalCode
    active          BOOLEAN DEFAULT TRUE,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,        -- Soft delete
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- FHIR content example:
-- {
--   "resourceType": "Patient",
--   "id": "uuid-here",
--   "meta": {
--     "versionId": "1",
--     "lastUpdated": "2026-05-19T10:30:00Z",
--     "profile": ["http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient"]
--   },
--   "identifier": [
--     {"system": "http://hospital.example/mrn", "value": "MRN-2026-00042"},
--     {"system": "http://hl7.org/fhir/sid/us-ssn", "value": "***-**-1234"}
--   ],
--   "name": [{"use": "official", "family": "Smith", "given": ["Jane", "Marie"]}],
--   "gender": "female",
--   "birthDate": "1985-03-15",
--   "address": [{"line": ["123 Main St"], "city": "Portland", "state": "OR", "postalCode": "97201"}],
--   "telecom": [
--     {"system": "phone", "value": "503-555-0142", "use": "mobile"},
--     {"system": "email", "value": "jane.smith@email.com"}
--   ],
--   "extension": [
--     {"url": "http://hl7.org/fhir/us/core/StructureDefinition/us-core-race",
--      "extension": [{"url": "ombCategory", "valueCoding": {"code": "2106-3", "display": "White"}}]},
--     {"url": "http://hl7.org/fhir/us/core/StructureDefinition/us-core-ethnicity",
--      "extension": [{"url": "ombCategory", "valueCoding": {"code": "2186-5", "display": "Not Hispanic or Latino"}}]}
--   ]
-- }

CREATE INDEX idx_patient_org ON "Patient"(organization_id);
CREATE INDEX idx_patient_name ON "Patient"(family, given);
CREATE INDEX idx_patient_birthdate ON "Patient"(birthdate);
CREATE INDEX idx_patient_identifier ON "Patient" USING GIN(identifier);
CREATE INDEX idx_patient_content ON "Patient" USING GIN(content jsonb_path_ops);
```

### Encounter

```sql
CREATE TABLE "Encounter" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    -- Extracted search parameters
    organization_id UUID,
    patient         UUID,                                  -- subject reference
    participant     UUID[],                                -- participant[].individual references
    status          TEXT,
    class_code      TEXT,                                  -- class.code
    type_code       TEXT[],                                -- type[].coding[].code
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    reason_code     TEXT[],                                -- reasonCode[].coding[].code
    location        UUID[],                                -- location[].location reference
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounter_patient ON "Encounter"(patient);
CREATE INDEX idx_encounter_org ON "Encounter"(organization_id);
CREATE INDEX idx_encounter_status ON "Encounter"(status);
CREATE INDEX idx_encounter_period ON "Encounter"(period_start, period_end);
CREATE INDEX idx_encounter_class ON "Encounter"(class_code);
CREATE INDEX idx_encounter_content ON "Encounter" USING GIN(content jsonb_path_ops);
```

### Condition

```sql
CREATE TABLE "Condition" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    -- Extracted search parameters
    organization_id UUID,
    patient         UUID,                                  -- subject reference
    encounter       UUID,                                  -- encounter reference
    code            TEXT[],                                -- code.coding[].code
    code_system     TEXT[],                                -- code.coding[].system
    clinical_status TEXT,                                   -- clinicalStatus.coding[].code
    verification_status TEXT,
    category        TEXT[],                                -- category[].coding[].code
    onset_date      TIMESTAMPTZ,
    recorded_date   DATE,
    asserter        UUID,                                  -- asserter reference
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_condition_patient ON "Condition"(patient);
CREATE INDEX idx_condition_code ON "Condition" USING GIN(code);
CREATE INDEX idx_condition_status ON "Condition"(clinical_status);
CREATE INDEX idx_condition_category ON "Condition" USING GIN(category);
CREATE INDEX idx_condition_content ON "Condition" USING GIN(content jsonb_path_ops);
```

### Observation

```sql
CREATE TABLE "Observation" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    -- Extracted search parameters
    organization_id UUID,
    patient         UUID,                                  -- subject reference
    encounter       UUID,                                  -- encounter reference
    code            TEXT[],                                -- code.coding[].code
    code_system     TEXT[],                                -- code.coding[].system
    category        TEXT[],                                -- category[].coding[].code
    status          TEXT,
    effective_date  TIMESTAMPTZ,                           -- effectiveDateTime or effectivePeriod.start
    value_quantity  NUMERIC,                               -- valueQuantity.value (if applicable)
    value_string    TEXT,                                   -- valueString (if applicable)
    value_code      TEXT[],                                -- valueCodeableConcept.coding[].code
    performer       UUID[],                                -- performer[] references
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_observation_patient ON "Observation"(patient);
CREATE INDEX idx_observation_encounter ON "Observation"(encounter);
CREATE INDEX idx_observation_code ON "Observation" USING GIN(code);
CREATE INDEX idx_observation_category ON "Observation" USING GIN(category);
CREATE INDEX idx_observation_effective ON "Observation"(effective_date);
CREATE INDEX idx_observation_status ON "Observation"(status);
CREATE INDEX idx_observation_content ON "Observation" USING GIN(content jsonb_path_ops);
```

### MedicationRequest

```sql
CREATE TABLE "MedicationRequest" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    -- Extracted search parameters
    organization_id UUID,
    patient         UUID,                                  -- subject reference
    encounter       UUID,                                  -- encounter reference
    requester       UUID,                                  -- requester reference
    medication_code TEXT[],                                -- medicationCodeableConcept.coding[].code
    status          TEXT,
    intent          TEXT,
    authoredon      TIMESTAMPTZ,
    category        TEXT[],
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_medrx_patient ON "MedicationRequest"(patient);
CREATE INDEX idx_medrx_status ON "MedicationRequest"(status);
CREATE INDEX idx_medrx_medication ON "MedicationRequest" USING GIN(medication_code);
CREATE INDEX idx_medrx_content ON "MedicationRequest" USING GIN(content jsonb_path_ops);
```

### AllergyIntolerance

```sql
CREATE TABLE "AllergyIntolerance" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    -- Extracted search parameters
    organization_id UUID,
    patient         UUID,
    code            TEXT[],
    clinical_status TEXT,
    verification_status TEXT,
    type            TEXT,
    category        TEXT[],
    criticality     TEXT,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_allergy_patient ON "AllergyIntolerance"(patient);
CREATE INDEX idx_allergy_status ON "AllergyIntolerance"(clinical_status);
CREATE INDEX idx_allergy_code ON "AllergyIntolerance" USING GIN(code);
CREATE INDEX idx_allergy_content ON "AllergyIntolerance" USING GIN(content jsonb_path_ops);
```

### Procedure

```sql
CREATE TABLE "Procedure" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    -- Extracted search parameters
    organization_id UUID,
    patient         UUID,
    encounter       UUID,
    code            TEXT[],
    code_system     TEXT[],
    status          TEXT,
    performed_date  TIMESTAMPTZ,
    performer       UUID[],
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_procedure_patient ON "Procedure"(patient);
CREATE INDEX idx_procedure_code ON "Procedure" USING GIN(code);
CREATE INDEX idx_procedure_date ON "Procedure"(performed_date);
CREATE INDEX idx_procedure_content ON "Procedure" USING GIN(content jsonb_path_ops);
```

### Immunization

```sql
CREATE TABLE "Immunization" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID,
    patient         UUID,
    encounter       UUID,
    vaccine_code    TEXT[],
    status          TEXT,
    occurrence_date TIMESTAMPTZ,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_immunization_patient ON "Immunization"(patient);
CREATE INDEX idx_immunization_vaccine ON "Immunization" USING GIN(vaccine_code);
CREATE INDEX idx_immunization_content ON "Immunization" USING GIN(content jsonb_path_ops);
```

### DiagnosticReport

```sql
CREATE TABLE "DiagnosticReport" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID,
    patient         UUID,
    encounter       UUID,
    code            TEXT[],
    category        TEXT[],
    status          TEXT,
    effective_date  TIMESTAMPTZ,
    issued          TIMESTAMPTZ,
    result          UUID[],                                -- Observation references
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_diagrpt_patient ON "DiagnosticReport"(patient);
CREATE INDEX idx_diagrpt_code ON "DiagnosticReport" USING GIN(code);
CREATE INDEX idx_diagrpt_status ON "DiagnosticReport"(status);
CREATE INDEX idx_diagrpt_content ON "DiagnosticReport" USING GIN(content jsonb_path_ops);
```

### ServiceRequest

```sql
CREATE TABLE "ServiceRequest" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID,
    patient         UUID,
    encounter       UUID,
    requester       UUID,
    code            TEXT[],
    category        TEXT[],
    status          TEXT,
    intent          TEXT,
    priority        TEXT,
    authored_on     TIMESTAMPTZ,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_servicerx_patient ON "ServiceRequest"(patient);
CREATE INDEX idx_servicerx_status ON "ServiceRequest"(status);
CREATE INDEX idx_servicerx_content ON "ServiceRequest" USING GIN(content jsonb_path_ops);
```

---

## Supporting Resources

### Organization

```sql
CREATE TABLE "Organization" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    name            TEXT,
    identifier      TEXT[],
    type_code       TEXT[],
    partof          UUID,                                  -- Parent organization
    active          BOOLEAN DEFAULT TRUE,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_name ON "Organization"(name);
CREATE INDEX idx_org_identifier ON "Organization" USING GIN(identifier);
CREATE INDEX idx_org_content ON "Organization" USING GIN(content jsonb_path_ops);
```

### Practitioner

```sql
CREATE TABLE "Practitioner" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID,
    family          TEXT,
    given           TEXT[],
    identifier      TEXT[],                                -- NPI, DEA, etc.
    active          BOOLEAN DEFAULT TRUE,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_practitioner_name ON "Practitioner"(family);
CREATE INDEX idx_practitioner_identifier ON "Practitioner" USING GIN(identifier);
CREATE INDEX idx_practitioner_content ON "Practitioner" USING GIN(content jsonb_path_ops);
```

### Appointment

```sql
CREATE TABLE "Appointment" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID,
    patient         UUID,
    practitioner    UUID[],
    status          TEXT,
    start_time      TIMESTAMPTZ,
    end_time        TIMESTAMPTZ,
    service_type    TEXT[],
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appointment_patient ON "Appointment"(patient);
CREATE INDEX idx_appointment_start ON "Appointment"(start_time);
CREATE INDEX idx_appointment_status ON "Appointment"(status);
CREATE INDEX idx_appointment_content ON "Appointment" USING GIN(content jsonb_path_ops);
```

---

## Versioning & History

### Resource Version History

```sql
CREATE TABLE resource_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   TEXT NOT NULL,                          -- 'Patient', 'Encounter', etc.
    resource_id     UUID NOT NULL,
    version_id      INTEGER NOT NULL,
    content         JSONB NOT NULL,                        -- Full FHIR resource at this version
    author_id       UUID,
    change_reason    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (resource_type, resource_id, version_id)
);

CREATE INDEX idx_rv_resource ON resource_version(resource_type, resource_id);
CREATE INDEX idx_rv_created ON resource_version(created_at);
```

---

## Billing (Non-FHIR Operational Tables)

### Claim (Operational)

```sql
CREATE TABLE "Claim" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,                        -- FHIR Claim resource
    organization_id UUID,
    patient         UUID,
    encounter       UUID,
    status          TEXT,
    type_code       TEXT,
    use_code        TEXT,
    total_value     NUMERIC(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_claim_patient ON "Claim"(patient);
CREATE INDEX idx_claim_status ON "Claim"(status);

-- EDI transaction tracking (operational, not FHIR)
CREATE TABLE edi_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    claim_id        UUID REFERENCES "Claim"(id),
    transaction_type TEXT NOT NULL CHECK (transaction_type IN ('837P', '837I', '837D', '835', '270', '271', '278')),
    direction       TEXT NOT NULL CHECK (direction IN ('outbound', 'inbound')),
    clearinghouse   TEXT,
    control_number  TEXT,
    raw_content     TEXT,                                   -- Raw X12 EDI content
    status          TEXT CHECK (status IN ('pending', 'sent', 'accepted', 'rejected', 'acknowledged')),
    submitted_at    TIMESTAMPTZ,
    response_at     TIMESTAMPTZ,
    error_details   JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edi_claim ON edi_transaction(claim_id);
CREATE INDEX idx_edi_type ON edi_transaction(transaction_type);
CREATE INDEX idx_edi_status ON edi_transaction(status);
```

---

## Security & Operations

### User Account

```sql
CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    practitioner_id UUID,                                  -- Links to Practitioner resource
    patient_id      UUID,                                  -- Links to Patient resource
    email           TEXT NOT NULL UNIQUE,
    password_hash   TEXT NOT NULL,
    role            TEXT NOT NULL,
    scopes          TEXT[],                                -- SMART on FHIR scopes granted
    mfa_enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_org ON user_account(organization_id);
```

### FHIR Audit Event

```sql
CREATE TABLE "AuditEvent" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,                        -- Full FHIR AuditEvent resource
    -- Extracted search parameters
    organization_id UUID,
    agent_who       UUID,                                  -- Who performed the action
    patient         UUID,                                  -- Whose data was accessed
    type_code       TEXT,                                  -- rest, audit, etc.
    subtype_code    TEXT[],                                -- read, vread, update, delete, etc.
    action          TEXT,                                  -- C, R, U, D, E
    outcome         TEXT,                                  -- 0 (success), 4 (minor failure), 8 (serious failure), 12 (major failure)
    recorded        TIMESTAMPTZ NOT NULL DEFAULT now(),
    entity_type     TEXT,                                  -- Resource type accessed
    entity_id       UUID,                                  -- Resource ID accessed
    source_ip       INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded);

CREATE INDEX idx_audit_agent ON "AuditEvent"(agent_who);
CREATE INDEX idx_audit_patient ON "AuditEvent"(patient);
CREATE INDEX idx_audit_action ON "AuditEvent"(action);
CREATE INDEX idx_audit_recorded ON "AuditEvent"(recorded);
CREATE INDEX idx_audit_entity ON "AuditEvent"(entity_type, entity_id);
```

---

## Multi-Tenancy via Row-Level Security

```sql
-- Enable RLS on all resource tables
ALTER TABLE "Patient" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Encounter" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Condition" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Observation" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "MedicationRequest" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "AllergyIntolerance" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Procedure" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Immunization" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "DiagnosticReport" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "ServiceRequest" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Appointment" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Claim" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "AuditEvent" ENABLE ROW LEVEL SECURITY;

-- Policy template (repeat for each table)
CREATE POLICY tenant_isolation ON "Patient"
    USING (organization_id = current_setting('app.current_organization_id')::uuid);
```

---

## Query Examples

### FHIR Search: Conditions by Code and Status

```sql
-- FHIR search: GET /Condition?patient=uuid&clinical-status=active&code=44054006
SELECT content
FROM "Condition"
WHERE patient = 'patient-uuid-here'
  AND clinical_status = 'active'
  AND code @> ARRAY['44054006']
  AND is_deleted = FALSE;
```

### FHIR Search: Observations by Category and Date Range

```sql
-- FHIR search: GET /Observation?patient=uuid&category=vital-signs&date=ge2026-01-01&date=le2026-05-19
SELECT content
FROM "Observation"
WHERE patient = 'patient-uuid-here'
  AND category @> ARRAY['vital-signs']
  AND effective_date >= '2026-01-01'
  AND effective_date <= '2026-05-19'
  AND is_deleted = FALSE
ORDER BY effective_date DESC;
```

### JSONB Deep Query: Find Patients by Extension (US Core Race)

```sql
-- Find patients with race extension = "2106-3" (White)
SELECT id, content->>'id' AS fhir_id, content->'name'->0->>'family' AS family
FROM "Patient"
WHERE content @> '{"extension": [{"url": "http://hl7.org/fhir/us/core/StructureDefinition/us-core-race", "extension": [{"url": "ombCategory", "valueCoding": {"code": "2106-3"}}]}]}'
  AND is_deleted = FALSE;
```

### Bulk Data Export (NDJSON)

```sql
-- FHIR $export: stream all Patient resources as NDJSON
COPY (
    SELECT content::text
    FROM "Patient"
    WHERE organization_id = 'org-uuid'
      AND is_deleted = FALSE
) TO STDOUT;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Clinical Resources | 10 | Patient, Encounter, Condition, Observation, MedicationRequest, AllergyIntolerance, Procedure, Immunization, DiagnosticReport, ServiceRequest |
| Administrative Resources | 3 | Organization, Practitioner, Appointment |
| Billing | 2 | Claim, edi_transaction |
| Versioning | 1 | resource_version |
| Security & Audit | 2 | user_account, AuditEvent |
| **Total** | **18** | Fewer tables than normalized model; clinical complexity lives in JSONB content |

---

## Key Design Decisions

1. **FHIR resource JSON is the source of truth.** The `content` JSONB column stores the complete, valid FHIR R4 JSON resource. The extracted columns are derived indexes — if they disagree with the content, the content wins. This eliminates the impedance mismatch between database and API.

2. **One table per FHIR resource type (not one generic table).** Unlike some FHIR servers that use a single `resource` table with a `type` discriminator, this model gives each resource type its own table with type-specific extracted search parameters. This provides better query performance (smaller table scans), more specific indexing, and clearer PostgreSQL statistics for the query planner.

3. **Extracted search parameters as typed columns.** The most commonly queried FHIR search parameters (patient, code, status, date, category) are extracted into typed PostgreSQL columns alongside the JSONB content. This avoids full-document GIN index scans for the majority of clinical queries while keeping the full JSONB for arbitrary deep queries.

4. **Array columns for multi-valued search parameters.** FHIR elements like `code.coding[].code` and `category[].coding[].code` can have multiple values. These are extracted into `TEXT[]` array columns with GIN indexes, enabling efficient containment queries (`@>`) that map directly to FHIR search semantics.

5. **Soft deletes with `is_deleted` flag.** FHIR's `delete` operation sets `is_deleted = TRUE` rather than removing the row. This preserves referential integrity and supports the FHIR `_history` endpoint. All queries filter on `is_deleted = FALSE`.

6. **Separate version history table.** Prior versions of resources are stored in `resource_version` rather than in the main resource table. This keeps the primary tables lean for operational queries while supporting FHIR `vread` and `_history` operations.

7. **Audit logging via FHIR AuditEvent resource.** Rather than a custom audit table, audit records are stored as FHIR AuditEvent resources. This means the audit trail is itself FHIR-searchable and exportable via the standard API, and aligns with IHE ATNA profile requirements.

8. **GIN indexes on content for arbitrary FHIR queries.** Every resource table has a `content jsonb_path_ops` GIN index. This supports containment queries (`@>`) for FHIR search parameters that are not extracted into typed columns — providing a fallback for any FHIR search parameter without requiring schema changes.
