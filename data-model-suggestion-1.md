# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Electronic Health Record (EHR) · Created: 2026-05-19

## Philosophy

This model follows the traditional normalized relational approach where every clinical concept — patients, encounters, conditions, observations, medications, allergies, procedures — gets its own dedicated table with strongly typed columns, foreign keys, and referential integrity constraints. The schema is designed to mirror the FHIR R4 resource model but decomposed into third normal form (3NF), with separate lookup/reference tables for coded values (SNOMED CT, LOINC, ICD-10, RxNorm, CPT).

This is the architecture used by legacy EHR platforms like Cerner Millennium and many hospital data warehouses. It provides maximum query flexibility via standard SQL joins, strong data integrity guarantees, and straightforward indexing for regulatory reporting. Every field is explicitly typed and constrained, making schema evolution deliberate and auditable.

The trade-off is rigidity: adding a new clinical concept (e.g., a new observation type or a jurisdiction-specific field) requires DDL changes, migration scripts, and deployment coordination. Multi-region or multi-specialty deployments that need flexible field sets will find this model constraining without additional extension mechanisms.

**Best for:** Regulatory-heavy deployments where data integrity, auditability, and complex cross-entity SQL reporting are paramount — hospital systems, government national EHR programs, and ONC-certified platforms.

**Trade-offs:**
- (+) Maximum referential integrity — foreign keys enforce valid relationships at the database level
- (+) Excellent for complex analytical queries, quality measure calculation, and regulatory reporting
- (+) Well-understood by database administrators and healthcare data analysts
- (+) Direct mapping to FHIR R4 resources for API exposure
- (-) Schema rigidity — every new concept requires DDL migration
- (-) High table count increases join complexity for simple clinical queries
- (-) Multi-specialty or multi-jurisdiction flexibility requires nullable columns or extension tables
- (-) Schema evolution across distributed deployments is operationally expensive

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Each core FHIR resource (Patient, Encounter, Condition, Observation, MedicationRequest, AllergyIntolerance, Procedure, DiagnosticReport, Immunization) maps to a dedicated table |
| US Core IG (STU8) | Must-support elements from US Core profiles define the required columns in each table |
| SNOMED CT | `code_system = 'http://snomed.info/sct'` in `coded_concept` lookup table for diagnoses and clinical findings |
| LOINC | `code_system = 'http://loinc.org'` in `coded_concept` for lab tests, vital signs, and document types |
| ICD-10-CM/PCS | `code_system = 'http://hl7.org/fhir/sid/icd-10-cm'` for billing diagnoses |
| RxNorm | `code_system = 'http://www.nlm.nih.gov/research/umls/rxnorm'` for medication codes |
| CPT | `code_system = 'http://www.ama-assn.org/go/cpt'` for procedure billing codes (AMA licence required) |
| ISO 3166-1/2 | `jurisdiction` columns use ISO 3166 codes for multi-region deployments |
| HIPAA Security Rule | `audit_log` table captures all ePHI access per 45 CFR 164.312(b) |
| X12 837/835 | `claim`, `claim_line`, `remittance` tables model EDI billing transactions |
| NCPDP SCRIPT | `prescription` table fields align with NewRx/RxRenewal message structure |

---

## Entity Management

### Organization (Tenant)

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id       UUID REFERENCES organization(id),
    name            TEXT NOT NULL,
    type            TEXT NOT NULL CHECK (type IN ('health-system', 'hospital', 'clinic', 'practice-group', 'department')),
    npi             VARCHAR(10),                          -- National Provider Identifier
    tax_id          VARCHAR(20),                          -- EIN for billing
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state           VARCHAR(2),
    postal_code     VARCHAR(10),
    country         VARCHAR(3) DEFAULT 'USA',             -- ISO 3166-1 alpha-3
    phone           TEXT,
    fax             TEXT,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organization_parent ON organization(parent_id);
CREATE INDEX idx_organization_npi ON organization(npi);
CREATE INDEX idx_organization_type ON organization(type);
```

### Practitioner

```sql
CREATE TABLE practitioner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    middle_name     TEXT,
    suffix          TEXT,                                  -- MD, DO, NP, PA, etc.
    npi             VARCHAR(10) UNIQUE,
    dea_number      VARCHAR(20),                           -- DEA for controlled substance prescribing
    specialty_code  TEXT,                                   -- NUCC taxonomy code
    specialty_name  TEXT,
    email           TEXT,
    phone           TEXT,
    license_state   VARCHAR(2),
    license_number  TEXT,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_practitioner_org ON practitioner(organization_id);
CREATE INDEX idx_practitioner_npi ON practitioner(npi);
CREATE INDEX idx_practitioner_name ON practitioner(last_name, first_name);
```

---

## Patient Management

### Patient

```sql
CREATE TABLE patient (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organization(id),
    mrn                 TEXT NOT NULL,                      -- Medical Record Number (org-scoped)
    first_name          TEXT NOT NULL,
    last_name           TEXT NOT NULL,
    middle_name         TEXT,
    prefix              TEXT,
    suffix              TEXT,
    date_of_birth       DATE NOT NULL,
    date_of_death       DATE,
    gender              TEXT NOT NULL CHECK (gender IN ('male', 'female', 'other', 'unknown')),  -- FHIR AdministrativeGender
    birth_sex           TEXT CHECK (birth_sex IN ('M', 'F', 'UNK')),                              -- US Core Birth Sex
    race                TEXT,                              -- OMB race category (US Core)
    ethnicity           TEXT,                              -- OMB ethnicity (US Core)
    preferred_language  TEXT,                              -- BCP 47 language tag
    ssn_last_four       VARCHAR(4),                        -- Last 4 of SSN (encrypted at rest)
    marital_status      TEXT,
    email               TEXT,
    phone_home          TEXT,
    phone_mobile        TEXT,
    address_line1       TEXT,
    address_line2       TEXT,
    city                TEXT,
    state               VARCHAR(2),
    postal_code         VARCHAR(10),
    country             VARCHAR(3) DEFAULT 'USA',
    emergency_contact_name  TEXT,
    emergency_contact_phone TEXT,
    emergency_contact_rel   TEXT,
    active              BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, mrn)
);

CREATE INDEX idx_patient_org ON patient(organization_id);
CREATE INDEX idx_patient_name ON patient(last_name, first_name);
CREATE INDEX idx_patient_dob ON patient(date_of_birth);
CREATE INDEX idx_patient_mrn ON patient(organization_id, mrn);
```

### Patient Identifier (Cross-References)

```sql
CREATE TABLE patient_identifier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    system          TEXT NOT NULL,                          -- e.g., 'http://hospital.example/mrn', 'http://hl7.org/fhir/sid/us-ssn'
    value           TEXT NOT NULL,
    type            TEXT,                                   -- MR, SS, DL, etc.
    assigner        TEXT,
    period_start    DATE,
    period_end      DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_identifier_patient ON patient_identifier(patient_id);
CREATE UNIQUE INDEX idx_patient_identifier_system_value ON patient_identifier(system, value);
```

### Insurance Coverage

```sql
CREATE TABLE coverage (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    organization_id     UUID NOT NULL REFERENCES organization(id),
    payer_name          TEXT NOT NULL,
    payer_id            TEXT,                               -- Payer identifier
    plan_name           TEXT,
    member_id           TEXT NOT NULL,
    group_number        TEXT,
    subscriber_name     TEXT,
    subscriber_relationship TEXT CHECK (subscriber_relationship IN ('self', 'spouse', 'child', 'other')),
    coverage_type       TEXT CHECK (coverage_type IN ('primary', 'secondary', 'tertiary')),
    period_start        DATE NOT NULL,
    period_end          DATE,
    active              BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_patient ON coverage(patient_id);
CREATE INDEX idx_coverage_member ON coverage(member_id);
```

---

## Clinical Documentation

### Encounter

```sql
CREATE TABLE encounter (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    organization_id     UUID NOT NULL REFERENCES organization(id),
    status              TEXT NOT NULL CHECK (status IN ('planned', 'arrived', 'triaged', 'in-progress', 'onleave', 'finished', 'cancelled', 'entered-in-error', 'unknown')),
    class_code          TEXT NOT NULL,                      -- AMB, IMP, EMER, HH, VR, etc. (ActCode)
    type_code           TEXT,                               -- SNOMED CT encounter type
    type_display        TEXT,
    priority            TEXT,                               -- R (routine), EM (emergency), UR (urgent)
    reason_code         TEXT,                               -- SNOMED CT or ICD-10 chief complaint
    reason_display      TEXT,
    period_start        TIMESTAMPTZ NOT NULL,
    period_end          TIMESTAMPTZ,
    discharge_disposition TEXT,                             -- SNF, home, AMA, etc.
    location_name       TEXT,
    location_id         UUID,
    service_type        TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounter_patient ON encounter(patient_id);
CREATE INDEX idx_encounter_practitioner ON encounter(practitioner_id);
CREATE INDEX idx_encounter_org ON encounter(organization_id);
CREATE INDEX idx_encounter_period ON encounter(period_start, period_end);
CREATE INDEX idx_encounter_status ON encounter(status);
```

### Condition (Problem List / Diagnoses)

```sql
CREATE TABLE condition (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    clinical_status     TEXT NOT NULL CHECK (clinical_status IN ('active', 'recurrence', 'relapse', 'inactive', 'remission', 'resolved')),
    verification_status TEXT CHECK (verification_status IN ('unconfirmed', 'provisional', 'differential', 'confirmed', 'refuted', 'entered-in-error')),
    category            TEXT NOT NULL CHECK (category IN ('problem-list-item', 'encounter-diagnosis', 'health-concern')),
    severity            TEXT CHECK (severity IN ('severe', 'moderate', 'mild')),
    code_system         TEXT NOT NULL,                      -- SNOMED CT or ICD-10-CM
    code                TEXT NOT NULL,
    display             TEXT NOT NULL,
    body_site_code      TEXT,                               -- SNOMED CT body site
    body_site_display   TEXT,
    onset_datetime      TIMESTAMPTZ,
    onset_age           INTEGER,
    abatement_datetime  TIMESTAMPTZ,
    recorded_date       DATE NOT NULL DEFAULT CURRENT_DATE,
    note                TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_condition_patient ON condition(patient_id);
CREATE INDEX idx_condition_encounter ON condition(encounter_id);
CREATE INDEX idx_condition_code ON condition(code_system, code);
CREATE INDEX idx_condition_category ON condition(category);
CREATE INDEX idx_condition_clinical_status ON condition(clinical_status);
```

### Observation (Vitals, Labs, Assessments)

```sql
CREATE TABLE observation (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    status              TEXT NOT NULL CHECK (status IN ('registered', 'preliminary', 'final', 'amended', 'corrected', 'cancelled', 'entered-in-error', 'unknown')),
    category_code       TEXT NOT NULL,                      -- vital-signs, laboratory, social-history, survey, exam, etc.
    code_system         TEXT NOT NULL,                      -- LOINC
    code                TEXT NOT NULL,                      -- LOINC code
    display             TEXT NOT NULL,
    -- Value (polymorphic — only one populated per row)
    value_quantity      NUMERIC,
    value_unit          TEXT,
    value_unit_code     TEXT,                               -- UCUM unit code
    value_string        TEXT,
    value_codeable_code TEXT,
    value_codeable_display TEXT,
    value_boolean       BOOLEAN,
    value_datetime      TIMESTAMPTZ,
    -- Reference ranges
    reference_low       NUMERIC,
    reference_high      NUMERIC,
    reference_text      TEXT,
    -- Interpretation
    interpretation_code TEXT,                               -- H, L, HH, LL, N, A, AA, etc.
    effective_datetime  TIMESTAMPTZ NOT NULL,
    issued              TIMESTAMPTZ,
    body_site_code      TEXT,
    method_code         TEXT,
    note                TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_observation_patient ON observation(patient_id);
CREATE INDEX idx_observation_encounter ON observation(encounter_id);
CREATE INDEX idx_observation_code ON observation(code_system, code);
CREATE INDEX idx_observation_category ON observation(category_code);
CREATE INDEX idx_observation_effective ON observation(effective_datetime);
```

### Allergy Intolerance

```sql
CREATE TABLE allergy_intolerance (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    clinical_status     TEXT NOT NULL CHECK (clinical_status IN ('active', 'inactive', 'resolved')),
    verification_status TEXT CHECK (verification_status IN ('unconfirmed', 'confirmed', 'refuted', 'entered-in-error')),
    type                TEXT CHECK (type IN ('allergy', 'intolerance')),
    category            TEXT CHECK (category IN ('food', 'medication', 'environment', 'biologic')),
    criticality         TEXT CHECK (criticality IN ('low', 'high', 'unable-to-assess')),
    code_system         TEXT NOT NULL,                      -- RxNorm, SNOMED CT, or NDFRT
    code                TEXT NOT NULL,
    display             TEXT NOT NULL,
    onset_datetime      TIMESTAMPTZ,
    recorded_date       DATE NOT NULL DEFAULT CURRENT_DATE,
    note                TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_allergy_patient ON allergy_intolerance(patient_id);
CREATE INDEX idx_allergy_code ON allergy_intolerance(code_system, code);
CREATE INDEX idx_allergy_status ON allergy_intolerance(clinical_status);
```

### Allergy Reaction

```sql
CREATE TABLE allergy_reaction (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    allergy_intolerance_id  UUID NOT NULL REFERENCES allergy_intolerance(id) ON DELETE CASCADE,
    substance_code          TEXT,
    substance_display       TEXT,
    manifestation_code      TEXT NOT NULL,                  -- SNOMED CT
    manifestation_display   TEXT NOT NULL,
    severity                TEXT CHECK (severity IN ('mild', 'moderate', 'severe')),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_allergy_reaction_allergy ON allergy_reaction(allergy_intolerance_id);
```

---

## Medications & Prescribing

### Medication Request (Orders / Prescriptions)

```sql
CREATE TABLE medication_request (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    prescriber_id       UUID NOT NULL REFERENCES practitioner(id),
    status              TEXT NOT NULL CHECK (status IN ('active', 'on-hold', 'cancelled', 'completed', 'entered-in-error', 'stopped', 'draft', 'unknown')),
    intent              TEXT NOT NULL CHECK (intent IN ('proposal', 'plan', 'order', 'original-order', 'reflex-order', 'filler-order', 'instance-order', 'option')),
    category            TEXT,                               -- inpatient, outpatient, community, discharge
    medication_code     TEXT NOT NULL,                      -- RxNorm code
    medication_display  TEXT NOT NULL,
    medication_system   TEXT NOT NULL DEFAULT 'http://www.nlm.nih.gov/research/umls/rxnorm',
    dosage_text         TEXT,
    dosage_route_code   TEXT,                               -- SNOMED CT route
    dosage_route_display TEXT,
    dosage_frequency    TEXT,                               -- e.g., 'BID', 'TID', 'Q8H'
    dosage_dose_value   NUMERIC,
    dosage_dose_unit    TEXT,                               -- UCUM unit
    dosage_duration_value NUMERIC,
    dosage_duration_unit TEXT,
    quantity_value      NUMERIC,
    quantity_unit       TEXT,
    refills_allowed     INTEGER DEFAULT 0,
    substitution_allowed BOOLEAN DEFAULT TRUE,
    authored_on         TIMESTAMPTZ NOT NULL DEFAULT now(),
    dispense_request_start DATE,
    dispense_request_end   DATE,
    reason_code         TEXT,                               -- ICD-10 or SNOMED CT
    reason_display      TEXT,
    note                TEXT,
    is_controlled       BOOLEAN NOT NULL DEFAULT FALSE,     -- Schedule II-V
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_medrx_patient ON medication_request(patient_id);
CREATE INDEX idx_medrx_encounter ON medication_request(encounter_id);
CREATE INDEX idx_medrx_prescriber ON medication_request(prescriber_id);
CREATE INDEX idx_medrx_status ON medication_request(status);
CREATE INDEX idx_medrx_medication ON medication_request(medication_code);
```

---

## Orders & Results

### Service Request (Lab / Imaging Orders)

```sql
CREATE TABLE service_request (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    requester_id        UUID NOT NULL REFERENCES practitioner(id),
    status              TEXT NOT NULL CHECK (status IN ('draft', 'active', 'on-hold', 'revoked', 'completed', 'entered-in-error', 'unknown')),
    intent              TEXT NOT NULL DEFAULT 'order',
    category            TEXT NOT NULL CHECK (category IN ('laboratory', 'imaging', 'procedure', 'referral', 'other')),
    priority            TEXT CHECK (priority IN ('routine', 'urgent', 'asap', 'stat')),
    code_system         TEXT NOT NULL,                      -- LOINC for labs, SNOMED CT for procedures
    code                TEXT NOT NULL,
    display             TEXT NOT NULL,
    reason_code         TEXT,                               -- ICD-10 or SNOMED CT
    reason_display      TEXT,
    occurrence_datetime TIMESTAMPTZ,
    authored_on         TIMESTAMPTZ NOT NULL DEFAULT now(),
    note                TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_servicerx_patient ON service_request(patient_id);
CREATE INDEX idx_servicerx_encounter ON service_request(encounter_id);
CREATE INDEX idx_servicerx_status ON service_request(status);
CREATE INDEX idx_servicerx_category ON service_request(category);
```

### Diagnostic Report

```sql
CREATE TABLE diagnostic_report (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    service_request_id  UUID REFERENCES service_request(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    status              TEXT NOT NULL CHECK (status IN ('registered', 'partial', 'preliminary', 'final', 'amended', 'corrected', 'appended', 'cancelled', 'entered-in-error', 'unknown')),
    category_code       TEXT NOT NULL,                      -- LAB, RAD, PATH, etc.
    code_system         TEXT NOT NULL,
    code                TEXT NOT NULL,                      -- LOINC panel code
    display             TEXT NOT NULL,
    effective_datetime  TIMESTAMPTZ,
    issued              TIMESTAMPTZ,
    conclusion          TEXT,
    conclusion_code     TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_diagreport_patient ON diagnostic_report(patient_id);
CREATE INDEX idx_diagreport_service_request ON diagnostic_report(service_request_id);
CREATE INDEX idx_diagreport_status ON diagnostic_report(status);

-- Junction: diagnostic report contains multiple observations
CREATE TABLE diagnostic_report_observation (
    diagnostic_report_id UUID NOT NULL REFERENCES diagnostic_report(id) ON DELETE CASCADE,
    observation_id       UUID NOT NULL REFERENCES observation(id),
    PRIMARY KEY (diagnostic_report_id, observation_id)
);
```

---

## Procedures & Immunizations

### Procedure

```sql
CREATE TABLE procedure (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    status              TEXT NOT NULL CHECK (status IN ('preparation', 'in-progress', 'not-done', 'on-hold', 'stopped', 'completed', 'entered-in-error', 'unknown')),
    code_system         TEXT NOT NULL,                      -- SNOMED CT or CPT
    code                TEXT NOT NULL,
    display             TEXT NOT NULL,
    category_code       TEXT,                               -- SNOMED CT procedure category
    performed_datetime  TIMESTAMPTZ,
    performed_start     TIMESTAMPTZ,
    performed_end       TIMESTAMPTZ,
    reason_code         TEXT,                               -- ICD-10 or SNOMED CT
    reason_display      TEXT,
    body_site_code      TEXT,                               -- SNOMED CT body site
    body_site_display   TEXT,
    outcome_code        TEXT,
    outcome_display     TEXT,
    note                TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_procedure_patient ON procedure(patient_id);
CREATE INDEX idx_procedure_encounter ON procedure(encounter_id);
CREATE INDEX idx_procedure_code ON procedure(code_system, code);
```

### Immunization

```sql
CREATE TABLE immunization (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    status              TEXT NOT NULL CHECK (status IN ('completed', 'entered-in-error', 'not-done')),
    vaccine_code        TEXT NOT NULL,                      -- CVX code
    vaccine_display     TEXT NOT NULL,
    occurrence_datetime TIMESTAMPTZ NOT NULL,
    lot_number          TEXT,
    expiration_date     DATE,
    site_code           TEXT,                               -- SNOMED CT body site
    route_code          TEXT,                               -- SNOMED CT route
    dose_quantity       NUMERIC,
    dose_unit           TEXT,
    is_subpotent        BOOLEAN DEFAULT FALSE,
    note                TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_immunization_patient ON immunization(patient_id);
CREATE INDEX idx_immunization_vaccine ON immunization(vaccine_code);
CREATE INDEX idx_immunization_date ON immunization(occurrence_datetime);
```

---

## Clinical Notes

### Clinical Note (SOAP / Progress Note)

```sql
CREATE TABLE clinical_note (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    author_id           UUID NOT NULL REFERENCES practitioner(id),
    status              TEXT NOT NULL CHECK (status IN ('preliminary', 'final', 'amended', 'entered-in-error')),
    type_code           TEXT NOT NULL,                      -- LOINC document type (e.g., 34117-2 for History and Physical)
    type_display        TEXT NOT NULL,
    title               TEXT,
    -- SOAP sections
    subjective          TEXT,
    objective           TEXT,
    assessment          TEXT,
    plan                TEXT,
    -- Full text fallback
    narrative           TEXT,
    -- AI metadata
    ai_generated        BOOLEAN NOT NULL DEFAULT FALSE,
    ai_model            TEXT,                               -- Model identifier if AI-generated
    ai_confidence       NUMERIC,
    clinician_reviewed  BOOLEAN NOT NULL DEFAULT FALSE,
    reviewed_at         TIMESTAMPTZ,
    authored_on         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_note_patient ON clinical_note(patient_id);
CREATE INDEX idx_note_encounter ON clinical_note(encounter_id);
CREATE INDEX idx_note_author ON clinical_note(author_id);
CREATE INDEX idx_note_type ON clinical_note(type_code);
CREATE INDEX idx_note_authored ON clinical_note(authored_on);
```

---

## Scheduling

### Appointment

```sql
CREATE TABLE appointment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    organization_id     UUID NOT NULL REFERENCES organization(id),
    encounter_id        UUID REFERENCES encounter(id),
    status              TEXT NOT NULL CHECK (status IN ('proposed', 'pending', 'booked', 'arrived', 'fulfilled', 'cancelled', 'noshow', 'entered-in-error', 'checked-in', 'waitlist')),
    service_type        TEXT,
    appointment_type    TEXT,                               -- routine, walkin, checkup, followup, emergency
    reason_code         TEXT,
    reason_display      TEXT,
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    minutes_duration    INTEGER,
    patient_instruction TEXT,
    note                TEXT,
    cancellation_reason TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appointment_patient ON appointment(patient_id);
CREATE INDEX idx_appointment_practitioner ON appointment(practitioner_id);
CREATE INDEX idx_appointment_start ON appointment(start_time);
CREATE INDEX idx_appointment_status ON appointment(status);
```

---

## Billing & Claims

### Claim

```sql
CREATE TABLE claim (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    organization_id     UUID NOT NULL REFERENCES organization(id),
    coverage_id         UUID REFERENCES coverage(id),
    status              TEXT NOT NULL CHECK (status IN ('draft', 'active', 'cancelled', 'entered-in-error')),
    type                TEXT NOT NULL CHECK (type IN ('institutional', 'professional', 'pharmacy', 'vision', 'dental')),
    use                 TEXT NOT NULL CHECK (use IN ('claim', 'preauthorization', 'predetermination')),
    claim_number        TEXT,                               -- Payer-assigned claim number
    total_amount        NUMERIC(12,2),
    currency            VARCHAR(3) DEFAULT 'USD',
    provider_npi        VARCHAR(10),
    billing_npi         VARCHAR(10),
    facility_npi        VARCHAR(10),
    service_date_start  DATE NOT NULL,
    service_date_end    DATE,
    submitted_at        TIMESTAMPTZ,
    -- X12 837 specific
    edi_transaction_id  TEXT,                               -- 837P/837I transaction control number
    clearinghouse       TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_patient ON claim(patient_id);
CREATE INDEX idx_claim_encounter ON claim(encounter_id);
CREATE INDEX idx_claim_status ON claim(status);
CREATE INDEX idx_claim_submitted ON claim(submitted_at);
```

### Claim Line Item

```sql
CREATE TABLE claim_line (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id            UUID NOT NULL REFERENCES claim(id) ON DELETE CASCADE,
    sequence            INTEGER NOT NULL,
    procedure_code      TEXT NOT NULL,                      -- CPT or HCPCS
    procedure_system    TEXT NOT NULL DEFAULT 'http://www.ama-assn.org/go/cpt',
    procedure_display   TEXT,
    modifier_1          VARCHAR(2),
    modifier_2          VARCHAR(2),
    modifier_3          VARCHAR(2),
    modifier_4          VARCHAR(2),
    diagnosis_pointer   TEXT,                               -- References claim-level diagnosis
    quantity            NUMERIC(10,2) DEFAULT 1,
    unit_price          NUMERIC(12,2),
    net_amount          NUMERIC(12,2),
    place_of_service    VARCHAR(2),                         -- CMS POS code
    revenue_code        VARCHAR(4),                         -- UB-04 revenue code (institutional)
    service_date        DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_line_claim ON claim_line(claim_id);
```

### Claim Diagnosis

```sql
CREATE TABLE claim_diagnosis (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id            UUID NOT NULL REFERENCES claim(id) ON DELETE CASCADE,
    sequence            INTEGER NOT NULL,
    diagnosis_code      TEXT NOT NULL,                      -- ICD-10-CM
    diagnosis_system    TEXT NOT NULL DEFAULT 'http://hl7.org/fhir/sid/icd-10-cm',
    diagnosis_display   TEXT,
    type                TEXT CHECK (type IN ('admitting', 'principal', 'secondary')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_dx_claim ON claim_diagnosis(claim_id);
```

### Remittance (ERA / 835)

```sql
CREATE TABLE remittance (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id            UUID REFERENCES claim(id),
    payer_name          TEXT NOT NULL,
    payer_id            TEXT,
    check_number        TEXT,
    payment_date        DATE NOT NULL,
    total_paid          NUMERIC(12,2) NOT NULL,
    patient_responsibility NUMERIC(12,2),
    adjustment_amount   NUMERIC(12,2),
    adjustment_reason   TEXT,                               -- CARC (Claim Adjustment Reason Code)
    remark_code         TEXT,                               -- RARC (Remittance Advice Remark Code)
    edi_transaction_id  TEXT,                               -- 835 transaction control number
    status              TEXT CHECK (status IN ('paid', 'denied', 'partial', 'adjusted')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_remittance_claim ON remittance(claim_id);
CREATE INDEX idx_remittance_payment_date ON remittance(payment_date);
```

---

## Security & Audit

### User Account

```sql
CREATE TABLE user_account (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organization(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    patient_id          UUID REFERENCES patient(id),
    email               TEXT NOT NULL UNIQUE,
    password_hash       TEXT NOT NULL,
    role                TEXT NOT NULL CHECK (role IN ('admin', 'physician', 'nurse', 'ma', 'front-desk', 'biller', 'patient')),
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    mfa_secret          TEXT,
    last_login_at       TIMESTAMPTZ,
    failed_login_count  INTEGER DEFAULT 0,
    locked_until        TIMESTAMPTZ,
    active              BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_org ON user_account(organization_id);
CREATE INDEX idx_user_email ON user_account(email);
CREATE INDEX idx_user_role ON user_account(role);
```

### Audit Log (HIPAA Compliance)

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID REFERENCES user_account(id),
    patient_id          UUID REFERENCES patient(id),       -- Which patient's data was accessed
    organization_id     UUID REFERENCES organization(id),
    action              TEXT NOT NULL,                      -- CREATE, READ, UPDATE, DELETE, EXPORT, PRINT, LOGIN, LOGOUT
    resource_type       TEXT NOT NULL,                      -- Table/FHIR resource type accessed
    resource_id         UUID,                               -- ID of the specific record
    description         TEXT,
    ip_address          INET,
    user_agent          TEXT,
    session_id          TEXT,
    outcome             TEXT NOT NULL CHECK (outcome IN ('success', 'failure', 'error')),
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

-- Partition by month for retention management
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional monthly partitions created via cron

CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_patient ON audit_log(patient_id);
CREATE INDEX idx_audit_action ON audit_log(action);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_recorded ON audit_log(recorded_at);
```

---

## Coded Concept Reference (Terminology)

```sql
CREATE TABLE coded_concept (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_system     TEXT NOT NULL,                          -- e.g., 'http://snomed.info/sct', 'http://loinc.org'
    code            TEXT NOT NULL,
    display         TEXT NOT NULL,
    version         TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    parent_code     TEXT,                                   -- For hierarchical navigation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (code_system, code)
);

CREATE INDEX idx_coded_concept_system ON coded_concept(code_system);
CREATE INDEX idx_coded_concept_display ON coded_concept USING gin(to_tsvector('english', display));
```

---

## Row-Level Security (Multi-Tenancy)

```sql
-- Enable RLS on all clinical tables
ALTER TABLE patient ENABLE ROW LEVEL SECURITY;
ALTER TABLE encounter ENABLE ROW LEVEL SECURITY;
ALTER TABLE condition ENABLE ROW LEVEL SECURITY;
ALTER TABLE observation ENABLE ROW LEVEL SECURITY;
-- ... repeat for all clinical tables

-- Policy: users can only see data for their organization
CREATE POLICY org_isolation ON patient
    USING (organization_id = current_setting('app.current_organization_id')::uuid);

CREATE POLICY org_isolation ON encounter
    USING (organization_id = current_setting('app.current_organization_id')::uuid);
-- ... repeat for all tables with organization_id
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Entity Management | 2 | organization, practitioner |
| Patient Management | 3 | patient, patient_identifier, coverage |
| Clinical Documentation | 6 | encounter, condition, observation, allergy_intolerance, allergy_reaction, clinical_note |
| Medications | 1 | medication_request |
| Orders & Results | 3 | service_request, diagnostic_report, diagnostic_report_observation |
| Procedures & Immunizations | 2 | procedure, immunization |
| Scheduling | 1 | appointment |
| Billing & Claims | 4 | claim, claim_line, claim_diagnosis, remittance |
| Security & Audit | 2 | user_account, audit_log |
| Reference Data | 1 | coded_concept |
| **Total** | **25** | |

---

## Key Design Decisions

1. **One table per FHIR resource type.** Each major FHIR R4 resource (Patient, Encounter, Condition, Observation, MedicationRequest, etc.) maps to a dedicated PostgreSQL table. This makes FHIR API implementation straightforward — each REST endpoint queries a single primary table.

2. **Polymorphic value columns on Observation.** Rather than using a single generic `value` column, the observation table has typed columns (`value_quantity`, `value_string`, `value_codeable_code`, `value_boolean`, `value_datetime`) matching FHIR's choice-type pattern. Only one is populated per row. This avoids JSONB for the most queried table in the system.

3. **Row-level security for multi-tenancy.** Every clinical table carries an `organization_id` foreign key. PostgreSQL RLS policies enforce tenant isolation at the database layer using `current_setting('app.current_organization_id')`, preventing data leakage even from buggy application code.

4. **Partitioned audit log.** The `audit_log` table is range-partitioned by `recorded_at` month. This supports HIPAA's 6-year retention requirement while allowing old partitions to be archived or moved to cold storage without affecting query performance on recent data.

5. **Explicit billing schema aligned to X12 837/835.** The claim, claim_line, claim_diagnosis, and remittance tables mirror the hierarchical structure of X12 EDI transactions, making claim generation and ERA posting straightforward mapping operations rather than complex transformations.

6. **Terminology reference table.** A single `coded_concept` table stores all terminology lookups (SNOMED CT, LOINC, ICD-10, RxNorm, CVX) with full-text search indexing on display names. This avoids scattering code lookups across clinical tables and provides a single point for terminology updates.

7. **AI provenance on clinical notes.** The `clinical_note` table includes `ai_generated`, `ai_model`, `ai_confidence`, and `clinician_reviewed` columns to track AI-authored content, supporting emerging regulatory requirements for AI transparency in clinical documentation.

8. **UUID primary keys throughout.** All tables use `gen_random_uuid()` for primary keys, supporting distributed systems, FHIR resource identity, and preventing enumeration attacks on clinical data.
