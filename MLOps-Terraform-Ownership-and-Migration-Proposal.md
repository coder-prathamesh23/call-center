# DSLZ Terraform: Joint Ownership and Implementation Plan

**Cloud Services and Application Team | DEV, QA and PROD | Meeting review: 26 October 2026**

## 1. What we will implement

We will retain **`project-repo` as the Application Team repository**, preserving its history, and transfer platform infrastructure ownership to a **Cloud Services repository**. We will follow the AI-LZ Terraform standards, adapting the environment sequence to DSLZ’s existing **DEV → QA → PROD** model.

We will preserve deployed resource names, IDs, data and configuration. Ownership handover will be separate from network changes, resource moves and provider upgrades, following reconciliation of code with live state.

## 2. Service ownership and treatment of deployed resources

| Service | Owner after handover | How we will handle the existing deployment |
|---|---|---|
| Subscriptions, governance and provider registrations | Cloud Services | We will retain the existing three subscriptions and central governance arrangements. |
| VNet, subnets, delegations and vWAN hub connection | Cloud Services | We will extract their declarations from `stacks/data-science-lz-core`, transfer state ownership and preserve connectivity. Hub-subscription write access will remain with Cloud Services. |
| NSGs, route tables, associations, private DNS zones and VNet links | Cloud Services | Cloud Services will own these resources. We will reconcile existing deployments and bring any unmanaged resources under their Terraform ownership without recreating them. |
| Dev Center, Dev Center project and Managed DevOps Pools | Cloud Services | We will transfer the `managed-devops-pool` module/stack and its state ownership, retaining pool names, subnet placement, ADO registration, authorised consumers, images and capacity. |
| Terraform state storage, containers, backend private endpoint and RBAC | Cloud Services | Cloud Services will provision or take over and manage the state-storage account in each environment’s subscription, including its containers, private connectivity and RBAC. All DSLZ stack state files for that environment will reside there; application data storage remains separate. |
| IAM bootstrap, custom roles and deployment-identity grants | Cloud Services | `stacks/iam` is already owned by Cloud Services. We will retain that ownership and existing state; any repository relocation will be coordinated by Cloud Services. |
| MLOps storage, ACR, Application Insights and workload Key Vaults | Application Team | We will retain these core-module resources and their workload private endpoints in an application shared-services stack. |
| AML workspace, training/image-build compute and managed-network outbound rules | Application Team | We will preserve resources and identities; Cloud Services will coordinate central-service permissions and connectivity approvals. |
| Workload Log Analytics, diagnostics, action groups and ML alerts | Application Team | We will preserve monitoring/alert configuration, destinations and Application Insights associations. |
| Project-local AI Foundry | Application Team | We will retain it only in environments where required; shared Foundry and Central AML Registry will be Cloud Services responsibilities. |
| Workload RBAC | Application Team with agreed delegation | We will retain `environment-rbac` with narrowly scoped role-assignment permissions. Where delegation is not approved, Cloud Services will own those assignments through requests. |
| ADO pipelines, policies, connections, variable groups and pipeline-secret vault | Application Team, coordinated with Cloud Services | We will manage these outside application Terraform per the guide; workload Key Vaults remain separate. |

**Resource groups:** Cloud Services will own the existing mixed `rg-core-<env>-dslz-westus3` groups; DEV group ownership will transfer from core, while QA/PROD groups remain with Cloud Services through IAM. Application resources will stay in place under separate state ownership. We will restrict application permissions on mixed groups; workload-only groups can use scoped Contributor. Physical moves will be assessed separately.

**Active pools:** we will retain `mdp-<env>-dslz-westus3` and existing consumers. Current Terraform YAML uses `azumanage-flz`; we will validate any execution-pool switch separately before retiring access. Cloud Services will retain independent backend/agent recovery access.

## 3. How we will evolve the code

| Current implementation | Target implementation |
|---|---|
| `stacks/iam` | Already Cloud Services-owned; we will retain ownership and reference its resources by name, without reading its state. |
| Core networking declarations | We will transfer these to Cloud Services networking roots, with no dependency on application state. |
| `managed-devops-pool` module and stack | Cloud Services agent/backend implementation; platform references replace core remote-state reads. We will initially retain the existing state boundary where practical. |
| Remaining core services and workload endpoints | `modules/mlops-shared-services` + `stack/mlops-shared-services`; we will map address changes explicitly. |
| AML, monitoring, Foundry and environment-RBAC add-ons | Flat, same-named `modules/<name>` and `stack/<name>` pairs. |
| `modules/add-ons/ml-alerts` | `modules/early-crew-dispatch-alerts`, matching its stack for POC change detection. |
| Existing pipeline and scan | POC `pipelines/applications-pipeline.yml` and templates, retaining the IaC scan under `pipelines/templates`. |

Each application root will contain `main.tf`, `data.tf`, `variables.tf`, `outputs.tf`, `providers.tf`, `backend.tf`, `versions.tf`, a committed provider lock file and **three** `env/<env>.tfvars` files. We will explicitly configure optional services per environment; the supplied Foundry stack currently has DEV configuration only.

Modules will define resources, accept explicit inputs without defaults, apply tags and contain no provider/backend blocks or platform/shared-service lookups. Roots will perform lookups and pass IDs into modules. We will retain the existing AzureRM/AzAPI resource implementations and map Terraform address changes without replacing Azure resources.

### Resource references and delivery order

We will replace all cross-stack `terraform_remote_state` reads with **name-based data sources in root `data.tf`**, including IAM, core, AML, monitoring and RBAC dependencies. Non-secret names, resource groups and subscription context will come from `env/<env>.tfvars`.

For example, the AML root will resolve the existing subnet:

```hcl
data "azurerm_subnet" "private_endpoints" {
  name                 = var.private_endpoints_subnet_name
  virtual_network_name = var.vnet_name
  resource_group_name  = var.network_resource_group_name
}
```

We will pass its `.id` to AML and resolve other dependencies similarly, using provider aliases for cross-subscription reads. Application code will not declare/import platform resources or read platform state; Cloud Services will use request inputs without consuming application state.

Our README contract will record names, scopes, configuration, owners and readiness. Dependencies will follow **ServiceNow request → platform delivery → application PR**. Shared services will precede first consumers in separate PRs; we will validate dependants explicitly because the pipeline does not determine dependency order.

## 4. Deployment controls we will adopt

| Area | Our implementation |
|---|---|
| Three-environment promotion | `feature/* → dev → qa → prod`. We will update triggers, environment lists, `resolve-env.yml` so QA accepts DEV, onboarding defaults/policies, tfvars and documentation together. No TEST resources will be introduced. |
| PR and CI | Branch-policy PR validation (`pr: none`) will run change-record checks, promotion guard and plans only. Merge CI will produce and apply saved plans; `batch: false` and `HEAD~1..HEAD` detection will remain. |
| Reviews and approvals | We will retain guide §6.3 policies: required Project Approvers, minimum one reviewer, self-approval allowed, vote reset and resolved comments. DEV permits squash/basic merges; QA/PROD permit basic merges only. DEV validation is path-filtered with 12-hour expiry; promotion validation is unfiltered and expires on branch updates. QA/PROD applies require environment approval; all environments use exclusive locks. |
| Protected resources | We will protect pipeline changes, restrict connections/variable groups/environments to their pipeline and authorise pools only for approved consumers. We will deny policy bypass, force push and policy editing except designated break-glass administration. |
| Change and secrets | We will retain the `CHG[0-9]{6,}` reference in PRs and CI merge messages. This validates a reference, not ServiceNow approval. We will use per-environment federated connections, Key Vault-linked variable groups and `TF_VAR_*` secrets, removing unused POC SQL mappings. |
| State and permissions | We will use isolated application/platform state access, AzureRM Entra ID/OIDC authentication, blob locking and `-lock-timeout=5m`. Application access will include platform Reader, required subnet/DNS join actions and backend Blob Data Contributor; workload role assignments need additional constrained RBAC permission. |

We will set `projectName: dslz`, map existing ADO resources, preview onboarding with `-WhatIf`, and enable real mode after readiness checks. The three-environment sequence and preservation of existing pool consumers will be documented DSLZ adaptations to the POC.

### State-storage location and ownership

**We will store every DSLZ stack’s state file in a Cloud Services-managed state-storage account within that stack’s environment subscription.** All DEV stacks will use the DEV account, all QA stacks the QA account, and all PROD stacks the PROD account. Each stack will retain its own state file; we will not create a separate account for every stack.

| Environment | Location of all its DSLZ stack state files | Subscription |
|---|---|---|
| DEV | State-storage account provisioned or taken over and managed by Cloud Services in DEV | `f956ebe7-509b-41be-8dc2-6310517151c1` |
| QA | State-storage account provisioned or taken over and managed by Cloud Services in QA | `6b92cf8c-b6d5-49b0-9e4e-955730029fb4` |
| PROD | State-storage account provisioned or taken over and managed by Cloud Services in PROD | `f8c03143-1254-442a-b504-2aae5a9c3310` |

Cloud Services will own the accounts, containers, private connectivity and **all backend RBAC assignments**. Application state will use container `tfstate` and keys `dslz/<stack>/<env>.tfstate`; Cloud Services platform state will use a separate restricted container in the same environment account. Cloud Services will grant each deployment identity the required container-scoped access, including Storage Blob Data Contributor for application state read/write/locking. The application identity will not receive access to platform state or other environments’ state.

Cloud Services will confirm the account names and resource groups and supply them to each environment’s pipeline configuration. The managed-pool code already declares `stmdpstatedevwestus301`, `stmdpstateqawestus301` and `stmdpstateprodwestus301` in these subscriptions. We will assess these existing accounts for reuse before provisioning replacements; their declaration alone does not establish current backend use.

**Migration baseline:** current remote-state references identify `stoazuflzwes001umsg` / `flz-tfstate` in `rg-azumanage-flz-state-westus3-001`; AML-state reads specify subscription `ece57219-cc9e-4eb0-a616-a637a191107f`. Actual writes are controlled by ADO `TF_BACKEND_*` variables. We will verify those values and preserve the active backend until the controlled migration to the environment accounts is complete. This is our explicit DSLZ implementation decision; the AI-LZ guide assigns storage ownership to Cloud Services but does not explicitly mandate its subscription location.

## 5. Handover sequence and protection of active environments

We will progress **DEV → QA → PROD**, with joint sign-off at each stage. Only infrastructure deployment writers will be paused for handover; workloads will remain running.

1. **Baseline:** inventory Azure IDs, state locations, Terraform addresses, owners and pipeline consumers; record destination mappings, current plans and workload checks. We will reconcile drift and protect state/configuration backups.
2. **Prepare:** refactor both repositories and replace remote-state dependencies while preserving deployed settings. We will validate names, permissions, agents and backend access before enabling new applies.
3. **Freeze and transfer:** pause legacy/new writers, cancel queued applies and invalidate saved plans. For resources crossing state boundaries, we will use reviewed `removed` blocks with `destroy = false`, followed by destination imports. We will accept only ownership changes, with no unintended Azure creates, updates, replacements or deletes. Same-state address changes will use `moved` blocks; whole-state handovers can retain addresses and use backend migration where needed.
4. **Migrate backend separately:** we will verify the configured shared backend versus pool-created storage, then migrate each environment’s states to its Cloud Services-managed account in that environment’s subscription, using the containers and keys defined in §4 and Terraform migration tooling. We will validate destination RBAC and private access before the move, update pipeline backend settings, and verify the migrated state before retiring source writers. Changing a key or folder alone does not migrate state. We will never duplicate the mixed core state into both repositories.
5. **Verify and cut over:** we will require no-change infrastructure plans and exactly one state owner per resource, then check pool jobs, state access, DNS/connectivity, AML training/inference, storage/ACR/Key Vault access, diagnostics and alerts. Only then will we enable the new deployment path and retire legacy writers/access.

We will not switch `network_mode` from creation to lookup or simply delete resource declarations while their old state still owns them: that can plan destruction. Cross-state handovers will be coordinated jointly because individual state locks do not make the transfer atomic.

**Rollback:** we will keep writers paused, reconcile destination ownership back to the original state/configuration without destroying resources, and rerun plans before restoring deployments. We will retain recovery access and backups, and never restore stale state blindly after Azure changes.

## 6. Items we will close jointly before execution

| Item | Agreed implementation work |
|---|---|
| DNS and subnet behaviour | We will preserve working settings during handover. Cloud Services will separately validate the guide’s `Enabled` private-endpoint policies against the current `Disabled` setting. We will agree one DNS zone-group writer: retain Azure Policy initially, then coordinate a Terraform handover or record the exception. |
| MLOps platform contract | We will document actual AML/storage/Key Vault/ACR/Foundry dependencies and central-service permissions, without deploying unrelated POC example services. |
| Backend and execution readiness | Cloud Services will confirm each environment account and provision the isolated containers and RBAC defined in §4. We will validate migration access to the old backend, private connectivity and Entra ID/OIDC backend authentication without shared keys. We will pin compatible CLI/providers on real agents; the POC installer runs only in demo mode. |
| Safe validation and documentation | We will add an explicit plan-only control: the POC’s manual `planAllStacks` flow can otherwise apply. We will correct tfvars placeholders, reconcile README/code differences and commit missing lock files. |

**Meeting outcome:** we will confirm service owners, restricted permissions, DNS responsibilities, the three-environment pipeline adaptation, the environment-local backend account names and migration windows. Completion will require independently managed repositories, no application ownership of platform resources or cross-stack state reads, successful workload checks and no unintended infrastructure changes from the handover.

**References:** supplied AI-LZ guide §§2–8/templates and MLOps code; [HashiCorp state refactoring](https://developer.hashicorp.com/terraform/language/state/refactor) and [backend migration](https://developer.hashicorp.com/terraform/cli/commands/init). Permissions: [Microsoft Contributor role](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/privileged#contributor). Live state and access checks will be completed during the baseline step.
