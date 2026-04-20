# Tamnoon GCP Onboarding Module

Terraform module that creates a dedicated Tamnoon service account with Workload Identity Federation (WIF) for keyless authentication from AWS, and assigns read-only IAM roles at the chosen scope (project, folder, or organization).

## Resources Created

| Resource | Description |
|----------|-------------|
| **Service Account** | `tamnoon-<service_account_suffix>` in the identity project |
| **Workload Identity Pool** | `tamnoon-pool-<identity_suffix>` — federation endpoint that accepts external tokens |
| **Workload Identity Provider** | `tamnoon-aws-<identity_suffix>` — AWS provider that validates STS tokens and extracts the caller's IAM role |
| **WIF Principal Binding** | Grants `roles/iam.workloadIdentityUser` to the trusted AWS role on the service account |
| **IAM Role Bindings** | Assigns 6 predefined read-only roles to the service account at the chosen scope |

## Prerequisites

### Terraform

- Terraform >= 1.3.0
- `hashicorp/google` provider >= 5.0.0, < 7.0.0

### APIs

The following APIs must be enabled on the identity project:

- `iam.googleapis.com` — IAM and service account management
- `cloudresourcemanager.googleapis.com` — project/folder/org hierarchy
- `sts.googleapis.com` — Security Token Service (WIF token exchange)
- `iamcredentials.googleapis.com` — SA credential generation for WIF

### Permissions

The identity running `terraform apply` needs full lifecycle permissions (create, read, update, delete) on the resources the module manages. Terraform reads every resource on each `plan`/`apply` to refresh state.

*Project data source:*

| Permission | Purpose |
|-----------|---------|
| `resourcemanager.projects.get` | Read identity project number (`data.google_project`) |

*Service account management:*

| Permission | Purpose |
|-----------|---------|
| `iam.serviceAccounts.create` | Create the Tamnoon service account |
| `iam.serviceAccounts.get` | Read service account state (Terraform refresh) |
| `iam.serviceAccounts.list` | List service accounts in the project |
| `iam.serviceAccounts.update` | Update service account attributes |
| `iam.serviceAccounts.delete` | Remove the service account (`terraform destroy`) |
| `iam.serviceAccounts.setIamPolicy` | Bind the WIF principal to the service account (used by `google_service_account_iam_member`) |

> Alternatively, `roles/iam.serviceAccountAdmin` covers all of the above.

*Workload Identity Federation:*

| Permission | Purpose |
|-----------|---------|
| `iam.googleapis.com/workloadIdentityPools.create` | Create the WIF pool |
| `iam.googleapis.com/workloadIdentityPools.get` | Read pool state (Terraform refresh) |
| `iam.googleapis.com/workloadIdentityPools.update` | Update pool configuration |
| `iam.googleapis.com/workloadIdentityPools.delete` | Remove the pool (`terraform destroy`) |
| `iam.googleapis.com/workloadIdentityPools.list` | List pools in the project |
| `iam.googleapis.com/workloadIdentityPoolProviders.create` | Create the AWS provider |
| `iam.googleapis.com/workloadIdentityPoolProviders.get` | Read provider state (Terraform refresh) |
| `iam.googleapis.com/workloadIdentityPoolProviders.update` | Update provider configuration |
| `iam.googleapis.com/workloadIdentityPoolProviders.delete` | Remove the provider (`terraform destroy`) |
| `iam.googleapis.com/workloadIdentityPoolProviders.list` | List providers in the pool |

> Alternatively, `roles/iam.workloadIdentityPoolAdmin` covers all of the above.

*IAM role bindings (scope-dependent) — the module uses `google_*_iam_member` (additive, no read needed):*

| Permission | Purpose |
|-----------|---------|
| `resourcemanager.projects.setIamPolicy` | Assign roles at project level |

For folder or org scope, add the corresponding permission at the target scope:
- **Folder**: `resourcemanager.folders.setIamPolicy`
- **Organization**: `resourcemanager.organizations.setIamPolicy`

See [Tamnoon Public Permissions](https://github.com/tamnoon-io/Tamnoon-Public-Permissions/blob/main/Cloud_Providers/GCP/gcp-onboarding-permissions.md#1-prerequisites) for full details.

## Usage

### Project scope

```hcl
module "tamnoon_onboarding" {
  source = "github.com/tamnoon-io/gcp-onboarding"

  identity_project_id   = "my-infra-project"
  trusted_aws_role_name = "gcp-onboarding-trust-<customer-tenant-id>"  # provided during onboarding
  project_ids           = "project-a;project-b;project-c"
}
```

### Folder scope

```hcl
module "tamnoon_onboarding" {
  source = "github.com/tamnoon-io/gcp-onboarding"

  identity_project_id   = "my-infra-project"
  trusted_aws_role_name = "gcp-onboarding-trust-<customer-tenant-id>"  # provided during onboarding
  folder_ids            = "123456789012;987654321098"
}
```

### Organization scope (recommended)

```hcl
module "tamnoon_onboarding" {
  source = "github.com/tamnoon-io/gcp-onboarding"

  identity_project_id   = "my-infra-project"
  trusted_aws_role_name = "gcp-onboarding-trust-<customer-tenant-id>"  # provided during onboarding
  organization_id       = "100623586402"
}
```

When `organization_id` is set, `project_ids` and `folder_ids` are ignored. Organization scope is recommended as it covers all projects (existing and future) through IAM inheritance.

### Running

Create a `main.tf` file with the module block from one of the examples above, then:

```bash
# Initialize — downloads the module from GitHub
terraform init

# Preview the changes
terraform plan \
  -var='identity_project_id=my-infra-project' \
  -var='trusted_aws_role_name=gcp-onboarding-trust-<customer-tenant-id>' \
  -var='project_ids=project-a;project-b'

# Apply
terraform apply \
  -var='identity_project_id=my-infra-project' \
  -var='trusted_aws_role_name=gcp-onboarding-trust-<customer-tenant-id>' \
  -var='project_ids=project-a;project-b'

# Retrieve outputs for Tamnoon onboarding completion
terraform output -json
```

Alternatively, define variables in a `terraform.tfvars` file to avoid passing `-var` on every command:

```hcl
# terraform.tfvars
identity_project_id   = "my-infra-project"
trusted_aws_role_name = "gcp-onboarding-trust-<customer-tenant-id>"
project_ids           = "project-a;project-b"
```

```bash
terraform init
terraform plan
terraform apply
terraform output -json
```

### Scope expansion

To add new projects or folders, update `project_ids` or `folder_ids` and re-run:

```bash
terraform plan    # verify only IAM bindings change
terraform apply
```

The service account and WIF resources remain unchanged — only new IAM bindings are added.

### Teardown

```bash
terraform destroy
```

This removes the Tamnoon service account, WIF pool/provider, and all IAM bindings created by the module.

## Variables

| Variable | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `identity_project_id` | `string` | Yes | — | GCP project ID where the Tamnoon service account and WIF resources are created |
| `trusted_aws_role_name` | `string` | Yes | — | AWS IAM role name, format: `gcp-onboarding-trust-<customer-tenant-id>` (provided during onboarding) |
| `service_account_suffix` | `string` | Yes* | — | Appended to `tamnoon-` for the service account ID (total must be <=30 chars). Use `federate-svc-account` |
| `identity_suffix` | `string` | Yes* | — | Used to derive pool ID (`tamnoon-pool-<suffix>`) and provider ID (`tamnoon-aws-<suffix>`, each <=32 chars). Use `federate` |
| `aws_account_id` | `string` | Yes* | — | AWS account ID trusted by the WIF provider. Use `112665896816` (Tamnoon) |
| `project_ids` | `string` | No | `""` | Semicolon-delimited GCP project IDs for project-scope bindings |
| `folder_ids` | `string` | No | `""` | Semicolon-delimited GCP folder IDs for folder-scope bindings |
| `organization_id` | `string` | No | `null` | GCP organization ID — when set, overrides `project_ids` and `folder_ids` |

> \* These variables have known values and should use defaults in a future update to `variables.tf` (see [Suggested Changes](#suggested-changes-to-variablestf)).

## Outputs

After `terraform apply`, the module outputs values that Tamnoon needs to complete the onboarding setup. Provide this JSON output back to Tamnoon via the UI.

```bash
terraform output -json
```

| Output | Description |
|--------|-------------|
| `service_account_id` | The Tamnoon service account unique ID |
| `workload_identity_pool_id` | The WIF pool ID |
| `workload_identity_provider_id` | The AWS WIF provider ID |
| `identity_project_number` | The identity project number |


## Roles Assigned

The module assigns these read-only roles to the Tamnoon service account at the chosen scope:

| Role | Purpose |
|------|---------|
| `roles/viewer` | Read-only access to all resources and project-level IAM policies |
| `roles/browser` | Navigate organization, folder, and project hierarchy |
| `roles/iam.securityReviewer` | Read IAM policies at all levels + SCC findings |
| `roles/cloudasset.viewer` | Search IAM bindings and resources across projects |
| `roles/logging.privateLogViewer` | Access Data Access Logs and filtered log views |
| `roles/serviceusage.serviceUsageConsumer` | View enabled APIs and service usage quotas |

See [Roles Assigned to Tamnoon Service Account](https://github.com/tamnoon-io/Tamnoon-Public-Permissions/blob/main/Cloud_Providers/GCP/gcp-onboarding-permissions.md#2-roles-assigned-to-tamnoon-service-account) for detailed justification of each role.

## Suggested Changes to `variables.tf`

The following variables have known, constant values and should be given defaults to simplify customer usage:

```hcl
variable "service_account_suffix" {
  # ... existing validations ...
  default = "federate-svc-account"
}

variable "identity_suffix" {
  # ... existing validations ...
  default = "federate"
}

variable "aws_account_id" {
  # ... existing validations ...
  default = "112665896816"
}
```

This reduces the required customer inputs from 5 to 2 (`identity_project_id` + `trusted_aws_role_name`) plus the scope variable.
