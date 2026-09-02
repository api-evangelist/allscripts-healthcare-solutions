---
name: Export a population with Veradigm FHIR Bulk Data
description: >-
  Run the asynchronous FHIR Bulk Data ($export) flow against the Veradigm (formerly Allscripts) FHIR R4 API
  as a backend System application, including the polling contract and the one published cancellation path.
api: fhir/allscripts-healthcare-solutions-veradigm-fhir-r4-capabilitystatement.json
operations:
  - Group.read
  - Group.search-type
  - '$export'
method: generated
generated: '2026-09-01'
source: >-
  https://developer.veradigm.com/Fhir/BulkData and
  fhir/allscripts-healthcare-solutions-veradigm-fhir-r4-capabilitystatement.json
---

# Export a population with Veradigm FHIR Bulk Data

## Preconditions you cannot work around

1. The registered FHIR application's **App Type must be `System`**. Patient and User application types
   cannot make bulk data requests at all.
2. **Backend authentication via JWKS must be configured.** You host a JWKS endpoint; Veradigm validates
   your signed client assertion against it. This is what lets you rotate keys without contacting Veradigm.
3. You must already know the **`Group` resource id**. Groups are created by the customer organization from
   segments in the Veradigm EHR Reporting module — you cannot create one through this API.

Authorize with `client_credentials` and `system/` scopes (e.g. `system/*.rs`, or the narrower per-resource
`system/Observation.rs`).

## 1. Kick off

```
GET [base]/Group/{groupId}/$export
Accept: application/fhir+json
```

Optionally narrow with `_type`:

```
GET [base]/Patient/$export?_type=Patient,Observation,Provenance
```

You get **`202 Accepted`, no body**. The status URL is in the **`Content-Location`** response header.
A client that treats non-`200` as failure breaks here.

## 2. Poll — and respect Retry-After

```
GET [Content Location URL]
```

While running:

- `Status: Accepted`
- `X-Progress: <percentage complete>`
- `Retry-After: <seconds>`

**Polling before `Retry-After` elapses returns `429 Too Many Requests`.** This is the only runtime
throttling signal Veradigm publishes anywhere in this API — there are no `X-RateLimit-*` headers and no
published requests-per-second figure.

## 3. Collect the output

On completion:

- `Status: OK`
- `Expires:` — when the files stop being downloadable
- Body: a manifest with `transactionTime`, `request`, `requiresAccessToken`, an `output[]` array of NDJSON
  file URLs by resource `type`, and an **`error[]`** array of `OperationOutcome` NDJSON files.

`error[]` being non-empty does **not** mean the job failed — an export can succeed for some data and not
other data. Always read both arrays.

```
GET [file url]     # NDJSON, one resource per line
```

You may download a package as many times as you need, but once it passes `Expires` it is gone.

## 4. Cancel if you no longer need it

```
DELETE [Content Location URL]
```

This is the **only reversal path published anywhere in this API**. Veradigm states no deadline for it — see
the `reversibility` block in `conventions/allscripts-healthcare-solutions-conventions.yml`. Do not assume a
window.

## Provenance rules

- Name **no** specific resource type → `Provenance` is included by default.
- Name `Provenance` explicitly → every other requested resource also carries provenance.
- Name resources but not `Provenance` → **no** resource carries provenance.

## Exportable resources

25 of the 31 resources Veradigm's CapabilityStatement declares support bulk export. See
`bulk_exportable` per entity in `data-model/allscripts-healthcare-solutions-data-model.yml`.
