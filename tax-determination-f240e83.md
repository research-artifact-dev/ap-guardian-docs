<!-- loiof240e832dbea427ab69f2e9a904de072 -->

# Tax Code Determination

AP Guardian determines the expected SAP tax code for every invoice using a pure function that takes contract type and vendor tax status as inputs and produces a tax code as output. The function contains no I/O, no model calls, and no clock — given the same inputs it returns the same code, which is what makes a posted invoice defensible in an audit two years later.


<a name="loiof240e832dbea427ab69f2e9a904de072__section_matrix"/>

## Tax Matrix

The tax matrix is defined in `policy.yaml` and covers the three codes in scope:

<table>
<tr>
<th valign="top">Contract Type</th>
<th valign="top">Vendor Tax Status</th>
<th valign="top">Tax Code</th>
<th valign="top">Meaning</th>
</tr>
<tr>
<td valign="top">MATERIAL</td>
<td valign="top">LOCAL_VAT_REGISTERED</td>
<td valign="top"><code>V3</code></td>
<td valign="top">Recoverable input VAT \(15%\)</td>
</tr>
<tr>
<td valign="top">SERVICE</td>
<td valign="top">LOCAL_VAT_REGISTERED</td>
<td valign="top"><code>V6</code></td>
<td valign="top">Non-recoverable input VAT \(15%\)</td>
</tr>
<tr>
<td valign="top">MATERIAL</td>
<td valign="top">FOREIGN_OR_NON_REGISTERED</td>
<td valign="top"><code>V0</code></td>
<td valign="top">Zero-rated / reverse charge</td>
</tr>
<tr>
<td valign="top">SERVICE</td>
<td valign="top">FOREIGN_OR_NON_REGISTERED</td>
<td valign="top"><code>V0</code></td>
<td valign="top">Zero-rated / reverse charge</td>
</tr>
</table>

`UNKNOWN` never counts as a match. An undetermined code routes the invoice to review rather than quietly agreeing with whatever the vendor wrote.

> ### Note:  
> Extending the tax matrix to the wider Nortech code set (V1, V5, 9F, 9G) is an edit to `policy.yaml`, not a code change.


<a name="loiof240e832dbea427ab69f2e9a904de072__section_vendor"/>

## Vendor Classification

The vendor master `taxClassification` field uses Nortech-specific labels. The policy maps these to the canonical tax status used by the tax matrix:

<table>
<tr>
<th valign="top">Vendor master classification</th>
<th valign="top">Canonical tax status</th>
</tr>
<tr>
<td valign="top"><code>LOCAL_MATERIAL</code></td>
<td valign="top">LOCAL_VAT_REGISTERED</td>
</tr>
<tr>
<td valign="top"><code>LOCAL_SERVICE</code></td>
<td valign="top">LOCAL_VAT_REGISTERED</td>
</tr>
<tr>
<td valign="top"><code>LOCAL_NON_RECOVERABLE</code></td>
<td valign="top">LOCAL_VAT_REGISTERED</td>
</tr>
<tr>
<td valign="top"><code>FOREIGN</code></td>
<td valign="top">FOREIGN_OR_NON_REGISTERED</td>
</tr>
<tr>
<td valign="top"><code>FOREIGN_NON_REGISTERED</code></td>
<td valign="top">FOREIGN_OR_NON_REGISTERED</td>
</tr>
</table>

Recoverability is a property of what is supplied, not of the vendor's registration, so all three local classifications map to the same registered status.


<a name="loiof240e832dbea427ab69f2e9a904de072__section_resolution"/>

## Contract Type Resolution

Contract type comes from the AI extraction layer. Resolution logic:

1.  If the extracted `contractType` confidence is below the policy minimum (0.85), the type is treated as `UNKNOWN`, and the expected tax code is `UNKNOWN`.
2.  If the extracted `contractType` is `UNKNOWN` (the model could not determine it), the system falls back to the material number prefix. Material numbers starting with `SV-` indicate a service line.
3.  The material number corroboration is only used as a fallback — a disagreement between the contract type reading and the material number prefix is surfaced as an extraction concern rather than silently resolved.

The AI supplies the contract type by reading the agreement. It does not supply the tax code — that mapping is policy, and policy is code plus configuration.


<a name="loiof240e832dbea427ab69f2e9a904de072__section_confidence"/>

## Confidence in Tax Determination

The confidence reported alongside the expected tax code is the confidence in the **inputs**, never in the rule. The mapping from contract type and vendor status to a tax code is a lookup and carries no uncertainty. What can be uncertain is the contract type reading, so the tax determination confidence equals the contract type extraction confidence — the only honest number to show next to a recommended code.

Vendor status comes from structured master data and is considered certain.


<a name="loiof240e832dbea427ab69f2e9a904de072__section_coverage"/>

## Dataset Coverage Note

The Nortech test dataset contains all 18 vendors in Saudi Arabia, so `V0` (foreign/non-registered) has zero examples in the standard 30-case evaluation. The Arabic robustness probe includes synthetic foreign-vendor scenarios. Extending to real V0 cases requires adding non-Saudi vendors to the dataset.

**Related Information**  


[Verification Pipeline](verification-pipeline-e493a6b.md)

[Policy Configuration](policy-configuration-ebd7cc7.md)

[AI-Based Contract Extraction](ai-contract-extraction-3608239.md)
