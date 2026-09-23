# Individual Activity 5: Advanced Input Design and Collection Transformations

**Course:** CSG 401 - Getting Started with Terraform in GCP  
**Activity type:** Individual guided practical  
**Suggested duration:** 3 hours  
**Cloud account required:** No  
**Billing account required:** No  
**External provider required:** No

## 1. Scenario

You are designing the Terraform configuration for a small GCP-style application environment. The design contains one network, several subnets, application virtual machines, and an optional administrative VM.

The activity uses Terraform's built-in `terraform_data` resource. It records representative infrastructure objects in local state but does not create real VPCs or VMs.

### Relationship to the earlier activities

Activities 1, 3, and 4 already introduced basic variables, simple validation, `count`, and `for_each`. Those features therefore appear here only as prerequisites for a more advanced objective. This activity does not repeat the earlier VM-count or module-composition exercises.

The new focus is:

- Nested object types with optional attributes
- Multiple validation rules on one variable
- Cross-variable relationship validation
- Nested `for` expressions and `flatten`
- Converting a generated list into a stable keyed map
- Filtering, grouping, merging, deduplication, and ordering
- Comparing `count` identity with `for_each` identity
- Observing the effect of changing a `for_each` key
- Exploring expressions interactively with `terraform console`
- Understanding where provider-level dynamic blocks fit

## 2. Learning outcomes

After completing the activity, you should be able to:

1. Model related settings with nested object and map types.
2. Apply optional attributes, multiple validations, and cross-variable validation.
3. Use locals to normalize, flatten, group, filter, and summarize collections.
4. Use conditional expressions to select values.
5. Use `count` for optional or nearly identical objects.
6. Use `for_each` for objects that require stable, meaningful keys.
7. Transform collections with `for` expressions, filtering, `flatten`, and `toset`.
8. Read resource addresses created by `count` and `for_each`.
9. Predict how changing collection keys affects a Terraform plan.

## 3. Concepts used

| Feature | Use in this activity |
|---|---|
| String variable | Project and environment names |
| Boolean variable | Enable or disable the administrative VM |
| Object variable | Network configuration |
| Map of objects | Subnets and application tiers |
| Validation | Reject unsupported environments, regions, CIDRs, and machine types |
| Optional attribute | Supply a default VM count inside each tier object |
| Locals | Naming, labels, flattened VM definitions, and totals |
| Conditional expression | Choose environment class and optional object count |
| `count` | Create zero or one administrative VM |
| `for_each` | Create keyed subnet and VM records |
| `for` expression | Transform, filter, and summarize collections |

Basic string and Boolean variables are included to keep the activity self-contained, but they are not treated as new material.

## 4. When to use `count` and `for_each`

Use `count` when instances are nearly identical or when a Boolean condition should create zero or one object. Instances receive numeric addresses such as `terraform_data.admin_vm[0]`.

Use `for_each` when every object has a meaningful, stable key and may have different settings. Instances receive addresses such as `terraform_data.subnet["web"]`.

Avoid changing `for_each` keys casually. Terraform treats a changed key as one object being removed and another being created, even if their attributes are similar.

## 5. Project structure

Create a new directory named `terraform-variables-loops-lab`:

```text
terraform-variables-loops-lab/
|-- versions.tf
|-- variables.tf
|-- locals.tf
|-- main.tf
|-- outputs.tf
|-- terraform.tfvars
`-- .gitignore
```

Open PowerShell in this directory.

## 6. Step 1 - Set the Terraform version

Create `versions.tf`:

```hcl
terraform {
  required_version = ">= 1.7.0"
}
```

The `terraform_data` resource belongs to Terraform's built-in provider, so no provider download or cloud credentials are required.

## 7. Step 2 - Define and validate variables

Create `variables.tf`:

```hcl
variable "project_name" {
  description = "Short lowercase name used as a prefix."
  type        = string
  default     = "student-app"

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,19}$", var.project_name))
    error_message = "project_name must be 3-20 lowercase letters, digits, or hyphens and must start with a letter."
  }
}

variable "environment" {
  description = "Deployment environment."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be dev, test, or prod."
  }
}

variable "region" {
  description = "Simulated GCP region."
  type        = string
  default     = "asia-south1"

  validation {
    condition = contains([
      "asia-south1",
      "asia-southeast1",
      "us-central1",
    ], var.region)
    error_message = "region must be asia-south1, asia-southeast1, or us-central1."
  }
}

variable "network" {
  description = "Network-level settings."
  type = object({
    routing_mode = string
    mtu          = number
  })

  default = {
    routing_mode = "REGIONAL"
    mtu          = 1460
  }

  validation {
    condition     = contains(["REGIONAL", "GLOBAL"], var.network.routing_mode)
    error_message = "network.routing_mode must be REGIONAL or GLOBAL."
  }

  validation {
    condition     = var.network.mtu >= 1300 && var.network.mtu <= 8896
    error_message = "network.mtu must be between 1300 and 8896."
  }
}

variable "subnets" {
  description = "Subnets keyed by a stable logical name."
  type = map(object({
    cidr                  = string
    private_google_access = optional(bool, true)
  }))

  default = {
    web = {
      cidr = "10.10.1.0/24"
    }
    app = {
      cidr = "10.10.2.0/24"
    }
  }

  validation {
    condition = length(var.subnets) > 0 && alltrue([
      for subnet in values(var.subnets) : can(cidrhost(subnet.cidr, 1))
    ])
    error_message = "At least one subnet is required and every subnet must contain a valid IPv4 CIDR."
  }

  validation {
    condition     = length(distinct([for subnet in values(var.subnets) : subnet.cidr])) == length(var.subnets)
    error_message = "Every subnet must use a unique CIDR value."
  }
}

variable "application_tiers" {
  description = "Application tiers keyed by a stable tier name."
  type = map(object({
    subnet         = string
    machine_type   = string
    instance_count = optional(number, 1)
    public         = optional(bool, false)
    tags           = optional(set(string), [])
  }))

  default = {
    frontend = {
      subnet         = "web"
      machine_type   = "e2-small"
      instance_count = 2
      public         = true
      tags           = ["http-server", "frontend"]
    }
    backend = {
      subnet         = "app"
      machine_type   = "e2-medium"
      instance_count = 2
      tags           = ["backend"]
    }
  }

  validation {
    condition = alltrue([
      for tier in values(var.application_tiers) :
      contains(["e2-micro", "e2-small", "e2-medium"], tier.machine_type)
    ])
    error_message = "machine_type must be e2-micro, e2-small, or e2-medium."
  }

  validation {
    condition = alltrue([
      for tier in values(var.application_tiers) :
      tier.instance_count >= 1 && tier.instance_count <= 5
    ])
    error_message = "Each tier must request between 1 and 5 instances."
  }

  validation {
    condition = alltrue([
      for tier in values(var.application_tiers) : contains(keys(var.subnets), tier.subnet)
    ])
    error_message = "Every application tier must reference a key that exists in subnets."
  }
}

variable "enable_admin_vm" {
  description = "Whether to create one simulated administrative VM."
  type        = bool
  default     = false
}
```

Notice that validation on `application_tiers` can refer to `var.subnets`. This detects broken relationships before any objects are created.

## 8. Step 3 - Transform inputs with locals

Create `locals.tf`:

```hcl
locals {
  name_prefix = "${var.project_name}-${var.environment}"

  environment_class = var.environment == "prod" ? "production" : "non-production"

  common_labels = {
    project     = var.project_name
    environment = var.environment
    managed_by  = "terraform"
  }

  vm_matrix = flatten([
    for tier_name, tier in var.application_tiers : [
      for instance_number in range(tier.instance_count) : {
        key             = "${tier_name}-${format("%02d", instance_number + 1)}"
        tier            = tier_name
        instance_number = instance_number + 1
        subnet          = tier.subnet
        machine_type    = tier.machine_type
        public          = tier.public
        tags            = tier.tags
      }
    ]
  ])

  vm_definitions = {
    for vm in local.vm_matrix : vm.key => vm
  }

  public_vm_keys = toset([
    for key, vm in local.vm_definitions : key if vm.public
  ])

  unique_tags = sort(tolist(toset(flatten([
    for tier in values(var.application_tiers) : tolist(tier.tags)
  ]))))

  total_application_vms = sum([
    for tier in values(var.application_tiers) : tier.instance_count
  ])
}
```

The nested `for` expressions create a list of VM definitions for every tier. `flatten` converts the nested lists into one list, and the next expression converts it into a map suitable for `for_each`.

## 9. Step 4 - Create the simulated environment

Create `main.tf`:

```hcl
resource "terraform_data" "network" {
  input = {
    name              = "${local.name_prefix}-vpc"
    region            = var.region
    routing_mode      = var.network.routing_mode
    mtu               = var.network.mtu
    environment_class = local.environment_class
    labels            = local.common_labels
  }

  lifecycle {
    precondition {
      condition     = local.total_application_vms <= 12
      error_message = "This training environment permits no more than 12 application VMs."
    }
  }
}

resource "terraform_data" "subnet" {
  for_each = var.subnets

  input = {
    name                  = "${local.name_prefix}-${each.key}-subnet"
    logical_key           = each.key
    cidr                  = each.value.cidr
    region                = var.region
    private_google_access = each.value.private_google_access
    network_id            = terraform_data.network.id
  }
}

resource "terraform_data" "application_vm" {
  for_each = local.vm_definitions

  input = {
    name            = "${local.name_prefix}-${each.key}"
    tier            = each.value.tier
    instance_number = each.value.instance_number
    machine_type    = each.value.machine_type
    subnet_key      = each.value.subnet
    subnet_id       = terraform_data.subnet[each.value.subnet].id
    public          = each.value.public
    tags            = sort(tolist(each.value.tags))
    labels          = merge(local.common_labels, { tier = each.value.tier })
  }
}

resource "terraform_data" "admin_vm" {
  count = var.enable_admin_vm ? 1 : 0

  input = {
    name         = "${local.name_prefix}-admin-01"
    machine_type = var.environment == "prod" ? "e2-small" : "e2-micro"
    subnet_key   = sort(keys(var.subnets))[0]
    subnet_id    = terraform_data.subnet[sort(keys(var.subnets))[0]].id
    labels       = merge(local.common_labels, { role = "administration" })
  }
}
```

Observe the addressing strategies:

- The network is a single address: `terraform_data.network`.
- Subnets and application VMs use meaningful map keys through `for_each`.
- The optional administrative VM uses numeric indexing through `count`.

## 10. Step 5 - Create useful outputs

Create `outputs.tf`:

```hcl
output "deployment_summary" {
  description = "Calculated summary of the simulated deployment."
  value = {
    prefix               = local.name_prefix
    environment_class    = local.environment_class
    region               = var.region
    subnet_count         = length(var.subnets)
    application_vm_count = local.total_application_vms
    admin_vm_count       = length(terraform_data.admin_vm)
    unique_tags          = local.unique_tags
  }
}

output "subnet_records" {
  description = "Subnet records keyed by logical subnet name."
  value = {
    for key, subnet in terraform_data.subnet : key => subnet.output
  }
}

output "vm_names_by_tier" {
  description = "VM names grouped by application tier."
  value = {
    for tier_name in sort(keys(var.application_tiers)) :
    tier_name => sort([
      for vm in values(terraform_data.application_vm) :
      vm.output.name if vm.output.tier == tier_name
    ])
  }
}

output "public_vm_names" {
  description = "Only VMs whose public flag is true."
  value = sort([
    for key in local.public_vm_keys : terraform_data.application_vm[key].output.name
  ])
}

output "admin_vm_name" {
  description = "Administrative VM name, or null when disabled."
  value       = try(terraform_data.admin_vm[0].output.name, null)
}
```

These outputs demonstrate transformation, grouping, filtering, ordering, and safe access to an optional resource.

## 11. Step 6 - Supply initial values

Create `terraform.tfvars`:

```hcl
project_name = "student-app"
environment  = "dev"
region       = "asia-south1"

network = {
  routing_mode = "REGIONAL"
  mtu          = 1460
}

subnets = {
  web = {
    cidr = "10.10.1.0/24"
  }
  app = {
    cidr                  = "10.10.2.0/24"
    private_google_access = true
  }
}

application_tiers = {
  frontend = {
    subnet         = "web"
    machine_type   = "e2-small"
    instance_count = 2
    public         = true
    tags           = ["frontend", "http-server"]
  }
  backend = {
    subnet         = "app"
    machine_type   = "e2-medium"
    instance_count = 2
    tags           = ["backend"]
  }
}

enable_admin_vm = false
```

Create `.gitignore`:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
crash.log
```

## 12. Step 7 - Initialize and inspect the plan

Run:

```powershell
terraform init
terraform fmt
terraform validate
terraform plan -out=initial.tfplan
terraform show initial.tfplan
```

The initial plan should propose:

- One network record
- Two subnet records
- Four application VM records
- No administrative VM

Apply the plan:

```powershell
terraform apply initial.tfplan
terraform state list
terraform output
```

Study the addresses. Subnet and VM addresses contain string keys, while no `admin_vm[0]` address exists yet.

## 13. Step 8 - Test validation failures

Validation errors should be readable and should stop the plan before infrastructure operations.

Run each experiment separately and restore the original value afterward.

### Experiment A: Unsupported environment

Change:

```hcl
environment = "production"
```

Run `terraform plan`. It should fail because the allowed value is `prod`, not `production`.

### Experiment B: Unknown subnet reference

Change the backend tier to:

```hcl
subnet = "database"
```

Run `terraform plan`. It should fail because `database` is not a key in `subnets`.

### Experiment C: Excessive instance count

Change one tier to:

```hcl
instance_count = 6
```

Run `terraform plan`. It should fail the per-tier range validation.

Restore all original values and confirm that `terraform validate` and `terraform plan` succeed.

## 14. Step 9 - Add objects with `for_each`

Add a database subnet to `subnets`:

```hcl
database = {
  cidr = "10.10.3.0/24"
}
```

Add a database tier:

```hcl
database = {
  subnet         = "database"
  machine_type   = "e2-small"
  instance_count = 1
  tags           = ["database", "internal"]
}
```

Run:

```powershell
terraform plan -out=expanded.tfplan
terraform show expanded.tfplan
terraform apply expanded.tfplan
terraform state list
```

Confirm that existing keyed objects remain at the same addresses and only the new keyed objects are added.

## 15. Step 10 - Enable an optional object with `count`

Change:

```hcl
enable_admin_vm = true
```

Run:

```powershell
terraform plan
terraform apply
terraform state show 'terraform_data.admin_vm[0]'
terraform output admin_vm_name
```

The Boolean conditional changes `count` from `0` to `1`. Set it back to `false`, run `terraform plan`, and observe that only `admin_vm[0]` is scheduled for removal. Apply the change.

## 16. Step 11 - Compare stable and unstable identity

Rename the key `frontend` to `web-tier` in `application_tiers` without changing its values. Run `terraform plan`.

Terraform plans to remove the VM addresses containing `frontend-*` and create addresses containing `web-tier-*`. The key is part of each object's identity. Rename the key back to `frontend` and verify that the plan no longer proposes those replacements.

This behavior is why stable business identifiers are better `for_each` keys than display names that change frequently.

## 17. Step 12 - Use the Terraform console

Run:

```powershell
terraform console
```

Evaluate these expressions one at a time:

```hcl
local.name_prefix
local.environment_class
local.total_application_vms
local.vm_matrix
keys(local.vm_definitions)
local.public_vm_keys
local.unique_tags
{ for name, subnet in var.subnets : name => subnet.cidr }
[for name, tier in var.application_tiers : name if tier.public]
```

Enter `exit` to close the console.

## 18. Optional design exercise: dynamic blocks

Some provider resources contain repeatable nested blocks. For example, a real Google Cloud firewall resource can contain multiple `allow` blocks. A `dynamic` block can generate those nested blocks from a collection:

```hcl
dynamic "allow" {
  for_each = var.firewall_rules

  content {
    protocol = allow.value.protocol
    ports    = allow.value.ports
  }
}
```

Do not add this fragment to the executable configuration: `terraform_data` has no `allow` nested block. The key distinction is:

- Resource-level `for_each` creates multiple resource instances.
- A `dynamic` block creates repeated nested configuration blocks inside one resource.
- Dynamic blocks cannot generate meta-argument blocks such as `lifecycle`.

## 19. Cleanup

Run:

```powershell
terraform destroy
terraform state list
```

The final state list should be empty. Because this activity creates only local `terraform_data` objects, cleanup does not contact Google Cloud.

## 20. Review questions

1. Why is a map usually preferable to a list for `for_each`?
2. What is the difference between a type constraint and a validation rule?
3. Why are optional object attributes useful?
4. When is `count` a better choice than `for_each`?
5. What happens to resource addresses when a `for_each` key changes?
6. Why is `flatten` needed when generating VMs with nested `for` expressions?
7. What does the `if` clause do in a `for` expression?
8. Why is `toset` useful for a collection of unique values?
9. What purpose do locals serve if they cannot be set from outside the configuration?
10. How does `try` make the optional administrative VM output safe?
11. Why should derived collections be deterministic and use stable keys?
12. How is a dynamic block different from resource-level `for_each`?

## 21. Individual completion checklist

Complete the following for your own learning; there is no submission requirement.

- [ ] Created primitive and complex variables.
- [ ] Observed at least two validation failures.
- [ ] Used locals and conditional expressions.
- [ ] Applied a configuration using `for_each`.
- [ ] Enabled and disabled a resource using `count`.
- [ ] Inspected keyed and indexed resource addresses.
- [ ] Added a subnet and application tier.
- [ ] Observed the effect of changing a `for_each` key.
- [ ] Evaluated collection expressions in the Terraform console.
- [ ] Destroyed the local practice objects.

## 22. Reference documentation

- [Input variables](https://developer.hashicorp.com/terraform/language/values/variables)
- [Variable block and validation](https://developer.hashicorp.com/terraform/language/block/variable)
- [Expressions](https://developer.hashicorp.com/terraform/language/expressions)
- [Conditional expressions](https://developer.hashicorp.com/terraform/language/expressions/conditionals)
- [`for_each` reference](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each)
- [Meta-arguments, including `count`](https://developer.hashicorp.com/terraform/language/meta-arguments)
- [Dynamic blocks](https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks)
- [Validation mechanisms](https://developer.hashicorp.com/terraform/language/validate)
