# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Electronic Health Record (EHR) · Created: 2026-05-19

## Philosophy

This model combines relational tables for operational CRUD with a property graph layer for relationship-heavy clinical queries. Core clinical data (patients, encounters, observations, medications) is stored in relational tables for transactional performance, while a parallel graph structure (implemented as `graph_node` and `graph_edge` tables in PostgreSQL, or optionally as a Neo4j/Apache AGE sidecar) captures the rich web of clinical relationships: patient-to-provider care teams, medication-condition associations, referral chains, family medical history connections, care gap dependencies, and AI-discovered correlations.

Healthcare data is inherently graph-shaped. A single patient connects to multiple providers, each encounter links to diagnoses that connect to medications that interact with other medications and allergies, referral chains span organizations, and family relationships carry genetic risk implications. Traditional relational queries for these traversals require complex recursive CTEs and multi-table joins. A graph layer makes these queries natural: "find all providers who treated patients with condition X who are also taking medication Y" or "trace the referral chain for this patient across all organizations."

The graph layer is also the natural substrate for AI-powered features: clinical knowledge graphs, drug interaction networks, conflict-of-interest detection between providers and pharmaceutical companies, and population health network analysis.

**Best for:** AI-native platforms that need clinical knowledge graphs, referral network analysis, drug interaction traversal, family history risk propagation, care team relationship management, and population health network analytics.

**Trade-offs:**
- (+) Natural representation of clinical relationships — care teams, referral chains, drug interactions, family history
- (+) Efficient multi-hop traversals without complex recursive CTEs
- (+) AI-friendly: clinical knowledge graphs, pattern discovery, and graph neural networks for risk prediction
- (+) Supports conflict-of-interest and relationship-disclosure queries for compliance
- (+) Population health network analysis (disease clusters, provider networks, geographic patterns)
- (-) Dual-write complexity — data must be maintained in both relational and graph layers
- (-) Graph consistency with relational source requires synchronization logic
- (-) Teams need graph query expertise (Cypher or Apache AGE's openCypher) in addition to SQL
- (-) Graph layer adds infrastructure complexity (additional database or PostgreSQL extension)
- (-) Not a standard pattern for ONC-certified EHRs — may require justification to auditors

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Relational tables map to FHIR resources for API exposure; graph edges represent FHIR references |
| US Core IG | Must-support elements stored in relational layer; graph layer adds relationship semantics on top |
| FHIR GraphDefinition | FHIR's GraphDefinition resource formally defines subgraphs; the graph layer implements these natively |
| SNOMED CT | SNOMED CT's ontology IS a graph (concept hierarchies, IS-A relationships); the graph layer can import SNOMED subsumption trees |
| RxNorm | Drug-drug interaction data from RxNorm modeled as graph edges between medication nodes |
| IHE PDQ/PIX | Patient identity cross-referencing across organizations modeled as graph edges |
| OMOP CDM | OMOP's concept_relationship and concept_ancestor tables are graph structures; this model generalizes that pattern |
| W3C PROV-O | Provenance relationships (wasGeneratedBy, wasDerivedFrom, wasAttributedTo) modeled as typed graph edges |

---

## Relational Layer (Operational CRUD)

### Core Tables

The relational layer uses a streamlined version of the normalized model (Suggestion 1) for operational CRUD. Only the essential tables are shown; the full set would include all tables from Suggestion 1.

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id       UUID REFERENCES organization(id),
    name            TEXT NOT NULL,
    type            TEXT NOT NULL,
    npi             VARCHAR(10),
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE practitioner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    npi             VARCHAR(10),
    specialty_code  TEXT,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE patient (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    mrn             TEXT NOT NULL,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    date_of_birth   DATE NOT NULL,
    gender          TEXT NOT NULL,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, mrn)
);

CREATE TABLE encounter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    practitioner_id UUID REFERENCES practitioner(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    status          TEXT NOT NULL,
    class_code      TEXT NOT NULL,
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE condition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    encounter_id    UUID REFERENCES encounter(id),
    clinical_status TEXT NOT NULL,
    category        TEXT NOT NULL,
    code_system     TEXT NOT NULL,
    code            TEXT NOT NULL,
    display         TEXT NOT NULL,
    onset_datetime  TIMESTAMPTZ,
    recorded_date   DATE DEFAULT CURRENT_DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    encounter_id    UUID REFERENCES encounter(id),
    status          TEXT NOT NULL,
    category_code   TEXT NOT NULL,
    code_system     TEXT NOT NULL,
    code            TEXT NOT NULL,
    display         TEXT NOT NULL,
    value_quantity  NUMERIC,
    value_unit      TEXT,
    value_string    TEXT,
    effective_datetime TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE medication_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    encounter_id    UUID REFERENCES encounter(id),
    prescriber_id   UUID NOT NULL REFERENCES practitioner(id),
    status          TEXT NOT NULL,
    medication_code TEXT NOT NULL,
    medication_display TEXT NOT NULL,
    dosage_text     TEXT,
    authored_on     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE allergy_intolerance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    clinical_status TEXT NOT NULL,
    category        TEXT,
    criticality     TEXT,
    code_system     TEXT NOT NULL,
    code            TEXT NOT NULL,
    display         TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Standard indexes on patient_id, encounter_id, code, status, etc. (same as Suggestion 1).

---

## Graph Layer

### Graph Node

```sql
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Identity: links to relational layer
    entity_type     TEXT NOT NULL,                          -- 'Patient', 'Practitioner', 'Organization', 'Encounter',
                                                           -- 'Condition', 'Medication', 'Allergen', 'LabTest',
                                                           -- 'SnomedConcept', 'LoincConcept', 'FamilyMember'
    entity_id       UUID,                                  -- FK to relational table (NULL for concept-only nodes)
    -- Node properties
    label           TEXT NOT NULL,                          -- Human-readable label
    code            TEXT,                                   -- Clinical code (SNOMED, LOINC, RxNorm, ICD-10)
    code_system     TEXT,                                   -- Code system URI
    properties      JSONB DEFAULT '{}',                    -- Flexible node properties
    -- Tenant
    organization_id UUID,                                  -- NULL for shared concept nodes (SNOMED, LOINC)
    -- Metadata
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Properties JSONB examples:
-- Patient node:   {"dateOfBirth": "1985-03-15", "gender": "female", "riskScore": 0.72}
-- Medication node: {"rxnormCode": "860975", "drugClass": "biguanide", "schedule": null}
-- SNOMED node:     {"sctid": "44054006", "fsn": "Type 2 diabetes mellitus (disorder)", "hierarchy": "disorder"}

CREATE INDEX idx_gn_entity ON graph_node(entity_type, entity_id);
CREATE INDEX idx_gn_code ON graph_node(code_system, code);
CREATE INDEX idx_gn_org ON graph_node(organization_id);
CREATE INDEX idx_gn_label ON graph_node USING gin(to_tsvector('english', label));
CREATE INDEX idx_gn_properties ON graph_node USING GIN(properties jsonb_path_ops);
```

### Graph Edge

```sql
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Relationship
    source_id       UUID NOT NULL REFERENCES graph_node(id),
    target_id       UUID NOT NULL REFERENCES graph_node(id),
    relationship    TEXT NOT NULL,                          -- See relationship taxonomy below
    -- Edge properties
    properties      JSONB DEFAULT '{}',                    -- Relationship-specific properties
    -- Temporal validity
    valid_from      TIMESTAMPTZ DEFAULT now(),
    valid_to        TIMESTAMPTZ,                            -- NULL = currently active
    -- Provenance
    source_system   TEXT,                                   -- 'ehr-ui', 'ai-inference', 'fhir-import', 'snomed-import'
    confidence      NUMERIC,                                -- AI confidence score (0.0-1.0) for inferred edges
    created_by      UUID,                                   -- User or system that created this edge
    -- Tenant
    organization_id UUID,
    -- Metadata
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Properties JSONB examples:
-- TREATS edge:       {"role": "attending", "startDate": "2026-01-15"}
-- DIAGNOSED_WITH:    {"severity": "moderate", "verificationStatus": "confirmed"}
-- INTERACTS_WITH:    {"severity": "major", "mechanism": "CYP3A4 inhibition", "evidence": "established"}
-- PRESCRIBED_FOR:    {"dosage": "500mg BID", "indication": "E11.9"}
-- FAMILY_HAS:        {"relationship": "mother", "ageAtOnset": 52}

CREATE INDEX idx_ge_source ON graph_edge(source_id);
CREATE INDEX idx_ge_target ON graph_edge(target_id);
CREATE INDEX idx_ge_relationship ON graph_edge(relationship);
CREATE INDEX idx_ge_source_rel ON graph_edge(source_id, relationship);
CREATE INDEX idx_ge_target_rel ON graph_edge(target_id, relationship);
CREATE INDEX idx_ge_temporal ON graph_edge(valid_from, valid_to);
CREATE INDEX idx_ge_org ON graph_edge(organization_id);
CREATE INDEX idx_ge_confidence ON graph_edge(confidence) WHERE confidence IS NOT NULL;
CREATE INDEX idx_ge_properties ON graph_edge USING GIN(properties jsonb_path_ops);
```

### Relationship Taxonomy

```sql
-- Clinical Relationships
-- Patient → Practitioner:   TREATED_BY, PRIMARY_CARE_PROVIDER, REFERRED_BY, REFERRED_TO
-- Patient → Condition:      DIAGNOSED_WITH, HISTORY_OF, FAMILY_HISTORY_OF
-- Patient → Medication:     TAKING, PREVIOUSLY_TOOK
-- Patient → Allergen:       ALLERGIC_TO
-- Patient → Encounter:      HAD_ENCOUNTER
-- Patient → Patient:        FAMILY_MEMBER (via FamilyMember proxy node)
-- Patient → Organization:   REGISTERED_AT

-- Encounter Relationships
-- Encounter → Condition:    RESULTED_IN_DIAGNOSIS
-- Encounter → Observation:  PRODUCED_OBSERVATION
-- Encounter → Medication:   PRESCRIBED_MEDICATION
-- Encounter → Procedure:    PERFORMED_PROCEDURE
-- Encounter → Practitioner: ATTENDED_BY

-- Clinical Knowledge
-- Condition → Condition:    COMORBID_WITH, COMPLICATION_OF, RISK_FACTOR_FOR
-- Medication → Medication:  INTERACTS_WITH, CONTRAINDICATED_WITH, ALTERNATIVE_TO
-- Medication → Condition:   INDICATED_FOR, CONTRAINDICATED_FOR
-- Medication → Allergen:    CROSS_REACTIVE_WITH
-- Condition → LabTest:      DIAGNOSTIC_FOR, MONITORING_FOR

-- Organizational
-- Practitioner → Organization:  AFFILIATED_WITH
-- Organization → Organization:  PARENT_OF, REFERRAL_NETWORK_WITH
-- Practitioner → Practitioner:  REFERS_TO, COLLABORATES_WITH

-- Ontology (SNOMED / ICD hierarchy)
-- SnomedConcept → SnomedConcept: IS_A, PART_OF, FINDING_SITE
-- LoincConcept → LoincConcept:   MEMBER_OF_PANEL

-- AI-Inferred
-- Patient → Condition:      AI_SUSPECTS (confidence score in properties)
-- Patient → Patient:        SIMILAR_PHENOTYPE (AI-computed similarity)
-- Condition → Condition:    AI_CORRELATED (discovered from population data)
```

---

## AI-Specific Graph Features

### AI Inference Log

```sql
CREATE TABLE ai_graph_inference (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_id         UUID REFERENCES graph_edge(id),        -- The edge this inference created/updated
    model_id        TEXT NOT NULL,                          -- AI model identifier
    model_version   TEXT NOT NULL,
    inference_type  TEXT NOT NULL,                          -- 'risk_prediction', 'drug_interaction', 'dx_suggestion', 'phenotype_similarity'
    input_summary   JSONB,                                 -- Summary of input features
    confidence      NUMERIC NOT NULL,                      -- 0.0 - 1.0
    explanation     TEXT,                                   -- Human-readable explanation
    clinician_reviewed BOOLEAN NOT NULL DEFAULT FALSE,
    reviewed_by     UUID,
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_aigi_edge ON ai_graph_inference(edge_id);
CREATE INDEX idx_aigi_model ON ai_graph_inference(model_id);
CREATE INDEX idx_aigi_type ON ai_graph_inference(inference_type);
CREATE INDEX idx_aigi_reviewed ON ai_graph_inference(clinician_reviewed);
```

### Clinical Knowledge Graph Import

```sql
-- Preloaded SNOMED CT subsumption hierarchy as graph nodes/edges
-- Example: importing the SNOMED "disorder" hierarchy

-- Step 1: Import SNOMED concepts as graph nodes
INSERT INTO graph_node (entity_type, label, code, code_system, organization_id, properties)
SELECT
    'SnomedConcept',
    fsn,                                                    -- Fully specified name
    concept_id::text,
    'http://snomed.info/sct',
    NULL,                                                   -- Shared across all orgs
    jsonb_build_object('hierarchy', hierarchy_tag, 'active', active)
FROM snomed_concept_staging
WHERE hierarchy_tag = 'disorder';

-- Step 2: Import IS-A relationships as graph edges
INSERT INTO graph_edge (source_id, target_id, relationship, source_system)
SELECT
    child_node.id,
    parent_node.id,
    'IS_A',
    'snomed-import'
FROM snomed_relationship_staging sr
JOIN graph_node child_node ON child_node.code = sr.source_id::text AND child_node.code_system = 'http://snomed.info/sct'
JOIN graph_node parent_node ON parent_node.code = sr.destination_id::text AND parent_node.code_system = 'http://snomed.info/sct'
WHERE sr.type_id = '116680003';  -- IS-A relationship type
```

---

## Graph Query Examples

### Using PostgreSQL Recursive CTEs (No Graph Extension Required)

#### Find All Providers in a Patient's Care Network (2 Hops)

```sql
-- Find all practitioners connected to patient X through encounters and referrals
WITH care_network AS (
    -- Start: patient node
    SELECT
        ge.target_id AS node_id,
        ge.relationship,
        gn.entity_type,
        gn.label,
        1 AS depth
    FROM graph_node gn_patient
    JOIN graph_edge ge ON ge.source_id = gn_patient.id
    JOIN graph_node gn ON gn.id = ge.target_id
    WHERE gn_patient.entity_type = 'Patient'
      AND gn_patient.entity_id = 'patient-uuid-here'
      AND ge.relationship IN ('TREATED_BY', 'REFERRED_TO', 'PRIMARY_CARE_PROVIDER')
      AND (ge.valid_to IS NULL OR ge.valid_to > now())

    UNION ALL

    -- Hop: provider → other providers they refer to
    SELECT
        ge.target_id,
        ge.relationship,
        gn.entity_type,
        gn.label,
        cn.depth + 1
    FROM care_network cn
    JOIN graph_edge ge ON ge.source_id = cn.node_id
    JOIN graph_node gn ON gn.id = ge.target_id
    WHERE gn.entity_type = 'Practitioner'
      AND ge.relationship IN ('REFERS_TO', 'COLLABORATES_WITH')
      AND cn.depth < 2
)
SELECT DISTINCT node_id, label, relationship, depth
FROM care_network
WHERE entity_type = 'Practitioner'
ORDER BY depth, label;
```

#### Drug Interaction Check

```sql
-- Check for interactions between all active medications for a patient
WITH patient_meds AS (
    SELECT gn_med.id AS med_node_id, gn_med.label, gn_med.code
    FROM graph_node gn_patient
    JOIN graph_edge ge ON ge.source_id = gn_patient.id AND ge.relationship = 'TAKING'
    JOIN graph_node gn_med ON gn_med.id = ge.target_id AND gn_med.entity_type = 'Medication'
    WHERE gn_patient.entity_type = 'Patient'
      AND gn_patient.entity_id = 'patient-uuid-here'
      AND (ge.valid_to IS NULL OR ge.valid_to > now())
)
SELECT
    m1.label AS medication_1,
    m2.label AS medication_2,
    ge.properties->>'severity' AS interaction_severity,
    ge.properties->>'mechanism' AS mechanism,
    ge.properties->>'evidence' AS evidence_level
FROM patient_meds m1
JOIN graph_edge ge ON ge.source_id = m1.med_node_id AND ge.relationship = 'INTERACTS_WITH'
JOIN patient_meds m2 ON m2.med_node_id = ge.target_id
WHERE m1.med_node_id < m2.med_node_id;  -- Avoid duplicate pairs
```

#### Family History Risk Propagation

```sql
-- Find conditions in patient's family history and their SNOMED ancestors
WITH family_conditions AS (
    SELECT
        gn_condition.code,
        gn_condition.label AS condition_name,
        ge_family.properties->>'relationship' AS family_relationship,
        ge_family.properties->>'ageAtOnset' AS age_at_onset
    FROM graph_node gn_patient
    JOIN graph_edge ge_family ON ge_family.source_id = gn_patient.id AND ge_family.relationship = 'FAMILY_HISTORY_OF'
    JOIN graph_node gn_condition ON gn_condition.id = ge_family.target_id
    WHERE gn_patient.entity_type = 'Patient'
      AND gn_patient.entity_id = 'patient-uuid-here'
),
risk_hierarchy AS (
    -- Walk up the SNOMED IS-A hierarchy to find broader risk categories
    SELECT
        fc.condition_name,
        fc.family_relationship,
        gn_parent.label AS risk_category,
        gn_parent.code AS risk_code,
        1 AS hierarchy_depth
    FROM family_conditions fc
    JOIN graph_node gn_cond ON gn_cond.code = fc.code AND gn_cond.code_system = 'http://snomed.info/sct'
    JOIN graph_edge ge_isa ON ge_isa.source_id = gn_cond.id AND ge_isa.relationship = 'IS_A'
    JOIN graph_node gn_parent ON gn_parent.id = ge_isa.target_id

    UNION ALL

    SELECT
        rh.condition_name,
        rh.family_relationship,
        gn_parent.label,
        gn_parent.code,
        rh.hierarchy_depth + 1
    FROM risk_hierarchy rh
    JOIN graph_node gn_cur ON gn_cur.code = rh.risk_code AND gn_cur.code_system = 'http://snomed.info/sct'
    JOIN graph_edge ge_isa ON ge_isa.source_id = gn_cur.id AND ge_isa.relationship = 'IS_A'
    JOIN graph_node gn_parent ON gn_parent.id = ge_isa.target_id
    WHERE rh.hierarchy_depth < 3
)
SELECT DISTINCT condition_name, family_relationship, risk_category
FROM risk_hierarchy
ORDER BY condition_name;
```

#### AI-Discovered Patient Phenotype Similarity

```sql
-- Find patients with similar clinical phenotypes (AI-inferred edges)
SELECT
    gn_similar.entity_id AS similar_patient_id,
    gn_similar.label AS patient_name,
    ge.confidence AS similarity_score,
    ge.properties->>'sharedConditions' AS shared_conditions,
    ge.properties->>'sharedMedications' AS shared_medications,
    ai.explanation
FROM graph_node gn_patient
JOIN graph_edge ge ON ge.source_id = gn_patient.id AND ge.relationship = 'SIMILAR_PHENOTYPE'
JOIN graph_node gn_similar ON gn_similar.id = ge.target_id
LEFT JOIN ai_graph_inference ai ON ai.edge_id = ge.id
WHERE gn_patient.entity_type = 'Patient'
  AND gn_patient.entity_id = 'patient-uuid-here'
  AND ge.confidence >= 0.8
ORDER BY ge.confidence DESC
LIMIT 10;
```

---

## Security & Audit

```sql
CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    practitioner_id UUID REFERENCES practitioner(id),
    email           TEXT NOT NULL UNIQUE,
    password_hash   TEXT NOT NULL,
    role            TEXT NOT NULL,
    mfa_enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES user_account(id),
    patient_id      UUID,
    organization_id UUID,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    graph_edge_id   UUID,                                  -- If the action involved a graph edge
    description     TEXT,
    ip_address      INET,
    outcome         TEXT NOT NULL CHECK (outcome IN ('success', 'failure', 'error')),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_patient ON audit_log(patient_id);
CREATE INDEX idx_audit_recorded ON audit_log(recorded_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Relational (Clinical) | 7 | organization, practitioner, patient, encounter, condition, observation, medication_request, allergy_intolerance |
| Graph Layer | 2 | graph_node, graph_edge |
| AI Inference | 1 | ai_graph_inference |
| Security & Audit | 2 | user_account, audit_log |
| **Total** | **12** | Plus the full relational set from Suggestion 1 for complete clinical coverage; graph layer adds 3 tables |

---

## Key Design Decisions

1. **Dual-layer architecture: relational for CRUD, graph for relationships.** The relational layer handles day-to-day clinical operations (scheduling, documentation, orders, billing) with strong transactional guarantees. The graph layer captures relationship semantics that are awkward in relational tables. Neither layer is redundant — they serve different query patterns.

2. **Graph implemented in PostgreSQL (not a separate database).** Using `graph_node` and `graph_edge` tables in PostgreSQL keeps the infrastructure simple: single database, single transaction boundary, familiar SQL. For teams that need advanced graph algorithms (PageRank, community detection, shortest path), Apache AGE (PostgreSQL extension for openCypher queries) or a Neo4j sidecar can be added without changing the schema.

3. **Temporal validity on edges.** Graph edges have `valid_from` and `valid_to` timestamps, enabling temporal graph queries: "who was this patient's care team on March 15?" or "what medications were they taking when this adverse event occurred?" This is critical for clinical timeline reconstruction.

4. **AI inference tracking.** The `ai_graph_inference` table records every AI-generated graph edge with model metadata, confidence score, and clinician review status. This creates a clear separation between clinician-asserted relationships and AI-inferred ones, supporting regulatory requirements and clinical trust.

5. **Confidence scores on edges.** AI-inferred edges (drug interaction predictions, phenotype similarity, suspected diagnoses) carry a numeric confidence score. Clinical workflows can filter by confidence threshold, showing high-confidence suggestions prominently and hiding low-confidence ones behind a "show more" interaction.

6. **SNOMED CT as a native graph.** The SNOMED CT ontology is imported into the graph layer as concept nodes with IS-A edges. This enables subsumption queries ("find all patients with any type of diabetes, including type 1, type 2, gestational, and all sub-types") without maintaining separate value set expansion logic.

7. **Relationship taxonomy is extensible.** The `relationship` column on `graph_edge` is a TEXT field, not an enum. New relationship types can be added without schema changes. The taxonomy above is the initial set; AI models may discover and propose new relationship types at runtime.

8. **Organization-scoped graph with shared concept nodes.** Patient and clinical nodes are scoped to an `organization_id` for multi-tenancy. Concept nodes (SNOMED, LOINC, RxNorm) have `organization_id = NULL` and are shared across all tenants, avoiding redundant imports of clinical terminology graphs.
