# Individual Activity 4: Terraform State Management and Safe Refactoring

**Course:** CSG401 - ACE Training  
**Main topic:** Terraform state  
**Activity type:** Individual guided practical  
**Suggested duration:** 3 hours  
**Cloud account required:** No  
**Billing account required:** No  
**External provider required:** No  
**Terraform version:** 1.7 or later

## 1. Purpose

Terraform state connects resource addresses in configuration to managed objects. In this activity, you will create a small local environment, inspect its state, change it safely, recover a removed state binding, replace an object, rename a resource with a `moved` block, explore workspace isolation, and design a remote GCS backend.

The executable portion uses Terraform's built-in `terraform_data` resource. It does not contact Google Cloud or create billable resources.

## 2. Learning outcomes

After completing the activity, you should be able to:

1. Explain why Terraform requires state.
2. Distinguish configuration, state, a plan, and real infrastructure.
3. Inspect state with supported Terraform commands.
4. Back up state before a risky operation.
5. Predict the effects of `state rm`, `state mv`, and `-replace`.
6. Rename a resource safely with a `moved` block.
7. Explain why sensitive output redaction does not automatically remove data from state.
8. Use workspaces to observe separate state snapshots.
9. Design secure remote state using the GCS backend.
10. Identify state commands that require exceptional care.

## 3. State model

```text
Configuration (.tf files)
        |
        | terraform plan compares
        v
Terraform state <------> Managed objects
        |
        `-- Resource addresses, object IDs, attributes,
            dependencies, provider references, and outputs
```

- **Configuration** describes the desired infrastructure.
- **State** records Terraform's current mapping and cached attributes.
- **Plan** describes the actions needed to reconcile configuration and state.
- **Managed objects** are the real resources. In this activity, they are local `terraform_data` objects.

Do not directly edit `terraform.tfstate`. Use Terraform commands or declarative `moved` blocks.

## 4. Safety rules

1. Perform this activity only in a new practice directory.
2. Never practise `state rm`, `state mv`, or `force-unlock` against production state.
3. Create a backup before every state-changing command.
4. Read the plan before applying it.
5. Do not store state, plan files, credentials, or secrets in Git.
6. Use only the fictional token included in this activity.
7. Do not use `-lock=false` with shared state.

## 5. State-management scenarios

| Scenario | Recommended approach |
|---|---|
| Inspect managed addresses | `terraform state list` |
| Inspect one object | `terraform state show ADDRESS` |
| Export a snapshot for backup or analysis | `terraform state pull` |
| Recreate one unhealthy object | `terraform plan/apply -replace=ADDRESS` |
| Stop managing an object without deleting it | `terraform state rm ADDRESS` |
| Change an address as a one-time operator action | `terraform state mv SOURCE DESTINATION` |
| Rename or move resources in reusable code | Add a `moved` block |
| Separate unrelated team environments | Separate backends/configurations; use workspaces only when appropriate |
| Collaborate on shared infrastructure | Use a secure remote backend with locking |
| Recover from accidental state changes | Restore a reviewed, protected state version |

## 6. Create the practice project

Create this structure:

```text
terraform-state-lab/
|-- main.tf
|-- variables.tf
|-- outputs.tf
|-- terraform.tfvars
`-- .gitignore
```

## 7. Step 1 - Define inputs

Create `variables.tf`:

```hcl
variable "environment" {
  description = "Training environment name."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be dev, test, or prod."
  }
}

variable "owner" {
  description = "Person or team responsible for the environment."
  type        = string
  default     = "student"
}

variable "services" {
  description = "Services represented in local Terraform state."

  type = map(object({
    port     = number
    replicas = number
  }))

  default = {
    web = {
      port     = 80
      replicas = 2
    }

    worker = {
      port     = 9000
      replicas = 1
    }
  }
}

variable "demo_token" {
  description = "Fictional value used only to demonstrate state sensitivity."
  type        = string
  sensitive   = true
  default     = "training-placeholder-not-a-secret"
}
```

## 8. Step 2 - Create local managed objects

Create `main.tf`:

```hcl
terraform {
  required_version = ">= 1.7.0"
}

resource "terraform_data" "environment" {
  input = {
    name  = var.environment
    owner = var.owner
  }
}

resource "terraform_data" "service" {
  for_each = var.services

  input = {
    name        = each.key
    environment = var.environment
    port        = each.value.port
    replicas    = each.value.replicas
  }
}

resource "terraform_data" "sensitivity_demo" {
  input = {
    token = var.demo_token
  }
}
```

## 9. Step 3 - Define outputs

Create `outputs.tf`:

```hcl
output "environment_record" {
  description = "Environment information recorded in state."
  value       = terraform_data.environment.output
}

output "service_records" {
  description = "Service information recorded in state."
  value = {
    for name, service in terraform_data.service :
    name => service.output
  }
}

output "demonstration_token" {
  description = "Redacted CLI output used to discuss sensitive state."
  value       = terraform_data.sensitivity_demo.output.token
  sensitive   = true
}
```

## 10. Step 4 - Supply values and protect local files

Create `terraform.tfvars`:

```hcl
environment = "dev"
owner       = "student"

services = {
  web = {
    port     = 80
    replicas = 2
  }

  worker = {
    port     = 9000
    replicas = 1
  }
}
```

Create `.gitignore`:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
state-backup*.json
crash.log
```

`.gitignore` reduces accidental commits but does not secure files already committed or copied elsewhere.

## 11. Step 5 - Initialize, plan, and create state

Run:

```powershell
terraform init
terraform fmt -check
terraform validate
terraform plan -out=initial.tfplan
terraform show initial.tfplan
terraform apply initial.tfplan
```

Confirm that Terraform creates four local objects:

```text
terraform_data.environment
terraform_data.sensitivity_demo
terraform_data.service["web"]
terraform_data.service["worker"]
```

Check the working directory. Terraform has created `terraform.tfstate`.

## 12. Step 6 - Inspect state safely

List all managed addresses:

```powershell
terraform state list
```

Inspect individual objects:

```powershell
terraform state show terraform_data.environment
terraform state show 'terraform_data.service["web"]'
```

Inspect outputs:

```powershell
terraform output
terraform output service_records
```

Create a state snapshot:

```powershell
terraform state pull | Set-Content -Encoding utf8 state-backup-01.json
```

Questions:

1. Which resource addresses use `for_each` keys?
2. Which provider manages `terraform_data`?
3. Which attributes are stored for each object?
4. Why is the state snapshot excluded from Git?

## 13. Step 7 - Observe sensitive-data behavior

Run:

```powershell
terraform output
```

Terraform displays the sensitive output as `<sensitive>`.

Now search the local state using the fictional value:

```powershell
Select-String -Path terraform.tfstate -Pattern "training-placeholder"
```

The placeholder appears in state even though the normal output is redacted. Marking a variable or output as `sensitive` hides it from routine CLI output; it does not automatically prevent storage in state.

Never use a real credential for this demonstration.

## 14. Step 8 - Change configuration and observe state history

Change the `owner` value in `terraform.tfvars`:

```hcl
owner = "platform-team"
```

Increase the web service replicas:

```hcl
web = {
  port     = 80
  replicas = 3
}
```

Run:

```powershell
terraform plan -out=update.tfplan
terraform show update.tfplan
terraform apply update.tfplan
terraform state show terraform_data.environment
terraform state show 'terraform_data.service["web"]'
```

Check for `terraform.tfstate.backup`. With local state, Terraform normally retains the previous snapshot as a backup.

## 15. Step 9 - Replace one object deliberately

Preview replacement of the web service record:

```powershell
terraform plan -replace='terraform_data.service["web"]'
```

Read the plan symbols and confirm that only the selected object will be replaced. Then execute the local-only replacement:

```powershell
terraform apply -replace='terraform_data.service["web"]'
```

Inspect its new ID:

```powershell
terraform state show 'terraform_data.service["web"]'
```

Use `-replace` when Terraform should intentionally recreate a managed object. It is safer and more reviewable than the older `terraform taint` workflow.

## 16. Step 10 - Remove and recover a state binding

Create another backup:

```powershell
terraform state pull | Set-Content -Encoding utf8 state-backup-02.json
```

Remove only the worker binding from state:

```powershell
terraform state rm 'terraform_data.service["worker"]'
```

This command does not modify configuration. For real infrastructure, it also does not delete the remote object; Terraform simply stops tracking that binding.

Inspect the result:

```powershell
terraform state list
terraform plan
```

Because the worker remains in configuration but is missing from state, the plan proposes creating it again. Restore the local training object:

```powershell
terraform apply
```

Confirm that the worker address has returned:

```powershell
terraform state list
```

In production, blindly applying after `state rm` can create a duplicate remote object or fail because an object already exists. Recovery may require importing the existing object rather than applying.

## 17. Step 11 - Rename a resource with a `moved` block

In `main.tf`, rename:

```hcl
resource "terraform_data" "environment" {
```

to:

```hcl
resource "terraform_data" "platform" {
```

In `outputs.tf`, change:

```hcl
value = terraform_data.environment.output
```

to:

```hcl
value = terraform_data.platform.output
```

Add this block to `main.tf`:

```hcl
moved {
  from = terraform_data.environment
  to   = terraform_data.platform
}
```

Run:

```powershell
terraform fmt
terraform validate
terraform plan
```

The plan should report an address move rather than destroy and create actions. Apply it:

```powershell
terraform apply
terraform state list
```

Expected new address:

```text
terraform_data.platform
```

Keep historical `moved` blocks in reusable configurations so users can upgrade safely from older versions.

## 18. Step 12 - Understand `terraform state mv`

The CLI alternative performs a one-time state operation:

```powershell
terraform state mv SOURCE_ADDRESS DESTINATION_ADDRESS
```

Use it when an operator must change an address directly and the change is not being distributed as reusable configuration. Configuration must also match the destination address before the next plan.

For shared modules and versioned configuration, prefer a `moved` block because it records the migration in code and applies consistently for every consumer.

Do not run an additional `state mv` in this exercise because the declarative `moved` block already completed the rename.

## 19. Step 13 - Explore workspace state isolation

List workspaces:

```powershell
terraform workspace list
```

Create and select a training workspace:

```powershell
terraform workspace new training
terraform state list
```

The new workspace has an empty state. Create its local objects:

```powershell
terraform apply -auto-approve -var='environment=test'
terraform state list
```

Return to the default workspace:

```powershell
terraform workspace select default
terraform state list
```

The default state still contains the original objects. Clean up the training workspace:

```powershell
terraform workspace select training
terraform destroy -auto-approve -var='environment=test'
terraform workspace select default
terraform workspace delete training
```

Workspaces provide multiple state instances for one configuration. They are not always the best boundary for environments with different access controls, credentials, backends, or ownership. Separate root configurations and backends often provide stronger isolation.

## 20. Step 14 - Design a GCS remote backend

Do not execute this section unless an instructor provides a pre-created state bucket and appropriate permissions.

Example backend configuration:

```hcl
terraform {
  backend "gcs" {
    bucket = "REPLACE_WITH_PRECREATED_STATE_BUCKET"
    prefix = "csg401/state-activity"
  }
}
```

The backend bucket must exist before `terraform init`. In a real project:

1. Use a dedicated state project or tightly controlled state bucket.
2. Enable Object Versioning for recovery.
3. Enforce public access prevention.
4. Use uniform bucket-level access.
5. Grant state access only to authorized administrators and automation identities.
6. Enable audit logging.
7. Use separate prefixes or buckets where stronger isolation is required.
8. Allow Terraform to use backend locking; do not disable locking for convenience.

After adding a backend to an existing local-state configuration, Terraform can migrate state during initialization:

```powershell
terraform init -migrate-state
```

Review the destination and backup before approving a real migration.

## 21. State-command scenarios

### Scenario A: A resource was deleted outside Terraform

Run a normal plan so Terraform refreshes state and detects the missing object. Review whether Terraform should recreate it or whether configuration should change. Do not start with `state rm`; the binding may already be updated during refresh.

### Scenario B: Terraform must stop managing an existing object

Remove the resource from configuration in a controlled change and use a `removed` block where appropriate, or use `state rm` as an operator action. Confirm that the real object must remain and document who will manage it afterward.

### Scenario C: A resource block was renamed

Use a `moved` block from the old address to the new address. Confirm the plan reports movement rather than replacement.

### Scenario D: A resource moved into a child module

Use a `moved` block with the complete destination address:

```hcl
moved {
  from = terraform_data.example
  to   = module.records.terraform_data.example
}
```

### Scenario E: Two engineers run Terraform concurrently

Use a remote backend that supports state locking. Terraform should prevent a second writer from changing state simultaneously. Do not bypass the lock with `-lock=false`.

### Scenario F: A previous process left a stale lock

First confirm that no operation is still running. Use `terraform force-unlock LOCK_ID` only for your own stale lock and only after verifying the backend and lock ID. Incorrect unlocking can permit concurrent writers and corrupt state.

### Scenario G: State contains a secret

Rotate the exposed secret, remove it from configuration where possible, restrict access to state, and migrate to an appropriate secret-management or ephemeral/write-only mechanism. Redacting CLI output is not sufficient.

### Scenario H: Local state is lost

Recover from a protected backup or remote object version. If no trusted state exists, rebuild bindings carefully with import rather than applying blindly and duplicating infrastructure.

## 22. Cleanup

Ensure the default workspace is selected:

```powershell
terraform workspace select default
```

Destroy the local training objects:

```powershell
terraform destroy -auto-approve
terraform state list
```

Expected result: no managed resource addresses remain.

The state file may still contain metadata and outputs. Treat all state-related files as sensitive until they are securely removed according to your lab policy.

## 23. Review questions

1. Why does Terraform need state instead of querying every cloud object from scratch?
2. What is the difference between removing configuration and running `state rm`?
3. Why is a `moved` block preferable when distributing a resource rename?
4. What does `-replace` change in a plan?
5. Why can a sensitive value still appear in state?
6. What protection does state locking provide?
7. Why should remote state enable versioning and access controls?
8. When are workspaces useful, and when are separate configurations safer?
9. Why can applying immediately after `state rm` be dangerous?
10. What evidence should you review before using `force-unlock`?

## 24. Individual completion checklist

No submission is required.

- [ ] I initialized and applied the local-only configuration.
- [ ] I listed and inspected individual state objects.
- [ ] I created state snapshots before state-changing operations.
- [ ] I observed that a sensitive output can still be stored in state.
- [ ] I updated configuration and inspected the resulting state.
- [ ] I replaced one selected object with `-replace`.
- [ ] I removed and recovered a worker state binding.
- [ ] I renamed a resource using a `moved` block without replacement.
- [ ] I created, inspected, cleaned, and deleted a training workspace.
- [ ] I can describe a secure GCS remote-state design.
- [ ] I destroyed all local training objects.

## 25. Reference documentation

- [Terraform state overview](https://developer.hashicorp.com/terraform/language/state)
- [Purpose of Terraform state](https://developer.hashicorp.com/terraform/language/state/purpose)
- [Terraform state commands](https://developer.hashicorp.com/terraform/cli/commands/state)
- [Moved block reference](https://developer.hashicorp.com/terraform/language/block/moved)
- [State locking](https://developer.hashicorp.com/terraform/language/state/locking)
- [Managing sensitive data](https://developer.hashicorp.com/terraform/language/manage-sensitive-data)
- [GCS backend](https://developer.hashicorp.com/terraform/language/backend/gcs)

