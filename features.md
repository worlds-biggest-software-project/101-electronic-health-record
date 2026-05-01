# Electronic Health Record (EHR) — Feature & Functionality Survey

> Candidate #101 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Epic Systems | Enterprise inpatient + ambulatory EHR | Proprietary (on-premise / hosted) | https://www.epic.com |
| Oracle Health (Cerner Millennium) | Enterprise inpatient + ambulatory EHR | Proprietary (on-premise / SaaS) | https://www.oracle.com/health |
| OpenEMR | Community EHR (ambulatory focus) | Open Source — GPL-2.0 | https://www.open-emr.org |
| OpenMRS | EHR platform (low-resource / global health) | Open Source — MPL-2.0 | https://openmrs.org |
| Medplum | Healthcare developer platform / headless EHR | Open Source — Apache-2.0 | https://www.medplum.com |
| Canvas Medical | Ambulatory specialty EHR (API-first) | Proprietary SaaS | https://www.canvasmedical.com |
| Elation Health | Primary care EHR (independent practices) | Proprietary SaaS | https://www.elationhealth.com |
| Athenahealth (AthenaOne) | Cloud ambulatory EHR + billing | Proprietary SaaS | https://www.athenahealth.com |
| DrChrono | Cloud EHR (independent practices, iPad-native) | Proprietary SaaS | https://www.drchrono.com |
| eClinicalWorks | Ambulatory EHR + AI documentation | Proprietary SaaS | https://www.eclinicalworks.com |

## Feature Analysis by Solution

### Epic Systems

**Core features**
- Comprehensive inpatient and ambulatory clinical documentation (SOAP notes, H&P, discharge summaries)
- CPOE (computerized provider order entry): medications, labs, imaging, referrals
- Clinical decision support: drug-drug interaction alerts, best practice advisories (BPAs), order sets
- MyChart patient portal: test results, appointment scheduling, messaging, billing
- Revenue cycle management: charge capture, claim submission, remittance, denial management
- Population health and care management: risk stratification, care gap identification, registries
- Interoperability: FHIR R4 API, SMART on FHIR, Care Everywhere (national record exchange network)
- Specialty modules: oncology (Beacon), surgery (OpTime), anesthesia, ICU, labor and delivery

**Differentiating features**
- Care Everywhere: Epic-to-Epic and Epic-to-non-Epic record sharing used by the majority of US hospitals
- Cosmos de-identified research database: pooled data from 230+ million patients across Epic sites
- 43.9% US inpatient hospital market share — dominant network effects for interoperability
- Epic App Orchard: SMART on FHIR app marketplace for third-party clinical and operational applications
- Cogito analytics platform: embedded predictive models (sepsis prediction, readmission risk, etc.)

**UX patterns**
- Role-specific workflows: separate interfaces for physicians, nurses, pharmacists, billing
- In-basket messaging system for care team communication
- Workflow-embedded documentation templates reducing free-text entry
- Mobile (Haiku for phone, Canto for tablet) for on-the-go chart access

**Integration points**
- HL7 FHIR R4 API and SMART on FHIR for third-party apps
- HL7 v2.x interfaces with lab, radiology, pharmacy, and ancillary systems
- Direct messaging and CDA document exchange for referrals and transitions of care
- Claims clearinghouse integrations (X12 837/835)

**Known gaps**
- Extremely high implementation cost and long go-live timelines (12–24 months typical)
- Complex customization requires dedicated IT staff and Epic-certified analysts
- Tightly closed ecosystem; limited external integration flexibility compared to FHIR-native platforms
- Small and independent practices effectively excluded by price

**Licence / IP notes**
- Proprietary; no source code disclosed. Epic is privately held.
- On-premise or Epic-hosted options; Epic retains ownership of the software.
- Customer data may reside in the Epic-hosted cloud; data portability contractual provisions vary.

---

### Oracle Health (Cerner Millennium)

**Core features**
- Inpatient and ambulatory clinical documentation across all care settings
- Millennium-based CPOE, clinical decision support, and order management
- Oracle Clinical Digital Assistant: voice-activated navigation, AI-generated draft notes from dictation/conversation
- Patient registration, scheduling, and bed management
- Revenue cycle: billing, coding, claims, and remittance processing
- Regulatory reporting and quality measure tracking (HEDIS, CMS)
- FHIR R4 API for patient access mandated under ONC rules

**Differentiating features**
- Oracle Clinical Digital Assistant launched 2025: generative AI + multimodal voice interface; early users report 4.5 minutes saved per patient and 20–40% documentation time reduction
- VA (Department of Veterans Affairs) deployment ongoing across US VA medical centers in 2026 — largest federal EHR rollout
- Oracle Cloud Infrastructure backing for scalability and data lake analytics
- Voice-first navigation as a design paradigm distinct from all other major EHRs

**UX patterns**
- Voice-activated commands for navigation, ordering, and note generation
- Contextual and conversational AI search within the clinical record
- Role-configured workflows for inpatient vs. ambulatory vs. emergency settings

**Integration points**
- HL7 FHIR R4 APIs (ONC-mandated and proprietary)
- HL7 v2.x and CDA for legacy system connectivity
- Cerner-to-non-Cerner HIE participation (CommonWell, Carequality)
- Oracle ERP and supply chain integration for hospital operations

**Known gaps**
- Large-scale VA deployment has experienced delays and challenges, raising implementation reliability concerns
- Enterprise pricing effectively limits to hospital and health-system buyers
- Market share has declined relative to Epic since 2022 acquisition; some health systems actively migrating away
- New AI-powered EHR platform (distinct from Millennium) still in ambulatory-only rollout as of 2026

**Licence / IP notes**
- Proprietary; Oracle Corporation ownership post-2022 acquisition of Cerner ($28.3B)
- No open-source components; cloud-hosted on Oracle Cloud Infrastructure

---

### OpenEMR

**Core features**
- Ambulatory clinical charting (SOAP notes, problem lists, medications, allergies)
- Appointment scheduling and patient demographics management
- Medical billing: charge entry, claim submission, ERA posting (X12 837/835)
- E-prescribing (EPCS-capable via Surescripts integration)
- Lab ordering and result management
- Patient portal: secure messaging, results, appointment requests
- FHIR R4 RESTful API with SMART on FHIR and OAuth2/OIDC (since v6.0, 2021)
- ONC HTI-1 certified as of January 30, 2026 (CHPL ID: 15.05.05.3115.OPEN.02.01.1.260130)

**Differentiating features**
- Most widely deployed open-source EHR globally; used in 100+ countries
- ONC certification maintained through community funding (OpenEMR Foundation)
- Full source code available enabling deep customization for national health programs and LMIC deployments
- Module marketplace for community-contributed clinical and billing extensions
- Deployed in both self-hosted (on-premise, cloud VPS) and managed cloud configurations

**UX patterns**
- Traditional web-based forms-driven interface; less modern than commercial competitors
- Configurable dashboards and custom form builders for specialty workflows
- Multi-language interface support

**Integration points**
- HL7 FHIR R4 and SMART on FHIR
- HL7 v2.x interfaces with labs and pharmacies
- Clearinghouse integrations for claim submission (Office Ally, Availity, others)
- Telehealth integrations (Zoom, Doxy.me) via module

**Known gaps**
- UI/UX significantly behind commercial peers; requires investment in custom theming for consumer-grade experience
- Clinical decision support is minimal compared to Epic / Oracle
- No native population health analytics or advanced risk stratification
- Implementation and ongoing maintenance requires technical staff; no vendor-managed upgrades
- Community support model; SLAs require paid commercial support contracts

**Licence / IP notes**
- **GPL-2.0** (GNU General Public License version 2).
- **Embedding concern — SIGNIFICANT**: GPL-2.0 is a strong copyleft licence. Any derivative work or software that statically or dynamically links OpenEMR code and is distributed must be released under GPL-2.0. An AI-native EHR that incorporates OpenEMR code directly would need to open-source its entire codebase under GPL-2.0 or seek a commercial licence exception (not currently offered by the OpenEMR Foundation).
- Safe usage model: run OpenEMR as a separate service and interact only via its published FHIR API — this avoids copyleft propagation.

---

### OpenMRS

**Core features**
- Patient demographics and clinical observations datastore (concept dictionary-driven)
- Encounter and visit management
- Clinical forms engine: configurable forms using HTML Form Entry or React-based form renderer
- FHIR R4 module for standard API access
- Reporting module: cohort-based reporting and data extraction
- Drug ordering and pharmacy dispensing

**Differentiating features**
- Concept dictionary model: all clinical data elements are defined as named concepts, enabling full localization for any language, terminology, or care context
- Purpose-designed for low-resource settings: operates on intermittent connectivity, low-spec hardware
- Deployed extensively in sub-Saharan Africa (Kenya, Rwanda, Uganda, Malawi), Southeast Asia, and Caribbean
- Large global implementer community (AMPATH, Partners In Health, Clinton Health Access Initiative, WHO-supported deployments)
- Module architecture: 200+ community modules for HIV care, TB management, maternal health, vaccination tracking

**UX patterns**
- Web-based interface designed for keyboard-driven data entry in high-volume clinic settings
- Offline-capable mobile clients (OpenMRS Android) for field data collection
- Reference Application UI as baseline; custom UIs built per-deployment

**Integration points**
- HL7 FHIR R4 via the FHIR module (SMART on FHIR in progress)
- HL7 v2.x for lab integrations (OpenELIS, others)
- DHIS2 integration for national health aggregate reporting
- GIS mapping modules for epidemiological surveillance

**Known gaps**
- Not designed for fee-for-service US billing; X12 EDI claim submission not supported
- Minimal or no revenue cycle management
- Advanced clinical decision support requires third-party modules with significant implementation effort
- UX designed for trained clinical users; not patient-facing or consumer-grade
- ONC certification not pursued (inappropriate for target market)

**Licence / IP notes**
- **MPL-2.0** (Mozilla Public License 2.0).
- MPL-2.0 is a **file-level copyleft** licence. Modified MPL-licensed files must remain open-source under MPL-2.0, but MPL code can be combined with proprietary code in a larger work without the proprietary code being required to open-source, provided the MPL and proprietary code are in separate files.
- **Low embedding concern**: An AI-native EHR can incorporate OpenMRS components, provided modified OpenMRS files are kept open and the proprietary application code is in distinct files. Legal review recommended before embedding.

---

### Medplum

**Core features**
- FHIR-native clinical data repository (CDR): all data stored as FHIR R4 resources
- FHIR RESTful API, GraphQL API, and subscription-based event streaming
- Medplum Auth: OAuth2 / OIDC / SMART on FHIR identity and access management
- Medplum Bots: server-side workflow automation in JavaScript/TypeScript without separate infrastructure
- React UI component library for building clinical interfaces
- HIPAA-compliant, SOC 2 Type II certified; BAA available for hosted deployments
- Supports 19+ million active patients; 120+ million workflow actions processed

**Differentiating features**
- Apache-2.0 licence: most permissive licence in this comparison; fully embeddable in proprietary products
- Developer-first platform; designed to be the backend infrastructure for custom health applications
- Self-hosted or fully managed cloud (app.medplum.com)
- "Headless EHR" paradigm: clinical data and workflow engine without opinionated clinical UI — consumers build their own
- Actively maintained on GitHub; strong developer community; 2026 recognized as revolutionary healthcare dev platform

**UX patterns**
- No end-user clinical UI out of the box; Medplum App is an admin/developer tool
- React component library provides UI primitives (FHIR resource forms, search, patient timeline)
- Builder-experience: clinical workflows designed in Medplum, consumed via custom UI

**Integration points**
- FHIR R4 APIs interoperate natively with Epic, Cerner, and all SMART on FHIR-compatible systems
- Webhook and event subscription system for real-time workflow triggers
- Pre-built connectors for lab orders, e-prescribing, and claims via third-party partner modules
- Open API enables any analytics or AI layer to sit on top

**Known gaps**
- No pre-built clinical UI means high front-end development investment to reach end-user readiness
- No native revenue cycle management (billing, claim scrubbing); requires integration with a billing system
- Clinical decision support rules engine not native; must be implemented via Bots or external CDSS
- ONC certification not yet obtained (as of 2026); would be needed for Meaningful Use-dependent customers

**Licence / IP notes**
- **Apache-2.0**: permissive licence with no copyleft. Code can be used, modified, and embedded in proprietary commercial products without obligation to open-source the surrounding codebase.
- **No embedding concern**. Ideal for an AI-native EHR that wants to leverage Medplum's FHIR CDR and infrastructure while retaining a proprietary clinical and AI layer.
- Patent grant clause in Apache-2.0 provides explicit patent rights from contributors.

---

### Canvas Medical

**Core features**
- Full clinical interface out-of-the-box: scheduling, Narrative Charting™, problem list, medications, labs, referrals
- Integrated billing: electronic claims submission and billing workflow
- FHIR API and SDK for customization and integration
- Population health tools: care gap lists, patient registries, outreach workflows
- Secure messaging and patient communication
- Telehealth integration

**Differentiating features**
- Decoupled EHR architecture: use the full Canvas clinical UI, or build a custom frontend while Canvas handles the compliant backend
- Named 2026 Best in KLAS for Ambulatory Specialty EHR
- Pricing based on monthly active patients (not per-provider seat); unlimited users and API calls included
- Implementation services and integrations bundled into single monthly fee
- Popular with digital health companies as a compliant backend while building custom patient-facing experiences

**UX patterns**
- Narrative Charting™: structured note-taking with natural-language flexibility
- API-first design: workflow triggers and automation available via the API to any authorized client
- Admin and clinical dashboards with care management task views

**Integration points**
- FHIR API for EHR-to-EHR data exchange
- SDK for custom application development
- Lab and e-prescribing integrations via third-party partners
- Telehealth and patient communication integrations

**Known gaps**
- Primarily ambulatory specialty focus; not designed for inpatient or hospital use cases
- Revenue cycle management less complete than Athenahealth or AdvancedMD
- Smaller customer base and ecosystem than Epic or Athenahealth
- Proprietary SaaS; no self-hosting option

**Licence / IP notes**
- Proprietary SaaS; Canvas Medical Inc.
- HIPAA BAA provided; SOC 2 certified

---

### Elation Health

**Core features**
- Primary care-focused clinical documentation (SOAP, problem list, medication management)
- Patient demographics, scheduling, and appointment management
- E-prescribing (controlled substances capable)
- Secure patient messaging and patient portal
- Care coordination: referral management and care transition documents
- Billing integration via RCM partners

**Differentiating features**
- Purpose-designed for independent primary care practices and direct primary care (DPC) models
- Strong reputation for physician-centric UX; high clinical satisfaction scores
- FHIR API supporting population health and care management overlays

**UX patterns**
- Clean, focused clinical chart designed for high-volume primary care workflows
- Minimal administrative overhead in clinical UI

**Integration points**
- FHIR R4 API
- HL7 v2.x lab interfaces
- RCM billing partner integrations
- Athenahealth and other clearinghouses for claims

**Known gaps**
- Limited inpatient, specialty, and enterprise capabilities
- Revenue cycle less integrated than Athenahealth
- Smaller marketplace ecosystem than Epic

**Licence / IP notes**
- Proprietary SaaS; Elation Health Inc.

---

### Athenahealth (AthenaOne)

**Core features**
- Cloud-native ambulatory EHR with integrated scheduling, billing, and patient engagement
- AthenaNet network: shared payer rules, denial patterns, and claim intelligence across all athenahealth customers
- Automated claim scrubbing using network-wide denial data
- Ambient Notes: AI-assisted clinical documentation from encounter audio
- Patient portal: messaging, bill pay, health history intake
- Population health: care gap reports, chronic disease registries, value-based care dashboards

**Differentiating features**
- Revenue cycle managed service model: athenahealth actively works denials and appeals on behalf of practices
- Network intelligence: payer rules and denial reasons learned from 160,000+ provider network shared in real time
- Cloud-native from inception (no on-premise version)
- % of collections pricing model aligns vendor incentives with practice revenue performance

**UX patterns**
- Task-based inbox driving daily clinical and administrative workflows
- Payer-specific claim editing suggested automatically before submission
- Patient engagement triggered by appointment workflow events

**Integration points**
- HL7 FHIR R4 and SMART on FHIR
- Marketplace for third-party EHR app integrations
- EDI 837/835 for claims and remittance
- Surescripts for e-prescribing

**Known gaps**
- Pricing (% of collections) can become expensive at high-revenue practices
- Less configurable than Epic for complex specialty workflows
- Inpatient use not supported

**Licence / IP notes**
- Proprietary SaaS; Athenahealth Inc. (held by Veritas Capital / Evergreen)

---

### DrChrono

**Core features**
- iPad-native and web EHR with clinical documentation, scheduling, and billing
- AI-powered documentation: ambient note generation, code suggestions
- Customizable clinical forms and templates for specialty workflows
- Integrated medical billing with claim submission and payment tracking
- Patient portal: online scheduling, intake forms, results access
- Telehealth: built-in video visits

**Differentiating features**
- iPad-native interface designed for mobile-first providers and concierge medicine
- AI-assisted note generation available at a lower price point than Epic/Oracle
- Part of EverCommerce portfolio since 2022; integration with business management tools

**UX patterns**
- Tablet-first clinical workflow; touch-optimized documentation
- Voice-to-text note dictation integrated

**Integration points**
- FHIR API for interoperability
- Surescripts e-prescribing
- Lab integrations and device connectivity
- Clearinghouse for claim submission

**Known gaps**
- Not suitable for large health systems or inpatient settings
- Billing capabilities less advanced than athenahealth's managed service model
- Customer support reviews mixed since EverCommerce acquisition

**Licence / IP notes**
- Proprietary SaaS; EverCommerce subsidiary (acquired 2022, ~$225M)

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Clinical documentation: SOAP notes, problem list, medication list, allergies, vital signs
- CPOE: lab orders, medication prescribing, imaging orders
- Appointment scheduling and patient demographics management
- Patient portal: secure messaging, result access, appointment requests
- E-prescribing (NCPDP SCRIPT; EPCS for controlled substances)
- HL7 FHIR R4 API (ONC-mandated for any US-market EHR)
- HIPAA-compliant data handling with audit logging and encryption
- Medical billing: charge capture, claim submission, ERA posting

### Differentiating Features
- AI ambient documentation (Oracle Clinical Digital Assistant, eClinicalWorks healow Genie, Athenahealth Ambient Notes)
- Network-intelligence claim scrubbing using cross-customer denial data (Athenahealth)
- Developer-first FHIR-native CDR as embeddable infrastructure (Medplum)
- Decoupled EHR architecture: compliant backend with custom frontend (Canvas Medical, Medplum)
- Global deployment for low-resource settings with offline capability (OpenMRS)
- Epic Care Everywhere network for national record exchange (Epic)
- Voice-first clinical interface with generative AI note generation (Oracle)

### Underserved Areas / Opportunities
- True ambient documentation natively integrated in an open, extensible EHR (not requiring third-party add-ons)
- AI-powered continuous problem list reconciliation and HCC coding suggestion at point of care
- Patient-facing conversational AI grounded in the patient's own FHIR record (beyond static portals)
- Cross-system FHIR brokering: intelligent translation between legacy HL7 v2, CDA, and FHIR R4 without fragile point-to-point integrations
- Predictive risk stratification surfaced directly in care management workflow without separate analytics platform
- Affordable, ONC-certified EHR for independent and small group practices that does not charge per-provider seat

### AI-Augmentation Candidates
- Ambient clinical documentation: listen to encounter audio, generate SOAP note, suggest billing codes
- HCC and problem list AI reconciliation: surface missing diagnoses from unstructured notes
- Prior authorization prediction: flag procedures likely to require PA before order submission
- Sepsis and deterioration prediction embedded in nursing workflow
- Patient-facing AI triage: structured symptom collection before appointment, pre-populating clinical note
- FHIR data normalization: AI translation layer mapping legacy data formats into FHIR R4 canonical resources

## Legal & IP Summary

- **GPL-2.0 (OpenEMR)**: Strong copyleft. Embedding or forking OpenEMR into a proprietary AI-native EHR would require the entire derivative work to be GPL-2.0 licensed. Safe approach: use OpenEMR only via its published FHIR API (network use does not trigger copyleft under GPL-2.0 / Affero distinction).
- **MPL-2.0 (OpenMRS)**: File-level copyleft only. Proprietary code in separate files can coexist. Lower embedding concern; legal review recommended before direct code incorporation.
- **Apache-2.0 (Medplum)**: Fully permissive. No copyleft. Can be embedded in a proprietary product without any open-source obligation. Strongly preferred as a foundation for an AI-native EHR platform.
- All commercial solutions (Epic, Oracle, Athenahealth, Canvas, DrChrono, Elation) are fully proprietary. No embedding rights; API-level integration only.
- **CPT code set (AMA)**: Required for billing but is a licensed dataset owned by the American Medical Association. Any EHR displaying or processing CPT codes requires an AMA licence. This is a significant ongoing IP cost.
- **ICD-10-CM/PCS**: Owned by the US government (CMS / NCHS); free to use in the US without licence.
- **SNOMED CT**: Licensed by SNOMED International; US National Library of Medicine provides a free US licence for domestic EHRs.

## Recommended Feature Scope

**Must-have (MVP)**:
- FHIR R4-native clinical data repository with complete patient demographics, problem list, medications, allergies, and clinical encounters
- Clinical documentation: structured SOAP/progress notes with customizable templates per specialty
- Appointment scheduling, patient demographics, and basic patient portal (results access, secure messaging)
- E-prescribing via NCPDP SCRIPT integration (Surescripts)
- Medical billing pipeline: charge capture, X12 837 claim generation, ERA/835 posting
- ONC certification path or SMART on FHIR compliance for Meaningful Use–adjacent customers

**Should-have (v1.1)**:
- AI ambient documentation: real-time encounter transcription with SOAP note draft generation
- Automated HCC / billing code suggestions from clinical note content
- Prior authorization detection and workflow initiation before order submission
- Population health dashboard: care gap lists, chronic disease registries, risk stratification scores
- Lab and imaging order integration with results inbox and abnormal-result alerting

**Nice-to-have (backlog)**:
- Patient-facing conversational AI grounded in the patient's FHIR record (replace static portal)
- Predictive sepsis / readmission risk model embedded in nursing / care-team workflow
- Cross-system FHIR normalization broker: auto-translate HL7 v2 and CDA feeds into FHIR resources
- Multi-language clinical interface and patient portal
- SMART on FHIR app marketplace for third-party clinical application integrations
