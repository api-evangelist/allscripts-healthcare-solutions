---
name: Register and promote a Veradigm FHIR application
description: >-
  Take a FHIR application from developer signup through registration, scope selection, sandbox testing and
  production access on the Veradigm Connect portal — including the choices that cannot be changed afterwards.
api: fhir/allscripts-healthcare-solutions-veradigm-fhir-r4-capabilitystatement.json
operations:
  - metadata
method: generated
generated: '2026-09-01'
source: >-
  https://developer.veradigm.com/Fhir/ProcessOverview and
  https://developer.veradigm.com/Fhir/FHIR_Sandboxes
---

# Register and promote a Veradigm FHIR application

This is an onboarding flow, not an API call sequence. Veradigm publishes no registration API — every step
below happens in the Veradigm Connect portal.

## 0. Know which program you are in

Veradigm sold TouchWorks EHR, Sunrise and Paragon to Harris/Altera in 2022. If you are integrating with
those, this is the wrong program — Veradigm's own documentation points you at `ADP@alterahealth.com`.
Veradigm Connect covers Veradigm EHR (formerly Allscripts Professional), Veradigm Practice Management and
Practice Fusion.

## 1. Sign up

<https://developer.veradigm.com/Account/RegisterSelf> — accept the User Agreement, provide a valid email.
The **Open** tier is free and gives full documentation plus sandbox access with no upfront fee. Unity API
access and certification require a paid **Integrator** tier (Bronze → Platinum Plus). See
`plans/allscripts-healthcare-solutions-plans-pricing.yml`.

## 2. Register the FHIR application

My Dashboard → My FHIR Applications → **+**. You will supply:

| Field | Notes |
|---|---|
| App Name | Appears in the customer's License Management Portal — make it identify your company AND product |
| **App Type** | `Patient`, `Provider` or `System`. `System` is required for Bulk Data. |
| App Description | Shown to EHR organizations deciding whether to authorize you |
| Additional info link | Usually your marketing site |
| JWKS URL | Your public key endpoint, for backend (System) authentication |
| Redirect URLs | Up to five. Desktop apps use `urn:ietf:wg:oauth:2.0:oob`; web apps use e.g. `https://yourdomain.com/callback` |
| Launch URLs | Up to three, for SMART launch |
| Client Type | Confidential (trusted) or Public (not trusted) |

Saving issues your **Client ID**, **Secret** and **Secret Expiration Date**.

## 3. Select Purpose of Use and scopes

Both are required before the app can be activated for a customer.

- Request only the scopes the application actually needs.
- Scopes must match the App Type — `patient/` scopes belong to Patient apps.
- **Never combine SMART v1 (`.read`) and SMART v2 (`.rs`) scopes on one application.** Veradigm will not
  approve it and it prolongs registration.
- Add `offline_access` only if you genuinely need a refresh token.

## 4. Test in the sandbox

Published base URLs and how to get test credentials are in
`sandbox/allscripts-healthcare-solutions-sandbox.yml`. Set up these environment variables — Veradigm's own
recommended set:

```
FhirURL      # FHIR server base
AuthURL      # from GET [base]/metadata
TokenURL     # from GET [base]/metadata
CallbackURL  # e.g. http://localhost/callback
ClientID
```

Provider test credentials come from a request form; patient test credentials from
`VeradigmConnect@veradigm.com`.

## 5. Request production access — the one-way door

**App Name, App Type and Purpose of Use cannot be changed once production access is granted.** Finalize
them and finish testing before clicking *Request Production Access*.

## 6. Wait for the customer, not for Veradigm

After Veradigm approves the app, **customers activate it themselves** in a separate client License
Management Portal. You cannot license your own application for a customer. If a customer asks how, point
them at the License Management Portal documentation — you will not be able to open it yourself.

## 7. Optional — certification

Certification publishes you on the Veradigm App Expo with a company and product page. It requires a
Security Questionnaire and a FHIR API Assessment sent to `VeradigmConnect@veradigm.com`, and is **not
available on the Open tier**.
