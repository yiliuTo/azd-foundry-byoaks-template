# Microsoft Foundry on customer-owned AKS

> [!IMPORTANT]
> This repository is a pre-production demo preview for product review. It is
> maintained in the style of an official Microsoft Foundry and Azure Developer
> CLI (`azd`) template, but it is not approved for production use or public
> distribution.

> [!WARNING]
> The default `azure.yaml` expresses the approved BYO-AKS onboarding design
> v0.10, not a working AKS-only path in the currently inspected runtime.
> Team integration `a85d6b1e9e26f1d9a3f538132799ceccd3a2bfd1` still requires
> all seven `agentHosting` fields in both its schema and synthesis.
> A compatible `azure.ai.projects` release implementing simplified input
> validation and resource preparation is required; its minimum version is
> not yet known. The existing `1.0.0-beta.9` constraint is the historical
> demo minimum and does **not** guarantee simplified-onboarding support.
> To retain the original fully specified demo contract, use the
> [advanced example](examples/azure.full.yaml) and the instructions below.

This is a **provision-only** sample for a new Microsoft Foundry account and
project connected to customer-owned Azure Kubernetes Service (AKS). It contains
no agent application source or deployable agent service. Template changes do
not implement extension preflight, automatic preparation, or agent deployment.

## Default inputs: AKS only

The default service configuration intentionally contains only the required
pre-existing AKS resource reference:

```yaml
services:
  project:
    host: azure.ai.project
    agentHosting:
      clusterResourceId: ${AZURE_AKS_CLUSTER_ID}
```

Storage, ACR, the agent subnet, and user-assigned managed identities (UAMIs)
are optional reuse inputs in design v0.10. They are omitted from the default,
not represented by empty environment-variable placeholders. A compatible
extension must resolve the hosting type/name defaults and prepare missing
supporting resources, identities, scoped grants, and configuration internally
during `azd provision`. It does not create or redesign the existing AKS cluster.

Review the [pinned prerequisite reference](https://github.com/taoxu0903/byo-aks-foundry-sample-agent/blob/4e4f1a55ae42006e6739ec16cacd19e4e3cc9b35/onboarding-guide/prerequisites.md)
for resource, access, and AKS requirements. In v0.10, Storage and ACR need not
already exist; supplied resources must still meet that reference's requirements.
Install the engineering team's supported `azd` build and extension bundle
before initialization. The private extension bundle is not included here.

## Initialize

Create an empty directory and initialize it from the template:

```powershell
New-Item -ItemType Directory my-foundry-byoaks
Set-Location my-foundry-byoaks
azd init `
  -t https://github.com/yiliuTo/azd-foundry-byoaks-template `
  -e <environment-name> `
  .
```

Repository access is required. The final positional `.` selects the current
directory. The historical `v0.1.0-demo` tag pins the earlier, fully specified
template; it does not select this minimal configuration.

`azd init` copies the template, creates the named azd environment, and selects
it as the current environment. If `-e` is omitted in interactive mode, azd
prompts for an environment name.

Set the sole required BYO resource reference:

```powershell
azd env set AZURE_AKS_CLUSTER_ID <aks-resource-id>
```

Subscription, location, and Foundry resource-group settings remain normal
provisioning inputs, not additional mandatory pre-existing BYO resource IDs.
`azd provision` prompts for the subscription and location when unset. For this
demo, select the AKS subscription and region. The original demo defaults the
Foundry resource group to `rg-<environment-name>`; an explicit dedicated name
is optional:

```powershell
azd env set AZURE_RESOURCE_GROUP <foundry-resource-group-name>
```

Do not commit concrete customer resource IDs or credentials to the template.

## Optional resource reuse (compatible release required)

Add a field under `services.project.agentHosting` only when reusing that
resource, and set its corresponding environment value. Setting an environment
value alone does **not** opt into reuse when its YAML binding is absent.

| Optional field under `services.project.agentHosting` | Environment value |
|---|---|
| `agentSubnetResourceId` | `AZURE_AGENT_SUBNET_ID` |
| `hostingManagementIdentityResourceId` | `AZURE_HOSTING_MANAGEMENT_IDENTITY_ID` |
| `workloadIdentityResourceId` | `AZURE_HOSTED_AGENTS_WORKLOAD_IDENTITY_ID` |
| `storageAccountResourceId` | `AZURE_HOSTED_AGENTS_STORAGE_ID` |

For example, to reuse only an existing storage account, add its binding while
leaving the other optional fields absent:

```yaml
services:
  project:
    host: azure.ai.project
    agentHosting:
      clusterResourceId: ${AZURE_AKS_CLUSTER_ID}
      storageAccountResourceId: ${AZURE_HOSTED_AGENTS_STORAGE_ID}
```

Then set that optional value:

```powershell
azd env set AZURE_HOSTED_AGENTS_STORAGE_ID <storage-account-resource-id>
```

This partial reuse example also requires simplified-onboarding support; the
currently inspected seven-field runtime rejects it. The full example below
preserves the old all-resources-supplied contract.

ACR uses ordinary registry selection, not an `agentHosting` property. For a
compatible release supporting registry reuse with a new Foundry account, the
optional existing-registry input is:

```powershell
azd env set AZURE_CONTAINER_REGISTRY_RESOURCE_ID <acr-resource-id>
```

Do not treat that setting as a proven new-account reuse path in the current
demo: the inspected runtime resolves registry-reuse inputs in its
existing-project branch, and derives ACR creation from hosted build-from-source
agents. This project-only template has no such agent, so it does not currently
trigger automatic ACR creation that way. Both optional ACR preparation and
new-account reuse need support in the compatible extension.

If adding service network configuration, put it at
`services.project.network`, alongside `agentHosting`, never inside it. This
template does not set a service network override.

## Provisioning contract in design v0.10

With a compatible extension, run:

```powershell
azd provision
```

The intended read-only preflight checks four groups before preparation:

1. The AKS resource ID is supplied.
2. AKS meets section 4 of the pinned prerequisite reference, with network
   policy enabled. This includes the supported cluster/network configuration,
   OIDC and workload identity, and a qualified dedicated agent pool with
   `KataVmIsolation` and `agents.ai.azure.com/sandbox-node=true`. There is no
   node-pool selector input.
3. Any supplied Storage or ACR meets section 2 of that reference.
4. The current operator has the **effective permissions** from section 3's
   first row and the registry/access-administrator capabilities. The resource
   group, managed-identity, VNet, AKS, and role-assignment capability baseline
   remains in force; simplified inputs do not waive it. Storage/ACR creation
   permissions are needed when those resources must be created, but are
   **not** required for a qualifying supplied Storage/ACR respectively.
   Resource inspection and scoped grant capability remain required for reuse.

After preflight, the compatible extension prepares missing supporting
resources, UAMIs, scoped grants, and configuration, then provisions Foundry.
Preflight and preparation are internal to `azd provision`, not extra user
commands or template hooks. Correct failed checks and rerun the same command.
This describes the approved design, not functionality added by this template
edit or verified against a released extension.

### Hosting and registry grants

The pinned reference specifies these assignments, not broad AKS Contributor
or Kubernetes cluster-admin grants as the hosting baseline:

| Identity | Scope | Role |
|---|---|---|
| Hosting-management UAMI | AKS resource | Reader |
| Hosting-management UAMI | AKS resource | Kubernetes Extension Contributor |
| Hosting-management UAMI | Workload UAMI resource | Managed Identity Federated Identity Credential Contributor |
| Workload UAMI | Storage account resource | Storage Blob Data Contributor |
| AKS kubelet identity | ACR | AcrPull, or Container Registry Repository Reader for ABAC-enabled ACR |

Existing qualifying assignments should be reused rather than removed.
Tenant/policy administrators must ensure that policy permits Agent Identity
Blueprint federated identity credentials through Microsoft Graph; automatic
tenant Graph policy changes are not part of template preparation.

## Advanced: original fully specified demo

For the original demo contract, use the engineering team's demo `azd` build
and private `azure.ai.projects` bundle (historical minimum `1.0.0-beta.9`).
Copy the [full configuration](examples/azure.full.yaml) over the root file:

```powershell
Copy-Item .\examples\azure.full.yaml .\azure.yaml
azd env set AZURE_AKS_CLUSTER_ID <aks-resource-id>
azd env set AZURE_AGENT_SUBNET_ID <agent-subnet-resource-id>
azd env set AZURE_HOSTING_MANAGEMENT_IDENTITY_ID <hosting-uami-resource-id>
azd env set AZURE_HOSTED_AGENTS_WORKLOAD_IDENTITY_ID <workload-uami-resource-id>
azd env set AZURE_HOSTED_AGENTS_STORAGE_ID <storage-account-resource-id>
azd provision
```

This example preserves all seven fields, including `hostingType: ManagedCluster`
and `name: primary`. Only this legacy path requires every resource value in
that block. Its hosting-management and workload UAMIs must be different, and
its storage account must be in the AKS/Foundry subscription and region.

The original demo requires an existing dedicated agent subnet, separate from
all AKS node/pod subnets and in the AKS nodes' VNet. It accepts a supported IPv4
prefix beginning with `10`, `172`, or `192` and either no delegation or an
existing `Microsoft.App/environments` delegation, not a conflicting service
delegation. Also meet the pinned reference's subnet sizing and connectivity
requirements.

In that demo flow, preflight precedes missing role assignments, supported
subnet delegation, and creation of a new Foundry resource group/account/project.
Do not pre-create the Foundry resource group for this original demo path.
The example does not enable missing-resource creation, establish greenfield
ACR reuse, or demonstrate a successful Azure deployment.

## Demo boundaries

- Only creation of a new Foundry account is supported. `agentHosting` cannot be
  combined with an existing-project `endpoint`.
- The intended workflow remains `azd init` -> `azd provision` -> `azd deploy`.
  This repository stops at provisioning: add actual agent sources and their
  supported service configuration before deploying an agent. `azd deploy`
  alone cannot turn this project-only template into an agent application.
- This sample uses the embedded `microsoft.foundry` provider. Ejected Bicep
  and Terraform are outside its scope.
- The demo does not add the sandbox label, create a Kata node pool, change AKS
  networking, replace a conflicting subnet delegation, or verify workload
  data-plane connectivity to storage.
- The template contains no concrete Azure resource IDs, credentials, tenant
  information, subscription IDs, or resource-group names.
