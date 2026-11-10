<!-- loiobe0d61ffcc234c37bcc1aa169d8f35e3 -->

# API Reference

AP Guardian exposes a REST API served by a Fastify Node.js application on port 3001. In a BTP deployment, all requests are routed through the approuter, which enforces XSUAA-issued JWT authentication before forwarding to the backend service. The API surface covers invoice queue retrieval, invoice verification, and natural-language queue querying.



<a name="loiobe0d61ffcc234c37bcc1aa169d8f35e3__section_endpoints"/>

## Endpoints

**AP Guardian REST API Endpoints**


<table>
<tr>
<th valign="top">

Method

</th>
<th valign="top">

Path

</th>
<th valign="top">

Description

</th>
</tr>
<tr>
<td valign="top">

GET

</td>
<td valign="top">

`/api/queue`

</td>
<td valign="top">

Returns the list of invoices pending verification, with their current status and metadata.

</td>
</tr>
<tr>
<td valign="top">

POST

</td>
<td valign="top">

`/api/verify`

</td>
<td valign="top">

Runs the full verification pipeline for a given invoice identifier. Returns the structured decision result including individual check outcomes, the recommended action, and a natural-language explanation.

</td>
</tr>
<tr>
<td valign="top">

POST

</td>
<td valign="top">

`/api/query`

</td>
<td valign="top">

Accepts a natural-language question about the invoice queue. The model returns a structured filter expression; the service re-renders the queue view. The model never returns a direct answer — only a filter.

</td>
</tr>
<tr>
<td valign="top">

GET

</td>
<td valign="top">

`/api/invoice/:id`

</td>
<td valign="top">

Returns the full detail record for a single invoice including the audit timeline and evidence drawer content.

</td>
</tr>
</table>



<a name="loiobe0d61ffcc234c37bcc1aa169d8f35e3__section_verify_request"/>

## POST /api/verify — Request and Response

### Request Body

```json
{
  "invoiceId": "INV-2026-0014"
}
```

### Response Body \(PASS Example\)

```json
{
  "invoiceId": "INV-2026-0014",
  "decision": "PASS",
  "recommendedAction": "POST",
  "findings": [],
  "explanation": "Invoice INV-2026-0014 passes all seven checks. The unit price of SAR 3,200 is within the ±10% cap stated in CNT-2026-014 clause 3.3. Tax code V3 is confirmed. Quantity matches the goods receipt.",
  "confidence": {
    "unitPrice": 0.97,
    "currency": 0.99,
    "paymentTerms": 0.95,
    "contractType": 0.98,
    "priceAdjustmentCapPercent": 0.91
  },
  "processingTimeMs": 14920,
  "policyVersion": "1.0.0",
  "modelDeploymentId": "da6e6901fe8fae8c"
}
```

### Response Body \(FAIL Example\)

```json
{
  "invoiceId": "INV-2026-0026",
  "decision": "FAIL",
  "recommendedAction": "BLOCK",
  "findings": [
    {
      "check": "UNIT_PRICE",
      "severity": "FAIL",
      "detail": "Variance of 10.87% exceeds the contract cap of 8% stated in CNT-2026-020 clause 3.3.",
      "invoiceValue": 3481.60,
      "contractValue": 3200.00,
      "capPercent": 8.0
    }
  ],
  "explanation": "Invoice INV-2026-0026 is blocked. The price variance of 10.87% exceeds the maximum ±8% adjustment permitted by contract CNT-2026-020. The invoice text notes 'price adjusted', but this does not override a cap exceedance.",
  "confidence": {
    "unitPrice": 0.96,
    "priceAdjustmentCapPercent": 0.93
  },
  "processingTimeMs": 16340,
  "policyVersion": "1.0.0",
  "modelDeploymentId": "da6e6901fe8fae8c"
}
```



<a name="loiobe0d61ffcc234c37bcc1aa169d8f35e3__section_finding_types"/>

## Finding Types

Each entry in the `findings` array carries a `check` identifier and a `severity` of `FAIL`, `REVIEW`, or `EXTRACTION`.


**Finding Check Identifiers**


<table>
<tr>
<th valign="top">

Check Identifier

</th>
<th valign="top">

Condition

</th>
<th valign="top">

Severity

</th>
</tr>
<tr>
<td valign="top">

`CURRENCY`

</td>
<td valign="top">

Invoice currency does not match contract currency.

</td>
<td valign="top">

FAIL → Block

</td>
</tr>
<tr>
<td valign="top">

`UNIT_PRICE`

</td>
<td valign="top">

Price variance exceeds contract cap, or contract states no cap and a price difference exists, or cap is within range but undocumented.

</td>
<td valign="top">

FAIL → Block; REVIEW → Park

</td>
</tr>
<tr>
<td valign="top">

`QUANTITY`

</td>
<td valign="top">

Invoice quantity differs from goods receipt quantity.

</td>
<td valign="top">

FAIL → Park

</td>
</tr>
<tr>
<td valign="top">

`PAYMENT_TERMS`

</td>
<td valign="top">

Invoice payment term differs from contract term.

</td>
<td valign="top">

FAIL → Park

</td>
</tr>
<tr>
<td valign="top">

`INCOTERM`

</td>
<td valign="top">

Invoice delivery term differs from contract delivery term.

</td>
<td valign="top">

FAIL → Park

</td>
</tr>
<tr>
<td valign="top">

`CONTRACT_VALIDITY`

</td>
<td valign="top">

Invoice date falls outside the contract validity window.

</td>
<td valign="top">

FAIL → Park

</td>
</tr>
<tr>
<td valign="top">

`TAX_CODE`

</td>
<td valign="top">

Invoice tax code differs from the deterministically derived expected code.

</td>
<td valign="top">

FAIL → Block

</td>
</tr>
<tr>
<td valign="top">

`EXTRACTION`

</td>
<td valign="top">

A critical field was extracted below the 0.85 confidence threshold.

</td>
<td valign="top">

Routes to human review; no downstream check run.

</td>
</tr>
</table>



<a name="loiobe0d61ffcc234c37bcc1aa169d8f35e3__section_n8n_integration"/>

## Orchestration Integration

The n8n workflow calls the API as follows:

```
Invoice intake trigger
  → GET /api/queue
  → for each pending invoice → POST /api/verify
      → branch on recommendedAction:
          POST   → MIRO \(post the document in S/4HANA\)
          PARK   → MIR7 \(park the document\)
          BLOCK  → MRBR \(block the document for review\)
```

The service is orchestrator-agnostic. Any HTTP client can drive it. In a production BTP landscape, SAP Build Process Automation is the recommended replacement for n8n, as it integrates natively with S/4HANA workflow and the Fiori inbox.



<a name="loiobe0d61ffcc234c37bcc1aa169d8f35e3__section_related"/>

**Related Information**  


[System Architecture](architecture-5d8a8f9.md)

[Evaluation and Results](evaluation-and-results-c7efc5d.md)

[Security and Privacy](security-and-privacy-f6e56e1.md)
