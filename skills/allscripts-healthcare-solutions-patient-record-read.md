---
name: Read a patient's clinical record from Veradigm FHIR
description: >-
  Authorize with SMART on FHIR against a specific Veradigm customer environment, then read a patient and
  their USCDI clinical resources from the Veradigm (formerly Allscripts) FHIR R4 API.
api: fhir/allscripts-healthcare-solutions-veradigm-fhir-r4-capabilitystatement.json
operations:
  - Patient.read
  - Patient.search-type
  - Condition.search-type
  - Observation.search-type
  - MedicationRequest.search-type
  - AllergyIntolerance.search-type
  - Immunization.search-type
  - DocumentReference.search-type
method: generated
generated: '2026-09-01'
source: >-
  Grounded in fhir/allscripts-healthcare-solutions-veradigm-fhir-r4-capabilitystatement.json (every resource
  and interaction below is declared there) plus https://developer.veradigm.com/Fhir/Introduction and
  https://developer.veradigm.com/Fhir/Searching.
---

# Read a patient's clinical record from Veradigm FHIR

## Before you start

**There is no single Veradigm API host.** Every customer organization has its own FHIR base URL, its own
authorize endpoint and its own token endpoint. Get the base URL from the
[Veradigm Endpoint Directory](https://developer.veradigm.com/Fhir/EndpointDirectory), which is downloadable
as an NDJSON FHIR Bundle of `Organization` resources.

- Endpoints ending **`/fhir`** are for **provider**-facing apps.
- Endpoints ending **`/open`** are for **patient**-facing apps.
- Picking the wrong one is a `404`, not a `403`.

## 1. Discover the environment

```
GET [base]/metadata
Accept: application/fhir+json
```

The `CapabilityStatement` that comes back is the contract for **that** environment. Read two things from it:

- `fhirVersion` — on a "Versionless" endpoint this tells you which release you get by default.
- `rest.security.extension` where `url` is
  `http://fhir-registry.smarthealthit.org/StructureDefinition/oauth-uris` — the `authorize` and `token`
  URIs for this customer.

Pin the FHIR release explicitly on every subsequent request rather than trusting the default:

```
Accept: application/fhir+json; fhirVersion=4.0
```

## 2. Authorize

Run SMART App Launch 2.0.0 authorization code with PKCE (`S256`) against the `authorize` and `token` URIs
you just read. Request the narrowest scopes that do the job — Veradigm's registration guidance is explicit:
*never request scopes the application does not need.*

For a patient-facing app reading a record:

```
launch/patient patient/Patient.rs patient/Condition.rs patient/Observation.rs
patient/MedicationRequest.rs patient/AllergyIntolerance.rs offline_access
```

**Do not mix SMART v1 and SMART v2 scopes on one registered application.** `.read` is v1, `.rs` is v2.
Veradigm will not approve an application that requests both. See
`scopes/allscripts-healthcare-solutions-scopes.yml` for all 237 published scopes.

## 3. Resolve the patient

```
GET [base]/Patient/{id}
```

or search — `Patient` declares 9 search parameters:

```
GET [base]/Patient?_id=INF-101
```

## 4. Pull the clinical resources

Each of these declares `read` and `search-type`:

```
GET [base]/Condition?patient={id}
GET [base]/Observation?patient={id}&category=vital-signs
GET [base]/MedicationRequest?patient={id}
GET [base]/AllergyIntolerance?patient={id}
GET [base]/Immunization?patient={id}
GET [base]/DocumentReference?patient={id}
```

Per-resource search parameters vary from 0 (`Medication`) to 11 (`Location`, `Practitioner`). Read the
`searchParam` list for each resource out of the CapabilityStatement rather than assuming the US Core set.

## 5. Page and filter correctly

- `_count` caps resources per page; follow the Bundle `link` relations for the next page.
- `_lastUpdated` is the incremental-sync parameter: `?_lastUpdated=ge2026-03-01`.
- **A date with no comparison operator means EXACT MATCH.** Use `eq gt ge lt le`; express a range by
  repeating the parameter: `?date=ge2026-01-01&date=le2026-01-31`.
- When a paged search uses `_include`, included resources are **repeated on every page** they relate to.
  De-duplicate by resource id; do not assume each appears once.

## 6. Handle the two behaviours that break naive clients

**Redacted is not null.** A `200` may contain elements marked `"redacted"`. Veradigm returns these when its
Clinical Authorization Service decides the caller may not see the value but that knowing the data *exists*
matters clinically. Surface it as withheld. Never normalise it to empty — the documentation says this
behaviour exists to prevent patient-safety issues.

**Provenance cannot be searched.** `Provenance` supports read-by-id only. To get provenance, hang it off
another resource:

```
GET [base]/AllergyIntolerance?_id=INF-101&_revinclude=Provenance:target
```

## Errors

| Status | Meaning | Do this |
|---|---|---|
| 401 | Token missing or expired | Re-authorize, or refresh if `offline_access` was granted |
| 403 | Scope missing, or the customer has not licensed your app | Check scopes; confirm activation in the customer's License Management Portal |
| 404 | Bad id, or wrong `/fhir` vs `/open` base | Re-check the Endpoint Directory |
| 413 | Request too large | Narrow with `_lastUpdated`, page with `_count`, or use Bulk Data |

Errors arrive as a FHIR `OperationOutcome` (`severity`, `code`, `description`) — **not** RFC 9457
`application/problem+json`.
