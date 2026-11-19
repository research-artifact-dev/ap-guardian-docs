<!-- loioe3b454fd878c42ac80d9fefab8715e97 -->

# Audit Trail

AP Guardian maintains an append-only audit trail that records every significant event in the invoice verification lifecycle. The trail answers the question "why was this invoice posted?" two years after the fact, without exposing any document content.


<a name="loioe3b454fd878c42ac80d9fefab8715e97__section_design"/>

## Design Principles

The `AuditLog` class (`src/audit/audit-log.ts`) enforces three properties:

-   **Append-only**: There is no `update` method and no `delete` method. An audit record that can be corrected is not an audit record.
-   **Content-free**: Records carry a correlation ID, the policy version, and the model deployment identifier. They never carry document content, vendor financial data, or contract text.
-   **Reconstructable**: The decision described by any audit event is reconstructable from the IDs and the policy version alone. The confidential body of the document stays out of the log entirely.


<a name="loioe3b454fd878c42ac80d9fefab8715e97__section_events"/>

## Audit Event Types

<table>
<tr>
<th valign="top">Event Type</th>
<th valign="top">When raised</th>
</tr>
<tr>
<td valign="top"><code>INVOICE_RECEIVED</code></td>
<td valign="top">The invoice enters the pipeline.</td>
</tr>
<tr>
<td valign="top"><code>REFERENCE_DATA_LOADED</code></td>
<td valign="top">PO, GR, vendor, and contract attachment resolved from the data source.</td>
</tr>
<tr>
<td valign="top"><code>EXTRACTION_STARTED</code></td>
<td valign="top">Mistral is called to read the contract.</td>
</tr>
<tr>
<td valign="top"><code>EXTRACTION_COMPLETED</code></td>
<td valign="top">Mistral returned a valid extraction.</td>
</tr>
<tr>
<td valign="top"><code>EXTRACTION_VALIDATED</code></td>
<td valign="top">The extracted JSON passed schema validation and evidence grounding.</td>
</tr>
<tr>
<td valign="top"><code>EXTRACTION_REJECTED</code></td>
<td valign="top">The extraction was rejected after two attempts. The invoice routes to REVIEW.</td>
</tr>
<tr>
<td valign="top"><code>TAX_CODE_CALCULATED</code></td>
<td valign="top">The expected tax code was determined by the tax engine.</td>
</tr>
<tr>
<td valign="top"><code>DISCREPANCY_CREATED</code></td>
<td valign="top">A comparison check found a discrepancy.</td>
</tr>
<tr>
<td valign="top"><code>DECISION_CALCULATED</code></td>
<td valign="top">The final PASS/REVIEW/FAIL decision and POST/PARK/BLOCK action were computed.</td>
</tr>
<tr>
<td valign="top"><code>EXPLANATION_GENERATED</code></td>
<td valign="top">Mistral wrote the natural-language explanation for the decision.</td>
</tr>
<tr>
<td valign="top"><code>EXPLANATION_CONTRADICTED</code></td>
<td valign="top">The explanation contradicted the decision. The explanation is replaced with a template response.</td>
</tr>
<tr>
<td valign="top"><code>HUMAN_REVIEWED</code></td>
<td valign="top">A human approved or rejected a held invoice via the dashboard.</td>
</tr>
</table>


<a name="loioe3b454fd878c42ac80d9fefab8715e97__section_structure"/>

## Event Structure

Each `AuditEvent` carries:

```typescript
{
  timestamp: string;        // ISO 8601
  correlationId: string;    // UUID linking all events for one invoice
  invoiceNumber: string;
  eventType: AuditEventType;
  actor: string;            // "system", "mistral", or the user who approved
  policyVersion: string | null;
  modelDeployment: string | null;  // "aicore:da6e6901fe8fae8c"
  confidence: number | null;
  result: string;           // Human-readable summary of what happened
}
```


<a name="loioe3b454fd878c42ac80d9fefab8715e97__section_sinks"/>

## Audit Sinks

The `AuditLog` supports pluggable sinks via the `AuditSink` interface. The default in-memory log serves the dashboard's audit timeline. The `JsonlAuditSink` writes one JSON object per line to a file in append mode — suitable for production log shipping to SAP Cloud Logging or a SIEM.

```typescript
interface AuditSink {
  write(event: AuditEvent): void;
}
```

Multiple sinks can be registered simultaneously. All sinks receive every event synchronously in the order they were registered.


<a name="loioe3b454fd878c42ac80d9fefab8715e97__section_query"/>

## Querying the Audit Trail

The `AuditLog.forInvoice(invoiceNumber)` method returns all events for one invoice in chronological order. This is what the dashboard's audit timeline panel renders — the complete lifecycle of a document from intake to decision.

**Related Information**  


[Privacy and Security](privacy-and-security-77cbf91.md)

[REST API Reference](api-reference-c4925c5.md)

[Verification Pipeline](verification-pipeline-e493a6b.md)
