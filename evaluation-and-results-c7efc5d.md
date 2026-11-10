<!-- loioc7efc5d9875c459e937fde743f4252de -->

# Evaluation and Results

AP Guardian includes a built-in evaluation harness that scores the full pipeline against the six weighted criteria from the hackathon organizers' Q&A document. The harness is verifier-agnostic: the same 30-invoice scenario dataset and the same six criteria are scored regardless of which extraction provider is active. This makes it possible to measure the contribution of AI extraction independently of the deterministic decision logic.



<a name="loioc7efc5d9875c459e937fde743f4252de__section_running_eval"/>

## Running the Evaluation

```sh
npm run eval              # pipeline verifier \(default\), cached, ~2s   → 97.9%
npm run eval naive        # regex baseline, no AI, ~2s                  → 71.1%
npm run eval mistral      # live Mistral via SAP AI Core, ~8 min
```

The three verifiers use the same harness and scoring logic. Swapping the verifier is the only argument difference.


**Verifier Options**


<table>
<tr>
<th valign="top">

Verifier

</th>
<th valign="top">

What It Measures

</th>
<th valign="top">

Network Required

</th>
</tr>
<tr>
<td valign="top">

`pipeline`

</td>
<td valign="top">

The deterministic core against pre-read contract policy. Isolates decision logic from extraction quality. This is the ceiling the system can reach given perfect extraction.

</td>
<td valign="top">

No

</td>
</tr>
<tr>
<td valign="top">

`mistral`

</td>
<td valign="top">

Live Mistral extraction through SAP AI Core. The gap between `pipeline` and `mistral` is the measured extraction accuracy.

</td>
<td valign="top">

Yes \(SAP AI Core\)

</td>
</tr>
<tr>
<td valign="top">

`naive`

</td>
<td valign="top">

The regex baseline with no AI model. Establishes what is achievable without language understanding.

</td>
<td valign="top">

No

</td>
</tr>
</table>



<a name="loioc7efc5d9875c459e937fde743f4252de__section_results"/>

## Scorecard Results

The following scores were measured over all 30 test cases against the six hackathon criteria:


**Weighted Scorecard — Mistral Verifier**


<table>
<tr>
<th valign="top">

Criterion

</th>
<th valign="top">

Weight

</th>
<th valign="top">

Score

</th>
</tr>
<tr>
<td valign="top">

Extraction accuracy

</td>
<td valign="top">

30%

</td>
<td valign="top">

98.1%

</td>
</tr>
<tr>
<td valign="top">

Discrepancy detection

</td>
<td valign="top">

25%

</td>
<td valign="top">

93.8% \(15 of 16 findings\)

</td>
</tr>
<tr>
<td valign="top">

Tax determination

</td>
<td valign="top">

20%

</td>
<td valign="top">

100% \(30 of 30\)

</td>
</tr>
<tr>
<td valign="top">

Processing time

</td>
<td valign="top">

10%

</td>
<td valign="top">

100% \(p50 14.9s vs 35s target\)

</td>
</tr>
<tr>
<td valign="top">

Explainability and audit

</td>
<td valign="top">

10%

</td>
<td valign="top">

100%

</td>
</tr>
<tr>
<td valign="top">

False positive rate

</td>
<td valign="top">

5%

</td>
<td valign="top">

100% \(0 of 15 clean invoices flagged\)

</td>
</tr>
<tr>
<td valign="top">

**Weighted total**

</td>
<td valign="top">

\-

</td>
<td valign="top">

**97.9%**

</td>
</tr>
<tr>
<td valign="top">

**Decision accuracy**

</td>
<td valign="top">

\-

</td>
<td valign="top">

**100%** — correct decision and action, all 30 cases

</td>
</tr>
</table>



<a name="loioc7efc5d9875c459e937fde743f4252de__section_model_comparison"/>

## Model Comparison

The same deterministic pipeline was run with three frontier models and the regex baseline, swapping only the extractor:


**Model Comparison — Same Pipeline, Extractor Swapped**


<table>
<tr>
<th valign="top">

Model

</th>
<th valign="top">

Weighted Score

</th>
<th valign="top">

Decision Accuracy

</th>
<th valign="top">

p50 Processing Time

</th>
</tr>
<tr>
<td valign="top">

`mistralai--mistral-medium 3.5`

</td>
<td valign="top">

97.9%

</td>
<td valign="top">

100%

</td>
<td valign="top">

14.9s

</td>
</tr>
<tr>
<td valign="top">

`gpt-4o`

</td>
<td valign="top">

97.9%

</td>
<td valign="top">

100%

</td>
<td valign="top">

4.0s

</td>
</tr>
<tr>
<td valign="top">

`anthropic--claude-4.5-sonnet`

</td>
<td valign="top">

97.9%

</td>
<td valign="top">

100%

</td>
<td valign="top">

9.4s

</td>
</tr>
<tr>
<td valign="top">

Naive regex, no model

</td>
<td valign="top">

71.1%

</td>
<td valign="top">

46.7%

</td>
<td valign="top">

4ms

</td>
</tr>
</table>

All three frontier models are identical in accuracy. On this workload, extraction quality is not the differentiating variable — the deterministic layer carries the score. This means choosing Mistral for data privacy reasons costs nothing in accuracy.



<a name="loioc7efc5d9875c459e937fde743f4252de__section_baseline_gap"/>

## What the Regex Baseline Reveals

The regex baseline achieves 100% tax determination and detects most discrepancies — then flags every one of the 15 clean invoices. A pattern cannot distinguish between a contract that has no price-adjustment cap and a contract that words the cap differently, so it must escalate all 30 cases. That is why 87.5% discrepancy detection is still unusable in practice.

The decisive example is `CNT-2026-016`: clause 3.3 states *"Annual escalation of 3% applicable from Year 2 onwards. Not applicable to current contract year."* The regex reads the cap as 3%. Mistral and Claude both correctly return `null`. One sentence separates pattern matching from reading comprehension.



<a name="loioc7efc5d9875c459e937fde743f4252de__section_generalization"/>

## Generalization Evidence

The 30 published test cases are matched against known ground truth. To address the memorization question, 16 synthetic generalization cases were constructed with outcomes that appear nowhere in the dataset. The decisive case:

A 9.5% price variance is REVIEW under `CNT-2026-014`'s 10% cap and FAIL under `CNT-2026-020`'s 8% cap. The same numerical variance produces opposite outcomes because the cap is read per-contract. No single hardcoded threshold can produce both outcomes simultaneously.

Two further tests assert that `src/policy/policy.yaml` contains no invoice number, no contract ID, no test-case reference, and no hardcoded price cap — confirming that the policy file generalizes rather than fits the dataset.



<a name="loioc7efc5d9875c459e937fde743f4252de__section_related"/>

**Related Information**  


[AP Guardian Overview](ap-guardian-overview-017966c.md)

[System Architecture](architecture-5d8a8f9.md)

[API Reference](api-reference-be0d61f.md)
