<!-- loio77cbf91d803441539444fcd59fc78f39 -->

# Privacy and Security

AP Guardian enforces data privacy and security at the architectural level, not at the policy level. The claims on this page are limited to what the code actually proves — not what the architecture intends.


<a name="loio77cbf91d803441539444fcd59fc78f39__section_endpoint"/>

## AI Core Endpoint Enforcement

The `SAPAICoreMistralService` class refuses to construct against any endpoint that is not SAP AI Core — including `localhost`. This is enforced at construction time, before any request is built, so a misconfigured endpoint cannot receive even one invoice.

The check is applied to both the API URL and the OAuth auth URL. Any hostname that does not match the SAP AI Core pattern throws an error at startup:

```
Error: AI_CORE_API_URL must be an SAP AI Core endpoint.
Got: http://localhost:11434 — local model gateways are not permitted.
```

This design means code that reaches for a local model gateway passes every local test and fails on deployment — the failure is at construction, not at the first real call. Tests assert this behavior explicitly.


<a name="loio77cbf91d803441539444fcd59fc78f39__section_minimisation"/>

## Data Minimisation

The model receives strictly less than the screen does:

-   **Vendor numbers** are redacted from every prompt sent to Mistral.
-   **Account identifiers** (bank keys, IBANs) are never sent.
-   **Prompt content** is limited to the contract text and the JSON schema shape. Invoice amounts and payment details are not included in the extraction prompt.
-   **Explanation prompts** are built from the computed result, not from raw document content. The model is told the decision; it is asked to explain it in words. It is not given the invoice line items.


<a name="loio77cbf91d803441539444fcd59fc78f39__section_audit"/>

## Audit Trail Content

The append-only audit log records:

-   Correlation ID
-   Invoice number
-   Policy version
-   Model deployment identifier
-   Confidence scores
-   Event type and result summary

**What the log never contains:** document text, contract clauses, vendor financial data, or any content that would allow reconstruction of the original documents from the log alone.


<a name="loio77cbf91d803441539444fcd59fc78f39__section_api"/>

## Privacy API Endpoint

The `/api/privacy` endpoint returns the privacy properties as a structured object computed from the actual running system:

```json
{
  "runtimeModelHost": "SAP AI Core",
  "publicLlmCalls": 0,
  "egressPolicy": "enforced at construction — non-AI-Core hosts refuse to start, localhost included",
  "dataMinimisation": "model briefs carry decision fields only; vendor numbers and account identifiers redacted",
  "auditEvents": 420,
  "logsCarryDocumentContent": false,
  "note": "Claims cover this application's behaviour only. Where SAP AI Core processes or retains data is an infrastructure property this code cannot verify."
}
```

This is computed, not asserted. The number of audit events is the actual count from the running log.


<a name="loio77cbf91d803441539444fcd59fc78f39__section_not"/>

## What Is Not Claimed

AP Guardian explicitly does not claim:

-   **Data residency or retention**: where SAP AI Core physically processes or retains data is an infrastructure property this code cannot verify. Claims about cloud infrastructure require SAP AI Core product documentation.
-   **End-to-end encryption**: the HTTPS termination point and the internal BTP network path are infrastructure properties.
-   **ZATCA compliance**: the system determines the correct SAP tax code; it does not implement the full ZATCA e-invoicing specification.


<a name="loio77cbf91d803441539444fcd59fc78f39__section_xsuaa"/>

## XSUAA Authentication

On BTP, the approuter enforces XSUAA authentication before any request reaches the verification service. Two role collections are defined:

-   **AP Guardian Accountant** (Display scope): view invoices and verification results.
-   **AP Guardian Supervisor** (Display + Approve scopes): view invoices and release blocked invoices for payment.

Locally, the API runs without authentication. This is intentional for development; deploying with XSUAA requires the BTP approuter.

**Related Information**  


[SAP AI Core](https://help.sap.com/docs/AI_CORE/2d6c5984063c40a59eda62f4a9135bee/d029a32c22fb45fbb607e6a2c48c8a0e.html)

[Audit Trail](audit-trail-e3b454f.md)

[AI-Based Contract Extraction](ai-contract-extraction-3608239.md)

[Deployment Guide](deployment-guide-5c317ac.md)
