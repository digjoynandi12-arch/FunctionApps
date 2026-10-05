# Migrating Azure Function Apps Between Resource Groups (Same Subscription)
### A step-by-step runbook, laid out slide by slide

---

## Slide 1: Overview and Key Facts

**What a resource group move does**
- A move within the same subscription changes only the resource's **metadata**, meaning its resource group and resource ID. The Function App is not redeployed or recreated.
- The app's **hostname (`<app>.azurewebsites.net`), code, app settings, and system-assigned managed identity stay the same**.
- The **Resource ID changes** from `/subscriptions/<sub>/resourceGroups/OLD-RG/...` to `/subscriptions/<sub>/resourceGroups/NEW-RG/...`. Anything that references the full resource ID must be updated afterward.
- The **region cannot change**. Resources keep their location. Moving to another region is a different process (recreate and redeploy).

**Typical downtime:** none in most cases. The app keeps running during the move. Plan a maintenance window anyway as a precaution.

**Lock period:** during the move, both the source and target resource groups are locked for write and delete operations. This usually lasts minutes and can last up to about 4 hours.

---

## Slide 2: What You Will Move Together

Identify the full set of resources that belong with the Function App:

| Resource | Move with the app? | Notes |
|---|---|---|
| Function App (`Microsoft.Web/sites`) | **Yes** | Deployment slots move automatically with it |
| App Service Plan / Hosting plan (`Microsoft.Web/serverFarms`) | **Yes (strongly recommended)** | The app and its plan should be moved in the same operation |
| Application Insights | Optional (recommended) | The connection string does not change |
| Log Analytics workspace | Optional | Only if dedicated to this app |
| Storage account (`AzureWebJobsStorage`) | Optional | The connection string does not change |
| Uploaded/imported TLS certificates (`Microsoft.Web/certificates`) | See Slide 9 | Need special handling |
| App Service Managed Certificates | Usually recreate | Bound to the app |
| User-assigned managed identity | Optional | It keeps working from either RG |
| Key Vault, VNet, Private Endpoints | Usually leave in place | Cross-RG references keep working |

> **Rule of thumb:** move the **Function App, its hosting plan, and all its slots in one operation**. If several Function Apps share one plan, they must all move together with that plan.

---

## Slide 3: Prerequisites Checklist

Complete these before starting:

1. **Permissions**
   - On the **source RG**, you need `Microsoft.Resources/subscriptions/resourceGroups/moveResources/action`. Contributor or Owner includes it.
   - On the **target RG**, you need `Microsoft.Resources/subscriptions/resourceGroups/write`. Contributor or Owner includes it.
2. **Target resource group exists** in the same subscription. Its region does not have to match the resources' region.
3. **No blocking locks.** A `ReadOnly` or `CanNotDelete` lock on the source RG, target RG, or the resources will block the move. Remove the locks temporarily.
4. **Resource providers registered** in the subscription: `Microsoft.Web`, plus `Microsoft.Insights` and `Microsoft.Storage` if those resources move too.
5. **Azure Policy check.** Confirm that no policy on the target RG will deny the incoming resources, for example a tag requirement or allowed-locations policy.
6. **Not on an App Service Environment (ASE).** Apps in an ASE cannot be moved this way.
7. **Tooling (for the terminal route):**
   - Azure CLI 2.50+ (`az version`), or
   - Az PowerShell module (`Get-Module Az -ListAvailable`).

---

## Slide 4: Pre-Migration Discovery (Terminal)

Log in and set context:

```bash
az login
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"
az account show --output table
```

Define variables (reused throughout this runbook):

```bash
SUB_ID=$(az account show --query id -o tsv)
SRC_RG="rg-source"
DST_RG="rg-target"
FUNC_APP="my-function-app"
```

Inventory the source resource group:

```bash
az resource list --resource-group $SRC_RG \
  --query "[].{Name:name, Type:type, Location:location}" -o table
```

Identify the Function App's hosting plan:

```bash
PLAN_ID=$(az functionapp show -g $SRC_RG -n $FUNC_APP --query serverFarmId -o tsv)
echo $PLAN_ID
```

Find all apps sharing that plan (all of them must move together):

```bash
az functionapp list --query "[?serverFarmId=='$PLAN_ID'].{Name:name, RG:resourceGroup}" -o table
az webapp list --query "[?serverFarmId=='$PLAN_ID'].{Name:name, RG:resourceGroup}" -o table
```

List deployment slots:

```bash
az functionapp deployment slot list -g $SRC_RG -n $FUNC_APP -o table
```

Check for uploaded certificates:

```bash
az webapp config ssl list -g $SRC_RG -o table
```

---

## Slide 5: Pre-Migration Discovery (Portal)

1. Go to **portal.azure.com**, then **Resource groups**, then select the **source RG**.
2. Note every resource related to the Function App: the app, the App Service plan, Application Insights, the storage account, and certificates.
3. Open the **Function App**, then **Overview**, and note the **App Service plan** name.
4. Open the **App Service plan**, then **Apps** (left menu), and confirm which apps live on it.
5. In the Function App, open **Deployment slots** and note any slots.
6. In the Function App, open **Certificates** (or **TLS/SSL settings** on older UIs) and note any uploaded certificates.
7. Under **Settings**, open **Locks** for the source RG, the target RG, and the Function App, and note any locks.

---

## Slide 6: Back Up and Capture Current State

Take a snapshot of configuration so you can verify after the move (and recover if needed).

**Export app settings and connection strings:**

```bash
az functionapp config appsettings list -g $SRC_RG -n $FUNC_APP -o json > appsettings-backup.json
az webapp config connection-string list -g $SRC_RG -n $FUNC_APP -o json > connstrings-backup.json
```

**Export the RG as an ARM template (for reference):**

```bash
az group export --name $SRC_RG > src-rg-template.json
```

**Record role assignments scoped directly to the Function App.** Assignments made directly on a resource do **not** move with it:

```bash
FUNC_ID=$(az functionapp show -g $SRC_RG -n $FUNC_APP --query id -o tsv)
az role assignment list --scope $FUNC_ID --include-inherited false -o json > rbac-backup.json
```

**Record the managed identity principal ID:**

```bash
az functionapp identity show -g $SRC_RG -n $FUNC_APP -o json > identity-backup.json
```

**Record diagnostic settings:**

```bash
az monitor diagnostic-settings list --resource $FUNC_ID -o json > diag-backup.json
```

**Record alerts that target the app.** Do this in the portal under **Monitor**, then **Alerts**, then **Alert rules**, and filter by the resource.

**Portal alternative for backup:** Open the Function App, go to **Automation**, then **Export template**, and click **Download**.

---

## Slide 7: Remove Blocking Locks (If Any)

**Terminal:**

```bash
# List locks
az lock list --resource-group $SRC_RG -o table
az lock list --resource-group $DST_RG -o table

# Delete a lock (note its details first so you can recreate it)
az lock delete --name "<LOCK_NAME>" --resource-group $SRC_RG
```

**Portal:**
1. Open the **resource group**, then **Settings**, then **Locks**.
2. Note the lock name, type, and notes.
3. Click **Delete** on each blocking lock.
4. Repeat for the target RG and for any individual resources.

> You will re-create these locks in Slide 15.

---

## Slide 8: Create the Target Resource Group (If Needed)

**Terminal:**

```bash
az group create --name $DST_RG --location "<region>"
```

**Portal:**
1. Go to **Resource groups** and click **+ Create**.
2. Select the same **Subscription**.
3. Enter the **Resource group** name.
4. Choose a **Region**. This is only the metadata location; it doesn't need to match the resources.
5. Click **Review + create**, then **Create**.

Apply any required tags now so policies don't block the move:

```bash
az group update --name $DST_RG --set tags.Environment=Prod tags.Owner=TeamX
```

---

## Slide 9: Handle Uploaded TLS/SSL Certificates (Only If Present)

Within the same subscription, a third-party certificate that you uploaded can sometimes block or complicate the move. You have two options.

**Option A: Leave the certificate behind (simplest).**
The app can move without the certificate, and SSL bindings continue to work.

**Option B: Move the certificate along with the app.**
1. Make sure you have a copy of the certificate (the `.pfx` file and its password).
2. Remove the SSL binding and delete the certificate from the app.
   - In the portal: open the **Function App**, then **Certificates**, then **Bring your own certificates (.pfx)**, select the certificate, and click **Delete**. Remove the bindings first under **Custom domains**.
3. Move the Function App (Slides 10 to 12).
4. Re-upload the certificate in the new RG and re-create the bindings (Slide 14).

**App Service Managed Certificates (free certificates):** if validation complains about them, delete them before the move and re-create them afterward. They are free and quick to re-issue.

---

## Slide 10: Validate the Move (Dry Run) from the Terminal

Validation tests the move **without making changes**. Always run it first.

**Collect the resource IDs to move:**

```bash
FUNC_ID=$(az functionapp show -g $SRC_RG -n $FUNC_APP --query id -o tsv)
PLAN_ID=$(az functionapp show -g $SRC_RG -n $FUNC_APP --query serverFarmId -o tsv)
AI_ID=$(az resource list -g $SRC_RG --resource-type "Microsoft.Insights/components" --query "[0].id" -o tsv)

echo $FUNC_ID; echo $PLAN_ID; echo $AI_ID
```

**Call the validation API:**

```bash
az rest --method post \
  --uri "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$SRC_RG/validateMoveResources?api-version=2021-04-01" \
  --body "{
    \"resources\": [\"$FUNC_ID\", \"$PLAN_ID\", \"$AI_ID\"],
    \"targetResourceGroup\": \"/subscriptions/$SUB_ID/resourceGroups/$DST_RG\"
  }" \
  --verbose
```

**Interpreting the result:**
- A **202 Accepted** response means validation is running. Copy the `Location` header URL from the verbose output and poll it:

```bash
az rest --method get --uri "<LOCATION_HEADER_URL>"
```

- An empty result, or a **204 No Content**, means **validation passed**.
- An error message names the blocking resource or reason. Fix it and re-validate.

**PowerShell equivalent:**

```powershell
Invoke-AzResourceAction -Action validateMoveResources `
  -ResourceId "/subscriptions/$SubId/resourceGroups/$SrcRg" `
  -Parameters @{
     resources = @($FuncId, $PlanId, $AiId)
     targetResourceGroup = "/subscriptions/$SubId/resourceGroups/$DstRg"
  } -Force
```

---

## Slide 11: Execute the Move from the Terminal

**Azure CLI:**

```bash
az resource move \
  --destination-group $DST_RG \
  --ids $FUNC_ID $PLAN_ID $AI_ID
```

- The command blocks until the move completes, usually in a few minutes.
- Do **not** cancel it or run other operations on either RG meanwhile.
- Deployment slots (`Microsoft.Web/sites/slots`) move automatically with the parent app. Do not list them separately.

**PowerShell:**

```powershell
Connect-AzAccount
Set-AzContext -Subscription "<SUBSCRIPTION_ID>"

$FuncId = (Get-AzWebApp -ResourceGroupName "rg-source" -Name "my-function-app").Id
$PlanId = (Get-AzWebApp -ResourceGroupName "rg-source" -Name "my-function-app").ServerFarmId
$AiId   = (Get-AzResource -ResourceGroupName "rg-source" -ResourceType "Microsoft.Insights/components").ResourceId

Move-AzResource -DestinationResourceGroupName "rg-target" `
  -ResourceId $FuncId, $PlanId, $AiId
```

Type **Y** to confirm, or add `-Force` to skip the prompt.

---

## Slide 12: Validate and Execute the Move from the Portal

**Method 1: From the source resource group (recommended for multiple resources)**
1. Open **Resource groups** and select the **source RG**.
2. Tick the checkboxes for the **Function App**, its **App Service plan**, and the optional resources (App Insights, storage, and so on).
3. On the top toolbar, click **Move**, then **Move to another resource group**.
4. On the **Source + target** tab, select the **target resource group** and click **Next**.
5. On the **Resources to move** tab, the portal **automatically validates** the move. Wait for every item to show **Succeeded**. If any item fails, click its status to read the error, fix it, and re-run validation. Then click **Next**.
6. On the **Review** tab, tick the acknowledgement checkbox ("I understand that tools and scripts associated with moved resources will not work until I update them to use new resource IDs").
7. Click **Move**.
8. Track progress with the **Notifications** bell at the top right. You'll see "Moving resources" and then "Moving resources complete".

**Method 2: From the Function App itself**
1. Open the **Function App** and go to **Overview**.
2. Next to **Resource group**, click **(move)**.
3. Select the target RG. The portal suggests adding related resources such as the plan; include them.
4. Continue with steps 5 to 8 above.

---

## Slide 13: Verify the Move

**Confirm the location of the resources:**

```bash
az resource list -g $DST_RG --query "[].{Name:name, Type:type}" -o table
az resource list -g $SRC_RG --query "[].{Name:name, Type:type}" -o table
```

**Confirm the app is healthy:**

```bash
az functionapp show -g $DST_RG -n $FUNC_APP \
  --query "{State:state, Host:defaultHostName, RG:resourceGroup, Plan:serverFarmId}" -o table

az functionapp function list -g $DST_RG -n $FUNC_APP -o table
```

**Compare settings with the backup:**

```bash
az functionapp config appsettings list -g $DST_RG -n $FUNC_APP -o json > appsettings-after.json
diff <(jq -S . appsettings-backup.json) <(jq -S . appsettings-after.json)
```

**Confirm the managed identity is unchanged.** The principal ID should match `identity-backup.json`:

```bash
az functionapp identity show -g $DST_RG -n $FUNC_APP --query principalId -o tsv
```

**Smoke test the functions:**
- **HTTP triggers:** `curl https://<app>.azurewebsites.net/api/<function>?code=<key>`
- **Timer, queue, and blob triggers:** check executions in **Monitor** or in **Application Insights**, then **Live Metrics**.
- **Portal:** open the **Function App**, then **Functions**, select a function, and use **Code + Test** followed by **Test/Run**.

**Check the Activity Log:** in the portal, go to **Monitor**, then **Activity log**, and filter on "Move resources" to confirm success.

---

## Slide 14: Post-Move Remediation (Things That Do Not Move Automatically)

**1. RBAC role assignments scoped directly to the moved resources**
Assignments made directly on the Function App or plan are **not** carried over. They become orphaned. Re-create them from `rbac-backup.json`:

```bash
NEW_FUNC_ID=$(az functionapp show -g $DST_RG -n $FUNC_APP --query id -o tsv)
az role assignment create --assignee "<PRINCIPAL_ID>" --role "<ROLE_NAME>" --scope $NEW_FUNC_ID
```

Also review assignments that were inherited from the **source RG**. They no longer apply, so grant equivalents on the **target RG** if needed. Clean up orphaned assignments under **Access control (IAM)**, where they show as "Identity not found" or "Unknown".

**2. Diagnostic settings**
Verify that they still exist. If any are missing, re-create them:

```bash
az monitor diagnostic-settings list --resource $NEW_FUNC_ID -o table
```

**3. Alert rules and action groups**
Alerts scoped to the old resource ID may stop firing. In the portal, go to **Monitor**, then **Alerts**, then **Alert rules**, edit each affected rule, and re-select the resource in the new RG.

**4. TLS certificates (if you deleted them in Slide 9)**

```bash
az functionapp config ssl upload -g $DST_RG -n $FUNC_APP \
  --certificate-file cert.pfx --certificate-password "<pwd>"
az functionapp config ssl bind -g $DST_RG -n $FUNC_APP \
  --certificate-thumbprint <THUMBPRINT> --ssl-type SNI
```

**5. Key Vault references and access**
These usually keep working because the identity is unchanged. If Key Vault access was granted via RBAC **at the source RG scope**, re-grant it on the vault or the target RG.

**6. Networking**
Verify VNet integration, private endpoints, and access restrictions under **Networking** in the Function App.

---

## Slide 15: Update Automation, IaC, and Re-Apply Locks

**CI/CD pipelines**
- **Azure DevOps:** update the `resourceGroupName` input in `AzureFunctionApp@2` tasks and in service connection scopes if they are RG-scoped.
- **GitHub Actions:** update any `az` commands that use `-g`. Publish-profile deployments keep working, but re-download the profile if you're unsure.
- **Service principals:** if their RBAC was scoped to the old RG, grant equivalent roles on the new RG.

**Infrastructure as Code**
- **Bicep/ARM:** change the target RG in deployment commands and parameter files.
- **Terraform:** the resource ID changed, so update the state:

```bash
# Update resource_group_name in the .tf files first, then:
terraform state rm azurerm_linux_function_app.func
terraform import azurerm_linux_function_app.func "/subscriptions/<sub>/resourceGroups/rg-target/providers/Microsoft.Web/sites/my-function-app"
# Repeat for the service plan, App Insights, etc.
terraform plan   # should show no changes
```

**Scripts, dashboards, and workbooks:** search for the old RG name or the old resource IDs and update them.

**Re-create locks:**

```bash
az lock create --name "<LOCK_NAME>" --lock-type CanNotDelete --resource-group $DST_RG
az lock create --name "<LOCK_NAME>" --lock-type CanNotDelete --resource-group $SRC_RG
```

---

## Slide 16: Clean Up the Source Resource Group

1. Confirm that nothing in the source RG still depends on the moved resources.
2. If the source RG is now empty and no longer needed:

```bash
az resource list -g $SRC_RG -o table   # should be empty
az group delete --name $SRC_RG --yes --no-wait
```

3. **Portal:** open the **source RG**, click **Delete resource group**, type the RG name to confirm, and click **Delete**.

> Wait at least a day or two of normal operation before deleting the old RG, in case you need to roll back.

---

## Slide 17: Troubleshooting Common Errors

| Error / Symptom | Cause | Fix |
|---|---|---|
| `ResourceMoveValidationFailed`: server farm/plan must move | The app and its plan are in different move batches | Include the plan **and every app on it** in the same move |
| `ScopeLocked` | A lock exists on the source/target RG or a resource | Remove the locks (Slide 7) |
| `RequestDisallowedByPolicy` | An Azure Policy on the target denies the resources | Add the required tags or fix the policy, then re-validate |
| Certificate-related validation failure | Uploaded or managed certificate in the move set | Follow Slide 9 |
| `AuthorizationFailed` | Missing move/write permissions | Get Contributor on both RGs |
| Move hangs or times out | Long-running operation | Wait up to 4 hours; check the Activity Log; don't retry mid-operation |
| App can't read Key Vault or Storage after the move | RBAC was granted at the old RG scope | Re-grant at the resource or new RG scope |
| Alerts stopped firing | The alert is scoped to the old resource ID | Re-target the alert rule |
| Validation fails for an ASE-hosted app | ASE apps aren't movable | Redeploy into the target RG instead |
| Hosting plan type isn't supported for move | Some newer plan SKUs may have restrictions | Check the Microsoft "Move operation support for resources" doc; fall back to recreate and redeploy |

---

## Slide 18: Rollback Plan

The move is reversible. If issues arise:

```bash
az resource move --destination-group $SRC_RG \
  --ids $(az functionapp show -g $DST_RG -n $FUNC_APP --query id -o tsv) \
        $(az functionapp show -g $DST_RG -n $FUNC_APP --query serverFarmId -o tsv)
```

Then reverse any RBAC, alert, and pipeline changes you made.

**Last-resort alternative (recreate instead of move):**
1. Deploy a new plan and Function App in the target RG from the exported template.
2. Restore the app settings from `appsettings-backup.json`.
3. Redeploy the code from CI/CD.
4. Switch over custom domains and DNS.
5. Decommission the old app.

> **Note:** a new app gets a **new** managed identity and a new default hostname if the name changes.

---

## Slide 19: One-Page Quick Reference (End-to-End Script)

```bash
#!/bin/bash
set -e
SUB_ID=$(az account show --query id -o tsv)
SRC_RG="rg-source"; DST_RG="rg-target"; FUNC_APP="my-function-app"

# 1. Backup
FUNC_ID=$(az functionapp show -g $SRC_RG -n $FUNC_APP --query id -o tsv)
PLAN_ID=$(az functionapp show -g $SRC_RG -n $FUNC_APP --query serverFarmId -o tsv)
az functionapp config appsettings list -g $SRC_RG -n $FUNC_APP > appsettings-backup.json
az role assignment list --scope $FUNC_ID > rbac-backup.json

# 2. Ensure the target RG exists
az group show -n $DST_RG >/dev/null 2>&1 || az group create -n $DST_RG -l "<region>"

# 3. Validate
az rest --method post \
  --uri "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$SRC_RG/validateMoveResources?api-version=2021-04-01" \
  --body "{\"resources\":[\"$FUNC_ID\",\"$PLAN_ID\"],\"targetResourceGroup\":\"/subscriptions/$SUB_ID/resourceGroups/$DST_RG\"}"

# 4. Move
az resource move --destination-group $DST_RG --ids $FUNC_ID $PLAN_ID

# 5. Verify
az functionapp show -g $DST_RG -n $FUNC_APP --query "{state:state,rg:resourceGroup}" -o table
```

**Final checklist:**

- [ ] Locks removed and re-applied
- [ ] Validation passed
- [ ] Move succeeded
- [ ] Functions tested
- [ ] RBAC re-created
- [ ] Alerts and diagnostics checked
- [ ] Certificates restored
- [ ] Pipelines and IaC updated
- [ ] Source RG cleaned up
