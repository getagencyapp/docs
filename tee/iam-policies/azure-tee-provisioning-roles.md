# Azure TEE Provisioning - Required RBAC Roles

This document describes the Azure RBAC (Role-Based Access Control) roles required for TEE provisioning.

## Overview

Azure uses RBAC roles rather than JSON policies like AWS. You'll need to assign these roles to a Service Principal or Managed Identity.

Ideally you'll want to start by creating a Resource Group called "Agency" and then either assigning the appropriate roles, or creating a custom role as defined below.

We then create the Service Principal as a Registered App to get the "Tenant ID", "Client ID" and "Client Secret". We can then assign the roles to the Registered App through the bash command interface in the Azure Portal. We used Option 2 for testing.

---

## Option 1: Use Built-in Roles (Recommended for Simplicity)

Assign these built-in roles at the **Resource Group** scope:

| Role | Purpose |
|------|---------|
| **Virtual Machine Contributor** | Create, manage, delete Confidential VMs |
| **Network Contributor** | Create VNets, NSGs, Public IPs |
| **Storage Blob Data Reader** | Read from storage buckets (for direct ingestion) |

### Azure CLI Commands

```bash
# Variables
SUBSCRIPTION_ID="your-subscription-id"
RESOURCE_GROUP="agency-tee-rg"
SERVICE_PRINCIPAL_ID="your-sp-object-id"

# Assign roles
az role assignment create \
    --assignee "$SERVICE_PRINCIPAL_ID" \
    --role "Virtual Machine Contributor" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

az role assignment create \
    --assignee "$SERVICE_PRINCIPAL_ID" \
    --role "Network Contributor" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

# Optional: For direct ingestion from Azure Blob Storage
az role assignment create \
    --assignee "$SERVICE_PRINCIPAL_ID" \
    --role "Storage Blob Data Reader" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
```

---

## Option 2: Custom Role (Least Privilege)

For maximum security, create a custom role with only the required permissions:

### Custom Role Definition

```json
{
    "properties": {
        "roleName": "Agency TEE Provisioner",
        "description": "Custom role for Agency TEE provisioning with least privilege",
        "type": "CustomRole",
        "permissions": [
            {
                "actions": [
                    "Microsoft.Compute/virtualMachines/read",
                    "Microsoft.Compute/virtualMachines/write",
                    "Microsoft.Compute/virtualMachines/delete",
                    "Microsoft.Compute/virtualMachines/start/action",
                    "Microsoft.Compute/virtualMachines/powerOff/action",
                    "Microsoft.Compute/virtualMachines/deallocate/action",
                    "Microsoft.Compute/virtualMachines/restart/action",
                    "Microsoft.Compute/disks/read",
                    "Microsoft.Compute/disks/write",
                    "Microsoft.Compute/disks/delete",
                    "Microsoft.Compute/snapshots/read",
                    "Microsoft.Compute/snapshots/write",
                    "Microsoft.Compute/snapshots/delete",
                    "Microsoft.Network/virtualNetworks/read",
                    "Microsoft.Network/virtualNetworks/write",
                    "Microsoft.Network/virtualNetworks/subnets/read",
                    "Microsoft.Network/virtualNetworks/subnets/write",
                    "Microsoft.Network/virtualNetworks/subnets/join/action",
                    "Microsoft.Network/networkSecurityGroups/read",
                    "Microsoft.Network/networkSecurityGroups/write",
                    "Microsoft.Network/networkSecurityGroups/delete",
                    "Microsoft.Network/networkSecurityGroups/join/action",
                    "Microsoft.Network/networkInterfaces/read",
                    "Microsoft.Network/networkInterfaces/write",
                    "Microsoft.Network/networkInterfaces/delete",
                    "Microsoft.Network/networkInterfaces/join/action",
                    "Microsoft.Network/publicIPAddresses/read",
                    "Microsoft.Network/publicIPAddresses/write",
                    "Microsoft.Network/publicIPAddresses/delete",
                    "Microsoft.Network/publicIPAddresses/join/action",
                    "Microsoft.Authorization/roleAssignments/write",
                    "Microsoft.Resources/subscriptions/resourceGroups/read"
                ],
                "notActions": [],
                "dataActions": [
                    "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read"
                ],
                "notDataActions": []
            }
        ],
        "assignableScopes": [
            "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/agency-tee-rg"
        ]
    }
}
```

### Create Custom Role via CLI

```bash
# Save the JSON above to agency-tee-role.json, then:
az role definition create --role-definition agency-tee-role.json

# Assign to service principal
az role assignment create \
    --assignee "$SERVICE_PRINCIPAL_APP_ID" \
    --role "Agency TEE Provisioner" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
```

---

## Required Credentials for Agency App

When configuring Azure in the Agency app, you'll need:

| Field | Description | How to Get |
|-------|-------------|------------|
| **Subscription ID** | Azure subscription identifier | Azure Portal → Subscriptions |
| **Tenant ID** | Azure AD tenant identifier | Azure Portal → Azure Active Directory → Overview |
| **Client ID** | Service Principal application ID | Create via `az ad sp create-for-rbac` |
| **Client Secret** | Service Principal password | Created with service principal |

### Create Service Principal

```bash
az ad sp create-for-rbac \
    --name "agency-tee-provisioner" \
    --role "Virtual Machine Contributor" \
    --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
```

Output contains `appId` (Client ID) and `password` (Client Secret).

---

## Confidential VM Requirements

For SEV-SNP Confidential VMs, additional considerations:

| Requirement | Details |
|-------------|---------|
| **VM Size** | Must be DCas_v5 or DCads_v5 series |
| **Regions** | East US, West Europe, and other [supported regions](https://docs.microsoft.com/azure/confidential-computing/confidential-vm-overview#regions) |
| **Image** | Must use a confidential computing compatible OS image |

---

## Verification

Test your credentials:

```bash
# Login with service principal
az login --service-principal \
    --username "$CLIENT_ID" \
    --password "$CLIENT_SECRET" \
    --tenant "$TENANT_ID"

# Verify access
az vm list --resource-group "$RESOURCE_GROUP" -o table

# Check Confidential VM availability
az vm list-skus --location eastus --size Standard_DC --output table
```
