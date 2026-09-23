# Individual Activity 6: Terraform Lifecycle and Dependency Management

**Course:** CSG 401 - Getting Started with Terraform in GCP  
**Activity type:** Individual guided practical  
**Suggested duration:** 3 hours  
**Cloud or billing account required:** No  
**External provider required:** No

## 1. Purpose

Build a local GCP-style service stack and investigate how Terraform orders operations, replaces objects, protects important objects, ignores selected changes, and enforces assumptions. The activity uses the built-in `terraform_data` resource and creates no cloud infrastructure.

This topic was not the main focus of Activities 1-5. Earlier activities created natural references, but this activity explicitly studies the dependency graph and lifecycle rules.

## 2. Learning outcomes

You will be able to:

1. Distinguish implicit from explicit dependencies.
2. Explain why file order does not control resource order.
3. Inspect Terraform's dependency graph.
4. Use `create_before_destroy`, `prevent_destroy`, `ignore_changes`, and `replace_triggered_by` appropriately.
5. Trigger a controlled replacement with `terraform_data.triggers_replace`.
6. Use preconditions and postconditions for lifecycle guarantees.
7. Recognize dependency cycles and unnecessary broad dependencies.

## 3. Scenario and structure

Model a release path containing a network, database, application release, application service, and monitoring setup:

```text
terraform-lifecycle-lab/
|-- variables.tf
|-- main.tf
|-- outputs.tf
|-- terraform.tfvars
`-- .gitignore
```

## 4. Define inputs

Create `variables.tf`:

```hcl
variable "environment" {
  type    = string
  default = "dev"

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be dev, test, or prod."
  }
}

variable "release_version" {
  type    = string
  default = "1.0.0"

  validation {
    condition     = can(regex("^[0-9]+\\.[0-9]+\\.[0-9]+$", var.release_version))
    error_message = "release_version must use semantic form such as 1.0.0."
  }
}

variable "database_tier" {
  type    = string
  default = "small"

  validation {
    condition     = contains(["small", "medium", "large"], var.database_tier)
    error_message = "database_tier must be small, medium, or large."
  }
}

variable "operator_note" {
  type    = string
  default = "managed-by-platform-team"
}

```

Lifecycle settings such as `prevent_destroy` require literal values because Terraform must evaluate lifecycle behavior before ordinary expressions.

## 5. Create the dependency graph

Create `main.tf`:

```hcl
terraform {
  required_version = ">= 1.7.0"
}

locals {
  prefix = "lifecycle-${var.environment}"
}

resource "terraform_data" "network" {
  input = {
    name = "${local.prefix}-vpc"
    cidr = "10.40.0.0/16"
  }
}

resource "terraform_data" "database" {
  input = {
    name       = "${local.prefix}-database"
    tier       = var.database_tier
    network_id = terraform_data.network.id
  }

  lifecycle {
    prevent_destroy = false

    precondition {
      condition     = var.environment != "prod" || var.database_tier != "small"
      error_message = "A production database cannot use the small tier."
    }

    postcondition {
      condition     = self.output.network_id == terraform_data.network.id
      error_message = "The database must retain the expected network relationship."
    }
  }
}

resource "terraform_data" "release" {
  input = {
    version = var.release_version
  }

  triggers_replace = [var.release_version]
}

resource "terraform_data" "application" {
  input = {
    name          = "${local.prefix}-application"
    release       = var.release_version
    database_id   = terraform_data.database.id
    operator_note = var.operator_note
  }

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [input.operator_note]
    replace_triggered_by  = [terraform_data.release]
  }
}

resource "terraform_data" "monitoring" {
  input = {
    name        = "${local.prefix}-monitoring"
    application = "${local.prefix}-application"
  }

  # The monitoring service depends on the application being operational,
  # but it deliberately does not copy an application attribute.
  depends_on = [terraform_data.application]
}
```

Dependencies created by value references are implicit and preferred. `depends_on` is reserved for the hidden behavioral dependency in the monitoring example.

## 6. Define outputs and values

Create `outputs.tf`:

```hcl
output "object_ids" {
  value = {
    network     = terraform_data.network.id
    database    = terraform_data.database.id
    release     = terraform_data.release.id
    application = terraform_data.application.id
    monitoring  = terraform_data.monitoring.id
  }
}

output "application_record" {
  value = terraform_data.application.output
}
```

Create `terraform.tfvars`:

```hcl
environment     = "dev"
release_version = "1.0.0"
database_tier   = "small"
operator_note   = "managed-by-platform-team"
```

Create `.gitignore`:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
graph.dot
plan.txt
```

## 7. Establish the baseline

```powershell
terraform init
terraform fmt
terraform validate
terraform plan -out=baseline.tfplan
terraform apply baseline.tfplan
terraform state list
terraform output object_ids
```

Record the five IDs temporarily for comparison during later steps.

## 8. Inspect implicit and explicit dependencies

Generate Terraform's graph description:

```powershell
terraform graph > graph.dot
Get-Content graph.dot
```

Identify these paths:

```text
network -> database -> application -> monitoring
release -> application
```

The direction of arrows in raw DOT output may look reversed depending on graph semantics, so identify which object must be completed first rather than relying only on visual direction.

Answer:

1. Which relationships come from attribute references?
2. Which relationship requires `depends_on`?
3. Would moving the monitoring block above the application block change ordering? Test your prediction if desired.

## 9. Observe controlled replacement

Change `release_version` to `1.1.0`, then run:

```powershell
terraform plan -out=release.tfplan
terraform show release.tfplan
```

Expected behavior:

- `terraform_data.release` is replaced because its `triggers_replace` value changed.
- `terraform_data.application` is replaced because of `replace_triggered_by`.
- Terraform plans the application with create-before-destroy ordering.
- The database and network are not replaced.

Apply and compare IDs:

```powershell
terraform apply release.tfplan
terraform output object_ids
```

## 10. Observe `ignore_changes`

Change only `operator_note` in `terraform.tfvars`:

```hcl
operator_note = "changed-outside-terraform-workflow"
```

Run `terraform plan`. Terraform should not plan an application update because `input.operator_note` is ignored after creation.

Important: `ignore_changes` does not remove the argument from initial creation. It tells Terraform not to reconcile later changes to that selected attribute. Use it only when another process legitimately shares responsibility for that field.

Restore the original note. The state continues to contain the original recorded value.

## 11. Test `prevent_destroy`

In the database lifecycle block, change:

```hcl
prevent_destroy = false
```

to:

```hcl
prevent_destroy = true
```

Apply this configuration, then run:

```powershell
terraform destroy
```

Terraform should reject the plan because the database is protected. This rule is useful for stateful objects, but it also blocks legitimate replacement and destroy operations.

Set `prevent_destroy` back to `false` and apply before continuing.

`prevent_destroy` is not absolute protection: removing the resource block also removes the rule from configuration. Organizational permissions, backups, review, and policy checks are still required.

## 12. Test preconditions

Set these values together:

```hcl
environment   = "prod"
database_tier = "small"
```

Run `terraform plan`. The database precondition should reject the combination. Change `database_tier` to `medium` and confirm the plan succeeds.

Return to the original `dev` and `small` values afterward.

## 13. Create and diagnose a dependency cycle

Temporarily add this line to the `network` resource:

```hcl
depends_on = [terraform_data.monitoring]
```

Run `terraform validate` or `terraform plan`. The new explicit dependency completes a cycle:

```text
network -> database -> application -> monitoring -> network
```

Terraform cannot determine a starting point and reports a cycle. Remove the temporary line and validate again.

## 14. Dependency design rules

| Situation | Preferred design |
|---|---|
| One resource consumes another resource's value | Direct reference; dependency is inferred |
| A behavioral dependency has no exchanged value | Narrow `depends_on` reference |
| An entire module is declared as a dependency | Avoid unless the whole module truly must complete first |
| Resource replacement must follow another resource | `replace_triggered_by` |
| A plain value should cause replacement | Put it in `terraform_data.triggers_replace` |
| Two objects depend on each other | Redesign boundaries; do not force a cycle |

Broad explicit dependencies can make plans more conservative and cause more values to remain unknown until apply.

## 15. Cleanup

Confirm `prevent_destroy = false`, then run:

```powershell
terraform apply -auto-approve
terraform destroy -auto-approve
terraform state list
```

## 16. Review questions

1. Why does Terraform not use file order to determine creation order?
2. When is `depends_on` justified?
3. Why can broad module dependencies make plans less precise?
4. What limitation must be considered before using `create_before_destroy`?
5. Why should `prevent_destroy` be only one layer of protection?
6. What shared-ownership situation could justify `ignore_changes`?
7. Why can `replace_triggered_by` reference managed resources but not plain variables directly?
8. At what stages are preconditions and postconditions evaluated?
9. What is a dependency cycle?
10. Which IDs changed when the release version changed, and why?

## 17. Individual completion checklist

There is no submission requirement.

- [ ] Created and applied the five-object stack.
- [ ] Inspected the dependency graph.
- [ ] Observed cascading controlled replacement.
- [ ] Tested `ignore_changes`.
- [ ] Confirmed that `prevent_destroy` blocks destruction.
- [ ] Triggered and corrected a precondition failure.
- [ ] Created and corrected a dependency cycle.
- [ ] Disabled protection and destroyed all local objects.

## 18. References

- [Lifecycle meta-argument](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle)
- [`depends_on` reference](https://developer.hashicorp.com/terraform/language/meta-arguments/depends_on)
- [Resource dependencies tutorial](https://developer.hashicorp.com/terraform/tutorials/configuration-language/dependencies)
- [Terraform validation mechanisms](https://developer.hashicorp.com/terraform/language/validate)
