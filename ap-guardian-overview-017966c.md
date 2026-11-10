<!-- loio017966cc63a24240afa39bbef59c1663 -->

# AP Guardian Overview

AP Guardian is an intelligent Accounts Payable prototype that automates invoice verification and tax compliance for Nortech Steel, a synthetic Saudi steel manufacturer. It uses a hybrid AI architecture: Mistral running inside SAP AI Core reads unstructured contract PDFs and reports extracted fields with confidence scores, while deterministic TypeScript performs every comparison, arithmetic operation, and posting decision.

AP Guardian was developed as an SAP hackathon prototype targeting the "Intelligent Accounts Payable: Automating Invoice Verification & Tax Compliance" challenge. It addresses the core pain point of manual contract-clause lookup — 15–20 minutes per invoice — for which standard SAP has no out-of-the-box solution.



<a name="loio017966cc63a24240afa39bbef59c1663__section_problem"/>

## The Problem AP Guardian Solves

An AP accountant at Nortech Steel opens the contract PDF for every inbound invoice, locates the price, payment-term, and Incoterm clauses, and cross-checks them against the invoice, purchase order \(PO\), and goods receipt \(GR\) by eye. This process affects only local material invoices that reference a contract \(invoice type GR60\), which is the highest-volume pain point. Standard SAP has no contract-clause extraction capability, so there is no off-the-shelf solution.

AP Guardian reduces that 15–20 minute manual task to a fully automated pipeline with a median processing time of 14.9 seconds per invoice.



<a name="loio017966cc63a24240afa39bbef59c1663__section_design"/>

## The Central Design Idea

> ### Note:  
> The fundamental architectural principle of AP Guardian is expressed in one sentence: **Probabilistic AI reads the contract. Deterministic code decides the invoice.**

There is a strict *truth boundary* in the architecture. Above it, Mistral reads documents and reports what it found with page evidence and a confidence score. Below it, pure TypeScript produces every number, comparison, and decision. Facts cross downward; decision authority never crosses upward. This means the model cannot hallucinate a tax rate that survives into a financial posting.

The rules AP Guardian enforces are not hardcoded — they are written into each contract in prose. Three contracts illustrate this:

-   `CNT-2026-014` permits a maximum price adjustment of ±10% per month.
-   `CNT-2026-020` permits ±8% per month.
-   `CNT-2026-013` contains no price-adjustment clause at all.

A hardcoded tolerance is wrong for all three simultaneously. The model reads the contract-specific cap; the deterministic engine applies it.



<a name="loio017966cc63a24240afa39bbef59c1663__section_scope"/>

## Scope and Invoice Type

AP Guardian covers **local material invoices \(GR60\) that reference a contract**. This is one of Nortech's eight invoice types, chosen because it is the highest-volume category and the pattern replicates to the remaining seven types.

The system is not a payment system, an S/4HANA clone, or a ZATCA tax engine. It recommends a posting action — Post, Park, or Block — but does not execute write-back to SAP S/4HANA.



<a name="loio017966cc63a24240afa39bbef59c1663__section_seven_checks"/>

## Seven Verification Checks

AP Guardian runs seven checks in a fixed order for every invoice. The order is architecturally significant: currency gating must suppress the price check before it runs, and the contract-side price gates must fire before the invoice-text adjustment check.


**AP Guardian Verification Checks**


<table>
<tr>
<th valign="top">

Check

</th>
<th valign="top">

Source of Truth

</th>
<th valign="top">

Outcome on Failure

</th>
</tr>
<tr>
<td valign="top">

1. Currency

</td>
<td valign="top">

Contract currency vs invoice currency

</td>
<td valign="top">

Block — price check suppressed

</td>
</tr>
<tr>
<td valign="top">

2. Unit Price

</td>
<td valign="top">

Contract price cap \(four sequential gates\)

</td>
<td valign="top">

Block \(FAIL\) or Park \(REVIEW\)

</td>
</tr>
<tr>
<td valign="top">

3. Quantity

</td>
<td valign="top">

Goods receipt quantity — not the PO

</td>
<td valign="top">

Park

</td>
</tr>
<tr>
<td valign="top">

4. Payment Terms

</td>
<td valign="top">

Contract payment terms \(ZB30/ZB45/ZB60\)

</td>
<td valign="top">

Park

</td>
</tr>
<tr>
<td valign="top">

5. Incoterm

</td>
<td valign="top">

Contract delivery terms

</td>
<td valign="top">

Park

</td>
</tr>
<tr>
<td valign="top">

6. Contract Validity

</td>
<td valign="top">

Invoice date within contract validity window

</td>
<td valign="top">

Park

</td>
</tr>
<tr>
<td valign="top">

7. Tax Code

</td>
<td valign="top">

Derived from contract type × vendor tax status

</td>
<td valign="top">

Block

</td>
</tr>
</table>

The meta-rule governing all checks: **money faults block; paperwork faults park.** Price, tax code, and currency mismatches stop the payment. Payment terms, Incoterm, and validity issues hold the document for review.



<a name="loio017966cc63a24240afa39bbef59c1663__section_related"/>

**Related Information**  


[System Architecture](architecture-5d8a8f9.md)

[Security and Privacy](security-and-privacy-f6e56e1.md)

[AI-Assisted Goods Receipt Analysis](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/e3dc5400c1cc41d1bc0ae0e7fd9aa5a2/30d87325d6964ec18da12bd2fe927009.html)

[Contract Creation from a Document](https://help.sap.com/docs/SAP_ARIBA_CONTRACTS/0dbb7f38db0c44049973d3819a6f67f9/4035a4f136764e3899d26b7e0aeef980.html "Use AI to automatically extract structured and unstructured data from contract documents.") :arrow_upper_right:
