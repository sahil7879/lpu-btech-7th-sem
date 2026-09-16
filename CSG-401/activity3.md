# Individual Activity 3: Terraform Modules and Composition with a Mocked Google Provider

**Course:** CSG401 - ACE Training  
**Main topic:** Organizing and reusing Terraform configuration with modules  
**Activity type:** Individual guided practical  
**Suggested duration:** 3 to 4 hours  
**GCP project required:** No  
**Billing account required:** No  
**Terraform version:** 1.7 or later

## 1. Purpose

In this activity, you will convert a repeated infrastructure design into reusable Terraform modules. A root module will compose three child modules and use each child module for both development and production environments.

You will create:

- A reusable `network` module
- A reusable `web_service` module
- A reusable `storage` module
- A root module that connects module outputs to module inputs
- Two module instances using `for_each`
- Automated tests using a mocked Google provider

No Google credentials or billable resources are required.

## 2. Learning outcomes

After completing the activity, you should be able to:

1. Distinguish a root module from a child module.
2. Define a clear module interface with variables and outputs.
3. Follow the standard `main.tf`, `variables.tf`, and `outputs.tf` module structure.
4. Reuse one module with different environment configurations.
5. Create multiple module instances with `for_each`.
6. Compose modules by passing one module's output into another module's input.
7. Explain provider inheritance and explicit provider mapping.
8. Test module behavior without creating Google Cloud resources.

## 3. Scenario

A college needs consistent development and production environments. Each environment requires:

- One custom-mode VPC
- One subnet with Private Google Access
- One internal firewall rule
- A configurable group of web VMs without public IP addresses
- One private, versioned Cloud Storage bucket

The development environment uses one small VM. Production uses two larger VMs. The same module source must implement both environments.

## 4. Module design

```text
Root module
|
+-- module.network["dev"]
|   `-- VPC, subnet, firewall
|
+-- module.web_service["dev"]
|   `-- VM instances using the dev subnet output
|
+-- module.storage["dev"]
|   `-- Private assets bucket
|
+-- module.network["prod"]
|   `-- VPC, subnet, firewall
|
+-- module.web_service["prod"]
|   `-- VM instances using the prod subnet output
|
`-- module.storage["prod"]
    `-- Private assets bucket
```

The root module coordinates the environment. Each child module owns one architectural responsibility.

## 5. Safety rules

1. Do not add Google credentials.
2. Do not use a real GCP project ID.
3. Confirm the test file contains `mock_provider "google" {}`.
4. Run plans only through `terraform test`.
5. Do not run `terraform apply` for this activity.
6. Mock tests validate configuration logic but do not reproduce GCP API behavior.

## 6. Create the directory structure

```text
terraform-modules-lab/
|-- main.tf
|-- variables.tf
|-- outputs.tf
|-- versions.tf
|-- terraform.tfvars
|-- modules/
|   |-- network/
|   |   |-- main.tf
|   |   |-- variables.tf
|   |   `-- outputs.tf
|   |-- web_service/
|   |   |-- main.tf
|   |   |-- variables.tf
|   |   `-- outputs.tf
|   `-- storage/
|       |-- main.tf
|       |-- variables.tf
|       `-- outputs.tf
`-- tests/
    `-- modules.tftest.hcl
```

Every directory containing `.tf` files is a separate Terraform module. Terraform does not automatically combine files from nested directories with the root module.

## 7. Step 1 - Configure Terraform and the provider

Create root `versions.tf`:

```hcl
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 7.0"
    }
  }
}
```

## 8. Step 2 - Define the network module interface

Create `modules/network/variables.tf`:

```hcl
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
    }
  }
}

variable "project_id" {
  description = "Project identifier supplied by the root module."
  type        = string
}

variable "region" {
  description = "Region for the subnetwork."
  type        = string
}

variable "name_prefix" {
  description = "Prefix applied to network resource names."
  type        = string
}

variable "subnet_cidr" {
  description = "IPv4 CIDR range for the environment subnet."
  type        = string

  validation {
    condition     = can(cidrhost(var.subnet_cidr, 1))
    error_message = "subnet_cidr must be a valid IPv4 CIDR range."
  }
}
```

Notice that the child module declares its provider requirement but does not contain a `provider "google"` configuration block. Provider configurations belong in the root module.

## 9. Step 3 - Implement the network module

Create `modules/network/main.tf`:

```hcl
resource "google_compute_network" "this" {
  project                 = var.project_id
  name                    = "${var.name_prefix}-vpc"
  auto_create_subnetworks = false
  routing_mode            = "REGIONAL"
}

resource "google_compute_subnetwork" "this" {
  project                  = var.project_id
  name                     = "${var.name_prefix}-subnet"
  region                   = var.region
  network                  = google_compute_network.this.id
  ip_cidr_range            = var.subnet_cidr
  private_ip_google_access = true
}

resource "google_compute_firewall" "allow_internal" {
  project = var.project_id
  name    = "${var.name_prefix}-allow-internal"
  network = google_compute_network.this.name

  direction     = "INGRESS"
  source_ranges = [var.subnet_cidr]

  allow {
    protocol = "tcp"
    ports    = ["22", "80"]
  }

  allow {
    protocol = "icmp"
  }
}
```

## 10. Step 4 - Publish network module outputs

Create `modules/network/outputs.tf`:

```hcl
output "network_name" {
  description = "Name of the VPC created by this module instance."
  value       = google_compute_network.this.name
}

output "subnetwork_name" {
  description = "Name passed to resources that need the environment subnet."
  value       = google_compute_subnetwork.this.name
}

output "subnetwork_cidr" {
  description = "CIDR range used by the subnet."
  value       = google_compute_subnetwork.this.ip_cidr_range
}

output "private_google_access" {
  description = "Whether Private Google Access is enabled."
  value       = google_compute_subnetwork.this.private_ip_google_access
}
```

These outputs form the module's public interface. The root module should consume outputs rather than reach into private resources inside the module.

## 11. Step 5 - Define the web service module interface

Create `modules/web_service/variables.tf`:

```hcl
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
    }
  }
}

variable "project_id" {
  description = "Project identifier supplied by the root module."
  type        = string
}

variable "zone" {
  description = "Zone for the web VM instances."
  type        = string
}

variable "name_prefix" {
  description = "Prefix applied to web resource names."
  type        = string
}

variable "subnetwork_name" {
  description = "Subnetwork received from the network module output."
  type        = string
}

variable "machine_type" {
  description = "Compute Engine machine type."
  type        = string
}

variable "instance_count" {
  description = "Number of web VM instances."
  type        = number

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 3
    error_message = "instance_count must be between 1 and 3."
  }
}
```

## 12. Step 6 - Implement the web service module

Create `modules/web_service/main.tf`:

```hcl
resource "google_service_account" "web" {
  project      = var.project_id
  account_id   = substr("${var.name_prefix}-web", 0, 30)
  display_name = "${var.name_prefix} web service"
}

resource "google_compute_instance" "web" {
  count = var.instance_count

  project      = var.project_id
  name         = "${var.name_prefix}-web-${count.index + 1}"
  machine_type = var.machine_type
  zone         = var.zone
  tags         = ["web"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 10
      type  = "pd-balanced"
    }
  }

  network_interface {
    subnetwork = var.subnetwork_name
  }

  service_account {
    email  = google_service_account.web.email
    scopes = ["cloud-platform"]
  }

  metadata = {
    enable-oslogin = "TRUE"
  }
}
```

The module accepts the subnet as an input. It does not create a network and does not know how the caller obtained the subnet name.

## 13. Step 7 - Publish web service outputs

Create `modules/web_service/outputs.tf`:

```hcl
output "instance_names" {
  description = "Names of all web instances created by this module."
  value       = google_compute_instance.web[*].name
}

output "instance_count" {
  description = "Number of web instances created by this module."
  value       = length(google_compute_instance.web)
}

output "machine_type" {
  description = "Machine type used by this module instance."
  value       = var.machine_type
}

output "subnetwork_name" {
  description = "Subnetwork supplied to this module instance."
  value       = var.subnetwork_name
}

output "has_external_ip" {
  description = "Whether any VM network interface defines access_config."
  value = anytrue([
    for instance in google_compute_instance.web :
    length(instance.network_interface[0].access_config) > 0
  ])
}
```

## 14. Step 8 - Define the storage module interface

Create `modules/storage/variables.tf`:

```hcl
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
    }
  }
}

variable "project_id" {
  description = "Project identifier supplied by the root module."
  type        = string
}

variable "bucket_name" {
  description = "Globally unique bucket name planned by the root module."
  type        = string
}

variable "location" {
  description = "Bucket location."
  type        = string
}

variable "environment" {
  description = "Environment label applied to the bucket."
  type        = string
}
```

## 15. Step 9 - Implement the storage module

Create `modules/storage/main.tf`:

```hcl
resource "google_storage_bucket" "this" {
  project                     = var.project_id
  name                        = var.bucket_name
  location                    = var.location
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"
  force_destroy               = false

  labels = {
    environment = var.environment
    managed_by  = "terraform"
  }

  versioning {
    enabled = true
  }
}
```

Create `modules/storage/outputs.tf`:

```hcl
output "bucket_name" {
  description = "Name of the bucket created by this module."
  value       = google_storage_bucket.this.name
}

output "public_access_prevention" {
  description = "Public access prevention setting."
  value       = google_storage_bucket.this.public_access_prevention
}

output "versioning_enabled" {
  description = "Whether object versioning is enabled."
  value       = google_storage_bucket.this.versioning[0].enabled
}
```

## 16. Step 10 - Define the root module inputs

Create root `variables.tf`:

```hcl
variable "project_id" {
  description = "Mock project identifier used only during tests."
  type        = string
  default     = "student-mock-project"
}

variable "environments" {
  description = "Configuration used to create one instance of each child module per environment."

  type = map(object({
    region         = string
    zone           = string
    subnet_cidr    = string
    machine_type   = string
    instance_count = number
  }))

  validation {
    condition = alltrue([
      for name in keys(var.environments) :
      contains(["dev", "test", "prod"], name)
    ])
    error_message = "Environment keys must be dev, test, or prod."
  }

  validation {
    condition = alltrue([
      for config in values(var.environments) :
      config.instance_count >= 1 && config.instance_count <= 3
    ])
    error_message = "Every environment must contain between 1 and 3 instances."
  }
}
```

The map key becomes the module instance key, such as `module.network["dev"]`.

## 17. Step 11 - Compose the child modules

Create root `main.tf`:

```hcl
provider "google" {
  project = var.project_id
}

module "network" {
  for_each = var.environments
  source   = "./modules/network"

  project_id  = var.project_id
  region      = each.value.region
  name_prefix = "${each.key}-modular"
  subnet_cidr = each.value.subnet_cidr

  providers = {
    google = google
  }
}

module "web_service" {
  for_each = var.environments
  source   = "./modules/web_service"

  project_id      = var.project_id
  zone            = each.value.zone
  name_prefix     = "${each.key}-modular"
  subnetwork_name = module.network[each.key].subnetwork_name
  machine_type    = each.value.machine_type
  instance_count  = each.value.instance_count

  providers = {
    google = google
  }
}

module "storage" {
  for_each = var.environments
  source   = "./modules/storage"

  project_id  = var.project_id
  bucket_name = "${var.project_id}-${each.key}-module-lab-assets"
  location    = each.value.region
  environment = each.key

  providers = {
    google = google
  }
}
```

The expression below creates an implicit dependency between two child-module instances:

```hcl
subnetwork_name = module.network[each.key].subnetwork_name
```

Terraform must obtain the network module's output before it can configure the web service module input.

## 18. Step 12 - Aggregate module outputs

Create root `outputs.tf`:

```hcl
output "environment_summary" {
  description = "Combined public interface for all environment module instances."

  value = {
    for name, config in var.environments : name => {
      network_name          = module.network[name].network_name
      subnetwork_name       = module.network[name].subnetwork_name
      subnetwork_cidr       = module.network[name].subnetwork_cidr
      private_google_access = module.network[name].private_google_access
      instance_names        = module.web_service[name].instance_names
      instance_count        = module.web_service[name].instance_count
      machine_type          = module.web_service[name].machine_type
      has_external_ip       = module.web_service[name].has_external_ip
      bucket_name           = module.storage[name].bucket_name
      public_access         = module.storage[name].public_access_prevention
      versioning_enabled    = module.storage[name].versioning_enabled
    }
  }
}
```

The root output hides the internal layout of the three child modules behind one environment-oriented interface.

## 19. Step 13 - Configure development and production

Create `terraform.tfvars`:

```hcl
project_id = "student-mock-project"

environments = {
  dev = {
    region         = "asia-south1"
    zone           = "asia-south1-a"
    subnet_cidr    = "10.30.1.0/24"
    machine_type   = "e2-micro"
    instance_count = 1
  }

  prod = {
    region         = "asia-south1"
    zone           = "asia-south1-b"
    subnet_cidr    = "10.30.2.0/24"
    machine_type   = "e2-small"
    instance_count = 2
  }
}
```

Both environments use the same module source. Their inputs create different module instances.

## 20. Step 14 - Test the composed modules

Create `tests/modules.tftest.hcl`:

```hcl
mock_provider "google" {}

run "reusable_environment_modules" {
  command = plan

  variables {
    project_id = "student-mock-project"

    environments = {
      dev = {
        region         = "asia-south1"
        zone           = "asia-south1-a"
        subnet_cidr    = "10.30.1.0/24"
        machine_type   = "e2-micro"
        instance_count = 1
      }

      prod = {
        region         = "asia-south1"
        zone           = "asia-south1-b"
        subnet_cidr    = "10.30.2.0/24"
        machine_type   = "e2-small"
        instance_count = 2
      }
    }
  }

  assert {
    condition     = toset(keys(output.environment_summary)) == toset(["dev", "prod"])
    error_message = "The root module must expose dev and prod environments."
  }

  assert {
    condition     = output.environment_summary.dev.network_name == "dev-modular-vpc"
    error_message = "The dev network module used the wrong name."
  }

  assert {
    condition     = output.environment_summary.prod.subnetwork_cidr == "10.30.2.0/24"
    error_message = "The prod CIDR was not passed through the network module."
  }

  assert {
    condition     = output.environment_summary.dev.instance_count == 1
    error_message = "The dev web module must create one planned VM."
  }

  assert {
    condition     = output.environment_summary.prod.instance_count == 2
    error_message = "The prod web module must create two planned VMs."
  }

  assert {
    condition     = output.environment_summary.prod.machine_type == "e2-small"
    error_message = "The prod module must receive the e2-small machine type."
  }

  assert {
    condition     = output.environment_summary.dev.subnetwork_name == module.web_service["dev"].subnetwork_name
    error_message = "The network output must be passed into the matching web module."
  }

  assert {
    condition     = output.environment_summary.dev.has_external_ip == false
    error_message = "The web module must not configure external IP addresses."
  }

  assert {
    condition     = output.environment_summary.prod.private_google_access == true
    error_message = "Private Google Access must be enabled by the network module."
  }

  assert {
    condition     = output.environment_summary.dev.public_access == "enforced"
    error_message = "The storage module must enforce public access prevention."
  }

  assert {
    condition     = output.environment_summary.prod.versioning_enabled == true
    error_message = "The storage module must enable object versioning."
  }
}

run "single_test_environment" {
  command = plan

  variables {
    environments = {
      test = {
        region         = "asia-south1"
        zone           = "asia-south1-c"
        subnet_cidr    = "10.30.3.0/24"
        machine_type   = "e2-micro"
        instance_count = 1
      }
    }
  }

  assert {
    condition     = length(output.environment_summary) == 1
    error_message = "One input map entry must create one instance of each child module."
  }

  assert {
    condition     = output.environment_summary.test.network_name == "test-modular-vpc"
    error_message = "The reusable network module must support the test environment."
  }
}

run "reject_unknown_environment" {
  command = plan

  variables {
    environments = {
      staging = {
        region         = "asia-south1"
        zone           = "asia-south1-a"
        subnet_cidr    = "10.30.4.0/24"
        machine_type   = "e2-micro"
        instance_count = 1
      }
    }
  }

  expect_failures = [var.environments]
}

run "reject_excessive_instance_count" {
  command = plan

  variables {
    environments = {
      prod = {
        region         = "asia-south1"
        zone           = "asia-south1-b"
        subnet_cidr    = "10.30.2.0/24"
        machine_type   = "e2-small"
        instance_count = 5
      }
    }
  }

  expect_failures = [var.environments]
}
```

## 21. Step 15 - Initialize, validate, and test

Run these commands from `terraform-modules-lab`:

```powershell
terraform init
terraform fmt -recursive
terraform fmt -check -recursive
terraform validate
terraform test
```

Expected test result:

```text
Success! 4 passed, 0 failed.
```

Inspect the module instances and simulated plan:

```powershell
terraform test -verbose
terraform graph
```

Find these module addresses in the output:

```text
module.network["dev"]
module.network["prod"]
module.web_service["dev"]
module.web_service["prod"]
module.storage["dev"]
module.storage["prod"]
```

## 22. Module usage scenarios

Modules solve different problems depending on who owns the infrastructure and how often it must be repeated. The following scenarios show where modules fit and how to use them.

### Scenario 1: Repeat the same architecture across environments

**Situation:** Development, testing, and production need the same network and application structure but different sizes and CIDR ranges.

**Use:** Call the same local module once for each environment. Pass environment-specific values as inputs.

```hcl
module "network" {
  for_each = var.environments
  source   = "./modules/network"

  name_prefix = "${each.key}-portal"
  region      = each.value.region
  subnet_cidr = each.value.subnet_cidr
  project_id  = var.project_id
}
```

Use this pattern when the architecture should remain consistent but capacity, naming, locations, or addresses vary.

### Scenario 2: Standardize infrastructure for several teams

**Situation:** Multiple development teams create Cloud Storage buckets. Security requires uniform bucket-level access, public access prevention, labels, and versioning on every bucket.

**Use:** Create an organization-owned storage module that contains the required controls. Teams supply only approved inputs.

```hcl
module "application_assets" {
  source = "git::https://example.com/platform/storage-module.git?ref=v2.1.0"

  project_id  = var.project_id
  bucket_name = var.assets_bucket_name
  location    = var.region
  environment = var.environment
}
```

This pattern turns organizational decisions into reusable code. Version the module so teams can upgrade deliberately.

### Scenario 3: Compose independent infrastructure layers

**Situation:** The network team owns VPC construction while the application team owns compute resources.

**Use:** Keep network and compute as separate modules. Pass the network module's output into the compute module.

```hcl
module "network" {
  source = "./modules/network"
  # Network inputs omitted
}

module "web_service" {
  source = "./modules/web_service"

  subnetwork_name = module.network.subnetwork_name
  # Other compute inputs omitted
}
```

The output-to-input reference documents the relationship and creates an implicit dependency. Prefer this flat composition over making the web module create its own network.

### Scenario 4: Create several similar components with `for_each`

**Situation:** One configuration needs a network module for `dev`, `test`, and `prod`, or a storage module for several applications.

**Use:** Supply a map to `for_each`. Terraform gives each module instance a stable key.

```hcl
module "storage" {
  for_each = var.environments
  source   = "./modules/storage"

  environment = each.key
  bucket_name = "${var.project_id}-${each.key}-assets"
  location    = each.value.region
  project_id  = var.project_id
}
```

The addresses become `module.storage["dev"]` and `module.storage["prod"]`. Stable keys are preferable to numeric indexes when environments may be added or removed.

### Scenario 5: Deploy the same module in several regions or projects

**Situation:** A service must be deployed in two regions or into separate development and production projects.

**Use:** Define aliased providers in the root module and explicitly map the appropriate provider into each module instance.

```hcl
provider "google" {
  alias   = "primary"
  project = var.primary_project_id
  region  = "asia-south1"
}

provider "google" {
  alias   = "secondary"
  project = var.secondary_project_id
  region  = "asia-southeast1"
}

module "primary_network" {
  source = "./modules/network"

  providers = {
    google = google.primary
  }

  project_id  = var.primary_project_id
  region      = "asia-south1"
  name_prefix = "primary"
  subnet_cidr = "10.40.1.0/24"
}

module "secondary_network" {
  source = "./modules/network"

  providers = {
    google = google.secondary
  }

  project_id  = var.secondary_project_id
  region      = "asia-southeast1"
  name_prefix = "secondary"
  subnet_cidr = "10.50.1.0/24"
}
```

Provider configurations remain in the root module. Child modules declare requirements and receive the selected configuration.

### Scenario 6: Make an architectural component optional

**Situation:** Production requires an additional module, but development does not.

**Use:** Put `for_each` or `count` on the module block rather than scattering conditional expressions throughout the child module.

```hcl
module "production_archive" {
  count  = var.environment == "prod" ? 1 : 0
  source = "./modules/storage"

  project_id  = var.project_id
  bucket_name = "${var.project_id}-production-archive"
  location    = var.region
  environment = var.environment
}
```

Because `count` changes the address, its outputs are accessed through an index, such as `module.production_archive[0].bucket_name`. Use this only when the whole component is optional.

### Scenario 7: Consume a registry module

**Situation:** A trusted and maintained module already implements the required architecture.

**Use:** Call the registry module and pin a compatible version range.

```hcl
module "example" {
  source  = "organization/module-name/google"
  version = "~> 3.2"

  # Module-specific inputs
}
```

Review the module source, inputs, outputs, provider requirements, release notes, and security posture before adoption. The `version` argument applies to registry modules, not local module paths.

### Scenario 8: Refactor existing resources into a module

**Situation:** A root configuration has become repetitive and resources should move into a child module without being recreated.

**Use:** Create the module, move the resource blocks, and add `moved` blocks that map old addresses to new module addresses.

```hcl
moved {
  from = google_compute_network.main
  to   = module.network.google_compute_network.this
}
```

Inspect the plan carefully. The expected plan should show address movement rather than destroy-and-create actions. Back up state and test the refactor in a controlled environment first.

### Scenario 9: Publish an internal module

**Situation:** A module has become a supported building block for other projects.

**Use:** Give it a focused purpose, document inputs and outputs, add examples and automated tests, tag releases, and publish it through a private registry or version-controlled source.

A publishable module normally includes:

```text
module-repository/
|-- README.md
|-- main.tf
|-- variables.tf
|-- outputs.tf
|-- versions.tf
|-- examples/
`-- tests/
```

Treat changes to required inputs, output names, resource addresses, and behavior as interface changes that can affect every consumer.

### Scenario 10: Avoid an unnecessary module

**Situation:** A proposed module only wraps one resource, renames all its arguments, and adds no validation, security policy, composition, or reusable behavior.

**Decision:** Use the resource directly. A thin wrapper increases indirection without creating a useful abstraction.

Create a module when it represents a meaningful architectural component, enforces standards, removes genuine repetition, or provides a stable interface. Do not create modules merely to reduce the number of resource blocks visible in the root directory.

### Scenario selection guide

| Need | Recommended approach |
|---|---|
| Same design with different values | Reuse one module with different inputs |
| Unknown or changing number of instances | Use `for_each` with stable keys |
| One optional component | Use `count` or filtered `for_each` on the module block |
| One module depends on another | Pass an output into an input |
| Different account, project, or region credentials | Map aliased providers from the root module |
| Organization-wide standards | Publish a versioned internal module |
| Move existing resources into a module | Use `moved` blocks and inspect the plan |
| Only one resource with no added behavior | Use the resource directly |

## 23. Module experiments

Complete the experiments individually and restore the working configuration afterward.

### Experiment 1: Add a test environment

Add a `test` entry to `terraform.tfvars`. Use CIDR `10.30.3.0/24`, zone `asia-south1-c`, one `e2-micro` instance, and the same region.

Predict which new module addresses will appear before running the tests.

### Experiment 2: Change one module instance

Change only the production `instance_count` from 2 to 3. Confirm that the development module instance remains unchanged.

### Experiment 3: Break module composition

Temporarily replace:

```hcl
subnetwork_name = module.network[each.key].subnetwork_name
```

with:

```hcl
subnetwork_name = "manually-typed-subnet"
```

Compare `terraform graph` before and after the change. Explain which implicit dependency disappeared.

### Experiment 4: Respect the module boundary

Try to reference this address from the root output:

```hcl
module.network["dev"].google_compute_network.this.name
```

Run `terraform validate`, interpret the error, and restore the output-based reference. A caller can use declared module outputs but cannot directly access resources inside a child module.

### Experiment 5: Remove an output

Temporarily remove `subnetwork_name` from `modules/network/outputs.tf`. Observe how the root module fails because the public interface changed.

### Experiment 6: Provider inheritance

Remove the three `providers` maps from the root module blocks. Run the tests. The default Google provider is inherited automatically. Explain when explicit provider mapping becomes necessary, such as selecting an aliased provider configuration.

## 24. Module design review

Answer these questions for your own revision:

1. What makes a directory a Terraform module?
2. Why should child modules declare provider requirements but not provider configurations?
3. How do variables and outputs form a module interface?
4. Why is a subnet output preferable to duplicating a subnet name in another module?
5. How does `for_each` change module addresses?
6. Why is a flat composition of small modules easier to reuse than a deeply nested module tree?
7. When would a module be an unnecessary wrapper?
8. Which module inputs should receive validation rules?
9. What changes when a local module is published to a registry?
10. What does the mocked provider test prove, and what still needs a real GCP integration test?

## 25. Individual completion checklist

No submission is required.

- [ ] I created the root module and all three child modules.
- [ ] Every child module has `main.tf`, `variables.tf`, and `outputs.tf`.
- [ ] Child modules declare provider requirements but contain no provider configuration blocks.
- [ ] The root module calls each child module with `for_each`.
- [ ] The web module receives its subnet from the network module output.
- [ ] Development and production use the same module sources with different inputs.
- [ ] Formatting and validation pass.
- [ ] All four mocked test runs pass.
- [ ] I completed at least four module experiments.
- [ ] I can explain module inputs, outputs, composition, reuse, and provider inheritance.

## 26. Limitations

The mocked provider validates HCL structure, provider schemas, module inputs, outputs, resource arguments, and assertions. It does not verify:

- GCP API enablement
- IAM permissions
- Quotas or regional capacity
- Network traffic or firewall enforcement
- VM boot behavior
- Bucket-name availability
- Billing or real service costs

Run a controlled GCP integration test before using the modules in production.

## 27. Reference documentation

- [Terraform modules overview](https://developer.hashicorp.com/terraform/language/modules)
- [Create reusable modules](https://developer.hashicorp.com/terraform/language/modules/develop)
- [Standard module structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure)
- [Module composition](https://developer.hashicorp.com/terraform/language/modules/develop/composition)
- [Provider configurations within modules](https://developer.hashicorp.com/terraform/language/modules/develop/providers)
- [Terraform provider mocking](https://developer.hashicorp.com/terraform/language/tests/mocking)
