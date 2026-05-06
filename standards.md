# Standards & API Reference

> Project: Electronic Health Record (EHR) · Generated: 2026-05-06

## Industry Standards & Specifications

### HL7 Standards

**HL7 FHIR R4 (v4.0.1)**
- **URL:** https://hl7.org/fhir/R4/
- The primary modern standard for EHR data exchange. All FHIR R4 resources are normative and stable. Mandated by the ONC 21st Century Cures Act Final Rule for any EHR claiming ONC certification. The base format for US Core, SMART on FHIR, and bulk data APIs. An AI-native EHR must implement a FHIR R4-native clinical data repository.

**HL7 FHIR R5 (v5.0.0)**
- **URL:** https://hl7.org/fhir/R5/
- Published STU; supersedes R4 with new resources and maturity improvements. Not yet mandated by US federal regulation but gaining adoption for new implementations. US Core v9.0.0 (development) is tracking R5. Architects should design for R4/R5 compatibility paths.

**HL7 FHIR Bulk Data Access IG (Release 2 / v3.0.0-ballot)**
- **URL:** https://hl7.org/fhir/uv/bulkdata/
- Defines the $export asynchronous operation for population-scale data extraction using NDJSON flat files. Required by CMS interoperability rules (CMS-0057-F) for payer Patient Access and Payer-to-Payer APIs. Essential for population health analytics, quality measure calculation, and ML training pipelines.

**US Core FHIR Implementation Guide (STU8 / v8.0.1)**
- **URL:** https://hl7.org/fhir/us/core/ (STU6.1.0 stable), https://build.fhir.org/ig/HL7/US-Core/ (v9.0.0 development)
- US-realm FHIR profiles constraining the minimum data elements required for ONC certification. Defines must-support obligations for Patient, Encounter, Condition, Observation, MedicationRequest, AllergyIntolerance, Immunization, Procedure, DiagnosticReport, and other resources. Any ONC-certifiable EHR must conform to US Core.

**HL7 SMART App Launch Framework (v2.2.0)**
- **URL:** https://hl7.org/fhir/smart-app-launch/
- OAuth 2.0 and OpenID Connect extension for launching clinical apps from within EHR contexts. Defines EHR launch, standalone launch, patient-facing app authorization, and backend services authentication. Mandated under ONC §170.315(g)(10) certification criterion. SMART proxy will retire in September 2026; SMART Enhanced (v1.0.0 or v2.0.0) required.

**HL7 v2.x (ADT / ORU / ORM messages)**
- **URL:** https://www.hl7.org/implement/standards/product_brief.cfm?product_id=185
- Legacy pipe-delimited message format still dominant for real-time event streaming: ADT (admission/discharge/transfer), ORU (lab results), ORM (orders). Pervasive in hospital integration engines (Mirth, Rhapsody, Azure Health Data Services). An AI-native EHR acting as a FHIR broker must ingest and translate HL7 v2 feeds.

**HL7 Clinical Document Architecture Release 2 (CDA R2) / C-CDA Edition 4**
- **URL:** https://www.hl7.org/implement/standards/product_brief.cfm?product_id=492
- XML-based standard for clinical document exchange (Continuity of Care Document, Discharge Summary, Referral Note, Progress Note, etc.). The Consolidated CDA (C-CDA) Edition 4 is the current US realm implementation guide. Used for transitions of care between providers and Meaningful Use documentation requirements. Sequoia/eHealthExchange carries billions of CDA documents annually.

---

### ISO Standards

**ISO/HL7 27932:2009 — HL7 Clinical Document Architecture (CDA R2)**
- **URL:** https://www.iso.org/standard/44509.html
- International ISO adoption of HL7 CDA R2; establishes CDA as an internationally recognized document interchange standard for EHRs.

**ISO 13606 (EHRcom) — Electronic Health Record Communication**
- **URL:** https://www.iso.org/standard/62305.html
- Multi-part ISO standard defining the architecture and information model for communicating EHR extracts between systems. Uses a dual-model approach (Reference Model + Archetype Object Model, adopted from openEHR). Widely adopted in Europe; the Archetype Object Model (AOM2) is part of ISO 13606-2. Complements FHIR for deep structural EHR interoperability, particularly in research and national EHR programs.

**ISO/IEC 27001 — Information Security Management Systems**
- **URL:** https://www.iso.org/standard/27001
- The baseline information security management standard. Required by enterprise healthcare buyers and SOC 2 Type II certification bodies. Covers access control, audit logging, encryption, and incident response — all mandatory for HIPAA compliance.

---

### W3C & IETF Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect Core 1.0**
- **URL:** https://datatracker.ietf.org/doc/html/rfc6749 | https://openid.net/specs/openid-connect-core-1_0.html
- The authorization and authentication foundations beneath SMART on FHIR. All FHIR-based EHR APIs must implement OAuth 2.0 bearer token authorization. OpenID Connect provides identity tokens for user authentication during EHR launch. ONC §170.315(g)(10) mandates TLS 1.2+ for all API endpoints.

**RFC 7617 / RFC 7235 — HTTP Authentication**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7617
- HTTP Basic and Bearer token authentication schemes used in FHIR REST API implementations.

**JSON Schema (draft-07 and later)**
- **URL:** https://json-schema.org/specification
- Used for validating FHIR JSON payloads and OpenAPI specification schemas. FHIR R4 resources are expressible as JSON Schema.

**W3C PROV-O (Provenance Ontology)**
- **URL:** https://www.w3.org/TR/prov-o/
- Relevant to AI-generated clinical documentation audit trails; ensures transparency and traceability for AI-authored content added to the EHR, which is a growing regulatory concern.

---

### Data Model & API Specifications

**FHIR RESTful API (FHIR HTTP)**
- **URL:** https://hl7.org/fhir/R4/http.html
- The HTTP-based RESTful interaction model defined within the FHIR specification: read, vread, update, patch, delete, history, search, create, and batch/transaction operations. Defines the URL convention `[base]/[ResourceType]/[id]`.

**OpenAPI 3.x Specification**
- **URL:** https://spec.openapis.org/oas/v3.1.0
- The standard format for documenting REST APIs. FHIR capability statements can be published as OpenAPI documents. Athenahealth, Canvas Medical, and Medplum expose OpenAPI-compatible API documentation.

**NCPDP SCRIPT Standard (v2023011)**
- **URL:** https://www.ncpdp.org/Standards-Development/Standards-Information/Standards-Roster
- The US standard for electronic prescribing transactions (NewRx, RxRenewal, RxChange, CancelRx, RxFill). Version 2023011 is the current release, available via Surescripts for early adopters as of early 2026. All e-prescribing integrations in the US transit through Surescripts-certified middleware; there is no direct developer API. Integration requires a certified e-prescribing vendor relationship.

**ANSI ASC X12 Version 005010 — EDI Healthcare Transactions**
- **URL:** https://x12.org/products/transaction-sets
- The HIPAA-mandated EDI format for healthcare claims and remittance: 837P (professional claim), 837I (institutional claim), 837D (dental claim), 835 (payment/remittance advice), 270/271 (eligibility inquiry/response), 278 (prior authorization). Required for Medicare/Medicaid and most commercial payer claim submission. Clearinghouse partners (Availity, Change Healthcare, Office Ally) provide API abstraction over raw X12 EDI.

**DICOM PS3.18 — Web Services (DICOMweb)**
- **URL:** https://www.dicomstandard.org/using/dicomweb
- RESTful services for medical imaging: WADO-RS (retrieve), STOW-RS (store), QIDO-RS (query). Allows browser and mobile access to imaging studies without full DICOM tooling. Implemented by cloud imaging platforms (Azure Health Data Services DICOM, AWS HealthImaging, Google Cloud Healthcare API). Required for EHR radiology integration.

---

### Security & Compliance Standards

**HIPAA Security Rule (45 CFR Part 164, Subpart C)**
- **URL:** https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html
- Mandates administrative, physical, and technical safeguards for electronic protected health information (ePHI). A proposed HIPAA Security Rule update (NPRM filed January 2025, targeting finalization by mid-2026) would require: encryption of ePHI at rest (AES-256) and in transit (TLS 1.2+); MFA for system access; annual vulnerability scans; network segmentation. Any EHR handling US patient data must maintain a HIPAA Business Associate Agreement (BAA) with all data subprocessors.

**HIPAA Privacy Rule (45 CFR Part 164, Subpart E)**
- **URL:** https://www.hhs.gov/hipaa/for-professionals/privacy/index.html
- Governs permissible uses and disclosures of protected health information. Dictates patient consent workflows, notice of privacy practices, and access rights that patient portal and AI features must respect.

**ONC Health IT Certification Program — §170.315(g)(10)**
- **URL:** https://www.healthit.gov/test-method/standardized-api-patient-and-population-services
- The ONC certification criterion for standardized API access. Requires a FHIR R4 patient and population API with SMART on FHIR authorization, US Core–conformant resource exposure, and bulk data export. Any EHR claiming ONC certification must pass CHPL testing against this criterion. As of January 2026, US Core 4.0.0 and Bulk Data v1.0.0 minimum versions have expired; current requirements reference US Core 6.1.0+.

**CMS Interoperability & Prior Authorization Final Rule (CMS-0057-F)**
- **URL:** https://www.cms.gov/newsroom/fact-sheets/cms-interoperability-prior-authorization-final-rule-cms-0057-f
- Requires Medicare Advantage, Medicaid managed care, CHIP, and QHP payers to implement FHIR Patient Access API, Provider Directory API, Provider Access API, Payer-to-Payer API, and Prior Authorization API. EHRs serving covered payers must support these APIs by January 2026 (reporting) and January 2027 (full API compliance). Directly drives EHR feature requirements for prior authorization automation.

**NIST SP 800-53 Rev. 5 — Security and Privacy Controls**
- **URL:** https://doi.org/10.6028/NIST.SP.800-53r5
- The US federal security control framework widely used by healthcare organizations to structure security programs. Complements HIPAA and ONC certification requirements. Relevant to system access controls, audit and accountability, incident response, and supply chain risk management for AI components.

---

### Terminology Standards

**SNOMED CT (Systematized Nomenclature of Medicine — Clinical Terms)**
- **URL:** https://www.snomed.org/ | NLM US affiliate: https://www.nlm.nih.gov/healthit/snomedct/
- The global standard clinical terminology for diagnoses, clinical findings, procedures, and body structures. Required for ONC certification. In the US, the National Library of Medicine provides a free licence to US-based EHR developers. FHIR ValueSets reference SNOMED CT for clinical concepts throughout US Core profiles.

**LOINC (Logical Observation Identifiers Names and Codes)**
- **URL:** https://loinc.org/
- Standard codes for laboratory tests, clinical measurements, vital signs, and clinical document types. Freely available. Required for ONC certification and laboratory result interoperability. FHIR Observation resources use LOINC codes as the primary coding system for observations.

**RxNorm**
- **URL:** https://www.nlm.nih.gov/research/umls/rxnorm/
- NLM's normalized drug nomenclature for clinical drugs and drug interactions. The standard drug code system for FHIR MedicationRequest and MedicationStatement resources. Required for e-prescribing interoperability.

**ICD-10-CM / ICD-10-PCS**
- **URL:** https://www.cms.gov/medicare/coding-billing/icd-10-codes
- US standard diagnosis (CM) and procedure (PCS) codes for billing and clinical documentation. Maintained by CMS and NCHS; freely usable in the US without licence. Required for X12 837 claim submission and ONC-certified clinical documentation.

**CPT (Current Procedural Terminology)**
- **URL:** https://www.ama-assn.org/practice-management/cpt
- AMA-owned and licensed procedure codes required for professional medical billing. Any EHR displaying or processing CPT codes requires an AMA licence. This is a significant ongoing IP cost for any commercial EHR.

**NLM Value Set Authority Center (VSAC)**
- **URL:** https://vsac.nlm.nih.gov/
- Authoritative repository of value sets used in clinical quality measures and ONC certification criteria. Provides FHIR Terminology Service API (REST) for programmatic access to value sets across SNOMED CT, LOINC, RxNorm, and other code systems. Required UMLS licence (free).

---

### Data Exchange Frameworks

**openEHR**
- **URL:** https://specifications.openehr.org/ | https://openehr.org/
- Open specification for EHR data persistence and query using Archetypes (formal clinical content models defined by clinicians). Widely adopted in national health programs (Norway, Brazil, UK NHS regions, Slovenia). Complements FHIR for complex structured EHR persistence; an openEHR CDR can expose a FHIR API facade. Not mandated in the US; relevant for global deployments.

**IHE Integration Profiles**
- **URL:** https://profiles.ihe.net/
- Integrating the Healthcare Enterprise (IHE) profiles for cross-organizational document sharing and patient identity management:
  - **XDS.b (Cross-Enterprise Document Sharing):** Registry/repository architecture for sharing CDA and other clinical documents across organizations.
  - **MHD (Mobile Health Documents / FHIR-based):** RESTful FHIR equivalent of XDS for mobile and lightweight clients.
  - **PIX/PDQ (Patient Identifier Cross-referencing / Patient Demographics Query):** Patient matching across organizational identity domains.
  - **PDQm (Patient Demographics Query for Mobile):** FHIR RESTful version of PDQ.
  - **ATNA (Audit Trail and Node Authentication):** Audit logging and TLS node authentication for HIE participants.

**Carequality Framework**
- **URL:** https://carequality.org/
- A national-level policy and technical framework (built on IHE profiles) enabling FHIR and CDA document exchange between health networks (Epic Care Everywhere, CommonWell, Surescripts, etc.). Necessary for any EHR claiming broad HIE connectivity.

**CommonWell Health Alliance**
- **URL:** https://www.commonwellalliance.org/
- Industry-led interoperability network competing/cooperating with Carequality; Oracle Health (Cerner) and many ambulatory EHR vendors participate. Uses FHIR and HL7 v2 for patient matching and record sharing.

---

## Similar Products — Developer Documentation & APIs

### Medplum

- **Description:** Open-source, FHIR-native healthcare developer platform. Acts as a headless EHR backend: clinical data repository, workflow automation (Bots), and auth layer. Apache-2.0 licenced; fully embeddable in proprietary products.
- **API Documentation:** https://www.medplum.com/docs/api/fhir
- **API Overview:** https://www.medplum.com/docs/api
- **SDKs/Libraries:** TypeScript/JavaScript SDK (auto-generated types for all FHIR R4 resources); Python client via FHIR REST; React UI component library
  - GitHub: https://github.com/medplum/medplum
- **Developer Guide:** https://www.medplum.com/docs (FHIR basics, Bots, authentication, clinical workflows)
- **Standards:** FHIR R4 (v4.0.1) native; SMART on FHIR v2; US Core; Bulk Data; HIPAA BAA available; SOC 2 Type II
- **Authentication:** OAuth 2.0 / OIDC; SMART on FHIR backend services; API key
- **Base URL (hosted):** https://api.medplum.com/fhir/R4

---

### Epic on FHIR (Open.Epic)

- **Description:** Epic's developer platform for building SMART on FHIR apps and integrating with Epic EHR instances via standard FHIR R4/R5 APIs. Dominant US inpatient EHR (43.9% market share); critical integration target for any health app.
- **API Documentation:** https://fhir.epic.com/Documentation
- **Developer Portal:** https://open.epic.com/
- **SDKs/Libraries:** No official SDK; standard FHIR R4 REST with Epic-specific extensions. Community FHIR client libraries (JavaScript, Python, .NET) work against Epic endpoints.
- **Developer Guide:** https://fhir.epic.com/ (includes sandbox, app registration, SMART launch guides)
- **Standards:** FHIR R4 (with R5 preview endpoints); SMART on FHIR v2.2; US Core; Bulk Data ($export); SMART Health Cards
- **Authentication:** SMART on FHIR (EHR launch + standalone launch); backend services (JWT/JWKS); OAuth 2.0

---

### Oracle Health (Cerner Millennium) — FHIR APIs

- **Description:** Second-largest US inpatient EHR (Oracle Health Millennium platform). Provides FHIR R4 APIs replacing deprecated DSTU2 APIs. Essential integration target for hospital-based digital health apps.
- **API Documentation:** https://docs.oracle.com/en/industries/health/millennium-platform-apis/mfrap/r4_overview.html
- **Developer Portal:** https://docs.oracle.com/en/industries/health/millennium-platform-apis/
- **SDKs/Libraries:** No official SDK; FHIR R4 REST. Developers register via CernerCare account at code.cerner.com.
- **Developer Guide:** https://docs.oracle.com/en/industries/health/millennium-platform-apis/ (FHIR app provisioning, authorization framework)
- **Standards:** FHIR R4; SMART on FHIR v1 and v2; US Core; Bulk Data; CommonWell / Carequality participation
- **Authentication:** OAuth 2.0 / SMART on FHIR; SMART v1 and SMART v2 scopes supported

---

### Athenahealth (AthenaOne)

- **Description:** Cloud-native ambulatory EHR with 160,000+ provider network. Provides both proprietary REST APIs (primary) and ONC-mandated FHIR R4 APIs (for patient and population access).
- **API Documentation:** https://docs.athenahealth.com/api
- **FHIR API Docs:** https://docs.athenahealth.com/api/docs/fhir-apis
- **Developer Portal:** https://www.athenahealth.com/developer-portal
- **SDKs/Libraries:** GitHub sample: https://github.com/athenahealth/apiserver-athenaFlex (athenaPractice/athenaFlow FHIR server samples)
- **Developer Guide:** https://docs.athenahealth.com/api/guides/overview (800+ proprietary REST endpoints; FHIR base URLs guide)
- **Standards:** FHIR R4 (ONC g(10)); US Core (athenaCore IG); SMART on FHIR; EDI 837/835 via clearinghouse; Surescripts e-prescribing
- **Authentication:** OAuth 2.0; SMART on FHIR for FHIR endpoints; API key for proprietary REST

---

### Canvas Medical

- **Description:** Ambulatory specialty EHR with API-first, decoupled architecture. FHIR R4 API plus a Python SDK for native plugin development. Best in KLAS 2026 for Ambulatory Specialty EHR.
- **API Documentation:** https://docs.canvasmedical.com/api/
- **Developer Portal:** https://docs.canvasmedical.com/
- **SDKs/Libraries:** Python Canvas SDK for server-side plugins; Bruno collection for API testing; FHIR REST API with 37 resources (21 writable)
- **Developer Guide:** https://docs.canvasmedical.com/learn/overview/ (quickstart, SDK overview, FHIR resource guides)
- **Standards:** FHIR R4; SMART on FHIR; 650+ real-time clinical and operational events; HIPAA BAA; SOC 2
- **Authentication:** OAuth 2.0; SMART on FHIR

---

### OpenEMR

- **Description:** Most widely deployed open-source EHR globally (100+ countries). ONC HTI-1 certified (January 2026, CHPL ID: 15.05.05.3115.OPEN.02.01.1.260130). Current stable release: OpenEMR 8.0.0 (February 2026). GPL-2.0 licenced — strong copyleft (see features.md Legal & IP Summary).
- **API Documentation:** https://www.open-emr.org/wiki/index.php/OpenEMR_7.0.2_API
- **GitHub (FHIR docs):** https://github.com/openemr/openemr/blob/master/Documentation/api/FHIR_API.md
- **SMART on FHIR docs:** https://github.com/openemr/openemr/blob/master/Documentation/api/SMART_ON_FHIR.md
- **SDKs/Libraries:** Standard FHIR R4 REST; Swagger/OpenAPI UI available in the OpenEMR installation at `/swagger/`
- **Developer Guide:** https://github.com/openemr/openemr/blob/master/API_README.md
- **Standards:** FHIR R4; US Core 8.0; SMART on FHIR v2.2.0; Bulk Export; HL7 v2.x; X12 837/835
- **Authentication:** OAuth 2.0 / OIDC; SMART on FHIR

---

### Azure Health Data Services (Microsoft)

- **Description:** Managed cloud FHIR server and DICOM service from Microsoft Azure. Not an EHR but a widely used FHIR-as-a-service backend for healthcare applications and EHR data aggregation.
- **FHIR Service docs:** https://learn.microsoft.com/en-us/azure/healthcare-apis/fhir/
- **DICOM Service docs:** https://learn.microsoft.com/en-us/azure/healthcare-apis/dicom/dicomweb-standard-apis-with-dicom-services
- **SMART on FHIR docs:** https://learn.microsoft.com/en-us/azure/healthcare-apis/fhir/smart-on-fhir
- **SDKs/Libraries:** Azure SDKs (Python, .NET, Java, JavaScript); Terraform/Bicep for IaC provisioning
- **Standards:** FHIR R4 / R4B; SMART on FHIR; DICOMweb (WADO-RS, STOW-RS, QIDO-RS); US Core; HL7 v2 via Azure API for HL7
- **Authentication:** Azure Active Directory (Entra ID); OAuth 2.0; SMART on FHIR

---

### Google Cloud Healthcare API

- **Description:** Managed FHIR, DICOM, and HL7 v2 data services on Google Cloud. Commonly used by digital health companies building FHIR-native backends.
- **FHIR API docs:** https://cloud.google.com/healthcare-api/docs/how-tos/fhir
- **DICOMweb docs:** https://cloud.google.com/healthcare-api/docs/how-tos/dicomweb
- **SDKs/Libraries:** Google Cloud Python, Node.js, Java, Go, .NET client libraries
- **Standards:** FHIR R4; DICOMweb; HL7 v2 messaging; SMART on FHIR; US Core
- **Authentication:** Google Cloud IAM; OAuth 2.0; SMART on FHIR

---

### AWS HealthImaging & AWS HealthLake

- **Description:** AWS services for medical imaging (HealthImaging with DICOMweb) and FHIR data lake (HealthLake). Used for large-scale EHR analytics and AI/ML pipelines on patient data.
- **HealthLake FHIR docs:** https://docs.aws.amazon.com/healthlake/
- **HealthImaging DICOM docs:** https://docs.aws.amazon.com/healthimaging/latest/devguide/dicomweb-retrieve.html
- **SDKs/Libraries:** AWS SDK (Python boto3, JavaScript, Java, .NET, Go); AWS CDK for infrastructure
- **Standards:** FHIR R4; DICOMweb (WADO-RS); SMART on FHIR (via Cognito)
- **Authentication:** AWS IAM; OAuth 2.0

---

## Notes

**Regulatory trajectory:** The US regulatory environment is tightening around FHIR adoption. CMS-0057-F mandates payer FHIR APIs from 2026–2027, the HIPAA Security Rule NPRM proposes stricter encryption and MFA requirements (targeting finalization mid-2026), and ONC is continuously advancing US Core and Bulk Data version requirements. Any AI-native EHR must be designed to track these evolving baselines rather than targeting a single static version.

**Surescripts access:** E-prescribing via NCPDP SCRIPT is exclusively mediated through Surescripts-certified middleware. There is no open developer API. An AI-native EHR must establish a relationship with a Surescripts-certified vendor (e.g., DoseSpot, DrFirst, Veradigm) to offer e-prescribing capability.

**CPT licensing:** CPT codes (AMA-owned) carry ongoing licence costs. An AI-native EHR must budget for an AMA CPT licence agreement before enabling billing code display or AI code suggestion features.

**FHIR R4 vs R5:** US federal mandates reference FHIR R4. R5 is not yet mandated but is the forward-looking standard. Designing internal data stores as FHIR R4 resources with an adapter layer for R5 is the recommended approach for new implementations in 2026.

**openEHR for global deployments:** Projects targeting non-US national health systems (EU, LATAM, Asia-Pacific) should evaluate openEHR as a persistence and semantic layer alongside FHIR APIs for exchange. The two are complementary and increasingly used together in national digital health programs.
