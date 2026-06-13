Secure Access Delegation & Least Privilege Storage Lab

🎯 Project Overview

This hands-on lab demonstrates modern, identity-centric cloud security by exploring and comparing the three primary methods of delegating access to data resources within Microsoft Azure.

The objective of this project was to transition a sandbox environment from insecure legacy shared keys to time-bound delegated signatures, and ultimately to enterprise-grade Role-Based Access Control (RBAC) integrated with Azure Active Directory / Microsoft Entra ID.

🏗️ Architecture & IAM Security Paradigms

               ┌──────────────────────────────────────────────────┐
               │        Private Azure Blob Container (Private)    │
               └────────────────────────┬─────────────────────────┘
                                        │
             ┌──────────────────────────┼──────────────────────────┐
             ▼                          ▼                          ▼
     [ SAS Token ]              [ Access Key 1/2 ]        [ Storage Blob Reader ]
   - Restricted Time          - Absolute Power          - Azure Active Directory
   - Restricted IPs           - Legacy / High Risk      - Fully Audited IAM
   - "Temporary Access"       - "Unsecure"              - "Best Practice"


Shared Access Signatures (SAS - Temporary Delegation): Time-bound, granular delegated access. Generates a token with an explicit expiration window and specific permission scopes (e.g., Read-Only).

Shared Access Keys (Legacy / High Risk): Provides unrestricted root access to the entire storage account. If leaked, an attacker gains complete control plane and data plane power over the resource.

Role-Based Access Control (RBAC - Modern Best Practice): Completely keyless access. Permissions are assigned directly to an Entra ID identity, managed via security principles, and fully audited.

🛠️ Tech Stack & Environment

Operating System: Ubuntu 26.04 LTS

Command Line Interface: Bash / Azure CLI (az cli)

Cloud Platform: Microsoft Azure (University Managed Tenant)

Identity Provider: Microsoft Entra ID (Azure Active Directory)

💻 Technical Implementation Steps

Step 1: Secure Storage Provisioning

We provisioned a standard locally redundant storage (LRS) workspace with public network access strictly disabled to align with zero-trust network boundary defense:

# Create the resource group
az group create --name rg-iam-storage-lab --location eastus

# Provision the locked-down Storage Account
az storage account create \
  --name stiamlab11758 \
  --resource-group rg-iam-storage-lab \
  --location eastus \
  --sku Standard_LRS \
  --allow-blob-public-access false


Step 2: Data Ingestion & Container Locking

We created a private container named finance-records and ingested a mock confidential ledger:

# Create the secure container
az storage container create \
  --name finance-records \
  --connection-string "..."

# Upload the sensitive text file
az storage blob upload \
  --container-name finance-records \
  --file ~/projects/payroll-ledger.txt \
  --name payroll-ledger.txt \
  --connection-string "..."


🔐 IAM Authentication Analysis & Verification

Method A: Scoped Shared Access Signatures (SAS)

We generated a time-bound, read-only SAS token expiring in a restricted window to securely delegate temporary access to a single file:

EXPIRY="2026-06-14T23:59:59Z"

SAS_TOKEN=$(az storage blob generate-sas \
  --container-name finance-records \
  --name payroll-ledger.txt \
  --account-name stiamlab11758 \
  --permissions r \
  --expiry $EXPIRY \
  --connection-string "..." \
  --output tsv)

# Build the temporary, secure URL
echo "https://stiamlab11758.blob.core.windows.net/finance-records/payroll-ledger.txt?${SAS_TOKEN}"


Verification Output: Pasting the resulting SAS URL into an external browser successfully allowed download of the raw private text file, confirming operational delegation boundaries.

Method B: Shared Key Extraction (Legacy)

We retrieved the primary account key to verify legacy configuration support:

az storage account keys list \
  --account-name stiamlab11758 \
  --resource-group rg-iam-storage-lab \
  --query "[0].value" \
  --output tsv


IAM Security Analysis: While straightforward, shared keys represent a legacy security vulnerability. They cannot be restricted to specific files, cannot be IP-restricted, and rotating a compromised key immediately breaks all integrated applications.

Method C: Enterprise Role-Based Access Control (RBAC)

We assigned the industry-standard Storage Blob Data Reader role directly to our Entra ID security principal, allowing secure access without managing secret keys or tokens:

# Retrieve the logged-in user principal name
USER_EMAIL=$(az ad signed-in-user show --query userPrincipalName --output tsv)

# Apply the RBAC role assignment
az role assignment create \
  --assignee $USER_EMAIL \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/fdf50646-2655-4838-8045-0ffa921638df/resourceGroups/rg-iam-storage-lab/providers/Microsoft.Storage/storageAccounts/stiamlab11758"


IAM Security Analysis: This represents true modern identity management. Access tokens are negotiated automatically on the fly via Active Directory. No keys or static tokens are embedded in applications, dramatically reducing secret exposure and aligning with the core Principle of Least Privilege.

🔍 Verification Evidence

Method B Verification: Time-Bound SAS Access

Below is verification of the browser successfully accessing the secure data container using only the scoped SAS token:

Method C Verification: Keyless RBAC Handshake

Below is the successful JSON response payload from Azure confirming that the Storage Blob Data Reader RBAC role mapping is fully active:
