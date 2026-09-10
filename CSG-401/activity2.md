# Individual Activity 2: Test an Autoscaled GCP Web Platform with Terraform Mock Providers

**Course:** CSG401 - ACE Training  
**Activity type:** Individual guided practical  
**Suggested duration:** 3 hours  
**GCP project required:** No  
**Billing account required:** No  
**Terraform version:** 1.7 or later

## 1. Purpose

In this activity, you will use Terraform to design and test a production-style web platform for Google Cloud. Terraform's mocked Google provider will validate the resource configuration without creating infrastructure or requiring Google credentials.

The planned environment contains:

- A custom VPC and regional subnet
- A dedicated service account for web instances
- A private Compute Engine instance template
- A managed instance group
- CPU-based autoscaling
- A load-balancer health check
- An external Application Load Balancer configuration
- A protected Cloud Storage bucket
- A least-privilege bucket IAM binding
- Automated positive and negative tests

The activity tests Terraform configuration and architecture decisions. It does not emulate real VM startup, autoscaling decisions, IAM enforcement, health-check traffic, or Google Front End behavior.

## 2. Learning outcomes

After completing this activity, you should be able to:

1. Model a multi-resource Google Cloud architecture in HCL.
2. Use an instance template and managed instance group.
3. Configure autoscaling boundaries and health checks.
4. Express resource dependencies through references.
5. Apply basic security controls to compute and storage resources.
6. Validate infrastructure requirements with `terraform test`.
7. Explain the difference between a mocked plan and a real integration test.

## 3. Scenario

A college portal must run as a scalable web application. The proposed design has these requirements:

- Resources must use the environment name as a prefix.
- The VPC must use custom subnet mode.
- Web instances must not receive external IP addresses.
- The managed instance group must start at its minimum size.
- Autoscaling must stay between 2 and 6 instances by default.
- CPU utilization must trigger scaling at 60 percent.
- The load balancer must check `/health` on TCP port 80.
- Only Google health-check address ranges may reach TCP port 80 on tagged instances.
- The assets bucket must block public access and enable versioning.
- The web service account may read objects but must not administer the bucket.

## 4. Safety rules

1. Do not configure Google credentials for this activity.
2. Do not replace the mock project ID with a real project ID.
3. Confirm that `tests/platform.tftest.hcl` contains `mock_provider "google" {}`.
4. Run the infrastructure plan only through `terraform test`.
5. Do not run `terraform apply` from this project.
6. Remember that `terraform test` can create real infrastructure when tests use a real provider.

## 5. Project structure

Create this folder structure:

```text
terraform-gcp-mock-platform/
|-- main.tf
|-- variables.tf
|-- outputs.tf
|-- versions.tf
|-- terraform.tfvars
`-- tests/
    `-- platform.tftest.hcl
```

Open a terminal inside `terraform-gcp-mock-platform`.

## 6. Step 1 - Declare Terraform and the Google provider

Create `versions.tf`:

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

This constraint permits compatible Google provider releases in the 7.x series while preventing an automatic upgrade to a future major version.

## 7. Step 2 - Declare and validate inputs

Create `variables.tf`:

```hcl
variable "project_id" {
  description = "Mock project identifier used only for configuration testing."
  type        = string
  default     = "student-mock-project"
}

variable "region" {
  description = "Region represented by the architecture."
  type        = string
  default     = "asia-south1"
}

variable "zone" {
  description = "Zone represented by the managed instance group."
  type        = string
  default     = "asia-south1-a"
}

variable "environment" {
  description = "Environment included in resource names."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be dev, test, or prod."
  }
}

variable "machine_type" {
  description = "Machine type used by the instance template."
  type        = string
  default     = "e2-micro"

  validation {
    condition     = contains(["e2-micro", "e2-small", "e2-medium"], var.machine_type)
    error_message = "machine_type must be e2-micro, e2-small, or e2-medium."
  }
}

variable "scaling" {
  description = "Autoscaling boundaries and target CPU utilization."
  type = object({
    min_instances = number
    max_instances = number
    target_cpu    = number
  })

  default = {
    min_instances = 2
    max_instances = 6
    target_cpu    = 0.60
  }

  validation {
    condition = (
      var.scaling.min_instances >= 2 &&
      var.scaling.max_instances <= 10 &&
      var.scaling.max_instances >= var.scaling.min_instances &&
      var.scaling.target_cpu >= 0.40 &&
      var.scaling.target_cpu <= 0.80
    )
    error_message = "Scaling requires min >= 2, max <= 10, max >= min, and target_cpu between 0.40 and 0.80."
  }
}
```

### Checkpoint

Explain why the scaling configuration uses one object instead of three unrelated variables.

## 8. Step 3 - Configure naming and the provider

Start `main.tf` with:

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

locals {
  prefix = "${var.environment}-college-portal"

  common_labels = {
    environment = var.environment
    application = "college-portal"
    managed_by  = "terraform"
  }
}
```

The provider configuration contains no credential block. The mock provider in the test file will replace it during test runs.

## 9. Step 4 - Create the network design

Append to `main.tf`:

```hcl
resource "google_compute_network" "main" {
  project                 = var.project_id
  name                    = "${local.prefix}-vpc"
  auto_create_subnetworks = false
  routing_mode            = "REGIONAL"
}

resource "google_compute_subnetwork" "web" {
  project                  = var.project_id
  name                     = "${local.prefix}-web-subnet"
  region                   = var.region
  network                  = google_compute_network.main.id
  ip_cidr_range            = "10.20.1.0/24"
  private_ip_google_access = true
}

resource "google_compute_firewall" "allow_health_checks" {
  project = var.project_id
  name    = "${local.prefix}-allow-health-checks"
  network = google_compute_network.main.name

  direction     = "INGRESS"
  source_ranges = ["35.191.0.0/16", "130.211.0.0/22"]
  target_tags   = ["web-backend"]

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }
}
```

### Checkpoint

Identify the references that create implicit dependencies on the VPC.

## 10. Step 5 - Create a dedicated service account

Append to `main.tf`:

```hcl
resource "google_service_account" "web" {
  project      = var.project_id
  account_id   = "${var.environment}-portal-web"
  display_name = "${var.environment} college portal web service"
}
```

A dedicated service account lets administrators grant only the permissions required by the application.

## 11. Step 6 - Create the instance template

Append to `main.tf`:

```hcl
resource "google_compute_instance_template" "web" {
  project      = var.project_id
  name_prefix  = "${local.prefix}-template-"
  machine_type = var.machine_type
  region       = var.region
  tags         = ["web-backend"]
  labels       = local.common_labels

  disk {
    source_image = "debian-cloud/debian-12"
    auto_delete  = true
    boot         = true
    disk_size_gb = 10
    disk_type    = "pd-balanced"
  }

  network_interface {
    subnetwork = google_compute_subnetwork.web.id
  }

  service_account {
    email  = google_service_account.web.email
    scopes = ["cloud-platform"]
  }

  metadata = {
    enable-oslogin = "TRUE"
  }

  metadata_startup_script = <<-EOT
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    echo "healthy" > /var/www/html/health
    systemctl enable --now nginx
  EOT

  lifecycle {
    create_before_destroy = true
  }
}
```

The `network_interface` block deliberately omits `access_config`. The planned instances therefore have no external IPv4 address on this interface.

The broad `cloud-platform` OAuth scope does not itself grant IAM permissions. The attached service account's IAM roles determine access.

## 12. Step 7 - Configure the managed instance group

Append to `main.tf`:

```hcl
resource "google_compute_instance_group_manager" "web" {
  project            = var.project_id
  name               = "${local.prefix}-mig"
  base_instance_name = "${local.prefix}-web"
  zone               = var.zone
  target_size        = var.scaling.min_instances

  version {
    instance_template = google_compute_instance_template.web.id
  }

  named_port {
    name = "http"
    port = 80
  }

  update_policy {
    type                         = "PROACTIVE"
    minimal_action               = "REPLACE"
    max_surge_fixed              = 1
    max_unavailable_fixed        = 0
    replacement_method           = "SUBSTITUTE"
    most_disruptive_allowed_action = "REPLACE"
  }
}

resource "google_compute_autoscaler" "web" {
  project = var.project_id
  name    = "${local.prefix}-autoscaler"
  zone    = var.zone
  target  = google_compute_instance_group_manager.web.id

  autoscaling_policy {
    min_replicas    = var.scaling.min_instances
    max_replicas    = var.scaling.max_instances
    cooldown_period = 60

    cpu_utilization {
      target = var.scaling.target_cpu
    }
  }
}
```

### Checkpoint

Explain why `create_before_destroy` is useful for an immutable instance template used by a managed instance group.

## 13. Step 8 - Configure health checking and load balancing

Append to `main.tf`:

```hcl
resource "google_compute_health_check" "web" {
  project             = var.project_id
  name                = "${local.prefix}-health-check"
  check_interval_sec  = 10
  timeout_sec         = 5
  healthy_threshold   = 2
  unhealthy_threshold = 3

  http_health_check {
    port         = 80
    request_path = "/health"
  }
}

resource "google_compute_backend_service" "web" {
  project               = var.project_id
  name                  = "${local.prefix}-backend"
  protocol              = "HTTP"
  port_name             = "http"
  timeout_sec           = 30
  load_balancing_scheme = "EXTERNAL_MANAGED"
  health_checks         = [google_compute_health_check.web.id]

  backend {
    group           = google_compute_instance_group_manager.web.instance_group
    balancing_mode  = "UTILIZATION"
    max_utilization = 0.80
  }
}

resource "google_compute_url_map" "web" {
  project         = var.project_id
  name            = "${local.prefix}-url-map"
  default_service = google_compute_backend_service.web.id
}

resource "google_compute_target_http_proxy" "web" {
  project = var.project_id
  name    = "${local.prefix}-http-proxy"
  url_map = google_compute_url_map.web.id
}

resource "google_compute_global_address" "web" {
  project = var.project_id
  name    = "${local.prefix}-ip"
}

resource "google_compute_global_forwarding_rule" "web" {
  project               = var.project_id
  name                  = "${local.prefix}-forwarding-rule"
  ip_address            = google_compute_global_address.web.address
  port_range            = "80"
  target                = google_compute_target_http_proxy.web.id
  load_balancing_scheme = "EXTERNAL_MANAGED"
}
```

This exercise uses HTTP to keep the resource graph focused. A real production deployment should normally add a managed certificate, an HTTPS proxy, port 443, and HTTP-to-HTTPS redirection.

## 14. Step 9 - Add protected object storage and IAM

Append to `main.tf`:

```hcl
resource "google_storage_bucket" "assets" {
  project                     = var.project_id
  name                        = "${var.project_id}-${var.environment}-portal-assets"
  location                    = var.region
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"
  force_destroy               = false
  labels                      = local.common_labels

  versioning {
    enabled = true
  }

  lifecycle_rule {
    condition {
      age            = 30
      with_state     = "ARCHIVED"
      matches_prefix = ["temporary/"]
    }

    action {
      type = "Delete"
    }
  }
}

resource "google_storage_bucket_iam_member" "web_asset_reader" {
  bucket = google_storage_bucket.assets.name
  role   = "roles/storage.objectViewer"
  member = "serviceAccount:${google_service_account.web.email}"
}
```

### Checkpoint

Explain why `roles/storage.objectViewer` is safer than `roles/storage.admin` for a web application that only reads assets.

## 15. Step 10 - Define useful outputs

Create `outputs.tf`:

```hcl
output "network_name" {
  description = "Name of the planned VPC."
  value       = google_compute_network.main.name
}

output "instance_group_name" {
  description = "Name of the planned managed instance group."
  value       = google_compute_instance_group_manager.web.name
}

output "autoscaling_range" {
  description = "Minimum and maximum planned instance counts."
  value = {
    minimum = google_compute_autoscaler.web.autoscaling_policy[0].min_replicas
    maximum = google_compute_autoscaler.web.autoscaling_policy[0].max_replicas
  }
}

output "health_check_path" {
  description = "HTTP path checked by the load balancer."
  value       = google_compute_health_check.web.http_health_check[0].request_path
}

output "assets_bucket_name" {
  description = "Name of the planned private assets bucket."
  value       = google_storage_bucket.assets.name
}

output "load_balancer_ip" {
  description = "Mock-generated load balancer IP during tests."
  value       = google_compute_global_address.web.address
}
```

## 16. Step 11 - Supply development values

Create `terraform.tfvars`:

```hcl
project_id   = "student-mock-project"
region       = "asia-south1"
zone         = "asia-south1-a"
environment  = "dev"
machine_type = "e2-micro"

scaling = {
  min_instances = 2
  max_instances = 6
  target_cpu    = 0.60
}
```

## 17. Step 12 - Write mock-provider tests

Create `tests/platform.tftest.hcl`:

```hcl
mock_provider "google" {}

run "development_platform" {
  command = plan

  variables {
    project_id   = "student-mock-project"
    region       = "asia-south1"
    zone         = "asia-south1-a"
    environment  = "dev"
    machine_type = "e2-micro"

    scaling = {
      min_instances = 2
      max_instances = 6
      target_cpu    = 0.60
    }
  }

  assert {
    condition     = google_compute_network.main.auto_create_subnetworks == false
    error_message = "The VPC must use custom subnet mode."
  }

  assert {
    condition     = google_compute_subnetwork.web.private_ip_google_access == true
    error_message = "Private Google Access must be enabled on the web subnet."
  }

  assert {
    condition     = google_compute_instance_template.web.machine_type == "e2-micro"
    error_message = "The development instance template must use e2-micro."
  }

  assert {
    condition     = length(google_compute_instance_template.web.network_interface[0].access_config) == 0
    error_message = "Web instances must not receive an external IP address."
  }

  assert {
    condition     = google_compute_instance_group_manager.web.target_size == 2
    error_message = "The managed instance group must start at the minimum size."
  }

  assert {
    condition     = google_compute_autoscaler.web.autoscaling_policy[0].min_replicas == 2
    error_message = "The autoscaler minimum must be two instances."
  }

  assert {
    condition     = google_compute_autoscaler.web.autoscaling_policy[0].max_replicas == 6
    error_message = "The autoscaler maximum must be six instances."
  }

  assert {
    condition     = google_compute_autoscaler.web.autoscaling_policy[0].cpu_utilization[0].target == 0.60
    error_message = "The autoscaler CPU target must be 60 percent."
  }

  assert {
    condition     = google_compute_health_check.web.http_health_check[0].request_path == "/health"
    error_message = "The load balancer must check the /health path."
  }

  assert {
    condition     = toset(google_compute_firewall.allow_health_checks.source_ranges) == toset(["35.191.0.0/16", "130.211.0.0/22"])
    error_message = "The firewall must use the documented health-check source ranges."
  }

  assert {
    condition     = google_storage_bucket.assets.public_access_prevention == "enforced"
    error_message = "Public access prevention must be enforced."
  }

  assert {
    condition     = google_storage_bucket.assets.versioning[0].enabled == true
    error_message = "Object versioning must be enabled."
  }

  assert {
    condition     = google_storage_bucket_iam_member.web_asset_reader.role == "roles/storage.objectViewer"
    error_message = "The web service account must receive only object-viewer access."
  }
}

run "production_platform" {
  command = plan

  variables {
    environment  = "prod"
    machine_type = "e2-medium"

    scaling = {
      min_instances = 3
      max_instances = 10
      target_cpu    = 0.55
    }
  }

  assert {
    condition     = startswith(google_compute_network.main.name, "prod-")
    error_message = "Production resource names must start with prod-."
  }

  assert {
    condition     = google_compute_instance_template.web.machine_type == "e2-medium"
    error_message = "The production test must use e2-medium."
  }

  assert {
    condition     = google_compute_autoscaler.web.autoscaling_policy[0].max_replicas == 10
    error_message = "The production test must permit up to ten instances."
  }
}

run "reject_invalid_scaling_order" {
  command = plan

  variables {
    scaling = {
      min_instances = 6
      max_instances = 3
      target_cpu    = 0.60
    }
  }

  expect_failures = [var.scaling]
}

run "reject_unsafe_cpu_target" {
  command = plan

  variables {
    scaling = {
      min_instances = 2
      max_instances = 6
      target_cpu    = 0.95
    }
  }

  expect_failures = [var.scaling]
}

run "reject_unapproved_machine_type" {
  command = plan

  variables {
    machine_type = "n2-standard-32"
  }

  expect_failures = [var.machine_type]
}
```

## 18. Step 13 - Initialize and test

Run these commands from `terraform-gcp-mock-platform`:

### Initialize the directory

```powershell
terraform init
```

Terraform downloads the Google provider schema. It does not create GCP resources.

### Format the configuration

```powershell
terraform fmt -recursive
terraform fmt -check -recursive
```

### Validate the configuration

```powershell
terraform validate
```

### Run all mocked tests

```powershell
terraform test
```

Expected result: five test runs pass without Google authentication.

### Inspect the simulated plans

```powershell
terraform test -verbose
```

Locate the following in the verbose output:

- Custom-mode network
- Private web subnet
- Dedicated service account
- Instance template without `access_config`
- Managed instance group initial size
- Minimum and maximum autoscaling values
- Health-check request path
- Backend service and URL map
- Protected storage bucket
- Object-viewer IAM role

## 19. Troubleshooting exercises

Complete the exercises individually. Restore the correct configuration after each exercise.

### Exercise 1: Public VM mistake

Add this block inside the instance template's `network_interface`:

```hcl
access_config {}
```

Run `terraform test`. Identify which assertion fails and explain the security effect of the change.

### Exercise 2: Excessive IAM permissions

Change the bucket role to:

```hcl
role = "roles/storage.admin"
```

Run the tests and explain why an application that only reads objects should not receive this role.

### Exercise 3: Incorrect health-check path

Change `/health` to `/status`. Observe the failed assertion and restore the required path.

### Exercise 4: Unsafe autoscaling range

Set the minimum to 7 and maximum to 4. Explain why validation should reject this before contacting a cloud API.

### Exercise 5: Open firewall source

Replace the health-check ranges with `0.0.0.0/0`. Add a new assertion that rejects this source range.

Suggested assertion:

```hcl
assert {
  condition     = !contains(google_compute_firewall.allow_health_checks.source_ranges, "0.0.0.0/0")
  error_message = "The health-check firewall rule must not allow the entire internet."
}
```

### Exercise 6: Missing dependency

Replace the instance template's subnet reference with a literal string. Compare the dependency graph before and after the change:

```powershell
terraform graph
```

Explain why a syntactically valid literal can weaken Terraform's dependency graph.

## 20. Architecture review questions

1. Why should backend instances avoid public IP addresses?
2. How does a health check affect load-balancer traffic distribution?
3. What happens when an instance template changes in a real managed instance group?
4. Why are minimum and maximum replica limits both necessary?
5. What is the relationship between OAuth scopes and IAM roles on a Compute Engine VM?
6. Why does uniform bucket-level access simplify permission management?
7. Which resources would require global scope and which require regional or zonal scope?
8. What additional resources are required to convert the HTTP design to HTTPS?
9. Which behaviors in this design cannot be verified by a mocked provider?
10. What tests should run later in a controlled GCP integration project?

## 21. Individual completion checklist

No submission is required. Use this checklist to review your progress.

- [ ] I created all five project files and the test file.
- [ ] I formatted and validated the configuration.
- [ ] All five mock-provider test runs pass.
- [ ] I verified that the instance template has no external IP configuration.
- [ ] I verified the autoscaling boundaries and CPU target.
- [ ] I verified the health-check path and firewall source ranges.
- [ ] I verified storage public-access prevention and least-privilege IAM.
- [ ] I completed at least four troubleshooting exercises.
- [ ] I answered the architecture review questions for my own revision.
- [ ] I can explain what the mock provider validates and what it cannot validate.

## 22. Important limitation

Passing these tests does not prove that the platform will deploy successfully or behave correctly in Google Cloud. A mocked provider uses the Google provider's schema and returns simulated computed values. It does not reproduce:

- API enablement and organization policies
- Quotas and regional capacity
- IAM permission enforcement
- VM boot or startup-script success
- Managed instance group repair and update behavior
- Health-check probes
- Autoscaling decisions
- Load-balancer traffic flow
- DNS, TLS certificates, latency, or availability
- Billing and service-specific costs

A real, controlled GCP integration test is still required before production use.

## 23. Reference documentation

- [Terraform provider mocking](https://developer.hashicorp.com/terraform/language/tests/mocking)
- [Terraform test command](https://developer.hashicorp.com/terraform/cli/commands/test)
- [Google provider documentation](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [Google Compute Engine instance template resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance_template)
- [Google Compute Engine backend service resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_backend_service)

