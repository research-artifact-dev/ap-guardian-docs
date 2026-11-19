<!-- loio353ac47f829d431195913e7a01019a60 -->

# Data Model

AP Guardian operates on six core domain types derived from the S/4HANA extract, plus eleven enumeration types, and stores master and transactional data in five SAP HANA Cloud tables.


<a name="loio353ac47f829d431195913e7a01019a60__section_enums"/>

## Enumerations

<table>
<tr>
<th valign="top">Type</th>
<th valign="top">Values</th>
<th valign="top">Description</th>
</tr>
<tr>
<td valign="top"><code>TaxCode</code></td>
<td valign="top"><code>V3</code> | <code>V6</code> | <code>V0</code> | <code>UNKNOWN</code></td>
<td valign="top">SAP tax codes in scope. UNKNOWN when evidence is missing.</td>
</tr>
<tr>
<td valign="top"><code>ContractType</code></td>
<td valign="top"><code>MATERIAL</code> | <code>SERVICE</code> | <code>UNKNOWN</code></td>
<td valign="top">What the contract supplies. Drives tax code determination.</td>
</tr>
<tr>
<td valign="top"><code>VendorTaxStatus</code></td>
<td valign="top"><code>LOCAL_VAT_REGISTERED</code> | <code>FOREIGN_OR_NON_REGISTERED</code> | <code>UNKNOWN</code></td>
<td valign="top">Canonical vendor tax standing, resolved from vendor master classification via policy.</td>
</tr>
<tr>
<td valign="top"><code>Decision</code></td>
<td valign="top"><code>PASS</code> | <code>REVIEW</code> | <code>FAIL</code></td>
<td valign="top">Deterministic verification outcome.</td>
</tr>
<tr>
<td valign="top"><code>RecommendedAction</code></td>
<td valign="top"><code>POST</code> | <code>PARK</code> | <code>BLOCK</code></td>
<td valign="top">What AP should do with the document in S/4HANA.</td>
</tr>
<tr>
<td valign="top"><code>DiscrepancyCategory</code></td>
<td valign="top"><code>UNIT_PRICE</code> | <code>QUANTITY</code> | <code>CURRENCY</code> | <code>PAYMENT_TERMS</code> | <code>INCOTERM</code> | <code>TAX_CODE</code> | <code>VAT_AMOUNT</code> | <code>QUANTITY_TOLERANCE</code> | <code>CONTRACT_VALIDITY</code> | <code>EXTRACTION</code></td>
<td valign="top">Which comparison produced a discrepancy.</td>
</tr>
<tr>
<td valign="top"><code>Severity</code></td>
<td valign="top"><code>INFO</code> | <code>LOW</code> | <code>MEDIUM</code> | <code>HIGH</code> | <code>CRITICAL</code></td>
<td valign="top">How much a discrepancy matters. Maps to Decision via policy.</td>
</tr>
<tr>
<td valign="top"><code>SourceOfTruth</code></td>
<td valign="top"><code>CONTRACT</code> | <code>PURCHASE_ORDER</code> | <code>GOODS_RECEIPT</code> | <code>VENDOR_MASTER</code> | <code>POLICY</code></td>
<td valign="top">Where an expected value came from. Shown to auditors for every field.</td>
</tr>
</table>


<a name="loio353ac47f829d431195913e7a01019a60__section_reference"/>

## Reference Data Types

**`Vendor`** — from vendor master:

-   `vendorNumber`, `name`, `country`, `taxClassification`, `vatRecoveryStatus`, `paymentTerms`, `currency`

**`PurchaseOrder`** — from PO data:

-   `poNumber`, `poItem`, `vendorNumber`, `materialNumber`, `materialDescription`, `quantity` (Decimal), `unitOfMeasure`, `netPrice` (Decimal), `currency`, `paymentTerms`, `incoterm`, `contractReference`, `taxCode`

**`GoodsReceipt`** — from GR data:

-   `grDocument`, `grDate`, `poNumber`, `poItem`, `materialNumber`, `quantity` (Decimal), `unitOfMeasure`, `vendorNumber`

**`Invoice`** — from invoice data:

-   `invoiceNumber`, `invoiceDate`, `vendorNumber`, `vendorName`, `amount` (Decimal), `currency`, `taxAmount` (Decimal), `taxCode`, `paymentTerms`, `poNumber`, `poItem`, `grDocument`, `quantity` (Decimal), `unitOfMeasure`, `description`

The `description` field (free-text from vendor) is load-bearing: it is the only field that separates a documented price adjustment from an undocumented one.

**`ContractAttachment`** — from Generic Object Services (GOS):

-   `poNumber`, `contractId`, `fileName`, `declaredLanguage`

The `declaredLanguage` column in GOS is unreliable — three contracts labeled "Arabic" contain no Arabic characters. Language is always detected from the actual text content.

**`InvoiceCase`** — the resolved bundle for one invoice:

-   `invoice`, `purchaseOrder`, `goodsReceipt` (nullable), `vendor`, `attachment` (nullable)


<a name="loio353ac47f829d431195913e7a01019a60__section_verification"/>

## Verification Output Types

**`ExtractedField<T>`** — a value the AI read out of a document:

-   `value: T | null`, `confidence: number` (0–1), `evidenceText: string | null`, `page: number | null`, `clauseRef: string | null`, `source: SourceOfTruth`

**`Discrepancy`** — one verified finding:

-   `id` (D00001–D99999), `category`, `severity`, `field`, `invoiceValue`, `expectedValue`, `sourceOfTruth`, `variancePercent` (Decimal | null), `financialImpact` (Decimal | null), `evidenceText`, `page`, `clauseRef`, `ruleId`, `explanation`

**`VerificationResult`** — the complete output for one invoice:

-   `invoiceNumber`, `correlationId`, `decision`, `recommendedAction`, `matchScore` (0–1), `fieldsCompared`, `fieldsMatched`, `discrepancies`, `observations`, `tax` (TaxDetermination), `extraction` (ContractExtraction | null), `policyVersion`, `explanation`, `durationMs`

> ### Note:  
> Money is never a float. All monetary amounts, prices, and percentages use the `Decimal` type (`src/domain/decimal.ts`), which performs exact string-based arithmetic. A numeric literal used in a price comparison is a bug.


<a name="loio353ac47f829d431195913e7a01019a60__section_hana"/>

## SAP HANA Cloud Tables

Five HDI tables store the Nortech master and transactional data:

<table>
<tr>
<th valign="top">Table</th>
<th valign="top">Rows</th>
<th valign="top">Key columns</th>
</tr>
<tr>
<td valign="top"><code>APG_VENDOR</code></td>
<td valign="top">18</td>
<td valign="top">VENDOR_NUMBER, NAME, COUNTRY, TAX_CLASSIFICATION, VAT_RECOVERY_STATUS, PAYMENT_TERMS, CURRENCY</td>
</tr>
<tr>
<td valign="top"><code>APG_PURCHASE_ORDER</code></td>
<td valign="top">25</td>
<td valign="top">PO_NUMBER, PO_ITEM, VENDOR_NUMBER, MATERIAL_NUMBER, QUANTITY, NET_PRICE, CURRENCY, PAYMENT_TERMS, INCOTERM, CONTRACT_REFERENCE, TAX_CODE</td>
</tr>
<tr>
<td valign="top"><code>APG_INVOICE</code></td>
<td valign="top">30</td>
<td valign="top">INVOICE_NUMBER, INVOICE_DATE, VENDOR_NUMBER, AMOUNT, CURRENCY, TAX_AMOUNT, TAX_CODE, PAYMENT_TERMS, PO_NUMBER, QUANTITY, DESCRIPTION</td>
</tr>
<tr>
<td valign="top"><code>APG_GOODS_RECEIPT</code></td>
<td valign="top">30</td>
<td valign="top">GR_DOCUMENT, GR_DATE, PO_NUMBER, PO_ITEM, MATERIAL_NUMBER, QUANTITY, UNIT_OF_MEASURE, VENDOR_NUMBER</td>
</tr>
<tr>
<td valign="top"><code>APG_CONTRACT_ATTACHMENT</code></td>
<td valign="top">25</td>
<td valign="top">PO_NUMBER, CONTRACT_ID, FILE_NAME, DECLARED_LANGUAGE</td>
</tr>
</table>

**Related Information**  


[Verification Pipeline](verification-pipeline-e493a6b.md)

[Policy Configuration](policy-configuration-ebd7cc7.md)

[Deployment Guide](deployment-guide-5c317ac.md)
