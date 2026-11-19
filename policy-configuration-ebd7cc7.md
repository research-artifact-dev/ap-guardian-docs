<!-- loioebd7cc76863041da9804b5bf1159c18a -->

# Policy Configuration

All thresholds, rules, and mappings in AP Guardian are defined in a single file: `src/policy/policy.yaml`. Every decision can be traced back to the exact policy version that produced it. A numeric literal anywhere else in `src/` is a bug.

> ### Caution:  
> Contract-specific values — especially price-adjustment caps — are **not** in this file. They are read out of each contract at verification time because they differ per agreement. `policy.yaml` contains only system-wide rules.


<a name="loioebd7cc76863041da9804b5bf1159c18a__section_version"/>

## Version

```yaml
version: "1.0.0"
```

The policy version is recorded on every decision audit event. Any posted invoice can be traced back to the exact rules that produced it.


<a name="loioebd7cc76863041da9804b5bf1159c18a__section_confidence"/>

## Confidence Settings

```yaml
confidence:
  minimum: 0.85
  criticalFields:
    - unitPrice
    - currency
    - paymentTerms
    - contractType
    - priceAdjustmentCapPercent
```

Fields extracted below 0.85 confidence are not compared — an `EXTRACTION` discrepancy is raised instead, routing the invoice to human review. This threshold comes directly from the organizers' Q&A: *"If the AI cannot locate a specific clause with sufficient confidence (threshold: 85%), it should flag the invoice for human review."*


<a name="loioebd7cc76863041da9804b5bf1159c18a__section_tax"/>

## Tax Matrix

```yaml
tax:
  matrix:
    - contractType: MATERIAL
      vendorTaxStatus: LOCAL_VAT_REGISTERED
      taxCode: V3
      meaning: Recoverable input VAT (15%)
    - contractType: SERVICE
      vendorTaxStatus: LOCAL_VAT_REGISTERED
      taxCode: V6
      meaning: Non-recoverable input VAT (15%)
    - contractType: MATERIAL
      vendorTaxStatus: FOREIGN_OR_NON_REGISTERED
      taxCode: V0
      meaning: Zero-rated / reverse charge
    - contractType: SERVICE
      vendorTaxStatus: FOREIGN_OR_NON_REGISTERED
      taxCode: V0
      meaning: Zero-rated / reverse charge
  vendorClassification:
    LOCAL_MATERIAL: LOCAL_VAT_REGISTERED
    LOCAL_SERVICE: LOCAL_VAT_REGISTERED
    LOCAL_NON_RECOVERABLE: LOCAL_VAT_REGISTERED
    FOREIGN: FOREIGN_OR_NON_REGISTERED
    FOREIGN_NON_REGISTERED: FOREIGN_OR_NON_REGISTERED
  serviceMaterialPrefixes:
    - SV-
```


<a name="loioebd7cc76863041da9804b5bf1159c18a__section_comparison"/>

## Comparison Rules

```yaml
comparison:
  currency:
    gatesPrice: true       # Currency mismatch suppresses the price check entirely

  quantity:
    sourceOfTruth: GOODS_RECEIPT
    tolerancePercent: 0.0  # GR quantity must match invoice quantity exactly

  price:
    matchTolerancePercent: 0.0
    noAdjustmentClause: FAIL
    beyondCap: FAIL
    withinCapDocumented: REVIEW
    withinCapUndocumented: FAIL

comparison_extra:
  vatAmount:
    tolerancePercentagePoints: 0.25   # 0.25 pp to account for per-line rounding
    outcome: FAIL
  quantityTolerance:
    outcome: REVIEW
```


<a name="loioebd7cc76863041da9804b5bf1159c18a__section_normalization"/>

## Normalization Rules

Models report the same fact in different words. Normalization is policy, not prompt engineering — a prompt can ask for a code and still get prose, and the comparison engine must never see the difference.

```yaml
normalization:
  paymentTerms:
    codePattern: '(?:^|[^A-Z0-9])(ZB[0-9]{2,3})(?![A-Z0-9])'
    netDaysToCode:
      "30": ZB30
      "45": ZB45
      "60": ZB60
  currency:
    aliases:
      "SAUDI RIYAL": SAR
      "SAUDI RIYALS": SAR
      "SR": SAR
      "US DOLLAR": USD
      "USD $": USD
  incoterm:
    codePattern: '(?:^|[^A-Z])(EXW|FCA|FAS|FOB|CFR|CIF|CPT|CIP|DAP|DPU|DDP|DDU)(?![A-Z])'
  unitOfMeasure:
    aliases:
      "طن متري": MT
      "طن": MT
      "METRIC TON": MT
      "METRIC TONNE": MT
      "TONNE": MT
      "برميل": DRM
      "براميل": DRM
      "DRUM": DRM
      "DRUMS": DRM
      "منصة": PAL
      "منصات": PAL
      "PALLET": PAL
      "PALLETS": PAL
      "رحلة": TRIP
      "رحلات": TRIP
      "TRIP": TRIP
      "TRIPS": TRIP
      "كيلوجرام": KG
      "KILOGRAM": KG
      "قطعة": PC
      "PIECE": PC
      "PIECES": PC
```


<a name="loioebd7cc76863041da9804b5bf1159c18a__section_severity"/>

## Severity Map

```yaml
severity:
  UNIT_PRICE: CRITICAL
  TAX_CODE: CRITICAL
  CURRENCY: CRITICAL
  PAYMENT_TERMS: HIGH
  INCOTERM: HIGH
  QUANTITY: HIGH
  VAT_AMOUNT: CRITICAL
  QUANTITY_TOLERANCE: MEDIUM
  CONTRACT_VALIDITY: MEDIUM
  EXTRACTION: MEDIUM
```


<a name="loioebd7cc76863041da9804b5bf1159c18a__section_decision"/>

## Decision and Action Rules

```yaml
decision:
  failAtOrAbove: HIGH      # CRITICAL or HIGH → FAIL
  reviewAtOrAbove: MEDIUM  # MEDIUM → REVIEW

action:
  blockCategories:
    - UNIT_PRICE
    - TAX_CODE
    - CURRENCY
    - VAT_AMOUNT
  failDefault: PARK
  review: PARK
  pass: POST
```


<a name="loioebd7cc76863041da9804b5bf1159c18a__section_observations"/>

## Observations Settings

```yaml
observations:
  vendorNameMatch: true
  taxArithmeticCheck: true
  expectedVatRatePercent: 15
```

Both observations are enabled but must never influence a decision. They fire on invoices the scenario map expects to PASS, so promoting either to a discrepancy costs false-positive score.

**Related Information**  


[Tax Code Determination](tax-determination-f240e83.md)

[Verification Pipeline](verification-pipeline-e493a6b.md)

[Data Model](data-model-353ac47.md)
