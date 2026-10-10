# Microsoft Foundry on customer-owned AKS

> [!IMPORTANT]
> This repository is a pre-production demo preview for product review. It is
> maintained in the style of an official Microsoft Foundry and Azure Developer
> CLI (`azd`) template, but it is not approved for production use or public
> distribution.

This template creates a new Microsoft Foundry account and project and connects
the account to customer-owned Azure Kubernetes Service (AKS) infrastructure.
The referenced AKS cluster, agent subnet, user-assigned managed identities, and
storage account remain customer-owned.

## What `azd provision` does

With the demo build of the `azure.ai.projects` extension installed,
`azd provision`:

1. Runs a read-only BYO AKS qualification preflight.
2. Reuses existing qualifying role assignments and creates only the missing
   assignments required by the public-preview contract.
3. Reuses an existing `Microsoft.App/environments` agent-subnet delegation or
   adds it when the subnet has no delegation.
4. Creates a new Microsoft Foundry resource group, account, and project only
   after the preflight succeeds.

The command does not create or redesign the customer's AKS cluster, virtual
network, managed identities, or storage account.

## Existing resources required

Before using this template, prepare:

- An AKS cluster with Azure CNI, OIDC issuer, workload identity, and Azure RBAC
  enabled.
- A node pool that has `workloadRuntime=KataVmIsolation` and the exact label
  `agents.ai.azure.com/sandbox-node=true` on the same pool.
- A dedicated agent subnet that is separate from every AKS node and pod subnet.
  For this demo, it must be in the same virtual network as the AKS node
  subnets.
- A hosting-management user-assigned managed identity.
- A different workload user-assigned managed identity.
- A storage account in the same subscription and region as AKS and Foundry.

The agent subnet must have a supported IPv4 prefix beginning with `10`, `172`,
or `192`. It may have no delegation or an existing
`Microsoft.App/environments` delegation, but not a conflicting service
delegation.

## Demo prerequisites

- The demo build of `azd` used by the Foundry/azd engineering team.
- The private `azure.ai.projects` extension bundle version
  `1.0.0-beta.9`, installed before initialization or provisioning.
- An Azure identity authorized to read all referenced resources, create the
  Foundry resource group and its resources, manage the required role
  assignments at their target scopes, and update the agent subnet delegation.

The extension bundle is intentionally not included in this repository.

## Initialize

Create an empty directory and initialize it from this private template:

```powershell
mkdir my-foundry-byoaks
Set-Location my-foundry-byoaks
azd init `
  -t https://github.com/yiliuTo/azd-foundry-byoaks-template `
  -e <environment-name> `
  .
```

Access to the private GitHub repository is required.

`azd init` copies the template, creates the named azd environment, and selects
it as the current environment. If `-e` is omitted in interactive mode, azd
prompts for an environment name.

## Bind existing Azure resources

Store references to the customer-owned resources in the environment created by
`azd init`:

```powershell
azd env set AZURE_AKS_CLUSTER_ID <aks-resource-id>
azd env set AZURE_AGENT_SUBNET_ID <agent-subnet-resource-id>
azd env set AZURE_HOSTING_MANAGEMENT_IDENTITY_ID <hosting-uami-resource-id>
azd env set AZURE_HOSTED_AGENTS_WORKLOAD_IDENTITY_ID <workload-uami-resource-id>
azd env set AZURE_HOSTED_AGENTS_STORAGE_ID <storage-account-resource-id>
```

These values are deployment inputs. Do not commit concrete customer resource
IDs to this template.

`azd provision` prompts for the Azure subscription and location when they are
not already set and persists the selections in the azd environment. Select the
subscription containing the AKS cluster and select the AKS region.

By default, the Foundry resource group is named `rg-<environment-name>`. To use
an explicit dedicated resource group, set it before provisioning:

```powershell
azd env set AZURE_RESOURCE_GROUP <foundry-resource-group-name>
```

Do not create the Foundry resource group in advance. The Foundry provisioning
deployment creates it after BYO AKS preparation succeeds.

## Provision

```powershell
azd provision
```

The command first reads and validates all existing inputs. If the preflight
fails, it reports the failed property and remediation and stops before creating
role assignments, changing the subnet, or creating Foundry resources.

After remediation, rerun the same command. A successful run displays whether
each required role assignment and the subnet delegation will be reused or
created before it submits the Foundry deployment.

## Required role assignments

The public-preview contract uses these four assignments:

| Identity | Scope | Role |
|---|---|---|
| Hosting-management UAMI | AKS resource | Reader |
| Hosting-management UAMI | AKS resource | Kubernetes Extension Contributor |
| Hosting-management UAMI | Workload UAMI resource | Managed Identity Federated Identity Credential Contributor |
| Workload UAMI | Storage account resource | Storage Blob Data Contributor |

Existing unconditional assignments at the target scope or an inherited parent
scope are reused. The demo does not remove any customer-owned role assignment.

## Demo boundaries

- Only creation of a new Foundry account is supported. `agentHosting` cannot be
  combined with an existing-project `endpoint`.
- Automatic BYO AKS preparation is supported only by the embedded
  `microsoft.foundry` provider. Ejected Bicep and Terraform are outside this
  demo.
- The demo does not add the sandbox label, create a Kata node pool, change AKS
  networking, replace a conflicting subnet delegation, or verify workload
  data-plane connectivity to storage.
- The template contains no concrete Azure resource IDs, credentials, tenant
  information, subscription IDs, or resource-group names.
