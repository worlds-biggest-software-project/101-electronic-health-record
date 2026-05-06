# Electronic Health Record (EHR)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, FHIR-first electronic health record that brings ambient documentation, predictive risk stratification, and intelligent interoperability to clinicians without enterprise pricing or vendor lock-in.

This project is a candidate open-source EHR designed around HL7 FHIR R4 and modern AI capabilities. It targets independent practices, digital health builders, and global health programs that are priced out of incumbent enterprise systems or constrained by closed ecosystems.

---

## Why EHR?

- Enterprise EHRs (Epic, Oracle Health, MEDITECH) carry total costs of ownership in the USD 10M–100M+ range and 12–24 month go-live timelines, effectively excluding small and independent practices.
- Cloud ambulatory EHRs charge USD 100–1,200 per provider per month or take a percentage of collections, scaling unfavourably as practice revenue grows.
- Existing open-source options have trade-offs: OpenEMR is GPL-2.0 (strong copyleft, hard to embed) with a dated UI; OpenMRS is purpose-built for low-resource settings and lacks US billing; Medplum is a developer-first headless platform with no clinical UI out of the box.
- True ambient documentation, point-of-care HCC reconciliation, and patient-facing conversational AI grounded in the patient's own record are not natively available in any open EHR today.
- Mandated FHIR R4 and SMART on FHIR APIs (US ONC 21st Century Cures Act) create an opening for an AI-native, standards-first alternative that interoperates with the installed base rather than replacing it.

---

## Key Features

### Clinical Data & Documentation

- FHIR R4-native clinical data repository: demographics, problem list, medications, allergies, encounters stored as FHIR resources.
- Structured SOAP and progress notes with customizable templates per specialty.
- Problem list, medication list, allergy list, and vitals capture aligned to SNOMED CT and LOINC.
- CPOE for medications, labs, and imaging orders.

### Practice Operations & Billing

- Appointment scheduling and patient demographics management.
- Charge capture and X12 837 claim generation with ERA/835 remittance posting.
- E-prescribing through NCPDP SCRIPT integration (Surescripts), EPCS-capable for controlled substances.
- Patient portal with secure messaging, results access, and appointment requests.

### Interoperability & Standards

- HL7 FHIR R4 RESTful API with SMART on FHIR (OAuth2 / OIDC) authorization.
- HL7 v2.x interfaces for legacy lab, radiology, and pharmacy connectivity.
- C-CDA document exchange for referrals and transitions of care.
- ONC certification path (45 CFR Part 170) and HIPAA-compliant audit logging and encryption.

### AI-Augmented Workflows

- Ambient clinical documentation: encounter audio transcribed into draft SOAP notes with suggested billing codes.
- Continuous problem-list and HCC reconciliation surfacing missing diagnoses from unstructured notes.
- Prior authorization detection before order submission.
- Population health dashboards: care gap lists, chronic disease registries, and risk stratification scores.

### Patient-Facing Intelligence

- Conversational AI grounded in the patient's own FHIR record for results, medications, and appointment questions.
- Structured AI symptom intake before appointments, pre-populating the clinical note.
- Multi-language clinical interface and patient portal.

---

## AI-Native Advantage

AI is treated as core infrastructure, not an add-on module. Ambient documentation drafts notes, orders, and codes in real time during the visit, attacking the documentation burden that drives clinician burnout. LLMs over longitudinal FHIR data flag patients at risk of sepsis, readmission, or non-adherence before deterioration, and continuously suggest HCCs and suspect diagnoses that affect risk-adjusted revenue. An AI-driven FHIR broker translates legacy HL7 v2 and CDA traffic into canonical FHIR resources, replacing fragile point-to-point integrations with intelligent normalization.

---

## Tech Stack & Deployment

- FHIR R4 / R4B as the primary data model; HL7 v2.x and C-CDA bridged via translation services.
- SMART on FHIR with OAuth2 / OIDC for app authorization and third-party integrations.
- Standards alignment: SNOMED CT, LOINC, DICOM, IHE XDS / PIX / PDQ profiles.
- Deployment modes: self-hosted (on-premise or customer cloud) and managed cloud, with HIPAA-compliant encryption and audit logging.
- An Apache-2.0 foundation such as Medplum's FHIR CDR is a candidate substrate, since GPL-2.0 (OpenEMR) propagates copyleft and MPL-2.0 (OpenMRS) imposes file-level constraints.

---

## Market Context

The global EHR market was approximately USD 35.89 billion in 2025 and is projected to reach USD 53.11 billion by 2033 at a ~5.1% CAGR, with North America accounting for 47.33% of spend (Grand View Research, 2025). Incumbent pricing spans USD 100–500 per provider per month for solo practices, USD 400–1,200 or a percentage of collections for groups, and USD 10M–100M+ total cost of ownership for enterprise health systems. Primary buyers are CMIOs and CIOs at health systems, practice managers at independent groups, digital health officers at government health ministries, and behavioral health practice owners.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
