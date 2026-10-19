# Self-provision in your own Azure subscription

This guide is for teams running the Agentic Inventory Planning MicroHack outside the managed MicroHack platform.

It assumes that:

- Your team repository has already been created from the template.
- You are reading this guide from that repository.
- You have opened the repository in its GitHub Codespace.

You will deploy the Azure resources, configure the application, and run the lab from the same Codespace.

> [!IMPORTANT]
> If your deployment is blocked by permissions, model availability, quota, or an organizational Azure Policy, stop after 10-15 minutes and ask the facilitator for a prepared environment. The purpose of the workshop is to learn agent patterns, not to debug subscription governance.

## Expected deployment

Each team receives an isolated environment:

| Resource | Default configuration |
|---|---|
| Resource group | One per team |
| Microsoft Foundry account | Azure AI Services, `S0` |
| Foundry project | `inventory-hack` |
| Model deployment | `gpt-5.4-mini`, version `2026-03-17` |
| Model SKU | `GlobalStandard`, capacity 100 |
| Azure Cosmos DB for NoSQL | Serverless, local authentication disabled |
| Application Insights | Workspace-based tracing |
| Log Analytics workspace | Observability backend |

Provisioning normally takes 10-20 minutes. Role assignments can take another 2-3 minutes to propagate.

## 1. Check your Azure prerequisites

The deploying user needs:

- An active Azure subscription.
- `Owner`, or `Contributor` plus `User Access Administrator`.
- Permission to create role assignments.
- Permission to register resource providers.
- `gpt-5.4-mini` version `2026-03-17` available in the selected region.
- At least 100 GlobalStandard capacity units available for that model and version.

The recommended region is:

```text
Sweden Central
```

Before deploying, check the Microsoft Foundry model catalog for your exact subscription and region. If the model or quota is unavailable, use a facilitator-provided environment.

## 2. Sign in from the Codespace

Two Azure authentication contexts are used:

- Azure CLI is used by the Python application through `DefaultAzureCredential`.
- Az PowerShell is used by the infrastructure deployment script.

First, sign in with Azure CLI:

```bash
az login --tenant <tenant-id> --use-device-code
az account set --subscription <subscription-id>
```

Confirm the selected subscription:

```bash
az account show --output table
```

Start PowerShell:

```bash
pwsh
```

Sign in to Az PowerShell and select the same subscription:

```powershell
Connect-AzAccount `
    -Tenant "<tenant-id>" `
    -UseDeviceAuthentication

Set-AzContext `
    -SubscriptionId "<subscription-id>"
```

Confirm the context:

```powershell
Get-AzContext
```

## 3. Register the required resource providers

Run this in the PowerShell session:

```powershell
$providers = @(
    "Microsoft.CognitiveServices",
    "Microsoft.DocumentDB",
    "Microsoft.OperationalInsights",
    "Microsoft.Insights"
)

foreach ($provider in $providers) {
    Register-AzResourceProvider `
        -ProviderNamespace $provider |
        Out-Null
}
```

Check their state:

```powershell
$providers | ForEach-Object {
    Get-AzResourceProvider `
        -ProviderNamespace $_ |
        Select-Object -First 1 `
            ProviderNamespace,
            RegistrationState
}
```

Wait until every provider reports `Registered`.

## 4. Resolve your Microsoft Entra object ID

The deployment grants the signed-in user access to Foundry and Cosmos DB.

```powershell
$userObjectId = (
    Get-AzADUser -SignedIn
).Id

$userObjectId
```

If this command cannot resolve your identity, ask your tenant administrator for your user object ID and assign it manually:

```powershell
$userObjectId = "<your-entra-user-object-id>"
```

## 5. Deploy the team environment

Choose a unique resource group name:

```powershell
$subscriptionId = "<subscription-id>"
$resourceGroup = "rg-inventory-hack-team01"
$location = "swedencentral"
```

Run the wrapper from the repository root:

```powershell
$result = .\labautomation\run-local.ps1 `
    -SubscriptionId $subscriptionId `
    -ResourceGroupName $resourceGroup `
    -Location $location `
    -AllowedEntraUserIds @($userObjectId)
```

The wrapper:

1. Creates the resource group.
2. Applies the `SecurityControl=Ignore` tag used by Microsoft internal MCAPS lab subscriptions.
3. Creates the Foundry account and `inventory-hack` project.
4. Deploys `gpt-5.4-mini`.
5. Creates Application Insights and Log Analytics.
6. Creates the Cosmos DB account, database, and containers.
7. Assigns the required data-plane roles.
8. Returns the endpoints needed by the application.

On non-MCAPS subscriptions, the `SecurityControl=Ignore` tag is normally a harmless no-op. It does not override Azure Policies defined by another organization.

## 6. Display and save the deployment values

```powershell
$credentials = $result |
    ForEach-Object {
        if ($_.HackboxCredential) {
            $_.HackboxCredential
        }
    }

$credentials |
    Format-Table name, value, note -AutoSize
```

You should receive:

| Output | Application setting |
|---|---|
| `FoundryProjectEndpoint` | `PROJECT_ENDPOINT` |
| `ModelDeploymentName` | `MODEL_DEPLOYMENT_NAME` |
| `CosmosEndpoint` | `COSMOS_ENDPOINT` |
| `ResourceGroupName` | Resource location and cleanup |

These values are endpoints and names, not passwords or account keys.

## 7. Verify Azure before continuing

Open the resource group in the Azure portal and confirm that it contains:

- An Azure AI Services account.
- A Foundry project.
- A Cosmos DB account.
- Application Insights.
- A Log Analytics workspace.

Open [Microsoft Foundry](https://ai.azure.com), select the `inventory-hack` project, and verify:

```text
gpt-5.4-mini = Succeeded
```

> [!CAUTION]
> Do not continue if the model deployment is missing or failed. The deployment script can report a warning and continue when model deployment or an RBAC assignment fails.

## 8. Configure the application

Exit PowerShell:

```powershell
exit
```

The Codespace post-create command has already copied `.env.example` to `src/.env`.

Open `src/.env` and set:

```env
PROJECT_ENDPOINT=<FoundryProjectEndpoint>
MODEL_DEPLOYMENT_NAME=gpt-5.4-mini

COSMOS_ENDPOINT=<CosmosEndpoint>
COSMOS_DATABASE=inventory

ENABLE_WEB_SEARCH=false
REASONING_EFFORT=minimal
```

Do not add quotes around the values, and never commit `src/.env`.

## 9. Start the Planner Console

```bash
cd src
uv run uvicorn ui.app:app --reload --port 8000
```

When Codespaces detects port 8000, select **Open in Browser**.

On first use, the application:

1. Connects to Cosmos DB with your Azure CLI identity.
2. Seeds the bundled inventory data.
3. Loads the current exposure and order book.

## 10. Readiness checkpoint

Do not start Challenge 2 until all checks pass:

- [ ] Azure CLI is signed in to the correct tenant and subscription.
- [ ] `gpt-5.4-mini` is `Succeeded`.
- [ ] `src/.env` contains the correct endpoints.
- [ ] The Planner Console opens.
- [ ] Inventory appears in the console.
- [ ] Cosmos DB contains seeded data.
- [ ] The order book loads.
- [ ] There are no `403` errors.

Continue with [Challenge 1](../challenges/challenge-01.md) after this checkpoint.

## 11. Grant access to multiple team members

If several members need to use Azure directly, pass every user object ID during deployment:

```powershell
$userIds = @(
    "<object-id-member-1>",
    "<object-id-member-2>",
    "<object-id-member-3>"
)

$result = .\labautomation\run-local.ps1 `
    -SubscriptionId $subscriptionId `
    -ResourceGroupName $resourceGroup `
    -Location $location `
    -AllowedEntraUserIds $userIds
```

Each user receives:

- Azure AI User.
- Cognitive Services User.
- Cosmos DB Built-in Data Contributor.

If only one person will run the Codespace, granting access to that person is sufficient.

## 12. Troubleshooting

| Symptom | Action |
|---|---|
| `DeploymentModelNotSupported` | The model/version is unavailable in that region; use another supported region or a facilitator environment |
| Model deployment fails | Check model availability and the exact model/version quota |
| Role assignment fails | The deploying account needs `Owner` or `User Access Administrator` |
| Cosmos DB returns `403` | Wait 2-3 minutes for RBAC propagation, verify the signed-in identity, and retry |
| Cosmos DB is unreachable | Check `publicNetworkAccess` and the subscription's Azure Policies |
| The console reports `not connected` | Check `src/.env`, then repeat `az login` |
| Deployment returns warnings | Verify the model, roles, and endpoints manually |
| Team is blocked for more than 15 minutes | Ask the facilitator for a prepared environment |

### Organizational network policies

The application connects from the Codespace to public Foundry and Cosmos DB endpoints.

If your organization requires private endpoints or disables public network access:

- `SecurityControl=Ignore` might have no effect outside Microsoft internal MCAPS subscriptions.
- Follow your organization's exemption process, or use a facilitator-provided environment.
- Do not attempt to weaken organizational policies without authorization.

## 13. Resume after stopping the Codespace

The Azure resources and Cosmos DB data persist after the Codespace stops.

Sign in again and restart the application:

```bash
az login --tenant <tenant-id> --use-device-code
az account set --subscription <subscription-id>

cd src
uv run uvicorn ui.app:app --reload --port 8000
```

## 14. Clean up

Delete the resource group after the workshop to stop charges.

From Az PowerShell:

```powershell
Remove-AzResourceGroup `
    -Name $resourceGroup `
    -Force
```

Or from Azure CLI:

```bash
az group delete \
    --name "rg-inventory-hack-team01" \
    --yes \
    --no-wait
```

Deleting the resource group removes all resources created by the lab.
