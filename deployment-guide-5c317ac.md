<!-- loio5c317ac815244cd5b41a6e5d836006b0 -->

# Deployment Guide

AP Guardian runs locally without any cloud services and deploys to SAP BTP Cloud Foundry as a multi-target application (MTA). This document covers both paths.


<a name="loio5c317ac815244cd5b41a6e5d836006b0__section_local"/>

## Local Development

**Prerequisites:** Node.js ≥ 20, npm.

**Install and test:**

```sh
npm install
npm test         # 104 unit tests, no network required
npm run eval     # scorecard against all 30 scenarios, offline → 97.9%
```

**Run the API (fixture mode, no credentials needed):**

```sh
npx tsx src/api/server.ts
```

Then open `http://localhost:3001`.

**Run with live Mistral extraction:**

```sh
AP_PROVIDER=mistral npx tsx src/api/server.ts
```

Startup takes approximately 80 seconds as the system reads all 25 contracts through SAP AI Core. Subsequent startups use the cache (~2s).


<a name="loio5c317ac815244cd5b41a6e5d836006b0__section_env"/>

## Environment Variables

Copy `.env.example` to `.env` and fill in the SAP AI Core binding. The file is git-ignored and must stay that way. The application refuses to start against any endpoint that is not SAP AI Core — including `localhost`. That is deliberate: a local model gateway is a development convenience that exists in no deployed environment, so code that reaches for it passes every local test and fails on deployment.

<table>
<tr>
<th valign="top">Variable</th>
<th valign="top">Required for</th>
<th valign="top">Description</th>
</tr>
<tr>
<td valign="top"><code>AI_CORE_AUTH_URL</code></td>
<td valign="top">Mistral mode</td>
<td valign="top">OAuth 2.0 token endpoint for SAP AI Core.</td>
</tr>
<tr>
<td valign="top"><code>AI_CORE_CLIENT_ID</code></td>
<td valign="top">Mistral mode</td>
<td valign="top">Client ID from the AI Core service binding.</td>
</tr>
<tr>
<td valign="top"><code>AI_CORE_CLIENT_SECRET</code></td>
<td valign="top">Mistral mode</td>
<td valign="top">Client secret from the AI Core service binding.</td>
</tr>
<tr>
<td valign="top"><code>AI_CORE_API_URL</code></td>
<td valign="top">Mistral mode</td>
<td valign="top">AI Core inference API base URL.</td>
</tr>
<tr>
<td valign="top"><code>AI_CORE_DEPLOYMENT_ID</code></td>
<td valign="top">Mistral mode</td>
<td valign="top">Deployment ID for the Mistral model. This is an identifier, not a secret — changing it is how the model is swapped without touching code.</td>
</tr>
<tr>
<td valign="top"><code>AI_CORE_RESOURCE_GROUP</code></td>
<td valign="top">Mistral mode</td>
<td valign="top">Resource group name. Defaults to <code>default</code>.</td>
</tr>
<tr>
<td valign="top"><code>AP_PROVIDER</code></td>
<td valign="top">All modes</td>
<td valign="top"><code>mistral</code> for live extraction; any other value uses the fixture cache. Defaults to <code>fixture</code>.</td>
</tr>
<tr>
<td valign="top"><code>AP_DATA</code></td>
<td valign="top">All modes</td>
<td valign="top"><code>hana</code> to read from SAP HANA Cloud; any other value reads the XLSX workbook. Auto-detects Cloud Foundry via <code>VCAP_SERVICES</code>.</td>
</tr>
<tr>
<td valign="top"><code>PORT</code></td>
<td valign="top">All modes</td>
<td valign="top">HTTP port. Defaults to <code>3001</code>.</td>
</tr>
</table>

On Cloud Foundry, `VCAP_SERVICES` delivers the AI Core binding automatically. The service reads from both the managed `aicore` service and user-provided service instances whose name matches `/ai.?core/i`.


<a name="loio5c317ac815244cd5b41a6e5d836006b0__section_hana"/>

## SAP HANA Cloud Setup

```sh
npm run hana:check    # verify connectivity and schema
npm run hana:deploy   # deploy HDI tables via the deployer module
npm run hana:load     # load the Nortech dataset from the XLSX workbook
```

The HDI deployer creates five tables under the `APG_` prefix. The `hana-load.ts` script reads the XLSX workbook and writes each sheet to its corresponding table.


<a name="loio5c317ac815244cd5b41a6e5d836006b0__section_btp"/>

## BTP Deployment

**Prerequisites:**

-   SAP BTP subaccount with Cloud Foundry environment enabled
-   Cloud Foundry CLI installed and logged in
-   MultiApps CF CLI plug-in installed (`cf install-plugin multiapps`)
-   SAP HANA Cloud instance available in the subaccount
-   SAP AI Core service instance (existing) or entitlement for a new one

**Build and deploy:**

```sh
npm ci
npm run build           # TypeScript → dist/
npm run build:ui        # Vite build → web/dist/
mbt build               # package as MTA archive (.mtar)
cf deploy ap-guardian_1.0.0.mtar
```

The `mta.yaml` configures parallel deployment (`enable-parallel-deployments: true`). The AI Core deployment ID and resource group are overridable at deploy time:

```sh
cf deploy ap-guardian_1.0.0.mtar \
  -e extensions.yaml   # override ai-core-deployment and ai-core-resource-group
```


<a name="loio5c317ac815244cd5b41a6e5d836006b0__section_security"/>

## Security Roles

Defined in `xs-security.json` and created as XSUAA role collections:

<table>
<tr>
<th valign="top">Role Collection</th>
<th valign="top">Scopes</th>
<th valign="top">Description</th>
</tr>
<tr>
<td valign="top">AP Guardian Accountant</td>
<td valign="top">Display</td>
<td valign="top">Read-only access to the invoice queue and verification results.</td>
</tr>
<tr>
<td valign="top">AP Guardian Supervisor</td>
<td valign="top">Display + Approve</td>
<td valign="top">Can view invoices and release blocked invoices for payment.</td>
</tr>
</table>


<a name="loio5c317ac815244cd5b41a6e5d836006b0__section_data"/>

## Test Data Placement

The Nortech dataset is excluded from version control. Place the organizer-supplied test data package in:

```
data/
  Nortech_Customer_data.xlsx
  contracts/
    CNT-2026-001_Agreement.pdf
    ...
    CNT-2026-025_Agreement.pdf
```

Run `npm test` after placement to verify the expected shape (30 invoices, 25 POs, 30 GRs, 18 vendors, 15 PASS / 2 REVIEW / 13 FAIL ground truth). If the organizers ship an updated dataset, failing tests will report the shape mismatch — update the expectations deliberately.

**Related Information**  


[Maintaining the Multitarget Application Development and Deployment Descriptors](https://help.sap.com/docs/HANA_CLOUD_DATABASE/c2b99f19e9264c4d9ae9221b22f6f589/b2e355a5137c4799932f776716b292c9.html)

[Setting Up a Schema or an HDI Container](https://help.sap.com/docs/HANA_CLOUD_CN/57175c34172a4f73b765f95f899c0094/2863434ddda042b8b8011a3f24856281.html)

[Architecture and Components](architecture-and-components-569c834.md)

[Privacy and Security](privacy-and-security-77cbf91.md)
