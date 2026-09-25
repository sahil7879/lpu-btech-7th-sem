# CSG 401 Progressive Capstone Lab: Build a GCP Environment with Terraform

**Lab type:** Individual, instructor-owned billed GCP account  
**Approach:** Begin with an empty directory and evolve the project one file at a time  
**Suggested duration:** 8-10 hours across multiple sessions  
**Primary workstation:** A Debian Linux VM created in the same GCP project  
**End product:** A modular, tested Terraform deployment with remote state, Registry/CFT integration, policy validation, and documented cleanup

## Lab architecture and VM count

The completed lab uses **two VMs**:

| VM | Created by | Purpose | Included in application Terraform state? |
|---|---|---|---|
| `terraform-admin` | Student through the Google Cloud Console | Terraform installation and administration workstation | No |
| `csg401-dev-web-01` | Terraform running inside `terraform-admin` | Worker/workload VM serving the demonstration web page | Yes |

The admin VM is the bootstrap machine. The worker VM is evidence that Terraform can provision infrastructure from inside the admin VM. Do not manually create the worker VM in the Console.

## 1. End goal

By the end of the lab, Terraform will manage:

- Required Google Cloud APIs through a pinned Cloud Foundation Toolkit module
- A custom-mode VPC network
- One regional subnet with Private Google Access
- A firewall rule allowing HTTP only from approved source ranges
- A dedicated runtime service account
- One small Compute Engine web VM
- One private Cloud Storage application bucket
- Reusable local network and workload modules
- Outputs for verification
- Remote Terraform state in a versioned Cloud Storage bucket
- Native Terraform tests and an optional CFT policy-validation step

The administration VM is created first in the Google Cloud Console and remains outside the main Terraform state. Terraform then runs inside that VM and creates all other lab resources. This avoids destroying the machine that is running Terraform while it is still needed for cleanup.

## 2. Progression

| Phase | Files or capability added | Main lesson |
|---|---|---|
| 0 | Administration VM | Safe Terraform workstation |
| 1 | `versions.tf` | Provider source and version constraints |
| 2 | `providers.tf` | Google provider and authentication |
| 3 | `network.tf` | First real managed resource |
| 4 | `variables.tf`, `terraform.tfvars` | Remove hard-coded values |
| 5 | `outputs.tf` | Expose useful results |
| 6 | `compute.tf`, `files/startup.sh` | Add identity and compute |
| 7 | `storage.tf` | Add secured storage |
| 8 | `backend.tf` | Migrate state to Cloud Storage |
| 9 | `modules/network/*` | Refactor without recreation |
| 10 | `modules/workload/*` | Compose reusable modules |
| 11 | Registry/CFT module | Use reviewed external modules |
| 12 | Tests and policy validation | Verify before applying |
| 13 | Final deployment | Apply, inspect, test, and document |
| 14 | Cleanup | Remove billable resources safely |

## 3. Cost and safety controls

This lab creates billable resources. Before beginning:

1. Use a dedicated training project, not a production project.
2. Create a small budget and billing alert in Cloud Billing.
3. Use one `e2-micro` VM for the administration workstation and one `e2-micro` workload VM.
4. Use standard persistent disks and a small disk size.
5. Keep Cloud Storage data minimal.
6. Do not allow SSH from `0.0.0.0/0`.
7. Do not grant `roles/owner` or `roles/editor` to workload service accounts.
8. Never place credentials, state files, plan files, or secrets in Git.
9. Run the cleanup phase immediately after completing the lab.
10. Check the console after cleanup for remaining VMs, disks, external IPs, buckets, and forwarding resources.

Record these values before starting:

```text
PROJECT_ID       = ______________________________
BILLING_ACCOUNT  = ______________________________
REGION           = asia-south1
ZONE             = asia-south1-a
ADMIN_VM         = terraform-admin
```

## 4. Phase 0 - Create the Terraform administration VM in the Console

Do not use the Google Cloud SDK for this phase. Perform all bootstrap work in the Google Cloud Console.

### 4.1 Select the project and enable bootstrap APIs

1. Open the Google Cloud Console.
2. Select the dedicated billed training project.
3. Open **APIs & Services > Library**.
4. Enable these APIs if they are not already enabled:
   - Compute Engine API
   - Identity and Access Management (IAM) API
   - Service Usage API
   - Cloud Resource Manager API
   - Cloud Storage API
5. Open **Billing > Budgets & alerts** and create a small training budget with appropriate alert thresholds.

Enabling an API does not by itself create a billable resource. The VMs, disks, external IP use, and stored data created later can incur charges.

### 4.2 Create the Terraform administration service account

1. Open **IAM & Admin > Service Accounts**.
2. Select **Create service account**.
3. Enter:
   - Service account name: `terraform-admin`
   - Service account ID: `terraform-admin`
   - Description: `Runs the CSG 401 Terraform capstone from the administration VM`
4. Grant only the training permissions needed by this lab:
   - Compute Network Admin
   - Compute Instance Admin (v1)
   - Service Account Admin
   - Service Account User
   - Storage Admin
   - Service Usage Admin
5. Finish without creating a service-account key.

These permissions are intentionally separated instead of using Owner or Editor. In a managed organization, an administrator should replace them with a custom role scoped to the exact lab operations.

### 4.3 Create the administration VM

1. Open **Compute Engine > VM instances**.
2. Select **Create instance**.
3. Configure:
   - Name: `terraform-admin`
   - Region: `asia-south1`
   - Zone: `asia-south1-a`
   - Machine type: `e2-micro`
   - Boot disk image: Debian 12
   - Boot disk type: Standard persistent disk
   - Boot disk size: 10 GB
4. Under **Identity and API access**:
   - Service account: select `terraform-admin`
   - Access scopes: **Allow full access to all Cloud APIs**
5. Under **Firewall**, do not select Allow HTTP or Allow HTTPS.
6. Under **Security**, keep Shielded VM, vTPM, and integrity monitoring enabled. Enable Secure Boot when supported by the selected image.
7. Under **Advanced options > Networking**:
   - Use the default network and subnet for this bootstrap VM.
   - Keep an ephemeral external IPv4 address so the VM can download Terraform packages.
8. Select **Create**.

The access scope allows the VM to request API tokens, while IAM roles on the attached service account determine what those tokens can actually do.

### 4.4 Connect through the Console

On the VM instances page, select **SSH** beside `terraform-admin`. The browser opens a terminal inside the VM. All Terraform commands in the remaining phases run in this browser SSH terminal.

If browser SSH is prohibited by an organization policy, use an administrator-approved console connection method. Do not open TCP port 22 to `0.0.0.0/0`.

### 4.5 Install Terraform on the VM

Run these commands inside the administration VM:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl wget gnupg git jq lsb-release

wget -O - https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=VERSION_CODENAME=).*' /etc/os-release) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update
sudo apt-get install -y terraform

terraform version
git --version
jq --version
```

### 4.6 Verify metadata-based authentication

Do not run `gcloud auth login` and do not create a credentials file. The Google provider automatically uses Application Default Credentials exposed by the attached VM service account through the metadata service.

Verify the attached identity without the Google Cloud SDK:

```bash
curl -sS \
  -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email
```

Expected result:

```text
terraform-admin@REPLACE_WITH_PROJECT_ID.iam.gserviceaccount.com
```

Verify that a short-lived access token is available without printing the token itself:

```bash
TOKEN_STATUS=$(curl -sS -o /dev/null -w "%{http_code}" \
  -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token)

test "$TOKEN_STATUS" = "200" && echo "Metadata credentials available"
```

The token is temporary and automatically rotated. Terraform retrieves it when required.

### 4.7 Optional Google Cloud SDK equivalents

The primary activity uses the Console. Keep these commands as instructor demonstrations, alternative automation, or troubleshooting references. They are not required for the main student path.

From Cloud Shell or a workstation with the Google Cloud SDK:

```bash
export PROJECT_ID="REPLACE_WITH_PROJECT_ID"
export REGION="asia-south1"
export ZONE="asia-south1-a"

gcloud config set project "$PROJECT_ID"

gcloud services enable \
  compute.googleapis.com \
  iam.googleapis.com \
  serviceusage.googleapis.com \
  cloudresourcemanager.googleapis.com \
  storage.googleapis.com
```

Create the administration service account and grant the same lab roles selected in the Console:

```bash
gcloud iam service-accounts create terraform-admin \
  --display-name="CSG 401 Terraform administration"

for ROLE in \
  roles/compute.networkAdmin \
  roles/compute.instanceAdmin.v1 \
  roles/iam.serviceAccountAdmin \
  roles/iam.serviceAccountUser \
  roles/storage.admin \
  roles/serviceusage.serviceUsageAdmin
do
  gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:terraform-admin@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="$ROLE"
done
```

Equivalent administration VM creation:

```bash
gcloud compute instances create terraform-admin \
  --project="$PROJECT_ID" \
  --zone="$ZONE" \
  --machine-type=e2-micro \
  --network=default \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-type=pd-standard \
  --boot-disk-size=10GB \
  --service-account="terraform-admin@${PROJECT_ID}.iam.gserviceaccount.com" \
  --scopes=cloud-platform \
  --shielded-vtpm \
  --shielded-integrity-monitoring
```

The SDK alternative must produce the same architecture and permissions as the Console procedure. Do not use both methods to create duplicate administration VMs.

## 5. Phase 1 - Start with an empty project directory

Inside the administration VM:

```bash
mkdir -p ~/csg401-capstone
cd ~/csg401-capstone
git init
```

Create `.gitignore` first:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
*.plan.json
terraform.tfvars
crash.log
override.tf
override.tf.json
```

Do commit `.terraform.lock.hcl` after initialization. Do not commit `terraform.tfvars` because it may contain project-specific data.

## 6. Phase 2 - Add `versions.tf`

Create `versions.tf`:

```hcl
terraform {
  required_version = ">= 1.7.0, < 2.0.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 7.46"
    }
  }
}
```

Initialize:

```bash
terraform init
terraform providers
terraform version
```

Checkpoint:

- `.terraform.lock.hcl` exists.
- The Google provider source is `hashicorp/google`.
- The selected provider version satisfies the declared constraint.

## 7. Phase 3 - Add `providers.tf`

Create `providers.tf`:

```hcl
provider "google" {
  project = "REPLACE_WITH_PROJECT_ID"
  region  = "asia-south1"
  zone    = "asia-south1-a"
}
```

Run:

```bash
terraform fmt
terraform validate
terraform plan
```

The plan reports no changes because a provider configures access but does not itself create infrastructure.

## 8. Phase 4 - Add the first resource in `network.tf`

Create `network.tf`:

```hcl
resource "google_compute_network" "main" {
  name                    = "csg401-dev-vpc"
  auto_create_subnetworks = false
  routing_mode            = "REGIONAL"
}
```

Run the full workflow:

```bash
terraform fmt -check
terraform validate
terraform plan -out=network.tfplan
terraform show network.tfplan
terraform apply network.tfplan
terraform state list
terraform state show google_compute_network.main
```

Do not continue until the console and state both show exactly one new VPC.

## 9. Phase 5 - Introduce variables one concern at a time

Create `variables.tf`:

```hcl
variable "project_id" {
  description = "Existing Google Cloud project used by the lab."
  type        = string
}

variable "region" {
  description = "Google Cloud region for regional resources."
  type        = string
  default     = "asia-south1"
}

variable "zone" {
  description = "Google Cloud zone for the web VM."
  type        = string
  default     = "asia-south1-a"
}

variable "environment" {
  description = "Short environment identifier."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be dev, test, or prod."
  }
}

variable "subnet_cidr" {
  description = "Primary IPv4 CIDR for the application subnet."
  type        = string
  default     = "10.20.0.0/24"

  validation {
    condition     = can(cidrhost(var.subnet_cidr, 1))
    error_message = "subnet_cidr must be a valid IPv4 CIDR."
  }
}

variable "allowed_http_cidrs" {
  description = "Source ranges allowed to reach the demonstration web server."
  type        = list(string)
  default     = ["0.0.0.0/0"]

  validation {
    condition = alltrue([
      for cidr in var.allowed_http_cidrs : can(cidrhost(cidr, 0))
    ])
    error_message = "Every allowed HTTP source must be a valid CIDR."
  }
}

variable "labels" {
  description = "Labels applied to supported resources."
  type        = map(string)
  default = {
    course     = "csg401"
    managed_by = "terraform"
  }
}
```

Create `terraform.tfvars`:

```hcl
project_id  = "REPLACE_WITH_PROJECT_ID"
region      = "asia-south1"
zone        = "asia-south1-a"
environment = "dev"

subnet_cidr = "10.20.0.0/24"

# For a stronger restriction, replace this with your public IP in /32 form.
allowed_http_cidrs = ["0.0.0.0/0"]

labels = {
  course      = "csg401"
  environment = "dev"
  managed_by  = "terraform"
  owner       = "instructor"
}
```

Refactor `providers.tf`:

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}
```

Refactor `network.tf` and add the subnet:

```hcl
locals {
  name_prefix = "csg401-${var.environment}"
}

resource "google_compute_network" "main" {
  name                    = "${local.name_prefix}-vpc"
  auto_create_subnetworks = false
  routing_mode            = "REGIONAL"
}

resource "google_compute_subnetwork" "main" {
  name                     = "${local.name_prefix}-subnet"
  project                  = var.project_id
  region                   = var.region
  network                  = google_compute_network.main.id
  ip_cidr_range            = var.subnet_cidr
  private_ip_google_access = true
}

resource "google_compute_firewall" "http" {
  name      = "${local.name_prefix}-allow-http"
  project   = var.project_id
  network   = google_compute_network.main.name
  direction = "INGRESS"
  priority  = 1000

  source_ranges = var.allowed_http_cidrs
  target_tags   = ["${local.name_prefix}-web"]

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }
}
```

Plan before applying:

```bash
terraform fmt
terraform validate
terraform plan -out=network-expanded.tfplan
terraform show network-expanded.tfplan
terraform apply network-expanded.tfplan
```

If Terraform proposes replacing the original VPC merely because its name changed, decide whether the rename is intentional. In this fresh lab it is acceptable; in a real environment, preserve names or use migration techniques carefully.

## 10. Phase 6 - Add `outputs.tf`

Create `outputs.tf`:

```hcl
output "network_name" {
  description = "Name of the application VPC."
  value       = google_compute_network.main.name
}

output "subnet_self_link" {
  description = "Self-link of the application subnet."
  value       = google_compute_subnetwork.main.self_link
}
```

Run:

```bash
terraform apply -auto-approve
terraform output
terraform output -json | jq
```

Outputs form an interface. Later, the root module will consume equivalent outputs from child modules.

## 11. Phase 7 - Add identity and compute

Create the startup-script directory:

```bash
mkdir -p files
```

Create `files/startup.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y apache2

cat >/var/www/html/index.html <<EOF
<!doctype html>
<html>
  <head><title>CSG 401 Terraform Capstone</title></head>
  <body>
    <h1>CSG 401 Terraform deployment is working</h1>
    <p>Environment: ${environment}</p>
  </body>
</html>
EOF

systemctl enable --now apache2
```

Create `compute.tf`:

```hcl
resource "google_service_account" "web" {
  project      = var.project_id
  account_id   = "${local.name_prefix}-web"
  display_name = "CSG 401 web runtime"
}

resource "google_compute_instance" "web" {
  name         = "${local.name_prefix}-web-01"
  project      = var.project_id
  zone         = var.zone
  machine_type = "e2-micro"
  tags         = ["${local.name_prefix}-web"]
  labels       = var.labels

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 10
      type  = "pd-standard"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.main.id

    access_config {}
  }

  service_account {
    email  = google_service_account.web.email
    scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }

  metadata = {
    enable-oslogin = "TRUE"
  }

  metadata_startup_script = templatefile("${path.module}/files/startup.sh", {
    environment = var.environment
  })

  shielded_instance_config {
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }

  depends_on = [google_compute_firewall.http]
}
```

The broad OAuth scope does not itself grant IAM permissions; the service account currently has no project roles. Avoid service-account keys.

Add to `outputs.tf`:

```hcl
output "web_external_ip" {
  description = "Ephemeral external IP of the demonstration web VM."
  value       = google_compute_instance.web.network_interface[0].access_config[0].nat_ip
}

output "web_url" {
  description = "HTTP URL for the demonstration page."
  value       = "http://${google_compute_instance.web.network_interface[0].access_config[0].nat_ip}"
}
```

Apply and test:

```bash
terraform fmt
terraform validate
terraform plan -out=compute.tfplan
terraform apply compute.tfplan

WEB_URL=$(terraform output -raw web_url)
echo "$WEB_URL"
curl --retry 12 --retry-delay 10 "$WEB_URL"
```

If `curl` fails initially, open **Compute Engine > VM instances**, select `csg401-dev-web-01`, and inspect its serial-port output and startup-script logs from the Console.

## 12. Phase 8 - Add secured storage

Create `storage.tf`:

```hcl
resource "google_storage_bucket" "application" {
  name                        = "${var.project_id}-${var.environment}-csg401-app"
  project                     = var.project_id
  location                    = var.region
  storage_class               = "STANDARD"
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"
  force_destroy               = true
  labels                      = var.labels

  versioning {
    enabled = true
  }

  lifecycle_rule {
    condition {
      age = 7
    }
    action {
      type = "Delete"
    }
  }
}

resource "google_storage_bucket_iam_member" "web_reader" {
  bucket = google_storage_bucket.application.name
  role   = "roles/storage.objectViewer"
  member = "serviceAccount:${google_service_account.web.email}"
}
```

Add to `outputs.tf`:

```hcl
output "application_bucket_name" {
  description = "Private application bucket."
  value       = google_storage_bucket.application.name
}
```

Run:

```bash
terraform plan -out=storage.tfplan
terraform apply storage.tfplan
terraform output -raw application_bucket_name
```

Open **Cloud Storage > Buckets** in the Console and confirm that the displayed bucket has uniform bucket-level access, public-access prevention, and versioning enabled.

## 13. Phase 9 - Migrate local state to a remote GCS backend

The state bucket is a bootstrap dependency and is created outside the application configuration so that deleting the application does not delete its own state.

Create it in the Console:

1. Open **Cloud Storage > Buckets**.
2. Select **Create**.
3. Use a globally unique name such as `PROJECT_ID-csg401-tfstate`.
4. Location type: Region.
5. Region: `asia-south1`.
6. Storage class: Standard.
7. Access control: Uniform.
8. Public access prevention: Enforced.
9. Create the bucket.
10. Open the new bucket, select **Protection**, and enable object versioning.
11. Record the exact bucket name before continuing.

Create `backend.tf`:

```hcl
terraform {
  backend "gcs" {
    bucket = "REPLACE_WITH_STATE_BUCKET_NAME"
    prefix = "csg401/capstone/dev"
  }
}
```

Migrate:

```bash
terraform init -migrate-state
terraform state list
```

Open the state bucket in the Console and confirm that objects now exist under the `csg401/capstone/dev` prefix.

Do not delete the local backup until the remote state has been verified. Restrict access to the state bucket because state can contain sensitive values.

## 14. Phase 10 - Refactor networking into a local module

Create the module directories:

```bash
mkdir -p modules/network
```

Create `modules/network/variables.tf`:

```hcl
variable "project_id" { type = string }
variable "region" { type = string }
variable "name_prefix" { type = string }
variable "subnet_cidr" { type = string }
variable "allowed_http_cidrs" { type = list(string) }
```

Create `modules/network/main.tf`:

```hcl
resource "google_compute_network" "main" {
  name                    = "${var.name_prefix}-vpc"
  project                 = var.project_id
  auto_create_subnetworks = false
  routing_mode            = "REGIONAL"
}

resource "google_compute_subnetwork" "main" {
  name                     = "${var.name_prefix}-subnet"
  project                  = var.project_id
  region                   = var.region
  network                  = google_compute_network.main.id
  ip_cidr_range            = var.subnet_cidr
  private_ip_google_access = true
}

resource "google_compute_firewall" "http" {
  name          = "${var.name_prefix}-allow-http"
  project       = var.project_id
  network       = google_compute_network.main.name
  direction     = "INGRESS"
  priority      = 1000
  source_ranges = var.allowed_http_cidrs
  target_tags   = ["${var.name_prefix}-web"]

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }
}
```

Create `modules/network/outputs.tf`:

```hcl
output "network_name" { value = google_compute_network.main.name }
output "network_id" { value = google_compute_network.main.id }
output "subnet_id" { value = google_compute_subnetwork.main.id }
output "subnet_self_link" { value = google_compute_subnetwork.main.self_link }
output "http_firewall_id" { value = google_compute_firewall.http.id }
```

Replace the contents of root `network.tf` with:

```hcl
module "network" {
  source = "./modules/network"

  project_id         = var.project_id
  region             = var.region
  name_prefix        = local.name_prefix
  subnet_cidr        = var.subnet_cidr
  allowed_http_cidrs = var.allowed_http_cidrs
}

moved {
  from = google_compute_network.main
  to   = module.network.google_compute_network.main
}

moved {
  from = google_compute_subnetwork.main
  to   = module.network.google_compute_subnetwork.main
}

moved {
  from = google_compute_firewall.http
  to   = module.network.google_compute_firewall.http
}
```

Update `compute.tf`:

```hcl
subnetwork = module.network.subnet_id
```

Also remove this old explicit dependency from the VM:

```hcl
depends_on = [google_compute_firewall.http]
```

The subnet reference already creates the required implicit network dependency. The web server does not require the firewall rule to exist before the VM can be created.

Replace the two original network outputs in root `outputs.tf`:

```hcl
output "network_name" {
  value = module.network.network_name
}

output "subnet_self_link" {
  value = module.network.subnet_self_link
}
```

Run:

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
```

The plan should show address moves and no network destroy/create operations. Do not apply if it proposes recreating the VPC, subnet, or firewall unexpectedly.

Apply the address migration:

```bash
terraform apply
terraform state list
```

## 15. Phase 11 - Refactor the workload into a second module

Create:

```text
modules/workload/
|-- main.tf
|-- variables.tf
|-- outputs.tf
`-- files/
    `-- startup.sh
```

Move the startup script conceptually into `modules/workload/files/startup.sh`. Use the same content created earlier.

Create `modules/workload/variables.tf`:

```hcl
variable "project_id" { type = string }
variable "region" { type = string }
variable "zone" { type = string }
variable "name_prefix" { type = string }
variable "environment" { type = string }
variable "subnet_id" { type = string }
variable "labels" { type = map(string) }
```

Create `modules/workload/main.tf` by moving the resources from `compute.tf` and `storage.tf`. Make these reference changes:

```hcl
# VM network interface
subnetwork = var.subnet_id

# Startup script path
metadata_startup_script = templatefile("${path.module}/files/startup.sh", {
  environment = var.environment
})
```

Replace every remaining `local.name_prefix` reference in the moved resources with `var.name_prefix`. Do not retain the removed root firewall dependency inside this module.

All remaining references in the moved resources continue to use their local resource names.

Create `modules/workload/outputs.tf`:

```hcl
output "web_external_ip" {
  value = google_compute_instance.web.network_interface[0].access_config[0].nat_ip
}

output "web_url" {
  value = "http://${google_compute_instance.web.network_interface[0].access_config[0].nat_ip}"
}

output "bucket_name" {
  value = google_storage_bucket.application.name
}

output "service_account_email" {
  value = google_service_account.web.email
}
```

Replace root `compute.tf` with:

```hcl
module "workload" {
  source = "./modules/workload"

  project_id  = var.project_id
  region      = var.region
  zone        = var.zone
  name_prefix = local.name_prefix
  environment = var.environment
  subnet_id   = module.network.subnet_id
  labels      = var.labels
}

moved {
  from = google_service_account.web
  to   = module.workload.google_service_account.web
}

moved {
  from = google_compute_instance.web
  to   = module.workload.google_compute_instance.web
}

moved {
  from = google_storage_bucket.application
  to   = module.workload.google_storage_bucket.application
}

moved {
  from = google_storage_bucket_iam_member.web_reader
  to   = module.workload.google_storage_bucket_iam_member.web_reader
}
```

Delete root `storage.tf` after its resources have been placed in the workload module.

Update the workload-related root outputs to reference `module.workload`.

```hcl
output "web_external_ip" { value = module.workload.web_external_ip }
output "web_url" { value = module.workload.web_url }
output "application_bucket_name" { value = module.workload.bucket_name }
output "web_service_account" { value = module.workload.service_account_email }
```

Run `terraform plan` and require move-only behavior before applying.

## 16. Phase 12 - Add a Registry/CFT module

Cloud Foundation Toolkit modules are published through the Terraform Registry. Use the `project_services` submodule to manage APIs rather than maintaining several individual API resources.

Create root `services.tf`:

```hcl
module "project_services" {
  source  = "terraform-google-modules/project-factory/google//modules/project_services"
  version = "18.3.0"

  project_id = var.project_id

  activate_apis = [
    "compute.googleapis.com",
    "iam.googleapis.com",
    "storage.googleapis.com",
  ]

  disable_services_on_destroy = false
  disable_dependent_services  = false
}
```

Add this dependency to the two root module calls:

```hcl
depends_on = [module.project_services]
```

Initialize and inspect what was downloaded:

```bash
terraform init -upgrade
terraform providers
terraform get
terraform plan
```

Registry/CFT review checklist:

- Confirm the namespace is `terraform-google-modules`.
- Confirm the module source repository belongs to the Google-maintained organization.
- Read the module inputs, outputs, provider requirements, license, release notes, and open issues.
- Pin the module version rather than using an unbounded latest version.
- Review the downloaded code; do not assume a public module is automatically suitable.
- Keep providers and backends in the root module, not inside reusable local modules.
- Ensure API disabling is false so teardown does not disrupt other project workloads.

## 17. Phase 13 - Add native verification

Create `tests/capstone.tftest.hcl`:

```hcl
mock_provider "google" {}

run "secure_structure" {
  command = plan

  override_resource {
    target = module.workload.google_compute_instance.web
    values = {
      network_interface = [{
        access_config = [{ nat_ip = "203.0.113.10" }]
      }]
    }
  }

  assert {
    condition     = module.network.network_name == "csg401-dev-vpc"
    error_message = "The network name must follow the environment naming convention."
  }

  assert {
    condition     = module.workload.bucket_name == "${var.project_id}-dev-csg401-app"
    error_message = "The application bucket name is not deterministic."
  }

  assert {
    condition     = module.workload.web_url == "http://203.0.113.10"
    error_message = "The root module must expose the workload URL."
  }
}
```

Run local checks:

```bash
terraform fmt -recursive -check
terraform validate
terraform test
terraform plan -out=reviewed.tfplan
terraform show -json reviewed.tfplan > reviewed.plan.json
jq '.resource_changes[] | {address, actions: .change.actions}' reviewed.plan.json
```

Mock tests validate configuration behavior without creating cloud resources. The ordinary saved plan still uses real provider data and must be reviewed before apply.

## 18. Phase 14 - Optional SDK-based CFT policy validation

The main lab does not require this step. It uses the Google Cloud SDK command `gcloud beta terraform vet`, which is a Preview capability and needs extra permissions to read project and IAM ancestry. Keep it as an instructor demonstration or advanced extension. It is separate from consuming CFT modules through Terraform.

Install or verify the component:

```bash
gcloud components install terraform-tools
gcloud beta terraform vet --help
```

Obtain an instructor-approved policy library:

```bash
git clone REPLACE_WITH_POLICY_LIBRARY_REPOSITORY policy-library
```

Validate the saved plan:

```bash
gcloud beta terraform vet reviewed.plan.json \
  --policy-library=./policy-library \
  --project="$PROJECT_ID" \
  --format=json
```

Do not apply when high-severity violations remain. If the command returns `403`, verify the documented `getIamPolicy`, project, folder, and ancestry permissions rather than granting broad roles blindly.

## 19. Phase 15 - Complete project structure

The repository should now resemble:

```text
csg401-capstone/
|-- .gitignore
|-- .terraform.lock.hcl
|-- backend.tf
|-- versions.tf
|-- providers.tf
|-- variables.tf
|-- terraform.tfvars
|-- network.tf
|-- compute.tf
|-- services.tf
|-- outputs.tf
|-- modules/
|   |-- network/
|   |   |-- main.tf
|   |   |-- variables.tf
|   |   `-- outputs.tf
|   `-- workload/
|       |-- main.tf
|       |-- variables.tf
|       |-- outputs.tf
|       `-- files/
|           `-- startup.sh
`-- tests/
    `-- capstone.tftest.hcl
```

Remove obsolete empty files and duplicate resource blocks. Each resource must be declared exactly once.

## 20. Phase 16 - Final deployment and verification

Run the quality gate:

```bash
terraform fmt -recursive -check
terraform validate
terraform test
terraform plan -out=final.tfplan
terraform show final.tfplan
terraform show -json final.tfplan > final.plan.json
```

If CFT policy validation is configured, run it against `final.plan.json`.

Apply the exact reviewed plan:

```bash
terraform apply final.tfplan
```

Verify from Terraform and the VM:

```bash
terraform state list
terraform output
terraform output -json | jq

curl --retry 12 --retry-delay 10 "$(terraform output -raw web_url)"
```

Then verify through the Console:

1. Open **VPC network > VPC networks** and locate the Terraform-created VPC and subnet.
2. Open **Compute Engine > VM instances** and confirm both VMs are visible:
   - `terraform-admin`, created manually
   - `csg401-dev-web-01`, created by Terraform
3. Open the worker VM details and verify its service account, network, tags, labels, Shielded VM settings, and external IP.
4. Open **Cloud Storage > Buckets** and inspect the application bucket protections.
5. Open **IAM & Admin > Service Accounts** and confirm the worker identity has no broad project role.

Optional SDK verification:

```bash
gcloud compute networks describe "$(terraform output -raw network_name)"
gcloud compute instances describe csg401-dev-web-01 --zone=asia-south1-a
gcloud storage buckets describe "gs://$(terraform output -raw application_bucket_name)"
```

Run a no-change plan:

```bash
terraform plan -detailed-exitcode
echo $?
```

Exit code `0` means no differences, `2` means changes exist, and `1` means an error occurred.

## 21. Troubleshooting checkpoints

| Symptom | Check |
|---|---|
| Provider authentication error | Attached `terraform-admin` service account, VM access scope, metadata endpoint, and assigned IAM roles |
| API disabled error | CFT project-services plan and Service Usage permissions |
| Bucket name conflict | Bucket names are globally unique; include the project ID |
| VM cannot serve HTTP | Firewall target tag, source range, startup script log, Apache status |
| Module refactor proposes recreation | Verify every `moved` source and destination address |
| Backend initialization fails | Bucket name, Storage Admin permission, metadata credentials, and prefix |
| Test has unknown computed values | Add a narrow `override_resource` value |
| `terraform vet` returns 403 | Required IAM/project/folder ancestry read permissions |
| Destroy blocked by bucket contents | Confirm this is the training bucket and `force_destroy = true` |

## 22. Phase 17 - Cleanup in the correct order

### 22.1 Destroy Terraform-managed resources

From `~/csg401-capstone`:

```bash
terraform plan -destroy -out=destroy.tfplan
terraform show destroy.tfplan
terraform apply destroy.tfplan
terraform state list
```

The state list should be empty. The state bucket and administration VM remain because they were bootstrapped outside this state.

### 22.2 Leave the administration VM

```bash
exit
```

Return to the Console and verify that the Terraform-created worker VM and application bucket no longer exist. The manually created admin VM and state bucket remain temporarily.

### 22.3 Delete the state bucket only after verification

State-bucket deletion is destructive and removes recovery history. Confirm the application has been destroyed and no other environment uses the bucket.

Using the primary Console method:

1. Open **Cloud Storage > Buckets**.
2. Open the state bucket and inspect its contents one final time.
3. Confirm that no other environment uses this bucket.
4. Delete all objects and versions.
5. Delete the bucket.

Optional SDK equivalent:

```bash
gcloud storage rm --recursive "gs://REPLACE_WITH_STATE_BUCKET/**"
gcloud storage buckets delete "gs://REPLACE_WITH_STATE_BUCKET"
```

### 22.4 Delete the administration VM

Using the primary Console method:

1. Open **Compute Engine > VM instances**.
2. Confirm that `csg401-dev-web-01` has already been destroyed by Terraform.
3. Select `terraform-admin`.
4. Select **Delete** and confirm deletion of the VM and its boot disk.
5. Open **IAM & Admin > Service Accounts** and delete `terraform-admin` if it is used only for this lab.

Do not delete the default VPC network merely because the admin VM used it; other resources or exercises may depend on it.

Optional SDK equivalent:

```bash
gcloud compute instances delete terraform-admin \
  --zone=asia-south1-a

gcloud iam service-accounts delete \
  "terraform-admin@REPLACE_WITH_PROJECT_ID.iam.gserviceaccount.com"
```

### 22.5 Final billing inspection

Check these pages in the Google Cloud console:

- Compute Engine instances and disks
- VPC networks, firewall rules, and external IP addresses
- Cloud Storage buckets
- IAM service accounts
- Billing reports and current project cost

API enablement can remain without direct charges, but disabled APIs may be appropriate if the entire training project is being retired.

## 23. Completion checklist

- [ ] Created and secured a dedicated Terraform administration VM.
- [ ] Installed and verified Terraform on the VM.
- [ ] Confirmed that Terraform used the admin VM's attached service account without a key file.
- [ ] Created the first VPC before introducing abstractions.
- [ ] Replaced hard-coded values with variables and validation.
- [ ] Added useful outputs.
- [ ] Added compute, service-account, firewall, and storage resources.
- [ ] Migrated local state to a versioned GCS backend.
- [ ] Refactored resources into local modules using `moved` blocks.
- [ ] Used a pinned Registry/CFT module.
- [ ] Ran formatting, validation, tests, plan review, and policy checks.
- [ ] Applied the exact reviewed plan.
- [ ] Confirmed a no-change final plan.
- [ ] Destroyed Terraform-managed resources.
- [ ] Deleted the state bucket, administration VM, its boot disk, and the lab-only administration service account.
- [ ] Confirmed no unexpected billable resources remain.

## 24. Reference documentation

- [Install Terraform](https://developer.hashicorp.com/terraform/install)
- [Authenticate Terraform to Google Cloud](https://cloud.google.com/docs/terraform/authentication)
- [Terraform on Google Cloud](https://cloud.google.com/docs/terraform)
- [Store Terraform state in Cloud Storage](https://cloud.google.com/docs/terraform/resource-management/store-state)
- [Google Cloud root-module practices](https://cloud.google.com/docs/terraform/best-practices/root-modules)
- [Google Cloud reusable-module practices](https://cloud.google.com/docs/terraform/best-practices/reusable-modules)
- [Terraform blueprints and Cloud Foundation Toolkit modules](https://cloud.google.com/docs/terraform/blueprints/terraform-blueprints)
- [CFT project-services Registry module](https://registry.terraform.io/modules/terraform-google-modules/project-factory/google/latest/submodules/project_services)
- [Validate policies with `gcloud beta terraform vet`](https://cloud.google.com/docs/terraform/policy-validation/validate-policies)
- [Cloud Storage uniform bucket-level access](https://cloud.google.com/storage/docs/uniform-bucket-level-access)
