<!-- loioc4e78b2ab7474cdba2f247162311cd84 -->

# Evaluation Framework

AP Guardian includes a built-in evaluation harness that scores the system against six weighted criteria using a swappable verifier. The harness runs offline (no network) with the fixture provider or live against SAP AI Core with the Mistral provider.


<a name="loioc4e78b2ab7474cdba2f247162311cd84__section_running"/>

## Running the Evaluation

```sh
npm run eval               # pipeline verifier (default, ~2s, offline)  → 97.9%
npm run eval naive         # regex baseline, no model (~2s, offline)    → 71.1%
npm run eval mistral       # live Mistral through SAP AI Core (~8 min)  → 97.9%
```

The pipeline verifier uses pre-read contract policy from the fixture file, isolating the decision logic from extraction quality. The mistral verifier reads every contract live through SAP AI Core — the gap between the two scores is the measured extraction accuracy.


<a name="loioc4e78b2ab7474cdba2f247162311cd84__section_criteria"/>

## Six Scoring Criteria

<table>
<tr>
<th valign="top">ID</th>
<th valign="top">Criterion</th>
<th valign="top">Weight</th>
<th valign="top">What is measured</th>
<th valign="top">Score (Mistral)</th>
</tr>
<tr>
<td valign="top">extraction</td>
<td valign="top">Extraction accuracy</td>
<td valign="top">30%</td>
<td valign="top">Seven clause fields (unitPrice, paymentTerms, currency, incoterm, unitOfMeasure, contractType, validTo) extracted with a non-null value AND a non-null evidence quote.</td>
<td valign="top">98.1%</td>
</tr>
<tr>
<td valign="top">discrepancy</td>
<td valign="top">Discrepancy detection</td>
<td valign="top">25%</td>
<td valign="top">Every mismatch in the ground truth found. 16 expected findings across 30 cases. False negatives are the fatal error.</td>
<td valign="top">93.8% \(15/16\)</td>
</tr>
<tr>
<td valign="top">tax</td>
<td valign="top">Tax determination</td>
<td valign="top">20%</td>
<td valign="top">Correct V3/V6/V0 determination for all 30 invoices.</td>
<td valign="top">100% \(30/30\)</td>
</tr>
<tr>
<td valign="top">latency</td>
<td valign="top">Processing time</td>
<td valign="top">10%</td>
<td valign="top">p95 at or under 35s target \(full marks\); zero at 60s limit. Linear between. Measured over all 30 cases.</td>
<td valign="top">100% \(p50 14.9s\)</td>
</tr>
<tr>
<td valign="top">explainability</td>
<td valign="top">Explainability and audit</td>
<td valign="top">10%</td>
<td valign="top">Every discrepancy has: expected value, rule ID, explanation, and — for CONTRACT source — a clause reference and page number.</td>
<td valign="top">100%</td>
</tr>
<tr>
<td valign="top">falsePositive</td>
<td valign="top">False positive rate</td>
<td valign="top">5%</td>
<td valign="top">Clean invoices \(0 expected discrepancies\) incorrectly flagged. Full marks at ≤5% rate; zero at 25%. 0 of 15 clean invoices flagged.</td>
<td valign="top">100%</td>
</tr>
</table>


<a name="loioc4e78b2ab7474cdba2f247162311cd84__section_results"/>

## Benchmark Results

**Weighted total by verifier:**

<table>
<tr>
<th valign="top">Verifier</th>
<th valign="top">Weighted score</th>
<th valign="top">Decision accuracy</th>
<th valign="top">p50 latency</th>
</tr>
<tr>
<td valign="top"><code>mistralai--mistral-medium 3.5</code></td>
<td valign="top">97.9%</td>
<td valign="top">100%</td>
<td valign="top">14.9s</td>
</tr>
<tr>
<td valign="top"><code>gpt-4o</code></td>
<td valign="top">97.9%</td>
<td valign="top">100%</td>
<td valign="top">4.0s</td>
</tr>
<tr>
<td valign="top"><code>anthropic--claude-4.5-sonnet</code></td>
<td valign="top">97.9%</td>
<td valign="top">100%</td>
<td valign="top">9.4s</td>
</tr>
<tr>
<td valign="top">naive regex, no model</td>
<td valign="top">71.1%</td>
<td valign="top">46.7%</td>
<td valign="top">4ms</td>
</tr>
</table>

All three frontier models are identical in accuracy. Extraction quality is not the differentiator — the deterministic layer carries the score. This means choosing Mistral for privacy costs nothing in accuracy.


<a name="loioc4e78b2ab7474cdba2f247162311cd84__section_baseline"/>

## What the Baseline Reveals

The naive regex baseline scores tax determination 100% correctly and finds most discrepancies — then flags **every one of the 15 clean invoices**. A pattern cannot distinguish *"this contract has no price-adjustment clause"* from *"this contract words the clause differently"*, so it must escalate all 30. That is why 87.5% detection is still unusable.

The decisive example from the dataset: `CNT-2026-016`'s pricing clause reads *"Annual escalation of 3% applicable from Year 2 onwards. Not applicable to current contract year."* The regex reads the cap as **3**. Mistral and Claude both correctly return `null`. One sentence, and it is the entire difference between pattern matching and reading.


<a name="loioc4e78b2ab7474cdba2f247162311cd84__section_generalisation"/>

## Generalisation Evidence

The evaluation includes 16 synthetic test cases (in `tests/generalisation.test.ts`) that appear nowhere in the dataset and whose outcomes are written nowhere. The decisive test:

> **A 9.5% variance is REVIEW under CNT-014's 10% cap and FAIL under CNT-020's 8% cap.** Same number, opposite outcomes, because the cap is read from the contract. No single fitted threshold produces both.

Two additional tests assert the policy file contains no invoice number, no contract ID, no test-case reference, and no price cap of its own — confirming the system does not memorize the dataset.


<a name="loioc4e78b2ab7474cdba2f247162311cd84__section_ground"/>

## Ground Truth

The ground truth file (`src/eval/ground-truth.ts`) contains 30 cases with the expected decision (PASS/REVIEW/FAIL), action (POST/PARK/BLOCK), and the set of expected discrepancy categories. Distribution: **15 PASS, 2 REVIEW, 13 FAIL**. The 16 discriminating findings: 5 price, 4 payment terms, 4 tax code, 2 currency, 1 Incoterm.

> ### Note:  
> Quantity has zero examples in the standard 30-case dataset. V0 (foreign/non-registered vendor) has zero examples — there are no foreign vendors in the Nortech dataset. Arabic has zero standard cases; the Arabic robustness is demonstrated separately via `npm run probe:arabic`.

**Related Information**  


[AP Guardian Overview](ap-guardian-overview-ef47983.md)

[AI-Based Contract Extraction](ai-contract-extraction-3608239.md)

[Verification Pipeline](verification-pipeline-e493a6b.md)
