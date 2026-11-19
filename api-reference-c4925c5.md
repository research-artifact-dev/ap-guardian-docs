<!-- loioc4925c540e904b89b3c3a4494b6285a1 -->

# REST API Reference

AP Guardian exposes a Fastify HTTP API on port 3001 (configurable via the `PORT` environment variable). The API serves invoice verification results, audit events, contract text, and the built Fiori UI. All routes except the UI static files are prefixed with `/api/`.


<a name="loioc4925c540e904b89b3c3a4494b6285a1__section_endpoints"/>

## Endpoints

**Summary of all endpoints:**

<table>
<tr>
<th valign="top">Method</th>
<th valign="top">Path</th>
<th valign="top">Description</th>
</tr>
<tr>
<td valign="top">GET</td>
<td valign="top"><code>/api/health</code></td>
<td valign="top">Service health check and configuration summary.</td>
</tr>
<tr>
<td valign="top">GET</td>
<td valign="top"><code>/api/summary</code></td>
<td valign="top">Dashboard summary: all invoices, discrepancy categories, tax mismatches, total financial exposure.</td>
</tr>
<tr>
<td valign="top">GET</td>
<td valign="top"><code>/api/invoices</code></td>
<td valign="top">List of all invoices with verification summary (sorted by invoice number).</td>
</tr>
<tr>
<td valign="top">GET</td>
<td valign="top"><code>/api/invoices/:id</code></td>
<td valign="top">Full invoice detail including discrepancies, audit timeline, and contract text.</td>
</tr>
<tr>
<td valign="top">GET</td>
<td valign="top"><code>/api/queue</code></td>
<td valign="top">Invoice numbers awaiting verification (the orchestrator's work list).</td>
</tr>
<tr>
<td valign="top">POST</td>
<td valign="top"><code>/api/verify</code></td>
<td valign="top">Verify one invoice on demand. Returns decision, action, and discrepancies.</td>
</tr>
<tr>
<td valign="top">POST</td>
<td valign="top"><code>/api/query</code></td>
<td valign="top">Natural-language query over the invoice queue.</td>
</tr>
<tr>
<td valign="top">POST</td>
<td valign="top"><code>/api/invoices/:id/reverify</code></td>
<td valign="top">Re-verify one invoice live through SAP AI Core (no cache).</td>
</tr>
<tr>
<td valign="top">GET</td>
<td valign="top"><code>/api/contracts/:id/text</code></td>
<td valign="top">Page-addressed contract text. Accepts <code>?lang=ar</code> for the Arabic version.</td>
</tr>
<tr>
<td valign="top">POST</td>
<td valign="top"><code>/api/translate-clause</code></td>
<td valign="top">Translate an Arabic clause to English for display (presentation only, no effect on decisions).</td>
</tr>
<tr>
<td valign="top">GET</td>
<td valign="top"><code>/api/privacy</code></td>
<td valign="top">Privacy statement computed from actual architecture properties.</td>
</tr>
</table>


<a name="loioc4925c540e904b89b3c3a4494b6285a1__section_verify"/>

## POST /api/verify

Verifies one invoice on demand. This is the entry point for orchestrators (n8n, SAP Build Process Automation, or any HTTP client).

**Request body:**

```json
{ "invoiceNumber": "INV-2026-0001" }
```

**Response (200):**

```json
{
  "invoiceNumber": "INV-2026-0001",
  "correlationId": "uuid",
  "decision": "PASS",
  "recommendedAction": "POST",
  "shouldPost": true,
  "shouldPark": false,
  "shouldBlock": false,
  "vendorName": "Al Rajhi Steel",
  "amount": "120000.00",
  "currency": "SAR",
  "poNumber": "PO-2026-001",
  "contractId": "CNT-2026-014",
  "matchScore": 1.0,
  "fieldsMatched": 6,
  "fieldsCompared": 6,
  "expectedTaxCode": "V3",
  "invoiceTaxCode": "V3",
  "discrepancyCount": 0,
  "discrepancies": [],
  "financialExposure": "0.00",
  "explanation": "All fields matched...",
  "policyVersion": "1.0.0",
  "modelDeployment": "aicore:da6e6901fe8fae8c",
  "durationMs": 14900
}
```

The flat `shouldPost` / `shouldPark` / `shouldBlock` booleans allow a workflow to branch without parsing nested structures.

**Error responses:**

<table>
<tr>
<th valign="top">Status</th>
<th valign="top">Condition</th>
</tr>
<tr>
<td valign="top">400</td>
<td valign="top"><code>invoiceNumber</code> is missing or empty.</td>
</tr>
<tr>
<td valign="top">404</td>
<td valign="top">Invoice number not found in the loaded dataset.</td>
</tr>
</table>


<a name="loioc4925c540e904b89b3c3a4494b6285a1__section_query"/>

## POST /api/query

Natural-language query over the invoice queue. The model returns a **filter**, not an answer — the count and the exposure are computed from the same verified results the dashboard uses. A question cannot make the system assert anything the deterministic layer did not establish.

**Request body:**

```json
{ "question": "Show me all invoices with tax code discrepancies" }
```

**Response (200):**

```json
{
  "question": "Show me all invoices with tax code discrepancies",
  "intent": { "categories": ["TAX_CODE"] },
  "parser": "mistral",
  "count": 4,
  "exposure": "45000.00",
  "invoices": [...]
}
```

**Error responses:**

<table>
<tr>
<th valign="top">Status</th>
<th valign="top">Condition</th>
</tr>
<tr>
<td valign="top">400</td>
<td valign="top"><code>question</code> is missing or empty.</td>
</tr>
<tr>
<td valign="top">422</td>
<td valign="top">The model could not produce a valid filter. A model that cannot parse the question must not silently return all invoices — that would look like an answer.</td>
</tr>
</table>


<a name="loioc4925c540e904b89b3c3a4494b6285a1__section_reverify"/>

## POST /api/invoices/:id/reverify

Re-verifies one invoice live through SAP AI Core, with no cache. The response reports how long the model actually took and which deployment answered — "live" is a measured number on the screen, not a claim in a slide.

**Response includes:**

-   `unchanged`: whether the decision and discrepancy count are identical to the cached result. An identical answer is the interesting result — it means the reading is reproducible.
-   `lowestConfidence`: the weakest field confidence across all extracted fields.
-   `elapsedMs`: actual wall-clock time for this re-verification.

Requires `AP_PROVIDER=mistral`. Returns 503 if the server is running in fixture mode.


<a name="loioc4925c540e904b89b3c3a4494b6285a1__section_contracts"/>

## GET /api/contracts/:id/text

Returns page-addressed contract text. Each page is a `{ page: number, text: string }` object. The `source` field records whether the text came from a PDF text layer, OCR, or SAP Document Information Extraction — an OCR-derived decision and one read from an embedded text layer carry different evidential weight, and a reviewer is entitled to know which.

Pass `?lang=ar` to request the Arabic version of the contract (available for 10 of the 25 contracts). The Arabic text was read from the original PDFs by the same extraction pipeline — the same clauses, the same confidence scores.


<a name="loioc4925c540e904b89b3c3a4494b6285a1__section_privacy"/>

## GET /api/privacy

Returns the privacy statement as a structured object computed from the actual architecture properties, not asserted from a slide:

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

**Related Information**  


[Orchestration Workflows](orchestration-workflows-668d71a.md)

[Audit Trail](audit-trail-e3b454f.md)

[Privacy and Security](privacy-and-security-77cbf91.md)
