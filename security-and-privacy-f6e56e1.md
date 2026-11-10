<!-- loiof6e56e1a5001456ba2517f511546fa51 -->

# Security and Privacy

AP Guardian is designed with a privacy-first architecture. Every security property described in this document is enforced in code and verified by automated tests. Claims are limited to what the code actually proves — properties of the infrastructure \(for example, where SAP AI Core physically processes data\) are not claimed because they cannot be verified by the application.



<a name="loiof6e56e1a5001456ba2517f511546fa51__section_authentication"/>

## Authentication and Authorization

All HTTP traffic between the browser and the backend service is routed through the SAP BTP Approuter. The approuter integrates with the XSUAA service instance \(`ap-guardian-uaa`\) and enforces OAuth 2.0 JWT authentication before forwarding any request to `ap-guardian-srv`. Unauthenticated requests are rejected at the approuter boundary.

The XSUAA instance is configured with `tenant-mode: dedicated`, providing per-tenant isolation. The `xs-security.json` defines the application scopes and role collections.

In the local development environment, authentication is bypassed. The application reads credentials from the `.env` file and connects directly to SAP AI Core. The `.env` file must never be committed to version control.



<a name="loiof6e56e1a5001456ba2517f511546fa51__section_ai_core_enforcement"/>

## SAP AI Core Endpoint Enforcement

`src/ai/sap-aicore-mistral-service.ts` refuses to construct against any endpoint that is not an SAP AI Core endpoint. This enforcement includes `localhost` and any local model gateway. The check is applied at construction time and is covered by automated tests.

The rationale is that a local model gateway is a development convenience that does not exist in any deployed environment. Code that reaches for it passes every local test and fails silently on deployment — which is the worst failure mode for a financial control system. By enforcing the endpoint at construction, the test suite itself prevents that class of misconfiguration.



<a name="loiof6e56e1a5001456ba2517f511546fa51__section_data_minimization"/>

## Data Minimization

The model receives strictly less information than the screen shows to a human reviewer:

-   Vendor numbers and account identifiers are redacted from every prompt before it is sent to SAP AI Core.
-   Bank keys and IBANs are never included in any prompt.
-   The explanation writer uses only the computed decision result to generate natural language — it does not re-send the raw contract text.

This is enforced in `src/ai/explanation-writer.ts` via a contradiction guard and prompt construction rules.



<a name="loiof6e56e1a5001456ba2517f511546fa51__section_audit_log"/>

## Audit Trail

`src/audit/audit-log.ts` implements an append-only audit trail. There is no `update` method and no `delete` method on the audit log type. Every verification event records:

-   Correlation ID
-   Policy version \(from `src/policy/policy.yaml`\)
-   Model deployment ID
-   Per-field confidence scores
-   Decision outcome and recommended action

No document content — no contract text, no invoice line items — is written to any log line. This means the audit log can be retained and inspected without exposing the underlying financial documents.



<a name="loiof6e56e1a5001456ba2517f511546fa51__section_confidence_gate"/>

## Confidence Gate

Any critical field extracted below the threshold of 0.85 confidence is treated as unread. The system raises an `EXTRACTION` concern and routes the invoice to a human reviewer without running any downstream verification check. This is enforced in the comparison engine before any check is executed.

The critical fields subject to this gate are: `unitPrice`, `currency`, `paymentTerms`, `contractType`, and `priceAdjustmentCapPercent`.

The rationale is that reporting a mismatch derived from a value the system admits it could not read is a false finding. The sharpest case is the price cap — an unread cap is indistinguishable from the contract containing no cap, and treating it as absent would block an invoice the contract actually permits.

> ### Caution:  
> Lowering the confidence threshold in `src/policy/policy.yaml` increases the risk of acting on incorrectly extracted fields. The 0.85 default was validated against the full 30-invoice dataset. Any reduction requires re-running the evaluation scorecard to measure the precision impact.



<a name="loiof6e56e1a5001456ba2517f511546fa51__section_not_claimed"/>

## What Is Not Claimed

-   **Where SAP AI Core processes data.** The application enforces that it sends data only to an SAP AI Core endpoint. The physical data residency and retention properties of the SAP AI Core infrastructure are not properties this code can verify.
-   **End-to-end encryption details.** All outbound calls use HTTPS. The TLS configuration of the SAP AI Core service endpoint is an infrastructure property.
-   **GDPR or ZATCA compliance.** AP Guardian is a prototype; it has not been assessed for regulatory compliance.



<a name="loiof6e56e1a5001456ba2517f511546fa51__section_related"/>

**Related Information**  


[AP Guardian Overview](ap-guardian-overview-017966c.md)

[System Architecture](architecture-5d8a8f9.md)

[Configuration and Deployment](configuration-and-deployment-84479ca.md)

[SAP AI Core Models](https://help.sap.com/docs/AI_CORE/d5d03749687449ad9768e46ea9e32b1e/6440777c196642dbb3171aa5c37b987e.html)
