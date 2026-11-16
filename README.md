# CrowdStrike IOA Activity Log Diagnostic Settings Policy

Deploys the CrowdStrike IOA Azure Policy, which streams each subscription's Activity Log
to an Event Hub for CrowdStrike Falcon Cloud Security Indicator of Attack (IOA)
assessment.

The policy is **"Activity Logs must be sent to CrowdStrike for IOA assessment"**, taken
unchanged from
[CrowdStrike/cs-azure-integration-bicep › policies/cs-ioa-policy](https://github.com/CrowdStrike/cs-azure-integration-bicep/tree/main/policies/cs-ioa-policy).
For every subscription in scope, it checks for a diagnostic setting that sends all eight
Activity Log categories to your Event Hub. If one is missing, the policy creates it:

`Administrative`, `Security`, `ServiceHealth`, `Alert`, `Recommendation`, `Policy`,
`Autoscale`, `ResourceHealth`

## Deploy

Pick one scope.

| Scope | Use when | Deploy |
|---|---|---|
| **Management group** | You want every subscription under a management group covered, including subscriptions added later | [![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmikedzikowski%2Fcs-azure-policy-diagnotic-settings%2Fmain%2FmanagementGroup.json) |
| **Subscription** | You want to cover one subscription | [![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmikedzikowski%2Fcs-azure-policy-diagnotic-settings%2Fmain%2Fsubscription.json) |

Each deployment creates:

1. The policy definition, stored at the selected management group or subscription
2. The policy assignment `cs-ioa-assignment` (display name *CrowdStrike IOA*), with a
   system-assigned managed identity
3. Role assignments at the selected scope for that identity, one for each role listed in
   the policy's `roleDefinitionIds`:
   - Monitoring Contributor
   - Lab Services Reader
   - Azure Event Hubs Data Owner
4. An **Azure Event Hubs Data Owner** role assignment for that identity on the Event Hub
   authorization rule. Remediation needs this to read the rule's keys, even when the
   Event Hub is in a different subscription
5. *Subscription scope only:* a remediation task that creates the diagnostic setting on
   the subscription (see [Remediate existing subscriptions](#remediate-existing-subscriptions))

The definition and assignment names match CrowdStrike's own Bicep module
(`modules/ioa/activityLogPolicy.bicep`). Deploying to a scope where that module already
ran updates the existing definition and assignment instead of creating duplicates.

## Prerequisites

**An Event Hub to receive the logs.** Create these before you deploy, or reuse the ones
from your CrowdStrike Azure registration:

- An Event Hubs namespace (Standard tier or higher)
- An event hub in that namespace
- A **namespace-level** shared access policy (authorization rule) with **Send** rights

**Permissions for the person deploying:**

| Where | Role |
|---|---|
| Target management group or subscription | **Owner**, or **Resource Policy Contributor** plus **User Access Administrator** |
| Resource group that contains the Event Hubs namespace | **Owner**, or **User Access Administrator**, plus rights to create deployments, e.g. **Contributor** |

**Room for one more diagnostic setting.** Azure allows at most 5 diagnostic settings per
subscription. Remediation fails on any subscription that already has 5. To check a
subscription:

```bash
az monitor diagnostic-settings subscription list --subscription <SUBSCRIPTION_ID> -o table
```

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| **Region** / `location` | Yes | — | Region where deployment metadata and the assignment's managed identity are stored |
| `eventHubAuthorizationRuleId` | Yes | — | Full resource ID of the namespace authorization rule, e.g. `/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-cs-eventhub/providers/Microsoft.EventHub/namespaces/ns-cs-activity/authorizationRules/cs-send` |
| `eventHubName` | Yes | — | The event hub's **name only**, e.g. `cs-activity-hub`, not a resource ID |
| `diagnosticSettingName` | No | `cs-monitor-activity-to-eventhub` | Name of the diagnostic setting created on each subscription |
| `policyEffect` | No | `DeployIfNotExists` | Set to `Disabled` to turn the policy off without removing it |
| `createRemediationTask` | No | `true` | *Subscription scope only.* Starts a remediation task after the assignment is created |

To find the authorization rule ID:

```bash
az eventhubs namespace authorization-rule show \
  --resource-group <RESOURCE_GROUP> --namespace-name <NAMESPACE> --name <RULE_NAME> \
  --query id -o tsv
```

> **Note:** The upstream policy stores its remediation deployments in `westeurope`.
> This is deployment metadata only. Your Activity Log data flows directly to the Event
> Hub you specify.

## Remediate existing subscriptions

A `DeployIfNotExists` policy creates the diagnostic setting automatically on subscriptions
**created or updated after** it's assigned. Subscriptions that already exist are only
fixed by a remediation task.

**Subscription scope:** the template starts a remediation task that runs a compliance scan
first, so there's nothing more to do. If it fails because the role assignments haven't
propagated yet, re-run it from **Policy → Remediation** after a few minutes.

**Management group scope:** Azure can't start a scan-first remediation task at management
group scope, so the template doesn't create one. Once the first compliance scan completes,
usually within 30 minutes:

1. In the Azure portal, open **Policy → Remediation**
2. Select the **CrowdStrike IOA** assignment
3. Select **Remediate**

Or with the Azure CLI:

```bash
MG="<MANAGEMENT_GROUP_ID>"

az policy remediation create \
  --name cs-ioa-remediation \
  --management-group "$MG" \
  --policy-assignment "/providers/Microsoft.Management/managementGroups/$MG/providers/Microsoft.Authorization/policyAssignments/cs-ioa-assignment"
```

## Deploy with the Azure CLI

The same templates deploy without the portal:

```bash
# Management group
az deployment mg create \
  --management-group-id <MANAGEMENT_GROUP_ID> \
  --location eastus \
  --template-uri https://raw.githubusercontent.com/mikedzikowski/cs-azure-policy-diagnotic-settings/main/managementGroup.json \
  --parameters eventHubAuthorizationRuleId="<RULE_ID>" eventHubName="<EVENT_HUB_NAME>"

# Subscription
az deployment sub create \
  --subscription <SUBSCRIPTION_ID> \
  --location eastus \
  --template-uri https://raw.githubusercontent.com/mikedzikowski/cs-azure-policy-diagnotic-settings/main/subscription.json \
  --parameters eventHubAuthorizationRuleId="<RULE_ID>" eventHubName="<EVENT_HUB_NAME>"
```

## Verify

```bash
# Compliance state for each subscription in scope
az policy state list --filter "policyAssignmentName eq 'cs-ioa-assignment'" \
  --query "[].{subscription:subscriptionId, state:complianceState}" -o table

# The diagnostic setting on a subscription
az monitor diagnostic-settings subscription list --subscription <SUBSCRIPTION_ID> \
  --query "value[].{name:name, eventHub:eventHubName}" -o table
```

## Before you deploy

- **Look for an existing assignment.** CrowdStrike's Azure registration Bicep
  (`cs-azure-integration-bicep`) may already have assigned this policy, or a similar one.
  Two assignments that write diagnostic settings for the same Event Hub can conflict.
  Check first:
  ```bash
  az policy assignment list --disable-scope-strict-match \
    --query "[?contains(displayName, 'CrowdStrike')].{name:name, scope:scope}" -o table
  ```
- **Assign at one scope only.** Assigning the policy at both a management group and a
  subscription beneath it creates two identities that manage the same diagnostic setting.

## Remove

```bash
SCOPE="/providers/Microsoft.Management/managementGroups/<MANAGEMENT_GROUP_ID>"   # or /subscriptions/<SUBSCRIPTION_ID>

PRINCIPAL_ID=$(az policy assignment show --name cs-ioa-assignment --scope "$SCOPE" \
  --query identity.principalId -o tsv)
DEFINITION_ID=$(az policy assignment show --name cs-ioa-assignment --scope "$SCOPE" \
  --query policyDefinitionId -o tsv)

# Remove the identity's role assignments, including the one on the Event Hub
az role assignment list --assignee "$PRINCIPAL_ID" --all --query "[].id" -o tsv \
  | xargs -n1 az role assignment delete --ids

az policy assignment delete --name cs-ioa-assignment --scope "$SCOPE"
az rest --method delete --url "https://management.azure.com${DEFINITION_ID}?api-version=2023-04-01"
```

Removing the policy **doesn't** delete diagnostic settings it already created. To delete
one:

```bash
az monitor diagnostic-settings subscription delete \
  --name cs-monitor-activity-to-eventhub --subscription <SUBSCRIPTION_ID> --yes
```

## Files

| File | Purpose |
|---|---|
| `managementGroup.json` | ARM template for management group scope |
| `subscription.json` | ARM template for subscription scope |

Both templates embed `cs-ioa-policy.json` from
[CrowdStrike/cs-azure-integration-bicep](https://github.com/CrowdStrike/cs-azure-integration-bicep/tree/main/policies/cs-ioa-policy)
without changes.
