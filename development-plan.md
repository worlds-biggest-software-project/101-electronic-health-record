# Electronic Health Record — Phased Development Plan
> Project: 101-electronic-health-record · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

### Language & Runtime
| Layer | Choice | Rationale |
|-------|--------|-----------|
| API Server | **TypeScript / Node.js (Fastify)** | Medplum (Apache-2.0) ecosystem is TypeScript-native; Fastify provides schema-based validation, high throughput, and OpenAPI generation; strong FHIR R4 type libraries available |
| Clinical UI | **React + TypeScript** | Medplum React component library provides FHIR-aware UI primitives; dominant framework for healthcare SaaS; largest hiring pool |
| Patient Portal | **Next.js (React)** | Server-side rendering for SEO and performance; API routes for BFF pattern; Vercel/self-host deployment flexibility |
| AI Services | **Python (FastAPI)** | ML ecosystem (transformers, scikit-learn, spaCy, medspacy) is Python-native; FastAPI for async inference endpoints; gRPC option for internal calls |
| Background Workers | **TypeScript (BullMQ on Redis)** | Shared codebase with API server; BullMQ for reliable job queuing (claim submission, HL7 v2 translation, FHIR bulk export) |

### Data Architecture — Hybrid Relational + JSONB (Data Model Suggestion 3) with Event Log
| Decision | Rationale |
|----------|-----------|
| **Primary store: PostgreSQL with FHIR-native JSONB tables** (Suggestion 3) | Fastest path to ONC-certifiable FHIR API; JSONB `content` column stores verbatim FHIR R4 resources; extracted search parameter columns provide indexed SQL queries without full-document scans; proven in production by Medplum, Fhirbase, and Aidbox |
| **Supplementary: append-only audit event table** (from Suggestion 2) | Immutable `audit_event` table satisfies HIPAA 45 CFR 164.312(b) without bolt-on audit; partitioned by month for retention management; supports temporal queries |
| **Graph layer deferred to Phase 9** (from Suggestion 4) | Graph-relational hybrid adds value for AI knowledge graphs and drug interaction traversal but introduces dual-write complexity; defer until clinical CRUD and FHIR API are stable |
| **PostgreSQL 16+** | JSONB, GIN indexes, row-level security, table partitioning, and pg_cron for partition management all available natively |
| **Redis 7+** | Session store, BullMQ job queue, FHIR subscription notifications, rate limiting |

### Terminology & Code Systems
| System | Source | Licence |
|--------|--------|---------|
| SNOMED CT | NLM UMLS (US licence — free) | Free for US-based EHR developers |
| LOINC | Regenstrief Institute download | Free (Regenstrief licence) |
| RxNorm | NLM UMLS | Free |
| ICD-10-CM/PCS | CMS/NCHS | Free (US government) |
| CVX (vaccines) | CDC | Free |
| CPT | AMA | **Paid licence required** — budget AMA licence before Phase 5 (billing) |

### Infrastructure & Deployment
| Component | Choice |
|-----------|--------|
| Container runtime | Docker + Docker Compose (dev); Kubernetes (staging/prod) |
| CI/CD | GitHub Actions |
| Object storage | S3-compatible (documents, CDA attachments, audio recordings) |
| Secrets management | HashiCorp Vault or AWS Secrets Manager |
| Observability | OpenTelemetry → Grafana (Loki + Tempo + Mimir) |
| TLS termination | Nginx or Envoy (TLS 1.2+ per HIPAA) |

### Project Structure
```
ehr/
├── packages/
│   ├── core/                 # Shared types, FHIR interfaces, validation
│   ├── server/               # Fastify API server
│   ├── worker/               # BullMQ background workers
│   ├── fhir/                 # FHIR R4 resource handlers, search, operations
│   ├── auth/                 # OAuth2 / OIDC / SMART on FHIR
│   ├── clinical-ui/          # React clinical application
│   ├── patient-portal/       # Next.js patient-facing app
│   └── ai/                   # Python AI/ML services
├── migrations/               # PostgreSQL migrations (node-pg-migrate)
├── seeds/                    # Terminology imports (SNOMED, LOINC, RxNorm)
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── infra/                    # Docker, Kubernetes, Terraform
├── docs/                     # Architecture decision records
└── package.json              # pnpm workspace root
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    │
    ├──► Phase 2: Patient & Clinical Core
    │        │
    │        ├──► Phase 3: FHIR R4 API & SMART on FHIR
    │        │        │
    │        │        ├──► Phase 4: Clinical Documentation & Notes
    │        │        │        │
    │        │        │        ├──► Phase 5: Billing & Revenue Cycle
    │        │        │        │        │
    │        │        │        │        └──► Phase 8: Population Health & Analytics
    │        │        │        │
    │        │        │        └──► Phase 6: AI Ambient Documentation
    │        │        │                 │
    │        │        │                 └──► Phase 7: AI Clinical Intelligence
    │        │        │                          │
    │        │        │                          └──► Phase 9: Knowledge Graph & Advanced AI
    │        │        │
    │        │        └──► Phase 10: Interoperability Bridge (HL7 v2, CDA)
    │        │
    │        └──► Phase 11: Patient Portal & Conversational AI
    │
    └──► Phase 12: ONC Certification & Compliance Hardening
```

**Critical path:** 1 → 2 → 3 → 4 → 6 → 7 → 9

---

## Phase 1: Foundation — Infrastructure, Auth, and Multi-Tenancy

**Goal:** Establish the monorepo, database, authentication, multi-tenant isolation, and audit logging so that all subsequent phases build on a secure, observable, HIPAA-ready foundation.

**Definition of Done:**
- [ ] Monorepo builds, lints, and passes CI with zero errors
- [ ] PostgreSQL schema deployed via migrations with RLS enforced
- [ ] OAuth2/OIDC login flow works for practitioner and admin roles
- [ ] MFA enrollment and verification operational
- [ ] Every authenticated request produces an audit log entry
- [ ] Row-level security prevents cross-tenant data access (integration test)
- [ ] Docker Compose starts all services with `docker compose up`
- [ ] OpenTelemetry traces visible in Grafana

### Task 1.1: Monorepo Scaffold and Build System

**What:** Initialize a pnpm workspace monorepo with TypeScript, ESLint, Prettier, and shared tsconfig.

**Design:**
```typescript
// packages/core/src/index.ts — re-exported shared types
export type { FhirResource, FhirBundle, FhirOperationOutcome } from './fhir-types';
export type { TenantContext, AuthenticatedUser } from './auth-types';
export { AppError, NotFoundError, ForbiddenError, ValidationError } from './errors';
```

```typescript
// packages/core/src/auth-types.ts
export interface TenantContext {
  organizationId: string;   // UUID
  userId: string;            // UUID
  role: UserRole;
  scopes: string[];          // SMART on FHIR scopes
}

export type UserRole = 'admin' | 'physician' | 'nurse' | 'ma' | 'front-desk' | 'biller' | 'patient';

export interface AuthenticatedUser {
  id: string;
  email: string;
  organizationId: string;
  practitionerId?: string;
  patientId?: string;
  role: UserRole;
  mfaVerified: boolean;
}
```

```jsonc
// pnpm-workspace.yaml
packages:
  - "packages/*"
  - "migrations"
  - "tests"
```

**Testing:**
- `pnpm install` completes without errors
- `pnpm -r build` compiles all packages
- `pnpm -r lint` passes with zero warnings
- `pnpm -r test` runs (empty test suites OK at this stage)

### Task 1.2: PostgreSQL Schema — Organizations, Users, and Audit Log

**What:** Create the initial migration establishing multi-tenant organization hierarchy, user accounts with MFA, and a partitioned audit log table.

**Design:**
```sql
-- Migration: 001_foundation.sql

CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE "Organization" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    name            TEXT NOT NULL,
    identifier      TEXT[],
    type_code       TEXT[],
    partof          UUID REFERENCES "Organization"(id),
    active          BOOLEAN DEFAULT TRUE,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_name ON "Organization"(name);
CREATE INDEX idx_org_content ON "Organization" USING GIN(content jsonb_path_ops);

CREATE TABLE user_account (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES "Organization"(id),
    email               TEXT NOT NULL UNIQUE,
    password_hash       TEXT NOT NULL,
    role                TEXT NOT NULL CHECK (role IN ('admin','physician','nurse','ma','front-desk','biller','patient')),
    scopes              TEXT[],
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    mfa_secret          TEXT,
    mfa_recovery_codes  TEXT[],
    last_login_at       TIMESTAMPTZ,
    failed_login_count  INTEGER DEFAULT 0,
    locked_until        TIMESTAMPTZ,
    active              BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_org ON user_account(organization_id);

CREATE TABLE audit_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID,
    user_id         UUID,
    patient_id      UUID,
    action          TEXT NOT NULL,          -- C, R, U, D, E (FHIR AuditEvent action codes)
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    outcome         TEXT NOT NULL DEFAULT '0',  -- 0=success, 4=minor, 8=serious, 12=major
    description     TEXT,
    ip_address      INET,
    user_agent      TEXT,
    session_id      TEXT,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

-- Create initial partition
CREATE TABLE audit_event_2026_06 PARTITION OF audit_event
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_audit_user ON audit_event(user_id);
CREATE INDEX idx_audit_patient ON audit_event(patient_id);
CREATE INDEX idx_audit_recorded ON audit_event(recorded_at);
CREATE INDEX idx_audit_resource ON audit_event(resource_type, resource_id);
```

**Testing:**
- Migration runs forward and rolls back cleanly
- `INSERT INTO "Organization"` succeeds; verify `content` JSONB stores valid FHIR Organization
- `INSERT INTO user_account` with invalid role is rejected by CHECK constraint
- Audit event partitions accept rows within the date range; rows outside range are rejected

### Task 1.3: Fastify Server with OAuth2/OIDC Authentication

**What:** Stand up the Fastify HTTP server with JWT-based authentication, OIDC discovery endpoint, and session management backed by Redis.

**Design:**
```typescript
// packages/server/src/app.ts
import Fastify from 'fastify';
import cors from '@fastify/cors';
import helmet from '@fastify/helmet';
import rateLimit from '@fastify/rate-limit';
import { authPlugin } from '@ehr/auth';
import { auditPlugin } from './plugins/audit';
import { tenantPlugin } from './plugins/tenant';

export async function buildApp() {
  const app = Fastify({ logger: true });

  await app.register(cors, { origin: process.env.CORS_ORIGIN });
  await app.register(helmet);
  await app.register(rateLimit, { max: 100, timeWindow: '1 minute' });
  await app.register(authPlugin);
  await app.register(tenantPlugin);
  await app.register(auditPlugin);

  return app;
}
```

```typescript
// packages/auth/src/index.ts
export interface TokenPayload {
  sub: string;              // user_account.id
  org: string;              // organization_id
  role: UserRole;
  scopes: string[];
  mfa: boolean;
  iat: number;
  exp: number;
}

export interface AuthRoutes {
  'POST /auth/login': { body: { email: string; password: string }; response: { accessToken: string; refreshToken: string } };
  'POST /auth/mfa/verify': { body: { code: string }; response: { accessToken: string } };
  'POST /auth/refresh': { body: { refreshToken: string }; response: { accessToken: string } };
  'POST /auth/logout': { response: { success: boolean } };
  'GET /.well-known/openid-configuration': { response: OIDCDiscoveryDocument };
}
```

**Testing:**
- `POST /auth/login` with valid credentials returns access + refresh tokens
- `POST /auth/login` with invalid password returns 401 and increments `failed_login_count`
- After 5 failed attempts, account is locked; `POST /auth/login` returns 423
- `POST /auth/mfa/verify` with valid TOTP code returns upgraded token with `mfa: true`
- Expired access token returns 401; refresh flow issues new access token
- `GET /.well-known/openid-configuration` returns valid OIDC discovery document
- Every authenticated request creates an `audit_event` row with correct `user_id`, `action`, and `ip_address`

### Task 1.4: Multi-Tenancy with Row-Level Security

**What:** Implement PostgreSQL RLS policies and a Fastify plugin that sets the tenant context on every database connection.

**Design:**
```typescript
// packages/server/src/plugins/tenant.ts
import { FastifyPluginAsync } from 'fastify';
import fp from 'fastify-plugin';
import { Pool } from 'pg';

const tenantPlugin: FastifyPluginAsync = async (app) => {
  app.decorateRequest('tenantId', '');

  app.addHook('preHandler', async (request) => {
    const token = request.user as TokenPayload;
    request.tenantId = token.org;
  });

  // Wrap DB pool.query to inject tenant context
  app.decorate('db', {
    async query(sql: string, params: unknown[], tenantId: string) {
      const client = await pool.connect();
      try {
        await client.query(`SET LOCAL app.current_organization_id = '${tenantId}'`);
        return await client.query(sql, params);
      } finally {
        client.release();
      }
    }
  });
};
```

```sql
-- Applied to every FHIR resource table (Phase 2+)
ALTER TABLE "Patient" ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON "Patient"
    USING (organization_id = current_setting('app.current_organization_id')::uuid);
```

**Testing:**
- Create two organizations A and B with patients in each
- Authenticate as org A user; `SELECT * FROM "Patient"` returns only org A patients
- Direct SQL bypass attempt (no `SET LOCAL`) returns zero rows when RLS is enforced
- Cross-tenant patient ID in URL path returns 404, not 403 (no information leakage)

### Task 1.5: Docker Compose and Observability

**What:** Create a Docker Compose file that starts PostgreSQL, Redis, the API server, and an OpenTelemetry collector with Grafana dashboards.

**Design:**
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ehr
      POSTGRES_USER: ehr
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports: ["5432:5432"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  api:
    build: { context: ., dockerfile: packages/server/Dockerfile }
    depends_on: [postgres, redis]
    environment:
      DATABASE_URL: postgres://ehr:${DB_PASSWORD}@postgres:5432/ehr
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET}
    ports: ["3000:3000"]

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./infra/otel-config.yaml:/etc/otelcol/config.yaml

  grafana:
    image: grafana/grafana:latest
    ports: ["3001:3000"]
    volumes:
      - ./infra/grafana/dashboards:/var/lib/grafana/dashboards

volumes:
  pgdata:
```

**Testing:**
- `docker compose up -d` starts all services within 60 seconds
- `curl http://localhost:3000/health` returns `{"status": "ok"}`
- PostgreSQL accepts connections on port 5432
- Redis PING returns PONG
- Grafana is accessible at port 3001
- API traces appear in Grafana Tempo after making authenticated requests

---

## Phase 2: Patient & Clinical Core

**Goal:** Implement the core FHIR resource tables and CRUD operations for Patient, Encounter, Condition, Observation, AllergyIntolerance, and MedicationRequest. This phase produces a working clinical data repository.

**Definition of Done:**
- [ ] All six FHIR resource tables created with JSONB `content` + extracted search parameters
- [ ] REST CRUD (create, read, update, delete) for each resource type
- [ ] FHIR search parameters (patient, code, status, date, category) return correct results
- [ ] Resource versioning: `vread` returns historical versions
- [ ] RLS enforced on all tables; cross-tenant isolation verified
- [ ] 90%+ unit test coverage on resource handlers

### Task 2.1: Patient Resource Table and CRUD

**What:** Create the Patient FHIR resource table and implement create/read/update/soft-delete with extracted search parameters.

**Design:**
```sql
-- Migration: 002_patient.sql
CREATE TABLE "Patient" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID NOT NULL,
    family          TEXT,
    given           TEXT[],
    birthdate       DATE,
    gender          TEXT,
    identifier      TEXT[],
    phone           TEXT[],
    email           TEXT[],
    address_city    TEXT,
    address_state   TEXT,
    address_postalcode TEXT,
    active          BOOLEAN DEFAULT TRUE,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE "Patient" ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON "Patient"
    USING (organization_id = current_setting('app.current_organization_id')::uuid);

CREATE INDEX idx_patient_org ON "Patient"(organization_id);
CREATE INDEX idx_patient_name ON "Patient"(family, given);
CREATE INDEX idx_patient_birthdate ON "Patient"(birthdate);
CREATE INDEX idx_patient_identifier ON "Patient" USING GIN(identifier);
CREATE INDEX idx_patient_content ON "Patient" USING GIN(content jsonb_path_ops);

CREATE TABLE resource_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    version_id      INTEGER NOT NULL,
    content         JSONB NOT NULL,
    author_id       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (resource_type, resource_id, version_id)
);
CREATE INDEX idx_rv_resource ON resource_version(resource_type, resource_id);
```

```typescript
// packages/fhir/src/resources/patient.ts
import { FhirPatient } from '@ehr/core';

export interface PatientRepository {
  create(patient: FhirPatient, orgId: string, authorId: string): Promise<FhirPatient>;
  read(id: string): Promise<FhirPatient | null>;
  vread(id: string, versionId: number): Promise<FhirPatient | null>;
  update(id: string, patient: FhirPatient, authorId: string): Promise<FhirPatient>;
  delete(id: string, authorId: string): Promise<void>;
  search(params: PatientSearchParams): Promise<FhirBundle>;
  history(id: string): Promise<FhirBundle>;
}

export interface PatientSearchParams {
  family?: string;
  given?: string;
  birthdate?: string;      // FHIR date prefix: eq, lt, gt, ge, le
  gender?: string;
  identifier?: string;     // system|value
  _count?: number;
  _offset?: number;
}

function extractSearchParams(patient: FhirPatient): Record<string, unknown> {
  return {
    family: patient.name?.[0]?.family ?? null,
    given: patient.name?.[0]?.given ?? [],
    birthdate: patient.birthDate ?? null,
    gender: patient.gender ?? null,
    identifier: patient.identifier?.map(id => id.value) ?? [],
    phone: patient.telecom?.filter(t => t.system === 'phone').map(t => t.value) ?? [],
    email: patient.telecom?.filter(t => t.system === 'email').map(t => t.value) ?? [],
    address_city: patient.address?.[0]?.city ?? null,
    address_state: patient.address?.[0]?.state ?? null,
    address_postalcode: patient.address?.[0]?.postalCode ?? null,
    active: patient.active ?? true,
  };
}
```

**Testing:**
- Create a Patient with full US Core profile; verify `content` JSONB matches input
- Read by ID returns the Patient; read with wrong org returns null (RLS)
- Update increments `version_id`; previous version stored in `resource_version`
- `vread` with version 1 returns original; version 2 returns updated
- Delete sets `is_deleted = TRUE`; subsequent read returns null
- Search by `family=Smith` returns matching patients; search by `birthdate=ge1985-01-01` filters correctly
- Search by `identifier=MRN-2026-00042` returns exact match

### Task 2.2: Encounter Resource Table and CRUD

**What:** Create the Encounter table and implement CRUD with search by patient, status, date range, and class.

**Design:**
```sql
-- Migration: 003_encounter.sql
CREATE TABLE "Encounter" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID NOT NULL,
    patient         UUID NOT NULL,
    participant     UUID[],
    status          TEXT,
    class_code      TEXT,
    type_code       TEXT[],
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    reason_code     TEXT[],
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE "Encounter" ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON "Encounter"
    USING (organization_id = current_setting('app.current_organization_id')::uuid);
```

```typescript
// packages/fhir/src/resources/encounter.ts
export interface EncounterSearchParams {
  patient?: string;
  status?: string;
  class?: string;
  date?: string;           // ge/le date range
  participant?: string;
  type?: string;
  _count?: number;
  _offset?: number;
}
```

**Testing:**
- Create an Encounter linked to an existing Patient
- Search `?patient=<uuid>&status=in-progress` returns active encounters
- Search `?date=ge2026-01-01&date=le2026-06-01` returns encounters in range
- Update encounter status from `in-progress` to `finished`; verify `period_end` populated
- Cross-patient encounter creation (patient from different org) rejected by RLS

### Task 2.3: Condition, Observation, AllergyIntolerance, MedicationRequest

**What:** Create the remaining four core clinical resource tables with CRUD and FHIR search.

**Design:**
```sql
-- Migration: 004_clinical_resources.sql
-- Condition, Observation, AllergyIntolerance, MedicationRequest tables
-- (Structure follows the same pattern as Patient and Encounter above)
-- Each table has: id, version_id, last_updated, content JSONB, organization_id,
-- patient UUID, extracted search parameters, is_deleted, created_at
-- Each table has RLS enabled with tenant_isolation policy
```

```typescript
// packages/fhir/src/resources/condition.ts
export interface ConditionSearchParams {
  patient?: string;
  'clinical-status'?: string;
  'verification-status'?: string;
  category?: string;       // problem-list-item | encounter-diagnosis
  code?: string;            // SNOMED or ICD-10 code
  'onset-date'?: string;
  _count?: number;
}

// packages/fhir/src/resources/observation.ts
export interface ObservationSearchParams {
  patient?: string;
  category?: string;       // vital-signs | laboratory | social-history
  code?: string;            // LOINC code
  date?: string;
  status?: string;
  'value-quantity'?: string; // gt/lt comparators
  _count?: number;
}

// packages/fhir/src/resources/medication-request.ts
export interface MedicationRequestSearchParams {
  patient?: string;
  status?: string;
  intent?: string;
  medication?: string;      // RxNorm code
  authoredon?: string;
  _count?: number;
}

// packages/fhir/src/resources/allergy-intolerance.ts
export interface AllergyIntoleranceSearchParams {
  patient?: string;
  'clinical-status'?: string;
  type?: string;
  category?: string;
  criticality?: string;
  code?: string;
  _count?: number;
}
```

**Testing:**
- Create Condition with SNOMED code `44054006` (Type 2 diabetes); verify search by code returns it
- Create Observation with LOINC code `85354-9` (blood pressure panel); verify search by category `vital-signs` returns it
- Create AllergyIntolerance with criticality `high`; verify search filters correctly
- Create MedicationRequest with RxNorm code `860975` (Metformin); verify search by status `active` returns it
- Each resource type: verify CRUD, versioning, soft delete, and RLS isolation

### Task 2.4: FHIR Validation Layer

**What:** Implement a validation service that validates incoming FHIR resources against US Core profiles before persistence.

**Design:**
```typescript
// packages/fhir/src/validation/validator.ts
export interface ValidationResult {
  valid: boolean;
  issues: ValidationIssue[];
}

export interface ValidationIssue {
  severity: 'error' | 'warning' | 'information';
  code: string;
  diagnostics: string;
  expression: string[];     // FHIRPath to the element
}

export interface FhirValidator {
  validate(resource: FhirResource, profile?: string): Promise<ValidationResult>;
  validateBundle(bundle: FhirBundle): Promise<ValidationResult>;
}

// Required validations for US Core Patient:
// - Patient.identifier MUST have at least one identifier
// - Patient.name MUST have at least one name with family
// - Patient.gender MUST be present
// - US Core Race extension must use OMB race categories
// - US Core Ethnicity extension must use OMB ethnicity categories
```

**Testing:**
- Valid US Core Patient passes validation with zero errors
- Patient missing `gender` fails with error on `Patient.gender`
- Patient with invalid race code fails with error on US Core Race extension
- Observation with missing `status` fails validation
- MedicationRequest with invalid `intent` value fails validation
- Bundle with mixed valid/invalid resources returns per-resource issues

---

## Phase 3: FHIR R4 API & SMART on FHIR

**Goal:** Expose a standards-compliant FHIR R4 RESTful API with SMART on FHIR authorization, capability statement, and FHIR search. This is the public-facing API that third-party apps, other EHRs, and the clinical UI will consume.

**Definition of Done:**
- [ ] FHIR RESTful API at `/fhir/R4/[ResourceType]/[id]` for all Phase 2 resource types
- [ ] CapabilityStatement at `/fhir/R4/metadata` accurately reflects supported resources and operations
- [ ] SMART on FHIR authorization (EHR launch + standalone launch) operational
- [ ] FHIR search with chained parameters, `_include`, `_revinclude`
- [ ] FHIR batch/transaction bundles processed atomically
- [ ] Content negotiation: `application/fhir+json` and `application/json`

### Task 3.1: FHIR REST Endpoint Router

**What:** Implement the Fastify route handler that maps FHIR HTTP interactions (read, vread, update, delete, create, search) to resource repositories.

**Design:**
```typescript
// packages/fhir/src/router.ts
export interface FhirRoutes {
  'GET /fhir/R4/metadata': CapabilityStatement;
  'GET /fhir/R4/:resourceType/:id': FhirResource;
  'GET /fhir/R4/:resourceType/:id/_history/:vid': FhirResource;
  'GET /fhir/R4/:resourceType/:id/_history': FhirBundle;
  'GET /fhir/R4/:resourceType': FhirBundle;            // search via GET
  'POST /fhir/R4/:resourceType/_search': FhirBundle;   // search via POST
  'POST /fhir/R4/:resourceType': FhirResource;         // create
  'PUT /fhir/R4/:resourceType/:id': FhirResource;      // update
  'DELETE /fhir/R4/:resourceType/:id': void;            // delete
  'POST /fhir/R4/': FhirBundle;                        // batch/transaction
}
```

```typescript
// packages/fhir/src/capability-statement.ts
export function buildCapabilityStatement(): CapabilityStatement {
  return {
    resourceType: 'CapabilityStatement',
    status: 'active',
    kind: 'instance',
    fhirVersion: '4.0.1',
    format: ['application/fhir+json'],
    rest: [{
      mode: 'server',
      security: {
        service: [{ coding: [{ system: 'http://hl7.org/fhir/restful-security-service', code: 'SMART-on-FHIR' }] }],
        extension: [{
          url: 'http://fhir-registry.smarthealthit.org/StructureDefinition/oauth-uris',
          extension: [
            { url: 'authorize', valueUri: '/auth/authorize' },
            { url: 'token', valueUri: '/auth/token' },
          ]
        }]
      },
      resource: [
        { type: 'Patient', interaction: [{ code: 'read' }, { code: 'vread' }, { code: 'update' }, { code: 'create' }, { code: 'delete' }, { code: 'search-type' }],
          searchParam: [
            { name: 'family', type: 'string' },
            { name: 'given', type: 'string' },
            { name: 'birthdate', type: 'date' },
            { name: 'gender', type: 'token' },
            { name: 'identifier', type: 'token' },
          ]
        },
        // ... Encounter, Condition, Observation, AllergyIntolerance, MedicationRequest
      ]
    }]
  };
}
```

**Testing:**
- `GET /fhir/R4/metadata` returns valid CapabilityStatement with all supported resources
- `GET /fhir/R4/Patient/<id>` returns FHIR Patient with correct `Content-Type: application/fhir+json`
- `GET /fhir/R4/Patient/<id>/_history/1` returns version 1 of the Patient
- `POST /fhir/R4/Patient` with valid body returns 201 with `Location` header
- `PUT /fhir/R4/Patient/<id>` with `If-Match` header enforces version concurrency
- `DELETE /fhir/R4/Patient/<id>` returns 204; subsequent GET returns 410 Gone
- Invalid resource type returns OperationOutcome with 404

### Task 3.2: SMART on FHIR Authorization Server

**What:** Implement the SMART on FHIR authorization endpoints for EHR launch and standalone launch flows per SMART App Launch v2.2.

**Design:**
```typescript
// packages/auth/src/smart.ts
export interface SmartAuthRoutes {
  'GET /auth/authorize': {
    query: {
      response_type: 'code';
      client_id: string;
      redirect_uri: string;
      scope: string;          // e.g., 'launch patient/*.read openid fhirUser'
      state: string;
      aud: string;            // FHIR server base URL
      launch?: string;        // EHR launch context token
    };
  };
  'POST /auth/token': {
    body: {
      grant_type: 'authorization_code' | 'refresh_token' | 'client_credentials';
      code?: string;
      refresh_token?: string;
      client_id?: string;
      client_secret?: string;
      redirect_uri?: string;
    };
    response: SmartTokenResponse;
  };
}

export interface SmartTokenResponse {
  access_token: string;
  token_type: 'Bearer';
  expires_in: number;
  scope: string;
  patient?: string;         // Patient ID in context (for patient-facing apps)
  encounter?: string;       // Encounter ID in context (for EHR launch)
  id_token?: string;        // OpenID Connect ID token
  refresh_token?: string;
}

export interface SmartAppRegistration {
  clientId: string;
  clientName: string;
  redirectUris: string[];
  scopes: string[];
  launchType: 'ehr' | 'standalone' | 'backend-service';
  jwksUri?: string;         // For backend services (asymmetric auth)
}
```

**Testing:**
- EHR launch flow: `GET /auth/authorize` with `launch` param → consent screen → callback with code → `POST /auth/token` returns access token with patient and encounter context
- Standalone launch flow: `GET /auth/authorize` with patient scope → patient picker → token with `patient` field
- Backend service flow: `POST /auth/token` with `client_credentials` grant and signed JWT assertion returns access token
- Token with `patient/Patient.read` scope can `GET /fhir/R4/Patient/<id>` but cannot `PUT`
- Token without `patient/Observation.read` scope returns 403 on `GET /fhir/R4/Observation`
- `GET /fhir/R4/.well-known/smart-configuration` returns SMART configuration JSON

### Task 3.3: FHIR Search Engine

**What:** Implement a FHIR search query parser that translates FHIR search parameters into PostgreSQL queries against extracted columns and JSONB content.

**Design:**
```typescript
// packages/fhir/src/search/engine.ts
export interface FhirSearchEngine {
  search(resourceType: string, params: Record<string, string | string[]>): Promise<FhirBundle>;
}

export interface SearchParameter {
  name: string;
  type: 'string' | 'token' | 'date' | 'reference' | 'quantity' | 'number' | 'uri' | 'composite';
  expression: string;     // FHIRPath expression
  column?: string;        // Extracted column name for direct SQL
  modifier?: string[];    // :exact, :contains, :missing, :not, :text
}

// Date prefix parsing: eq, ne, lt, le, gt, ge, sa, eb, ap
export function parseDatePrefix(value: string): { comparator: string; date: string } {
  const prefixes = ['eq', 'ne', 'lt', 'le', 'gt', 'ge', 'sa', 'eb', 'ap'];
  for (const p of prefixes) {
    if (value.startsWith(p)) return { comparator: p, date: value.slice(2) };
  }
  return { comparator: 'eq', date: value };
}

// Token search: system|code, |code, system|, code
export function parseTokenParam(value: string): { system?: string; code?: string } {
  if (value.includes('|')) {
    const [system, code] = value.split('|', 2);
    return { system: system || undefined, code: code || undefined };
  }
  return { code: value };
}
```

**Testing:**
- `GET /fhir/R4/Patient?family=Smith` returns patients with family name Smith
- `GET /fhir/R4/Patient?family:exact=Smith` does case-sensitive match
- `GET /fhir/R4/Observation?patient=<uuid>&category=vital-signs&date=ge2026-01-01` returns correct results
- `GET /fhir/R4/Condition?code=http://snomed.info/sct|44054006` returns conditions with exact system|code match
- `GET /fhir/R4/Patient?_count=10&_offset=20` returns page 3 with correct `Bundle.link` navigation
- `GET /fhir/R4/Patient?_include=Patient:general-practitioner` includes referenced Practitioners in bundle
- `GET /fhir/R4/Patient?_revinclude=Condition:patient` includes Conditions referencing each Patient

### Task 3.4: FHIR Batch and Transaction Bundles

**What:** Implement processing of FHIR Bundle resources with `type: batch` and `type: transaction`, with transaction bundles executing atomically within a PostgreSQL transaction.

**Design:**
```typescript
// packages/fhir/src/bundle/processor.ts
export interface BundleProcessor {
  process(bundle: FhirBundle): Promise<FhirBundle>;
}

// Transaction bundles:
// 1. All entries processed within a single PostgreSQL transaction
// 2. If any entry fails, the entire transaction rolls back
// 3. Conditional references (urn:uuid:) resolved within the transaction
// 4. Response bundle contains per-entry outcomes

// Batch bundles:
// 1. Each entry processed independently
// 2. Failures do not affect other entries
// 3. Response bundle contains per-entry outcomes (mix of success/failure)
```

**Testing:**
- Transaction bundle with 3 creates (Patient, Encounter, Condition) succeeds atomically
- Transaction bundle with an invalid resource rolls back all entries; no partial state
- Conditional references (`urn:uuid:` within the bundle) resolve correctly
- Batch bundle with 1 valid and 1 invalid entry: valid entry persists, invalid returns OperationOutcome
- Transaction bundle entry order is respected per FHIR spec (DELETE before POST before PUT)

---

## Phase 4: Clinical Documentation & Notes

**Goal:** Implement structured clinical note authoring (SOAP notes), appointment scheduling, and the clinical UI for encounter-based documentation workflows.

**Definition of Done:**
- [ ] Clinical note resource (DocumentReference) with SOAP sections persisted as FHIR resource
- [ ] Appointment scheduling CRUD with status transitions
- [ ] Clinical UI: patient chart view with problem list, medication list, allergy list, vitals timeline
- [ ] Clinical UI: encounter documentation workflow (start encounter → document → sign note)
- [ ] Note signing produces a final, immutable version with author attestation

### Task 4.1: DocumentReference and Appointment Resources

**What:** Create the DocumentReference (clinical notes) and Appointment FHIR resource tables.

**Design:**
```sql
-- Migration: 005_documentation.sql
CREATE TABLE "DocumentReference" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID NOT NULL,
    patient         UUID NOT NULL,
    encounter       UUID,
    author          UUID[],
    type_code       TEXT[],             -- LOINC document type
    category        TEXT[],
    status          TEXT,               -- current, superseded, entered-in-error
    date            TIMESTAMPTZ,
    -- AI metadata (custom extension in FHIR content)
    ai_generated    BOOLEAN DEFAULT FALSE,
    ai_model        TEXT,
    ai_confidence   NUMERIC,
    clinician_reviewed BOOLEAN DEFAULT FALSE,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE "DocumentReference" ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON "DocumentReference"
    USING (organization_id = current_setting('app.current_organization_id')::uuid);

CREATE TABLE "Appointment" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID NOT NULL,
    patient         UUID,
    practitioner    UUID[],
    status          TEXT,
    start_time      TIMESTAMPTZ,
    end_time        TIMESTAMPTZ,
    service_type    TEXT[],
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE "Appointment" ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON "Appointment"
    USING (organization_id = current_setting('app.current_organization_id')::uuid);
```

**Testing:**
- Create DocumentReference with SOAP sections in content; verify storage and retrieval
- Search DocumentReference by patient, type, date, author
- Appointment status transitions: proposed → booked → arrived → fulfilled (valid) and booked → cancelled (valid)
- Invalid status transition (fulfilled → booked) rejected with OperationOutcome
- Appointment time overlap detection: warn when booking overlaps existing appointment

### Task 4.2: Clinical Note Authoring Service

**What:** Implement a service that manages the lifecycle of clinical notes: draft creation, section editing, co-signing, and final attestation.

**Design:**
```typescript
// packages/fhir/src/services/note-service.ts
export interface ClinicalNoteService {
  createDraft(encounterId: string, authorId: string, noteType: string): Promise<DocumentReference>;
  updateSection(noteId: string, section: NoteSection, content: string): Promise<DocumentReference>;
  submitForReview(noteId: string): Promise<DocumentReference>;
  cosign(noteId: string, cosignerId: string): Promise<DocumentReference>;
  attest(noteId: string, authorId: string): Promise<DocumentReference>;  // Final, immutable
  addAiDraft(noteId: string, aiContent: AiNoteContent): Promise<DocumentReference>;
}

export type NoteSection = 'subjective' | 'objective' | 'assessment' | 'plan';

export interface AiNoteContent {
  subjective?: string;
  objective?: string;
  assessment?: string;
  plan?: string;
  modelId: string;
  confidence: number;
  sourceType: 'ambient' | 'dictation' | 'template';
}

// Note lifecycle states:
// draft → preliminary → final (attested)
// draft → preliminary → amended → final
// Any state → entered-in-error (administrative correction)
```

**Testing:**
- Create draft note for an encounter; verify status is `preliminary` with `docStatus` extension
- Update subjective section; verify content JSONB updated, version incremented
- Attest note; verify status changes to `current`, `authenticator` set, no further edits allowed
- Attempt to edit attested note returns 409 Conflict
- AI draft: `addAiDraft` populates sections with `ai_generated=true`, `clinician_reviewed=false`
- After clinician edits AI draft and attests, `clinician_reviewed` set to `true`

### Task 4.3: Clinical UI — Patient Chart

**What:** Build the React clinical UI for the patient chart view showing problem list, medication list, allergy list, vitals timeline, and encounter history.

**Design:**
```typescript
// packages/clinical-ui/src/components/PatientChart.tsx
export interface PatientChartProps {
  patientId: string;
}

// Sub-components:
// - PatientBanner: name, DOB, gender, MRN, allergies summary, photo
// - ProblemList: active conditions from Condition?category=problem-list-item&clinical-status=active
// - MedicationList: active medications from MedicationRequest?status=active
// - AllergyList: active allergies from AllergyIntolerance?clinical-status=active
// - VitalsTimeline: recent observations from Observation?category=vital-signs (chart + table)
// - EncounterHistory: paginated encounter list from Encounter?_sort=-date

// Uses @medplum/react FHIR-aware components where available
// Falls back to custom components for EHR-specific workflows
```

```typescript
// packages/clinical-ui/src/hooks/useFhirSearch.ts
export function useFhirSearch<T extends FhirResource>(
  resourceType: string,
  params: Record<string, string>,
  options?: { enabled?: boolean; refetchInterval?: number }
): {
  data: FhirBundle<T> | undefined;
  isLoading: boolean;
  error: Error | null;
  refetch: () => void;
};
```

**Testing:**
- Patient chart renders with all five sections populated from FHIR API
- Problem list displays SNOMED CT display names with clinical status badges
- Medication list shows drug name, dosage, frequency, prescriber
- Allergy list shows substance, criticality badge (high = red), reaction
- Vitals timeline chart displays blood pressure, heart rate, temperature trends
- Encounter history is paginated; clicking an encounter navigates to encounter detail
- Empty state: new patient with no data shows appropriate empty messages

### Task 4.4: Clinical UI — Encounter Documentation Workflow

**What:** Build the encounter documentation screen: start encounter, record vitals, document SOAP note, add diagnoses, place orders, and sign/attest the note.

**Design:**
```typescript
// packages/clinical-ui/src/components/EncounterWorkflow.tsx
export interface EncounterWorkflowProps {
  patientId: string;
  encounterId?: string;     // Existing encounter or create new
}

// Workflow steps:
// 1. Start Encounter (or resume existing)
//    - Create Encounter resource with status=in-progress
//    - Link to Appointment if scheduled
// 2. Record Vitals
//    - Vital signs form: BP, HR, Temp, RR, SpO2, Height, Weight, BMI
//    - Each vital creates an Observation resource
// 3. Document Note
//    - SOAP editor with rich text (Subjective, Objective, Assessment, Plan)
//    - AI draft button (disabled until Phase 6)
// 4. Add Diagnoses
//    - SNOMED CT / ICD-10 search typeahead
//    - Creates Condition resources linked to encounter
// 5. Place Orders
//    - Medication orders (RxNorm search) → MedicationRequest
//    - Lab orders (LOINC search) → ServiceRequest
// 6. Sign & Close
//    - Attest note → status=final
//    - Close encounter → status=finished
```

**Testing:**
- Full workflow: start encounter → record BP → write SOAP note → add diagnosis → sign → close
- Vitals saved as individual Observation resources with correct LOINC codes
- SNOMED CT typeahead returns relevant conditions; selecting one creates Condition resource
- Note attestation locks the note; encounter status transitions to `finished`
- Partially completed encounter preserved as `in-progress`; can resume later
- Concurrent edit detection: two clinicians editing same encounter → conflict resolution

---

## Phase 5: Billing & Revenue Cycle

**Goal:** Implement charge capture, claim generation (X12 837), remittance posting (X12 835), and coverage/eligibility management.

**Definition of Done:**
- [ ] Claim resource with line items, diagnoses, and modifiers persisted
- [ ] X12 837P claim generation from encounter data
- [ ] X12 835 remittance parsing and posting
- [ ] Coverage (insurance) resource CRUD
- [ ] Charge capture linked to encounter diagnoses and procedures
- [ ] Claim status dashboard in clinical UI

### Task 5.1: Coverage and Claim Resources

**What:** Create the Coverage (insurance) and Claim FHIR resource tables with billing-specific operational columns.

**Design:**
```sql
-- Migration: 006_billing.sql
CREATE TABLE "Coverage" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID NOT NULL,
    patient         UUID NOT NULL,
    payor           TEXT[],
    subscriber_id   TEXT,
    status          TEXT,
    type_code       TEXT,
    period_start    DATE,
    period_end      DATE,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "Claim" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    content         JSONB NOT NULL,
    organization_id UUID NOT NULL,
    patient         UUID NOT NULL,
    encounter       UUID,
    status          TEXT,
    type_code       TEXT,             -- institutional, professional, pharmacy
    use_code        TEXT,             -- claim, preauthorization, predetermination
    total_value     NUMERIC(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE TABLE edi_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    claim_id        UUID REFERENCES "Claim"(id),
    transaction_type TEXT NOT NULL CHECK (transaction_type IN ('837P','837I','837D','835','270','271','278')),
    direction       TEXT NOT NULL CHECK (direction IN ('outbound','inbound')),
    clearinghouse   TEXT,
    control_number  TEXT,
    raw_content     TEXT,
    status          TEXT CHECK (status IN ('pending','sent','accepted','rejected','acknowledged')),
    submitted_at    TIMESTAMPTZ,
    response_at     TIMESTAMPTZ,
    error_details   JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**
- Create Coverage with payer, member ID, group number; verify search by patient
- Create Claim with line items (CPT codes, diagnosis pointers, modifiers)
- Claim total computed correctly from line items
- EDI transaction status transitions tracked correctly
- Coverage eligibility period enforced: expired coverage flagged on claim creation

### Task 5.2: X12 837P Claim Generator

**What:** Implement a service that generates X12 837P (professional claim) EDI files from FHIR Claim resources.

**Design:**
```typescript
// packages/worker/src/billing/x12-837p-generator.ts
export interface X12837PGenerator {
  generate(claim: FhirClaim, coverage: FhirCoverage, provider: FhirPractitioner, org: FhirOrganization): string;
}

// X12 837P segment structure:
// ISA - Interchange Control Header
// GS  - Functional Group Header
// ST  - Transaction Set Header (837)
// BHT - Beginning of Hierarchical Transaction
// Loop 1000A - Submitter
// Loop 1000B - Receiver
// Loop 2000A - Billing Provider HL
// Loop 2010AA - Billing Provider Name
// Loop 2000B - Subscriber HL
// Loop 2010BA - Subscriber Name
// Loop 2300 - Claim Information
// Loop 2310 - Rendering Provider
// Loop 2400 - Service Line (per claim_line)
// SE  - Transaction Set Trailer
// GE  - Functional Group Trailer
// IEA - Interchange Control Trailer

export interface X12Segment {
  id: string;
  elements: string[];
  toString(): string;        // pipe-delimited: 'CLM*12345*100*...'
}
```

**Testing:**
- Generate 837P from a claim with 3 line items; validate segment count and structure
- ISA/IEA envelope control numbers match
- CLM segment contains correct claim amount and place of service
- SV1 segments contain correct CPT codes, modifiers, and line charges
- DTP segments contain correct service dates
- Generated EDI passes X12 syntax validation (segment terminators, element counts)

### Task 5.3: X12 835 Remittance Parser

**What:** Implement a parser that reads X12 835 (Electronic Remittance Advice) files and posts payments and adjustments to claims.

**Design:**
```typescript
// packages/worker/src/billing/x12-835-parser.ts
export interface X12835Parser {
  parse(ediContent: string): RemittanceAdvice;
}

export interface RemittanceAdvice {
  payerName: string;
  payerId: string;
  checkNumber: string;
  paymentDate: string;
  claims: RemittanceClaim[];
}

export interface RemittanceClaim {
  claimNumber: string;
  patientControlNumber: string;
  status: 'paid' | 'denied' | 'partial' | 'adjusted';
  totalCharged: number;
  totalPaid: number;
  patientResponsibility: number;
  adjustments: RemittanceAdjustment[];
  lineItems: RemittanceLineItem[];
}

export interface RemittanceAdjustment {
  groupCode: string;        // CO (contractual), PR (patient resp), OA (other adj)
  reasonCode: string;       // CARC code
  amount: number;
}
```

**Testing:**
- Parse sample 835 file; verify payer name, check number, and payment date extracted
- Per-claim amounts match 835 CLP segment values
- Adjustment reason codes (CARC) correctly mapped
- Posting updates Claim status and creates payment records
- Denied claim triggers status update and stores denial reason for follow-up

### Task 5.3: Charge Capture Workflow

**What:** Implement the charge capture service that auto-generates claim line items from encounter diagnoses and procedures.

**Design:**
```typescript
// packages/fhir/src/services/charge-capture.ts
export interface ChargeCaptureService {
  captureFromEncounter(encounterId: string): Promise<FhirClaim>;
  addLineItem(claimId: string, item: ChargeItem): Promise<FhirClaim>;
  removeLineItem(claimId: string, lineSequence: number): Promise<FhirClaim>;
  submitClaim(claimId: string): Promise<EdiTransaction>;
}

export interface ChargeItem {
  procedureCode: string;     // CPT code
  procedureSystem: string;
  modifiers?: string[];
  diagnosisPointers: number[];
  quantity: number;
  unitPrice: number;
  placeOfService: string;
  serviceDate: string;
}
```

**Testing:**
- Auto-capture from encounter with 2 diagnoses and 1 procedure creates claim with correct pointers
- Adding/removing line items updates claim total
- Submit triggers 837P generation and EDI transaction creation with status `pending`
- Claim without at least one diagnosis rejected with validation error
- Claim without coverage attached warns about self-pay

---

## Phase 6: AI Ambient Documentation

**Goal:** Implement real-time encounter audio transcription and AI-generated SOAP note drafts. This is the first AI-augmented feature and the primary differentiator identified in research.

**Definition of Done:**
- [ ] Audio capture from encounter (browser MediaRecorder API)
- [ ] Real-time speech-to-text transcription (streaming)
- [ ] AI SOAP note generation from transcript
- [ ] Draft note injected into encounter documentation workflow with `ai_generated=true`
- [ ] Clinician review and edit workflow before attestation
- [ ] AI provenance metadata (model, confidence, source) stored on DocumentReference

### Task 6.1: Audio Capture and Streaming Infrastructure

**What:** Implement browser-based audio capture and WebSocket streaming to the AI transcription service.

**Design:**
```typescript
// packages/clinical-ui/src/components/AmbientRecorder.tsx
export interface AmbientRecorderProps {
  encounterId: string;
  patientId: string;
  onTranscriptUpdate: (transcript: TranscriptSegment[]) => void;
  onNoteGenerated: (note: AiNoteContent) => void;
}

export interface TranscriptSegment {
  speaker: 'clinician' | 'patient' | 'unknown';
  text: string;
  startTime: number;       // seconds from recording start
  endTime: number;
  confidence: number;
}

// Browser: MediaRecorder API → WebSocket → AI service
// Audio format: Opus in WebM container, 16kHz mono
// WebSocket protocol: binary audio frames + JSON control messages
```

```typescript
// packages/ai/src/transcription/websocket.py (Python)
# FastAPI WebSocket endpoint
# Receives audio chunks via WebSocket
# Streams to speech-to-text model (Whisper or cloud STT)
# Returns TranscriptSegment JSON messages back to client
# Stores audio chunks in S3 for audit trail (HIPAA)
```

**Testing:**
- Audio capture starts/stops cleanly in Chrome and Firefox
- WebSocket connection established with authentication token
- Transcript segments streamed back within 2 seconds of speech
- Speaker diarization distinguishes clinician and patient voices
- Audio stored in S3 with encryption at rest (AES-256)
- Network interruption: reconnect and resume without data loss

### Task 6.2: AI SOAP Note Generator

**What:** Implement the AI service that takes a complete encounter transcript and generates a structured SOAP note draft.

**Design:**
```python
# packages/ai/src/note_generation/soap_generator.py
from dataclasses import dataclass

@dataclass
class SOAPNote:
    subjective: str
    objective: str
    assessment: str
    plan: str
    suggested_diagnoses: list['SuggestedDiagnosis']
    suggested_cpt_codes: list['SuggestedCode']
    confidence: float
    model_id: str

@dataclass
class SuggestedDiagnosis:
    code: str                # SNOMED CT or ICD-10
    code_system: str
    display: str
    confidence: float

@dataclass
class SuggestedCode:
    code: str                # CPT code
    display: str
    confidence: float

class SOAPNoteGenerator:
    async def generate(
        self,
        transcript: list[TranscriptSegment],
        patient_context: PatientContext,  # Active problems, meds, allergies
        encounter_type: str,
    ) -> SOAPNote:
        """
        1. Format transcript with speaker labels and timestamps
        2. Include patient context (problem list, medications, allergies) for grounding
        3. Prompt LLM to generate structured SOAP sections
        4. Extract suggested diagnoses (SNOMED CT codes) from assessment
        5. Extract suggested CPT codes from plan
        6. Return structured SOAPNote with confidence scores
        """
        ...
```

**Testing:**
- Given a 10-minute diabetes follow-up transcript, generates SOAP note with all 4 sections populated
- Subjective section captures patient-reported symptoms from patient speaker segments
- Objective section includes vitals mentioned in clinician speech
- Assessment suggests ICD-10 `E11.9` (Type 2 diabetes) with confidence > 0.8
- Plan section includes medication adjustments discussed
- Confidence score reflects transcript quality (clear audio > 0.85, noisy < 0.6)
- Patient context grounding: AI references existing medications in plan section
- No hallucinated diagnoses not supported by transcript content

### Task 6.3: AI Draft Integration with Note Workflow

**What:** Connect the AI SOAP note generator to the clinical note authoring workflow so that AI drafts appear in the encounter documentation with clear provenance.

**Design:**
```typescript
// packages/fhir/src/services/ai-note-integration.ts
export interface AiNoteIntegration {
  injectAiDraft(encounterId: string, soapNote: SOAPNote): Promise<DocumentReference>;
  acceptDraft(noteId: string, clinicianId: string): Promise<DocumentReference>;
  rejectDraft(noteId: string, clinicianId: string, reason: string): Promise<void>;
  getDraftComparison(noteId: string): Promise<DraftComparison>;
}

export interface DraftComparison {
  aiDraft: SOAPSections;
  clinicianEdits: SOAPSections;
  diffSummary: string;       // Human-readable summary of changes
}

// FHIR extension for AI provenance:
// {
//   "url": "https://ehr.example/fhir/StructureDefinition/ai-provenance",
//   "extension": [
//     { "url": "model", "valueString": "ehr-soap-v2.1" },
//     { "url": "confidence", "valueDecimal": 0.87 },
//     { "url": "source", "valueCode": "ambient" },
//     { "url": "clinicianReviewed", "valueBoolean": true },
//     { "url": "reviewedAt", "valueDateTime": "2026-05-25T14:30:00Z" }
//   ]
// }
```

**Testing:**
- AI draft injected into encounter note with `ai_generated=true`, `clinician_reviewed=false`
- Clinical UI shows AI-generated sections with visual indicator (badge/highlight)
- Clinician edits AI draft; `clinician_reviewed` set to `true` on save
- Draft comparison shows diff between AI original and clinician edits
- Rejecting AI draft removes it from note; audit event logged
- Attested note with AI content includes full AI provenance extension in FHIR resource
- Suggested diagnoses appear as "AI Suggested" in the diagnosis picker (distinct from clinician-entered)

---

## Phase 7: AI Clinical Intelligence

**Goal:** Implement AI-powered clinical decision support: continuous problem list / HCC reconciliation, prior authorization prediction, and sepsis / readmission risk scoring.

**Definition of Done:**
- [ ] HCC reconciliation runs on every encounter close, surfacing suspect conditions
- [ ] Prior authorization prediction flags orders likely to require PA before submission
- [ ] Risk stratification model scores patients for sepsis and 30-day readmission risk
- [ ] All AI suggestions displayed with confidence scores and clinician review workflow
- [ ] AI inference audit trail: every suggestion logged with model, confidence, and outcome

### Task 7.1: HCC and Problem List Reconciliation Engine

**What:** Build an AI service that analyzes encounter notes and historical data to surface missing or suspect diagnoses that should be on the problem list, with emphasis on HCC codes that affect risk-adjusted revenue.

**Design:**
```python
# packages/ai/src/clinical_intelligence/hcc_reconciliation.py
@dataclass
class HCCSuggestion:
    icd10_code: str
    icd10_display: str
    hcc_category: int
    hcc_description: str
    evidence: list[str]       # Quoted text from notes supporting this suggestion
    confidence: float
    source_notes: list[str]   # DocumentReference IDs containing evidence
    raf_impact: float         # Estimated RAF score impact

class HCCReconciliationEngine:
    async def reconcile(
        self,
        patient_id: str,
        encounter_id: str,
        current_problem_list: list[Condition],
        encounter_notes: list[DocumentReference],
        historical_claims: list[Claim],
    ) -> list[HCCSuggestion]:
        """
        1. Extract clinical concepts from encounter notes (NLP/LLM)
        2. Map concepts to ICD-10-CM codes
        3. Cross-reference against CMS HCC model V28
        4. Identify HCC codes present in notes but missing from problem list
        5. Identify HCC codes captured in prior years but not recaptured this year
        6. Rank by RAF impact and evidence confidence
        """
        ...
```

**Testing:**
- Patient with diabetes notes but no diabetes on problem list: suggests `E11.9` with evidence quotes
- Patient with prior-year HCC for COPD not recaptured: flags for recapture
- Suggestions include RAF impact estimate (e.g., 0.318 for HCC 19)
- No false suggestions for conditions explicitly refuted in notes
- Clinician can accept (adds to problem list), dismiss, or defer each suggestion
- All suggestions and outcomes logged in audit trail

### Task 7.2: Prior Authorization Prediction

**What:** Build a prediction model that flags service requests likely to require prior authorization based on payer rules and historical denial data.

**Design:**
```python
# packages/ai/src/clinical_intelligence/prior_auth_predictor.py
@dataclass
class PriorAuthPrediction:
    service_request_id: str
    requires_pa: bool
    confidence: float
    payer_name: str
    reason: str               # e.g., "MRI requires PA for Aetna PPO plans"
    estimated_turnaround_days: int
    supporting_documentation: list[str]  # Required clinical docs
    alternative_codes: list[str]         # Codes that may not require PA

class PriorAuthPredictor:
    async def predict(
        self,
        service_request: ServiceRequest,
        patient_coverage: Coverage,
        diagnosis_codes: list[str],
    ) -> PriorAuthPrediction:
        """
        1. Look up payer-specific PA rules from rules database
        2. Check procedure code against PA-required code lists
        3. Cross-reference with diagnosis (some dx exempt certain procedures)
        4. Score confidence based on historical approval/denial data
        5. Suggest alternatives if PA-exempt codes exist
        """
        ...
```

**Testing:**
- MRI order with Aetna PPO coverage flags as PA-required with > 0.9 confidence
- Routine lab order (CBC) does not flag for PA
- Prediction includes estimated turnaround time and required documentation list
- Alternative code suggestion: if CT without contrast does not require PA, it appears as an option
- Payer rules database can be updated without model retraining

### Task 7.3: Risk Stratification Scoring

**What:** Implement patient risk scoring for sepsis (inpatient), 30-day readmission, and medication non-adherence.

**Design:**
```python
# packages/ai/src/clinical_intelligence/risk_scoring.py
@dataclass
class RiskScore:
    patient_id: str
    risk_type: str            # 'sepsis', 'readmission', 'non-adherence'
    score: float              # 0.0 - 1.0
    risk_level: str           # 'low', 'moderate', 'high', 'critical'
    contributing_factors: list[RiskFactor]
    recommended_actions: list[str]
    model_id: str
    scored_at: str            # ISO datetime

@dataclass
class RiskFactor:
    factor: str               # e.g., "Elevated WBC count", "3 ED visits in 30 days"
    weight: float             # Contribution to overall score
    data_source: str          # FHIR resource ID

class RiskScoringEngine:
    async def score_patient(
        self,
        patient_id: str,
        risk_type: str,
    ) -> RiskScore:
        """
        Input features (from FHIR data):
        - Demographics: age, gender, comorbidity count
        - Vitals: latest and trending (BP, HR, Temp, RR, SpO2)
        - Labs: WBC, lactate, creatinine, procalcitonin (sepsis)
        - Utilization: ED visits, hospitalizations in past 90 days
        - Medications: count, complexity, controlled substances
        - Social determinants: insurance type, language, zip code
        """
        ...
```

**Testing:**
- Sepsis risk: patient with elevated WBC, tachycardia, and fever scores > 0.7 (high)
- Readmission risk: patient with 2 admissions in 30 days scores > 0.6 (moderate-high)
- Contributing factors list includes specific data points with FHIR resource references
- Recommended actions are clinically appropriate (e.g., "Order blood cultures and lactate")
- Scores refresh when new observations are added to the patient record
- Model predictions logged with full feature vector for retrospective analysis

---

## Phase 8: Population Health & Analytics

**Goal:** Implement population health dashboards, care gap identification, chronic disease registries, quality measure calculation, and FHIR Bulk Data Export.

**Definition of Done:**
- [ ] FHIR Bulk Data `$export` operation produces NDJSON files
- [ ] Care gap identification: surfaces patients missing preventive screenings
- [ ] Chronic disease registry: filterable patient cohorts by condition, medication, risk score
- [ ] Quality measure dashboards: CMS/HEDIS measure performance tracking
- [ ] Analytics UI: charts, tables, and drill-down views

### Task 8.1: FHIR Bulk Data Export

**What:** Implement the FHIR Bulk Data Access `$export` operation for system-level, patient-level, and group-level export as NDJSON files.

**Design:**
```typescript
// packages/fhir/src/operations/bulk-export.ts
export interface BulkExportOperation {
  // POST /fhir/R4/$export (system-level)
  // POST /fhir/R4/Patient/$export (all patients)
  // POST /fhir/R4/Group/<id>/$export (group of patients)
  initiateExport(params: BulkExportParams): Promise<BulkExportJob>;
  getStatus(jobId: string): Promise<BulkExportStatus>;
  downloadFile(jobId: string, fileIndex: number): ReadableStream;
  deleteExport(jobId: string): Promise<void>;
}

export interface BulkExportParams {
  _outputFormat?: string;    // application/fhir+ndjson (default)
  _since?: string;           // Only resources updated after this datetime
  _type?: string[];          // Resource types to include
}

export interface BulkExportStatus {
  transactionTime: string;
  request: string;
  requiresAccessToken: boolean;
  output: BulkExportFile[];
  error: BulkExportFile[];
  status: 'accepted' | 'in-progress' | 'complete' | 'error';
}
```

**Testing:**
- `POST /fhir/R4/$export` returns 202 Accepted with `Content-Location` header
- Polling status endpoint returns `in-progress` then `complete`
- NDJSON output contains one JSON resource per line
- `_since` parameter filters to recently updated resources only
- `_type=Patient,Condition` exports only those resource types
- Export files are encrypted at rest in S3
- Large export (10,000 patients) completes within acceptable time; progress reported

### Task 8.2: Care Gap Engine

**What:** Build a service that identifies patients missing recommended preventive care based on clinical guidelines (e.g., USPSTF recommendations).

**Design:**
```typescript
// packages/fhir/src/services/care-gaps.ts
export interface CareGapEngine {
  evaluatePatient(patientId: string): Promise<CareGap[]>;
  evaluatePopulation(orgId: string, filters?: PopulationFilter): Promise<CareGapSummary>;
}

export interface CareGap {
  patientId: string;
  measureId: string;        // e.g., 'CMS130v12' (Colorectal Cancer Screening)
  measureTitle: string;
  status: 'open' | 'closed' | 'excluded';
  dueDate: string;
  lastPerformed?: string;
  recommendation: string;
  priority: 'routine' | 'overdue' | 'urgent';
}

// Example measures:
// CMS130: Colorectal Cancer Screening (ages 45-75, colonoscopy/FIT)
// CMS125: Breast Cancer Screening (women 50-74, mammogram every 2 years)
// CMS122: Diabetes HbA1c Control (diabetics, HbA1c < 9%)
// CMS165: Blood Pressure Control (hypertension, BP < 140/90)
```

**Testing:**
- Patient age 50 with no colonoscopy in past 10 years: open gap for CMS130
- Patient with HbA1c of 9.5%: open gap for CMS122 (diabetes control)
- Patient with mammogram last year: closed gap for CMS125
- Population-level summary shows % of patients with open gaps per measure
- Filters: by provider panel, by condition cohort, by risk score range

### Task 8.3: Population Health Dashboard UI

**What:** Build the analytics dashboard showing care gap summaries, chronic disease registries, and quality measure performance.

**Design:**
```typescript
// packages/clinical-ui/src/components/PopulationHealth.tsx
// Dashboard sections:
// 1. Quality Measures Overview — bar chart of measure performance (% patients meeting target)
// 2. Care Gap Worklist — filterable table of patients with open gaps, sortable by priority
// 3. Chronic Disease Registry — cohort builder (condition + medication + risk score filters)
// 4. Risk Stratification Distribution — histogram of risk scores across patient panel
// 5. Provider Comparison — benchmarking individual provider panels against org averages

// Drill-down: click any measure → patient list → click patient → patient chart
```

**Testing:**
- Dashboard loads within 3 seconds for an organization with 5,000 patients
- Quality measure bars show correct numerator/denominator counts
- Care gap worklist filters by measure, provider, and priority
- Chronic disease registry: filter for "Type 2 diabetes + on insulin + HbA1c > 8" returns correct cohort
- Risk score histogram updates dynamically as date range is changed
- Provider comparison chart shows performance relative to organization mean

---

## Phase 9: Knowledge Graph & Advanced AI

**Goal:** Implement the graph-relational hybrid layer (Data Model Suggestion 4) for clinical knowledge graphs, drug interaction traversal, and AI-discovered clinical correlations.

**Definition of Done:**
- [ ] `graph_node` and `graph_edge` tables operational with synchronization from relational layer
- [ ] SNOMED CT ontology imported as graph nodes with IS-A edges
- [ ] Drug interaction graph populated from RxNorm
- [ ] AI phenotype similarity model generates graph edges with confidence scores
- [ ] Clinical UI surfaces graph-derived insights (drug interactions, related conditions)
- [ ] AI inference audit trail for all graph-generated recommendations

### Task 9.1: Graph Layer Schema and Sync

**What:** Create the graph node/edge tables and implement synchronization triggers that mirror relational CRUD operations into graph nodes.

**Design:**
```sql
-- Migration: 010_graph_layer.sql
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     TEXT NOT NULL,
    entity_id       UUID,
    label           TEXT NOT NULL,
    code            TEXT,
    code_system     TEXT,
    properties      JSONB DEFAULT '{}',
    organization_id UUID,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_node(id),
    target_id       UUID NOT NULL REFERENCES graph_node(id),
    relationship    TEXT NOT NULL,
    properties      JSONB DEFAULT '{}',
    valid_from      TIMESTAMPTZ DEFAULT now(),
    valid_to        TIMESTAMPTZ,
    source_system   TEXT,
    confidence      NUMERIC,
    created_by      UUID,
    organization_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_graph_inference (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_id         UUID REFERENCES graph_edge(id),
    model_id        TEXT NOT NULL,
    model_version   TEXT NOT NULL,
    inference_type  TEXT NOT NULL,
    input_summary   JSONB,
    confidence      NUMERIC NOT NULL,
    explanation     TEXT,
    clinician_reviewed BOOLEAN NOT NULL DEFAULT FALSE,
    reviewed_by     UUID,
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**
- Creating a Patient in relational layer auto-creates a graph node with `entity_type='Patient'`
- Creating a MedicationRequest creates `TAKING` edge between Patient and Medication nodes
- Deleting a MedicationRequest sets `valid_to` on the `TAKING` edge
- Sync lag: graph nodes created within 1 second of relational writes
- Graph nodes for concept-only entities (SNOMED concepts) have `organization_id = NULL`

### Task 9.2: Clinical Knowledge Graph Import

**What:** Import SNOMED CT and RxNorm terminologies as graph nodes with hierarchical and interaction edges.

**Design:**
```python
# packages/ai/src/knowledge_graph/terminology_import.py
class TerminologyImporter:
    async def import_snomed(self, rf2_path: str) -> ImportResult:
        """
        1. Parse SNOMED CT RF2 distribution
        2. Create graph_node for each active concept in disorder/finding/procedure hierarchies
        3. Create IS_A edges from concept_relationship (typeId=116680003)
        4. Create FINDING_SITE edges (typeId=363698007)
        """
        ...

    async def import_rxnorm_interactions(self, rxnorm_path: str) -> ImportResult:
        """
        1. Parse RxNorm drug interaction data (RXNREL with rela=interacts_with)
        2. Create Medication graph_node for each clinical drug
        3. Create INTERACTS_WITH edges with severity and mechanism in properties
        """
        ...
```

**Testing:**
- SNOMED CT import: `44054006` (Type 2 diabetes) has IS_A edge to `73211009` (Diabetes mellitus)
- Walking IS-A hierarchy from Type 2 diabetes reaches `64572001` (Disease) within 5 hops
- RxNorm interaction: Warfarin INTERACTS_WITH Aspirin with severity `moderate`
- Drug interaction check for a patient on Warfarin + Aspirin returns the interaction
- Import is idempotent: running twice does not create duplicate nodes

### Task 9.3: AI Phenotype Similarity and Discovery

**What:** Implement an AI model that discovers clinical similarities between patients and generates graph edges representing shared phenotypes.

**Design:**
```python
# packages/ai/src/knowledge_graph/phenotype_similarity.py
class PhenotypeSimilarityEngine:
    async def compute_similarities(
        self,
        target_patient_id: str,
        candidate_pool: list[str],   # Patient IDs to compare against
        min_confidence: float = 0.8,
    ) -> list[SimilarPatient]:
        """
        1. Extract patient feature vector: active conditions, medications, demographics, labs
        2. Compute cosine similarity against candidate pool feature vectors
        3. Create SIMILAR_PHENOTYPE edges for pairs above min_confidence
        4. Log in ai_graph_inference with full explanation
        """
        ...
```

**Testing:**
- Two patients with identical condition and medication profiles score > 0.95 similarity
- Similarity edge includes `sharedConditions` and `sharedMedications` in properties
- AI inference logged with model version and feature summary
- Clinician can view similar patients from patient chart
- Privacy: similarities only computed within same organization (RLS enforced)

---

## Phase 10: Interoperability Bridge — HL7 v2 & CDA

**Goal:** Implement inbound/outbound HL7 v2.x message translation and C-CDA document generation/parsing for interoperability with legacy systems, labs, pharmacies, and HIE networks.

**Definition of Done:**
- [ ] HL7 v2 ADT (admission/discharge/transfer) messages parsed and translated to FHIR Encounter
- [ ] HL7 v2 ORU (lab results) messages parsed and translated to FHIR Observation + DiagnosticReport
- [ ] HL7 v2 ORM (orders) messages generated from FHIR ServiceRequest
- [ ] C-CDA Continuity of Care Document (CCD) generated from patient FHIR data
- [ ] C-CDA documents parsed and imported as FHIR resources
- [ ] MLLP listener for HL7 v2 TCP connections

### Task 10.1: HL7 v2 Parser and FHIR Translator

**What:** Build a bidirectional HL7 v2 parser/serializer and mapping engine that translates between HL7 v2 segments and FHIR resources.

**Design:**
```typescript
// packages/worker/src/interop/hl7v2-parser.ts
export interface HL7v2Parser {
  parse(message: string): HL7v2Message;
  serialize(message: HL7v2Message): string;
}

export interface HL7v2Message {
  msh: MSHSegment;          // Message Header
  segments: HL7v2Segment[];
  messageType: string;       // ADT^A01, ORU^R01, ORM^O01, etc.
}

// packages/worker/src/interop/hl7v2-to-fhir.ts
export interface HL7v2ToFhirMapper {
  mapADT(message: HL7v2Message): { patient: FhirPatient; encounter: FhirEncounter };
  mapORU(message: HL7v2Message): { observations: FhirObservation[]; report: FhirDiagnosticReport };
  mapORM(serviceRequest: FhirServiceRequest): HL7v2Message;  // FHIR → v2 (outbound orders)
}

// Mapping rules:
// PID → Patient (name, DOB, gender, address, identifiers)
// PV1 → Encounter (class, status, attending, location, admit date)
// OBR → DiagnosticReport (universal service ID → LOINC, result status)
// OBX → Observation (value type, value, units, reference range, abnormal flag)
// ORC → ServiceRequest (order control, order status, quantity/timing)
```

**Testing:**
- Parse sample ADT^A01 message; verify Patient name, DOB, MRN extracted correctly
- Parse sample ORU^R01 with 5 OBX segments; verify each maps to an Observation with correct LOINC code
- Outbound ORM^O01 generated from ServiceRequest contains correct ORC and OBR segments
- Special characters in HL7 v2 (escape sequences `\F\`, `\S\`, `\T\`, `\R\`, `\E\`) handled correctly
- Multi-segment messages with repeating groups parsed correctly

### Task 10.2: C-CDA Generator and Parser

**What:** Implement C-CDA document generation (CCD, Discharge Summary, Referral Note) from FHIR data and C-CDA import parsing.

**Design:**
```typescript
// packages/worker/src/interop/ccda-generator.ts
export interface CCDAGenerator {
  generateCCD(patientId: string): Promise<string>;           // XML string
  generateDischargeSummary(encounterId: string): Promise<string>;
  generateReferralNote(serviceRequestId: string): Promise<string>;
}

// packages/worker/src/interop/ccda-parser.ts
export interface CCDAParser {
  parse(ccdaXml: string): CCDADocument;
  toFhir(doc: CCDADocument): FhirBundle;  // Converts to FHIR resources
}

export interface CCDADocument {
  patient: CCDAPatient;
  problems: CCDAProblem[];
  medications: CCDAMedication[];
  allergies: CCDAAllergy[];
  results: CCDAResult[];
  procedures: CCDAProcedure[];
  encounters: CCDAEncounter[];
  immunizations: CCDAImmunization[];
}
```

**Testing:**
- Generated CCD validates against C-CDA Edition 4 schematron rules
- CCD contains all active problems, medications, and allergies for the patient
- Discharge summary includes encounter diagnoses, procedures, and discharge instructions
- Parsed C-CDA from an external system creates correct FHIR resources
- Coded entries in C-CDA (SNOMED, LOINC, RxNorm) map to correct FHIR code systems
- Round-trip: generate CCD → parse → compare FHIR resources → match original data

### Task 10.3: MLLP Listener and Message Router

**What:** Implement an MLLP (Minimum Lower Layer Protocol) TCP listener that receives HL7 v2 messages from lab instruments, pharmacy systems, and other ancillary systems.

**Design:**
```typescript
// packages/worker/src/interop/mllp-listener.ts
export interface MLLPListener {
  start(port: number): Promise<void>;
  stop(): Promise<void>;
  onMessage(handler: (message: HL7v2Message, ack: (response: HL7v2Message) => void) => Promise<void>): void;
}

// MLLP frame: <VT> message <FS><CR>
// VT = 0x0B (vertical tab)
// FS = 0x1C (file separator)
// CR = 0x0D (carriage return)

// Message routing:
// ADT^A01 (Admit) → Create/update Patient + Encounter
// ADT^A03 (Discharge) → Update Encounter status to finished
// ADT^A08 (Update) → Update Patient demographics
// ORU^R01 (Lab result) → Create Observations + DiagnosticReport
// ORM^O01 (Order) → Update ServiceRequest status
```

**Testing:**
- MLLP listener accepts TCP connection on configured port
- HL7 v2 message wrapped in MLLP framing is correctly extracted
- ACK (MSA) message sent back with correct message control ID
- Lab result ORU creates Observations searchable via FHIR API within 5 seconds
- ADT message from external system updates patient demographics
- Malformed message returns NAK with error description
- Concurrent connections (multiple lab instruments) handled without message loss

---

## Phase 11: Patient Portal & Conversational AI

**Goal:** Build the patient-facing portal with secure messaging, results access, appointment scheduling, and conversational AI grounded in the patient's own FHIR record.

**Definition of Done:**
- [ ] Patient registration and login (separate from clinical users)
- [ ] View lab results, medications, allergies, and immunizations
- [ ] Secure messaging between patient and care team
- [ ] Self-scheduling appointments from available slots
- [ ] Conversational AI: patient can ask questions about their record
- [ ] AI-powered symptom intake before appointments
- [ ] Multi-language support (English, Spanish as initial languages)

### Task 11.1: Patient Portal Application

**What:** Build the Next.js patient portal with SMART on FHIR patient-facing authorization.

**Design:**
```typescript
// packages/patient-portal/src/app/layout.tsx
// Next.js App Router layout

// Pages:
// /portal/dashboard    — Overview: upcoming appointments, recent results, unread messages
// /portal/results      — Lab results and diagnostic reports with reference ranges
// /portal/medications  — Active medication list with refill request
// /portal/messages     — Secure messaging thread with care team
// /portal/appointments — View/cancel upcoming; schedule new from available slots
// /portal/health-summary — Problem list, allergies, immunizations
// /portal/ask          — Conversational AI interface

// Authorization: SMART on FHIR standalone launch with patient/*.read scopes
// The portal uses the same FHIR API as the clinical UI but with patient-scoped tokens
```

**Testing:**
- Patient logs in with email/password; receives SMART token with `patient/<patientId>/*.read` scope
- Dashboard shows next appointment, 3 most recent lab results, unread message count
- Lab results page displays abnormal results with red flags and reference ranges
- Medication page shows drug name, dosage, prescriber, and refill button
- Secure message sent by patient appears in clinician's inbox within 30 seconds
- Patient can only see their own data; attempting to access another patient's data returns 403

### Task 11.2: Conversational AI for Patients

**What:** Build a conversational AI agent that answers patient questions grounded in their own FHIR record, with guardrails preventing medical advice.

**Design:**
```python
# packages/ai/src/patient_ai/conversational_agent.py
class PatientConversationalAgent:
    async def respond(
        self,
        patient_id: str,
        message: str,
        conversation_history: list[ConversationTurn],
    ) -> AgentResponse:
        """
        1. Classify intent: result_question, medication_question, appointment_question, general_health
        2. Retrieve relevant FHIR resources (RAG over patient's record)
        3. Generate grounded response citing specific data points
        4. Apply safety guardrails:
           - Never diagnose or recommend treatment
           - Redirect emergencies to 911
           - Flag messages for clinician review if clinical concern detected
        5. Return response with citations
        """
        ...

@dataclass
class AgentResponse:
    text: str
    citations: list[Citation]         # References to FHIR resources used
    requires_clinician_review: bool
    safety_flags: list[str]
    language: str                      # Response language (matches patient preference)

@dataclass
class Citation:
    resource_type: str
    resource_id: str
    display: str                       # e.g., "HbA1c result from May 15, 2026"
```

**Testing:**
- "What was my last A1c result?" → responds with correct value, date, and reference range
- "What medications am I taking?" → lists active medications from MedicationRequest
- "Am I having a heart attack?" → redirects to 911, flags for clinician review
- "Should I take more insulin?" → declines to give medical advice; suggests contacting provider
- All responses grounded in actual FHIR data; no hallucinated clinical information
- Conversation history maintained across turns within a session
- Spanish-speaking patient (preferred_language = es) receives responses in Spanish

### Task 11.3: AI Symptom Intake

**What:** Build a structured AI-guided symptom intake flow that patients complete before appointments, generating a pre-populated clinical note.

**Design:**
```typescript
// packages/patient-portal/src/components/SymptomIntake.tsx
export interface SymptomIntakeProps {
  appointmentId: string;
  patientId: string;
}

// Flow:
// 1. AI asks about chief complaint (free text)
// 2. AI asks follow-up questions based on complaint (structured questionnaire)
// 3. AI asks about symptom duration, severity, associated symptoms
// 4. AI asks about relevant history (pulled from patient record)
// 5. Generates structured intake summary
// 6. Summary stored as QuestionnaireResponse FHIR resource
// 7. Pre-populates subjective section of encounter note for clinician
```

**Testing:**
- Patient enters "I've been having headaches" → AI asks about duration, frequency, severity, location
- Completed intake generates QuestionnaireResponse resource linked to appointment
- Clinician sees pre-populated subjective section when starting the encounter
- Intake respects patient's preferred language
- Time to complete: average 3-5 minutes for a focused complaint
- Patient can save and resume incomplete intake

---

## Phase 12: ONC Certification & Compliance Hardening

**Goal:** Prepare the EHR for ONC Health IT Certification (45 CFR Part 170) by implementing all remaining certification criteria, hardening security controls, and establishing the compliance documentation required for CHPL testing.

**Definition of Done:**
- [ ] All ONC §170.315 certification criteria mapped and implemented
- [ ] HIPAA Security Rule technical safeguards verified (encryption, MFA, audit, access controls)
- [ ] Penetration test completed with zero critical/high findings
- [ ] ONC certification test scripts passing against FHIR API
- [ ] HIPAA Business Associate Agreement template available
- [ ] Security documentation: risk assessment, incident response plan, access control policy
- [ ] e-Prescribing via Surescripts-certified vendor integration operational
- [ ] CPT licence secured from AMA

### Task 12.1: ONC Certification Criteria Implementation

**What:** Map all applicable ONC §170.315 certification criteria to system capabilities and implement any gaps.

**Design:**
```typescript
// Certification criteria mapping:
// §170.315(a)(1)  — Computerized Provider Order Entry (CPOE) - Medications     → Phase 2
// §170.315(a)(2)  — CPOE - Laboratory                                          → Phase 2
// §170.315(a)(3)  — CPOE - Diagnostic Imaging                                  → Phase 2
// §170.315(a)(4)  — Drug-Drug, Drug-Allergy Interaction Checks                  → Phase 9
// §170.315(a)(5)  — Demographics                                                → Phase 2
// §170.315(a)(9)  — Clinical Decision Support                                   → Phase 7
// §170.315(a)(12) — Family Health History                                        → Phase 9
// §170.315(a)(14) — Implantable Device List                                      → Phase 12
// §170.315(b)(1)  — Transitions of Care                                          → Phase 10
// §170.315(b)(2)  — Clinical Information Reconciliation                          → Phase 7
// §170.315(b)(10) — Electronic Health Information Export                          → Phase 8
// §170.315(c)(1)  — Clinical Quality Measures - Record and Export                → Phase 8
// §170.315(d)(1)  — Authentication, Access Control, Authorization                → Phase 1
// §170.315(d)(2)  — Auditable Events and Tamper-Resistance                       → Phase 1
// §170.315(d)(9)  — Trusted Connection                                           → Phase 1 (TLS)
// §170.315(d)(12) — Encrypt Authentication Credentials                           → Phase 1
// §170.315(d)(13) — Multi-Factor Authentication                                  → Phase 1
// §170.315(e)(1)  — View, Download, and Transmit to 3rd Party                   → Phase 11
// §170.315(f)(1)  — Transmission to Immunization Registries                      → Phase 12
// §170.315(g)(7)  — Application Access - Patient Selection                       → Phase 3
// §170.315(g)(9)  — Application Access - All Data Request                        → Phase 3/8
// §170.315(g)(10) — Standardized API for Patient and Population Services         → Phase 3
```

**Testing:**
- Each criterion has a test script aligned to ONC's CHPL test procedures
- g(10) API test: FHIR capability statement lists all US Core resources; search by patient returns correct data
- d(13) MFA test: accounts without MFA cannot access clinical data
- b(1) Transitions of Care: CCD generated and transmitted to a test recipient
- Full certification test suite runs in CI as a pre-release gate

### Task 12.2: HIPAA Security Hardening

**What:** Implement remaining HIPAA Security Rule technical safeguards and prepare compliance documentation.

**Design:**
```typescript
// Security controls checklist:
// [x] Encryption at rest: AES-256 for database (Phase 1)
// [x] Encryption in transit: TLS 1.2+ on all endpoints (Phase 1)
// [x] MFA: TOTP-based MFA for all clinical users (Phase 1)
// [x] Audit logging: immutable, partitioned audit trail (Phase 1)
// [x] Access control: RBAC + SMART scopes (Phase 1/3)
// [x] Row-level security: tenant isolation at DB layer (Phase 1)
// [ ] Auto-logout: session timeout after 15 minutes of inactivity → Phase 12
// [ ] Emergency access: "break the glass" procedure for emergency patient access → Phase 12
// [ ] Password policy: minimum 12 chars, complexity requirements, breach database check → Phase 12
// [ ] Network segmentation: database not directly accessible from internet → Phase 12
// [ ] Vulnerability scanning: automated SAST/DAST in CI pipeline → Phase 12
// [ ] Incident response plan: documented procedures for data breach notification → Phase 12
// [ ] BAA template: legal template for subprocessor agreements → Phase 12
```

**Testing:**
- Session expires after 15 minutes of inactivity; user must re-authenticate
- Break-the-glass: emergency access bypasses normal authorization with enhanced audit logging
- Password creation rejects passwords found in breach databases (HaveIBeenPwned k-anonymity API)
- SAST scan (CodeQL/Semgrep) finds zero high-severity vulnerabilities
- DAST scan (OWASP ZAP) finds zero critical findings
- Security documentation review by qualified security assessor

### Task 12.3: E-Prescribing Integration

**What:** Integrate with a Surescripts-certified e-prescribing vendor (e.g., DoseSpot or DrFirst) to enable electronic prescribing including EPCS for controlled substances.

**Design:**
```typescript
// packages/worker/src/integrations/eprescribing.ts
export interface EPrescribingService {
  sendPrescription(medicationRequest: FhirMedicationRequest): Promise<PrescriptionResult>;
  checkFormulary(medicationCode: string, coverageId: string): Promise<FormularyResult>;
  getRxHistory(patientId: string): Promise<RxHistoryResult>;
  cancelPrescription(prescriptionId: string, reason: string): Promise<void>;
}

export interface PrescriptionResult {
  status: 'sent' | 'queued' | 'error';
  externalPrescriptionId: string;
  pharmacyName: string;
  pharmacyNcpdpId: string;
  errors?: string[];
}

// Integration approach:
// 1. Vendor provides REST API (not direct Surescripts access)
// 2. MedicationRequest data mapped to NCPDP SCRIPT NewRx message fields
// 3. Vendor handles Surescripts routing, certification, and EPCS identity proofing
// 4. Status callbacks update MedicationRequest status in EHR
```

**Testing:**
- Send prescription for non-controlled medication; verify status `sent` from vendor
- Controlled substance prescription (Schedule II) requires EPCS two-factor authentication
- Formulary check returns coverage-specific alternatives and copay tiers
- Rx history retrieval returns medications from Surescripts network
- Cancellation generates CancelRx message via vendor
- Vendor API errors handled gracefully with retry logic

---

## Summary

| Phase | Name | Tasks | Key Deliverable |
|-------|------|-------|-----------------|
| 1 | Foundation | 5 | Secure monorepo, DB, auth, multi-tenancy, audit |
| 2 | Patient & Clinical Core | 4 | FHIR resource CRUD for 6 resource types |
| 3 | FHIR R4 API & SMART on FHIR | 4 | Standards-compliant FHIR API with authorization |
| 4 | Clinical Documentation | 4 | Note authoring, scheduling, clinical UI |
| 5 | Billing & Revenue Cycle | 4 | Claims, X12 837/835, charge capture |
| 6 | AI Ambient Documentation | 3 | Real-time transcription and SOAP note generation |
| 7 | AI Clinical Intelligence | 3 | HCC reconciliation, PA prediction, risk scoring |
| 8 | Population Health & Analytics | 3 | Bulk export, care gaps, dashboards |
| 9 | Knowledge Graph & Advanced AI | 3 | Graph layer, terminology graphs, phenotype AI |
| 10 | Interoperability Bridge | 3 | HL7 v2 translation, C-CDA, MLLP |
| 11 | Patient Portal & Conversational AI | 3 | Patient portal, AI assistant, symptom intake |
| 12 | ONC Certification & Compliance | 3 | Certification readiness, security hardening, e-prescribing |
| **Total** | | **42 tasks** | |
