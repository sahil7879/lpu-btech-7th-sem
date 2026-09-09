# Student Activity: Test Terraform in a GCP-Like Environment Without Billing

**Course:** CSG401 - ACE Training  
**Activity type:** Individual guided practical  
**Suggested duration:** 3 to 4 hours  
**Participation:** Individual  
**Cloud account required:** No  
**Billing account required:** No

## 1. Purpose

In this activity, you will design and test a small Google Cloud infrastructure configuration without creating real cloud resources. Terraform's mock-provider feature will simulate the Google provider's resource schema and computed values.

You will model:

- One custom-mode VPC network
- Two regional subnets
- An internal firewall rule
- A configurable set of Compute Engine VM instances
- Inputs, outputs, dependencies, modules, and automated tests

This lab tests infrastructure code and design decisions. It does **not** reproduce Google's real control plane, packet routing, IAM enforcement, quotas, VM boot process, or service availability.

## 2. Learning outcomes

After completing this activity, you should be able to:

1. Explain Infrastructure as Code and the Terraform workflow.
2. Write HCL for Google Cloud compute and networking resources.
3. Use variables, validation rules, locals, meta-arguments, dependencies, and outputs.
4. Organize reusable infrastructure as a Terraform module.
5. Run automated Terraform tests with a mocked Google provider.
6. Inspect and modify local Terraform state safely.
7. Evaluate an infrastructure design using Associate Cloud Engineer principles.

## 3. Safety rules

Follow these rules throughout the activity:

1. Use Terraform **1.7 or later**. Provider mocking is not available in older versions.
2. Do not add Google Cloud credentials.
3. Do not run `terraform apply` in the GCP mock project.
4. Run GCP checks only through `terraform test`.
5. The only `terraform apply` in this activity is inside the separate `state-sandbox` folder. It uses Terraform's built-in `terraform_data` resource and creates no cloud resources.
6. Confirm that the test file contains `mock_provider "google" {}` before running the tests.

> **Important:** `terraform test` can create real resources when a real provider is used. This activity is billing-free only because the Google provider is mocked.

## 4. Prerequisites

Install the following software:

- Terraform CLI 1.7 or later
- Visual Studio Code or another text editor
- Internet access for the initial provider download

Verify Terraform:

```powershell
terraform version
```

The displayed version must be 1.7.0 or newer.

## 5. Scenario

Your college is preparing a small web application environment on Google Cloud. The proposed development environment has these requirements:

- A custom VPC named from the environment.
- A web subnet and an application subnet in one region.
- No automatically created subnetworks.
- Internal communication allowed only within `10.10.0.0/16`.
- Two web VMs by default.
- VM count must be configurable from 1 to 5.
- Environment must be one of `dev`, `test`, or `prod`.
- Resource names must include the environment.
- Useful resource attributes must be exposed as outputs.
- The design must be reusable as a module.
- Tests must not contact Google Cloud or require credentials.

## 6. Target architecture

```text
Mocked Google Cloud project
|
+-- VPC: <environment>-student-vpc
    |
    +-- Web subnet: 10.10.1.0/24
    |   |
    |   +-- Web VM 1
    |   +-- Web VM 2
    |
    +-- Application subnet: 10.10.2.0/24
    |
    +-- Internal firewall rule: source 10.10.0.0/16
```

## 7. Create the project structure

Create the following folders and empty files:

```text
terraform-gcp-mock-lab/
|-- main.tf
|-- variables.tf
|-- outputs.tf
|-- versions.tf
|-- terraform.tfvars
|-- modules/
|   `-- student_environment/
|       |-- main.tf
|       |-- variables.tf
|       `-- outputs.tf
|-- tests/
|   `-- environment.tftest.hcl
`-- state-sandbox/
    |-- main.tf
    `-- variables.tf
```

Open a terminal in `terraform-gcp-mock-lab`.

## 8. Part A - Build the mocked GCP environment

### Step 1: Declare the Terraform and provider versions

Add the following to `versions.tf`:

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

**Checkpoint:** Identify why both Terraform and provider versions are constrained.

### Step 2: Declare root input variables

Add the following to `variables.tf`:

```hcl
variable "project_id" {
  description = "A non-production project identifier used only by the mock test."
  type        = string
  default     = "student-mock-project"
}

variable "region" {
  description = "Region used in the simulated design."
  type        = string
  default     = "asia-south1"
}

variable "zone" {
  description = "Zone used for the simulated VM instances."
  type        = string
  default     = "asia-south1-a"
}

variable "environment" {
  description = "Deployment environment name."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be dev, test, or prod."
  }
}

variable "vm_count" {
  description = "Number of simulated web VMs."
  type        = number
  default     = 2

  validation {
    condition     = var.vm_count >= 1 && var.vm_count <= 5
    error_message = "vm_count must be between 1 and 5."
  }
}
```

### Step 3: Call a reusable module

Add the following to the root `main.tf`:

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

module "student_environment" {
  source = "./modules/student_environment"

  project_id  = var.project_id
  region      = var.region
  zone        = var.zone
  environment = var.environment
  vm_count    = var.vm_count
}
```

**Checkpoint:** The root module describes *which* environment to create. The child module describes *how* to create it.

### Step 4: Declare module variables

Add the following to `modules/student_environment/variables.tf`:

```hcl
variable "project_id" {
  type = string
}

variable "region" {
  type = string
}

variable "zone" {
  type = string
}

variable "environment" {
  type = string
}

variable "vm_count" {
  type = number
}
```

### Step 5: Define the network resources

Add the following to `modules/student_environment/main.tf`:

```hcl
locals {
  name_prefix = "${var.environment}-student"

  common_labels = {
    environment = var.environment
    managed_by  = "terraform"
    course      = "csg401"
  }
}

resource "google_compute_network" "main" {
  project                 = var.project_id
  name                    = "${local.name_prefix}-vpc"
  auto_create_subnetworks = false
  routing_mode            = "REGIONAL"
}

resource "google_compute_subnetwork" "web" {
  project       = var.project_id
  name          = "${local.name_prefix}-web-subnet"
  region        = var.region
  network       = google_compute_network.main.name
  ip_cidr_range = "10.10.1.0/24"
}

resource "google_compute_subnetwork" "app" {
  project       = var.project_id
  name          = "${local.name_prefix}-app-subnet"
  region        = var.region
  network       = google_compute_network.main.name
  ip_cidr_range = "10.10.2.0/24"
}

resource "google_compute_firewall" "allow_internal" {
  project = var.project_id
  name    = "${local.name_prefix}-allow-internal"
  network = google_compute_network.main.name

  direction     = "INGRESS"
  source_ranges = ["10.10.0.0/16"]

  allow {
    protocol = "tcp"
  }

  allow {
    protocol = "icmp"
  }
}
```

**Checkpoint:** Locate the implicit dependencies between the VPC, subnets, and firewall rule.

### Step 6: Define configurable VM instances

Append the following to `modules/student_environment/main.tf`:

```hcl
resource "google_compute_instance" "web" {
  count = var.vm_count

  project      = var.project_id
  name         = "${local.name_prefix}-web-${count.index + 1}"
  machine_type = "e2-micro"
  zone         = var.zone
  labels       = local.common_labels

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 10
      type  = "pd-balanced"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.web.name
  }

  metadata = {
    enable-oslogin = "TRUE"
  }
}
```

Notice that there is no `access_config` block. In a real deployment, that means the VMs are not assigned ephemeral external IPv4 addresses through this interface.

### Step 7: Define module outputs

Add the following to `modules/student_environment/outputs.tf`:

```hcl
output "network_name" {
  description = "Name of the simulated VPC."
  value       = google_compute_network.main.name
}

output "subnet_names" {
  description = "Names of the simulated subnets."
  value = {
    web = google_compute_subnetwork.web.name
    app = google_compute_subnetwork.app.name
  }
}

output "vm_names" {
  description = "Names of the simulated web VMs."
  value       = google_compute_instance.web[*].name
}
```

### Step 8: Pass module attributes to root outputs

Add the following to the root `outputs.tf`:

```hcl
output "network_name" {
  description = "Name of the simulated VPC."
  value       = module.student_environment.network_name
}

output "subnet_names" {
  description = "Names of the simulated subnets."
  value       = module.student_environment.subnet_names
}

output "vm_names" {
  description = "Names of the simulated web VMs."
  value       = module.student_environment.vm_names
}
```

### Step 9: Supply development values

Add the following to `terraform.tfvars`:

```hcl
project_id  = "student-mock-project"
region      = "asia-south1"
zone        = "asia-south1-a"
environment = "dev"
vm_count    = 2
```

The project ID is only a string used by the mock test. Do not replace it with a real project ID.

## 9. Part B - Write automated infrastructure tests

Add the following to `tests/environment.tftest.hcl`:

```hcl
mock_provider "google" {}

run "development_environment" {
  command = plan

  variables {
    project_id  = "student-mock-project"
    region      = "asia-south1"
    zone        = "asia-south1-a"
    environment = "dev"
    vm_count    = 2
  }

  assert {
    condition     = module.student_environment.network_name == "dev-student-vpc"
    error_message = "The VPC name must contain the dev environment prefix."
  }

  assert {
    condition     = length(module.student_environment.vm_names) == 2
    error_message = "The development environment must contain two web VMs."
  }

  assert {
    condition     = module.student_environment.subnet_names.web == "dev-student-web-subnet"
    error_message = "The web subnet name is incorrect."
  }

  assert {
    condition     = module.student_environment.subnet_names.app == "dev-student-app-subnet"
    error_message = "The application subnet name is incorrect."
  }

  assert {
    condition     = google_compute_network.main.auto_create_subnetworks == false
    error_message = "The VPC must not create automatic subnetworks."
  }
}

run "production_scale" {
  command = plan

  variables {
    project_id  = "student-mock-project"
    region      = "asia-south1"
    zone        = "asia-south1-a"
    environment = "prod"
    vm_count    = 5
  }

  assert {
    condition     = length(module.student_environment.vm_names) == 5
    error_message = "The production test must create five planned VM instances."
  }

  assert {
    condition     = startswith(module.student_environment.network_name, "prod-")
    error_message = "Production resource names must start with prod-."
  }
}

run "reject_invalid_environment" {
  command = plan

  variables {
    environment = "qa"
  }

  expect_failures = [var.environment]
}

run "reject_excessive_vm_count" {
  command = plan

  variables {
    vm_count = 6
  }

  expect_failures = [var.vm_count]
}
```

### Correction challenge

One assertion above refers to `google_compute_network.main` as though the resource were in the root module. It is actually inside the child module. Terraform should report an undeclared-resource error.

Replace that assertion with this output-based assertion:

```hcl
assert {
  condition     = module.student_environment.network_name == "dev-student-vpc"
  error_message = "The expected custom VPC was not planned."
}
```

Explain why a root module cannot directly access a child module's private resources and why outputs form the module's public interface.

## 10. Part C - Initialize, format, validate, and test

Run these commands from `terraform-gcp-mock-lab`:

### Step 1: Initialize the working directory

```powershell
terraform init
```

Expected result: Terraform downloads the Google provider and creates `.terraform.lock.hcl`. It does not create a GCP project or contact a project API.

### Step 2: Check formatting

```powershell
terraform fmt -check -recursive
```

If files need formatting, run:

```powershell
terraform fmt -recursive
```

Then repeat the check.

### Step 3: Validate the configuration

```powershell
terraform validate
```

Expected result after completing the correction challenge: `Success! The configuration is valid.`

### Step 4: Run the mocked tests

```powershell
terraform test
```

Expected result: all four test runs pass, and no Google credentials are requested.

### Step 5: Inspect the simulated plan

```powershell
terraform test -verbose
```

In the output, locate:

- The VPC and its custom subnet mode
- Both subnet CIDR ranges
- The firewall direction and source range
- Two development VM instances
- The labels applied to each VM
- The absence of an external-IP `access_config` block

### Step 6: Review the dependency graph

```powershell
terraform graph
```

Optional: copy the output into an online or locally installed Graphviz viewer. Identify which expressions created implicit dependencies.

## 11. Part D - Safe local state exercise

This part uses Terraform's built-in `terraform_data` resource. It creates local state only and does not use Google Cloud.

### Step 1: Create the state sandbox configuration

Add the following to `state-sandbox/variables.tf`:

```hcl
variable "environment" {
  type    = string
  default = "dev"
}
```

Add the following to `state-sandbox/main.tf`:

```hcl
terraform {
  required_version = ">= 1.7.0"
}

resource "terraform_data" "environment_record" {
  input = {
    environment  = var.environment
    network_name = "${var.environment}-student-vpc"
    subnet_count = 2
    vm_count     = 2
  }
}

output "environment_record" {
  value = terraform_data.environment_record.output
}
```

### Step 2: Initialize and create local state

```powershell
cd state-sandbox
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

This apply operation records a built-in data resource in a local state file. It does not call a cloud API.

### Step 3: Inspect state

```powershell
terraform state list
terraform state show terraform_data.environment_record
terraform output
```

Answer:

1. What is the resource address?
2. Which attributes are stored in state?
3. Why can state contain sensitive information in a real project?

### Step 4: Observe a controlled change

```powershell
terraform plan -var='environment=test'
```

Record what Terraform proposes to change. Then apply the local-only change:

```powershell
terraform apply -var='environment=test'
```

### Step 5: Remove the sandbox resource

```powershell
terraform destroy -var='environment=test'
```

Confirm the prompt only when the plan shows `terraform_data.environment_record`. Afterward, run:

```powershell
terraform state list
cd ..
```

Expected result: the state contains no managed resources.

## 12. Part E - Troubleshooting challenges

Complete any three challenges. For each one, capture the error, explain its cause, and document the fix.

### Challenge 1: Invalid environment

Change `environment` to `qa`. Determine which validation rule rejects it.

### Challenge 2: Unsafe VM scale

Change `vm_count` to `10`. Explain why early variable validation is useful.

### Challenge 3: Broken dependency

Replace the web subnet reference in the VM with a misspelled resource name. Interpret Terraform's error and restore the correct reference.

### Challenge 4: Overlapping subnets

Change the application subnet to `10.10.1.0/24`. Does the mock provider detect the overlap? Explain why a schema-aware mock cannot reproduce every server-side GCP validation.

### Challenge 5: Public VM design

Research what adding an `access_config {}` block would do. Do not add it permanently. Explain the security and cost considerations of external IP addresses.

### Challenge 6: Module boundary

Try to reference a child-module resource directly from the root test. Explain why only module outputs should be used as the public contract.

## 13. Part F - ACE design review

Answer the following questions in 2 to 4 sentences each:

1. Why is a custom-mode VPC preferred here over an auto-mode VPC?
2. Is allowing every TCP port from `10.10.0.0/16` least privilege? Propose a safer rule.
3. How would administrators reach private VMs in a real project without assigning public IP addresses?
4. Why should a production VM use a dedicated service account rather than the default Compute Engine service account?
5. Where should production Terraform state be stored, and how should access be controlled?
6. What GCP features cannot be proven by this mock test?
7. What additional integration test would you run in a billing-controlled GCP sandbox before production?

## 14. Individual completion checklist

Use this checklist to review your own work. No submission is required.

- [ ] I created the complete root-module and child-module structure.
- [ ] I ran `terraform fmt -check -recursive` successfully.
- [ ] I ran `terraform validate` successfully.
- [ ] I corrected the intentional module-boundary error.
- [ ] I ran all mocked tests successfully without Google credentials.
- [ ] I inspected the verbose simulated plan.
- [ ] I identified the implicit dependencies in the resource graph.
- [ ] I completed the local-only state exercise and destroyed its resource.
- [ ] I attempted at least three troubleshooting challenges.
- [ ] I answered the ACE design-review questions for my own revision.
- [ ] I can explain what this mock test proves and what it cannot prove.

Keep credentials, service-account keys, access tokens, real project IDs, `.terraform/` provider binaries, and state files out of any shared location.

## 15. Instructor notes

- Deliberately keep the incorrect private-resource assertion in the first test when distributing the activity. It provides a guided troubleshooting event.
- Ask students to prove that they have no Google credentials configured or use a controlled lab machine.
- The Google provider must still be downloaded during `terraform init`; cache it on lab machines if student internet access is restricted.
- Do not describe mocked tests as a GCP emulator. They validate Terraform configuration against provider schemas and simulate computed values, but they do not run Google Cloud APIs.
- Use a separate, policy-restricted GCP sandbox only for a later integration lab requiring real VPC, IAM, VM, quota, or networking behavior.

## 16. Syllabus alignment

| CSG401 topic | Activity evidence |
|---|---|
| Infrastructure as Code and Terraform overview | Purpose, scenario, and workflow |
| Author phase and Terraform commands | Authoring tasks; init, fmt, validate, test, graph |
| Terraform Validator concepts | Variable validation and automated assertions |
| Resources and resource blocks | VPC, subnets, firewall, and VM definitions |
| Variables and best practices | Typed, described, validated inputs |
| Meta-arguments | Configurable VM creation with `count` |
| Resource dependencies | References between network, subnet, firewall, and VM |
| Outputs and best practices | Child-module and root output contracts |
| Modules and reuse | Parameterized `student_environment` module |
| Terraform state | Local-only state sandbox |
| ACE projects, IAM, compute, storage, networking, operations | Design-review questions and stated mock limitations |

## 17. Reference documentation

- [Terraform provider mocking](https://developer.hashicorp.com/terraform/language/tests/mocking)
- [Terraform test command](https://developer.hashicorp.com/terraform/cli/commands/test)
- [Terraform testing overview](https://developer.hashicorp.com/terraform/cli/test)
- [Google provider documentation](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
