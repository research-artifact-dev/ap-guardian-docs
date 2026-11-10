<!-- loio84479ca28b07469890ec708009a24744 -->

# Configuration and Deployment

AP Guardian is configured through environment variables and a central policy file, and is deployed to SAP BTP Cloud Foundry using the Multi-Target Application \(MTA\) build and deploy toolchain. This guide covers local setup, environment configuration, and the full BTP deployment procedure.



<a name="loio84479ca28b07469890ec708009a24744__section_prerequisites"/>

## Prerequisites

Before running or deploying AP Guardian, ensure the following are in place:

-   Node.js 20 or higher is installed.
-   The SAP AI Core service instance is provisioned in your BTP subaccount with a Mistral Medium 3.5 deployment created and its deployment ID noted.
-   The Cloud Foundry CLI is installed and you are logged in to the target organization and space.
-   The MultiApps CF CLI plugin \(`cf deploy`\) is installed.
-   The Cloud MTA Build Tool \(MBT\) is installed.
-   The Nortech dataset is available — see `data/README.md` for the required directory layout. The data is confidential and is not included in the repository.



<a name="loio84479ca28b07469890ec708009a24744__section_env_config"/>

## Environment Configuration

Copy `.env.example` to `.env` and populate the SAP AI Core binding values:

```sh
cp .env.example .env
```

The `.env` file must never be committed to version control. It is listed in `.gitignore`. The application refuses to start against any endpoint that is not an SAP AI Core endpoint — `localhost` and local model gateways are rejected at construction time.


**Required Environment Variables**


<table>
<tr>
<th valign="top">

Variable

</th>
<th valign="top">

Description

</th>
</tr>
<tr>
<td valign="top">

`AICORE_SERVICE_KEY`

</td>
<td valign="top">

Full JSON service key for the SAP AI Core instance \(clientid, clientsecret, url, serviceurls\).

</td>
</tr>
<tr>
<td valign="top">

`AI_CORE_DEPLOYMENT_ID`

</td>
<td valign="top">

The deployment ID of the Mistral model \(for example, `da6e6901fe8fae8c`\). Changing this value is how the model is swapped without touching code.

</td>
</tr>
<tr>
<td valign="top">

`AI_CORE_RESOURCE_GROUP`

</td>
<td valign="top">

The AI Core resource group \(default: `default`\).

</td>
</tr>
<tr>
<td valign="top">

`AP_PROVIDER`

</td>
<td valign="top">

The extraction provider to use. Set to `mistral` for live extraction or `pipeline` to use pre-read contract policy \(no network required\).

</td>
</tr>
<tr>
<td valign="top">

`AP_DATA`

</td>
<td valign="top">

Data source. Set to `xlsx` for the local workbook or `hana` for SAP HANA Cloud.

</td>
</tr>
</table>



<a name="loio84479ca28b07469890ec708009a24744__section_local_run"/>

## Running Locally

Use the following commands to run the application in a local environment:

```sh
npm install
npm test                    # 104 unit tests, no network required
npm run eval                # scorecard against all 30 scenarios, cached
npm run eval naive          # regex baseline, no AI
npm run eval mistral        # live Mistral extraction via SAP AI Core

npx tsx src/api/server.ts                        # UI on cached data
AP_PROVIDER=mistral npx tsx src/api/server.ts   # UI with live Mistral
```

After starting the server, open `http://localhost:3001` in a browser.

> ### Tip:  
> Running `npm run eval` with the default `pipeline` provider \(no network\) gives the deterministic ceiling score in approximately 2 seconds. Use this to verify logic changes without consuming SAP AI Core capacity.



<a name="loio84479ca28b07469890ec708009a24744__section_policy_config"/>

## Policy Configuration

All numeric thresholds in AP Guardian are stored in `src/policy/policy.yaml`. This is the single source of truth for:

-   The confidence minimum for critical fields \(default: 0.85\)
-   The tax code matrix \(contract type × vendor tax status → tax code\)
-   The severity map \(which finding types are FAIL vs REVIEW\)
-   The action map \(FAIL → Block, REVIEW → Park, PASS → Post\)
-   Field normalization rules for payment terms and Incoterms

A numeric literal anywhere else in the `src/` tree is a bug and will be flagged in code review.



<a name="loio84479ca28b07469890ec708009a24744__section_btp_deploy"/>

## BTP Deployment

AP Guardian is packaged and deployed as an MTA archive. The `mta.yaml` descriptor defines five modules: the verification service, the HDI database deployer, the Fiori UI, the UI deployer, and the approuter.

### Building the MTA Archive

```sh
npm run build
mbt build
```

The build produces an `.mtar` file in the `mta_archives/` directory.

### Deploying to Cloud Foundry

```sh
cf login -a <CF_API_ENDPOINT> -o <ORG> -s <SPACE>
cf deploy mta_archives/ap-guardian_1.0.0.mtar
```

The deployment creates or updates all five modules and provisions the required BTP service instances \(HANA HDI container, XSUAA, HTML5 repository, Destination service\). The AI Core service instance must already exist; the `mta.yaml` references it as an existing service named `default_aicore`.

> ### Note:  
> The AI Core deployment ID and resource group can be overridden at deploy time using the `-e` flag without rebuilding the archive: `cf deploy mta_archives/ap-guardian_1.0.0.mtar -e override.mtaext`. This is the supported way to point the same archive at a different model.

### Swapping the AI Model

To use a different model deployment without changing code:

1.  Choose *SAP AI Core Launchpad* \> *ML Operations* \> *Deployments* and note the new deployment ID.
2.  Set `AI_CORE_DEPLOYMENT_ID` in the environment or in an `.mtaext` override file.
3.  Redeploy with `cf deploy`.



<a name="loio84479ca28b07469890ec708009a24744__section_related"/>

**Related Information**  


[System Architecture](architecture-5d8a8f9.md)

[Security and Privacy](security-and-privacy-f6e56e1.md)

[Deploying Applications on SAP BTP](https://help.sap.com/docs/BTP/df50977d8bfa4c9a8a063ddb37113c43/866ab13d5f8e48cdaac6d70e55e76e09.html)

[Deploy a Multitarget Application with Command-Line Tools](https://help.sap.com/docs/HANA_CLOUD_DATABASE/c2b99f19e9264c4d9ae9221b22f6f589/854a00eb1193486383fc4a97b998dabd.html "Install a multi-target application on SAP HANA Cloud using the CF CLI MultiApps plugin.") :arrow_upper_right:

[Build and Deploy Content to HTML5 Application Repository](https://help.sap.com/docs/Portal_Service/956efb6db08c450bb303a0d316b72f10/439431592b8e465c834e64741746e013.html)
