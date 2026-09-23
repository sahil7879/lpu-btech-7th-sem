# Individual Activity 7: Terraform Security and Policy Checks

**Course:** CSG 401 - Getting Started with Terraform in GCP  
**Activity type:** Individual guided practical  
**Suggested duration:** 3-4 hours  
**Cloud or billing account required:** No  
**External provider required:** No  
**Additional software required:** PowerShell and Terraform

## 1. Purpose

Treat security requirements as executable policy. You will model a GCP-style storage service, generate a machine-readable Terraform plan, run a local policy gate, correct deliberately insecure settings, and add native Terraform tests.

Earlier activities used validation to catch input mistakes. This activity introduces a different concern: organization-wide security rules that evaluate the proposed infrastructure as a whole and can block an automated workflow.

## 2. Learning outcomes

You will be able to:

1. Distinguish syntax validation, configuration tests, and security-policy checks.
2. Convert a Terraform plan to JSON.
3. Enforce security rules against planned values.
4. Separate blocking policy failures from advisory checks.
5. Correct insecure public access, encryption, identity, retention, and labeling settings.
6. Write native `.tftest.hcl` assertions and expected failures.
7. Explain how the same checks become a CI/CD quality gate.

## 3. Policy model

```text
Terraform configuration
        |
        +-- terraform fmt       -> formatting
        +-- terraform validate  -> language and internal consistency
        +-- terraform test      -> expected configuration behavior
        `-- plan JSON policy    -> organization security requirements
```

Passing `terraform validate` does not mean a configuration is secure. Valid Terraform can still represent a public, unencrypted, or over-privileged design.

## 4. Project structure

```text
terraform-security-policy-lab/
|-- variables.tf
|-- main.tf
|-- outputs.tf
|-- insecure.tfvars
|-- secure.tfvars
|-- policy-check.ps1
|-- tests/
|   `-- security.tftest.hcl
`-- .gitignore
```

## 5. Define security-related inputs

Create `variables.tf`:

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

variable "storage" {
  description = "GCP-style storage security configuration."
  type = object({
    public_access      = bool
    encryption         = string
    versioning_enabled = bool
    retention_days     = number
  })
}

variable "service_account" {
  description = "Runtime identity configuration."
  type = object({
    name  = string
    roles = set(string)
  })
}

variable "labels" {
  type = map(string)
}
```

These variables intentionally contain only type constraints. The policy layer, rather than variable validation, will identify the security violations.

## 6. Model the proposed infrastructure

Create `main.tf`:

```hcl
terraform {
  required_version = ">= 1.7.0"
}

locals {
  required_labels = toset(["owner", "environment", "data_classification"])
  supplied_labels = toset(keys(var.labels))
  missing_labels  = setsubtract(local.required_labels, local.supplied_labels)
}

resource "terraform_data" "storage_service" {
  input = {
    name               = "policy-${var.environment}-storage"
    public_access      = var.storage.public_access
    encryption         = var.storage.encryption
    versioning_enabled = var.storage.versioning_enabled
    retention_days     = var.storage.retention_days
    labels             = var.labels
  }
}

resource "terraform_data" "runtime_identity" {
  input = {
    name  = var.service_account.name
    roles = sort(tolist(var.service_account.roles))
  }
}

check "security_advisories" {
  assert {
    condition     = var.storage.versioning_enabled
    error_message = "Advisory: object versioning should be enabled."
  }

  assert {
    condition     = var.storage.retention_days >= 7
    error_message = "Advisory: retention should be at least 7 days."
  }
}
```

`check` assertions issue warnings and continue the operation. They are suitable for advisory or ongoing health checks. The separate policy script below implements blocking requirements.

Create `outputs.tf`:

```hcl
output "security_summary" {
  value = {
    public_access      = terraform_data.storage_service.output.public_access
    encryption         = terraform_data.storage_service.output.encryption
    versioning_enabled = terraform_data.storage_service.output.versioning_enabled
    retention_days     = terraform_data.storage_service.output.retention_days
    identity_name      = terraform_data.runtime_identity.output.name
    roles              = terraform_data.runtime_identity.output.roles
    missing_labels     = local.missing_labels
  }
}
```

## 7. Create insecure and secure inputs

Create `insecure.tfvars`:

```hcl
environment = "prod"

storage = {
  public_access      = true
  encryption         = "provider-managed"
  versioning_enabled = false
  retention_days     = 0
}

service_account = {
  name  = "default"
  roles = ["roles/owner"]
}

labels = {
  environment = "prod"
}
```

Create `secure.tfvars`:

```hcl
environment = "prod"

storage = {
  public_access      = false
  encryption         = "customer-managed"
  versioning_enabled = true
  retention_days     = 30
}

service_account = {
  name = "application-runtime"
  roles = [
    "roles/storage.objectViewer",
    "roles/logging.logWriter",
  ]
}

labels = {
  owner               = "platform-team"
  environment         = "prod"
  data_classification = "internal"
}
```

## 8. Create a blocking policy gate

Create `policy-check.ps1`:

```powershell
param(
    [Parameter(Mandatory = $true)]
    [string]$PlanJson
)

$ErrorActionPreference = "Stop"
$plan = Get-Content -LiteralPath $PlanJson -Raw | ConvertFrom-Json
$resources = @($plan.planned_values.root_module.resources)

$storage = $resources |
    Where-Object { $_.address -eq "terraform_data.storage_service" } |
    Select-Object -First 1

$identity = $resources |
    Where-Object { $_.address -eq "terraform_data.runtime_identity" } |
    Select-Object -First 1

if (-not $storage -or -not $identity) {
    Write-Error "Required planned resources were not found."
}

$failures = [System.Collections.Generic.List[string]]::new()
$storageInput = $storage.values.input
$identityInput = $identity.values.input

if ($storageInput.public_access -ne $false) {
    $failures.Add("SEC-001: Production storage must not allow public access.")
}

if ($storageInput.encryption -ne "customer-managed") {
    $failures.Add("SEC-002: Production storage must use customer-managed encryption.")
}

if ($storageInput.versioning_enabled -ne $true) {
    $failures.Add("SEC-003: Storage versioning must be enabled.")
}

if ([int]$storageInput.retention_days -lt 7) {
    $failures.Add("SEC-004: Storage retention must be at least 7 days.")
}

if ($identityInput.name -eq "default") {
    $failures.Add("IAM-001: Workloads must use a dedicated service account.")
}

$prohibitedRoles = @("roles/owner", "roles/editor")
foreach ($role in @($identityInput.roles)) {
    if ($role -in $prohibitedRoles) {
        $failures.Add("IAM-002: Prohibited broad role assigned: $role")
    }
}

$requiredLabels = @("owner", "environment", "data_classification")
foreach ($label in $requiredLabels) {
    if (-not $storageInput.labels.PSObject.Properties.Name.Contains($label)) {
        $failures.Add("GOV-001: Required label is missing: $label")
    }
}

if ($failures.Count -gt 0) {
    Write-Host "POLICY RESULT: FAIL" -ForegroundColor Red
    $failures | ForEach-Object { Write-Host " - $_" -ForegroundColor Red }
    exit 1
}

Write-Host "POLICY RESULT: PASS" -ForegroundColor Green
exit 0
```

The script reads planned values, accumulates all violations, prints actionable policy IDs, and returns a non-zero exit code when the deployment must be blocked.

## 9. Scan the insecure plan

Create `.gitignore`:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
*.plan.json
```

Run:

```powershell
terraform init
terraform fmt
terraform validate
terraform plan -var-file=insecure.tfvars -out=insecure.tfplan
terraform show -json insecure.tfplan > insecure.plan.json
powershell -ExecutionPolicy Bypass -File .\policy-check.ps1 -PlanJson .\insecure.plan.json
```

Expected result: the policy gate fails and reports public access, encryption, versioning, retention, default identity, broad IAM, and missing labels. Terraform may also print advisory warnings from the `check` block.

The failed exit code is intentional. In CI/CD it would prevent the apply stage from running.

## 10. Correct and rescan the design

Run the same workflow with the secure values:

```powershell
terraform plan -var-file=secure.tfvars -out=secure.tfplan
terraform show -json secure.tfplan > secure.plan.json
powershell -ExecutionPolicy Bypass -File .\policy-check.ps1 -PlanJson .\secure.plan.json
```

Expected result: `POLICY RESULT: PASS`.

Only after policy passes, apply the reviewed plan:

```powershell
terraform apply secure.tfplan
terraform output security_summary
```

## 11. Add native Terraform tests

Create `tests/security.tftest.hcl`:

```hcl
run "secure_production_design" {
  command = plan

  variables {
    environment = "prod"

    storage = {
      public_access      = false
      encryption         = "customer-managed"
      versioning_enabled = true
      retention_days     = 30
    }

    service_account = {
      name  = "application-runtime"
      roles = ["roles/storage.objectViewer", "roles/logging.logWriter"]
    }

    labels = {
      owner               = "platform-team"
      environment         = "prod"
      data_classification = "internal"
    }
  }

  assert {
    condition     = terraform_data.storage_service.input.public_access == false
    error_message = "Secure storage must remain private."
  }

  assert {
    condition     = terraform_data.storage_service.input.encryption == "customer-managed"
    error_message = "Secure storage must use customer-managed encryption."
  }

  assert {
    condition     = !contains(terraform_data.runtime_identity.input.roles, "roles/owner")
    error_message = "The runtime identity must not receive the owner role."
  }

  assert {
    condition     = length(local.missing_labels) == 0
    error_message = "All required governance labels must be supplied."
  }
}

run "advisory_checks_warn_for_weak_settings" {
  command = plan

  expect_failures = [check.security_advisories]

  variables {
    environment = "dev"

    storage = {
      public_access      = false
      encryption         = "customer-managed"
      versioning_enabled = false
      retention_days     = 1
    }

    service_account = {
      name  = "development-runtime"
      roles = ["roles/storage.objectViewer"]
    }

    labels = {
      owner               = "student"
      environment         = "dev"
      data_classification = "training"
    }
  }
}
```

Run:

```powershell
terraform test
```

The first run contains blocking test assertions. The second deliberately supplies weak settings and declares the advisory check as an expected failure, allowing the test suite to verify that the warning is produced. During ordinary plan and apply operations, top-level `check` assertions warn and continue. In a real policy system, decide explicitly which controls are mandatory and which are advisory.

## 12. Policy classification exercise

Classify each control:

| Control | Preventive, detective, or corrective? | Blocking or advisory? |
|---|---|---|
| Deny public storage | Preventive | Blocking |
| Require customer-managed encryption | Preventive | Blocking |
| Warn when retention is under 30 days | Detective | Advisory or blocking by risk class |
| Remove an excessive role after deployment | Corrective | Automated with approval |
| Require ownership labels | Preventive | Blocking |

Security controls should have a documented owner, policy ID, severity, remediation message, exception process, and review date.

## 13. Exceptions and suppressions

Do not delete or silently disable a failed rule simply to make a pipeline green. A defensible exception should record:

- The exact policy ID
- Business justification
- Scope of the exception
- Approver
- Expiry date
- Compensating controls

Keep exceptions narrow and time-bound. Reassess them automatically when possible.

## 14. CI/CD quality-gate design

A practical pipeline sequence is:

```text
terraform fmt -check
        |
terraform init
        |
terraform validate
        |
terraform test
        |
terraform plan -out=reviewed.tfplan
        |
terraform show -json reviewed.tfplan
        |
blocking policy check
        |
manual approval for production
        |
terraform apply reviewed.tfplan
```

Apply the same saved plan that was scanned and approved. Creating a fresh plan after approval can introduce an unchecked change.

## 15. Optional industry-tool extension

Teams commonly add static analysis tools such as Checkov or TFLint. They provide maintained rule libraries and CI integrations, but they are additional installations and their rule sets change over time. If your instructor provides one, compare its findings with the local policy gate and document false positives or missing organization-specific requirements.

Checkov quick-start example after installation:

```powershell
checkov -d . --framework terraform
```

The custom `terraform_data` model may not match provider-specific Checkov policies. Use a real provider configuration or an instructor-provided sample when evaluating provider rules.

## 16. Cleanup

```powershell
terraform destroy -auto-approve -var-file=secure.tfvars
terraform state list
```

## 17. Review questions

1. Why can syntactically valid Terraform still be insecure?
2. What is the benefit of checking the saved plan rather than only source files?
3. Why must an automated policy gate return a non-zero exit code?
4. What is the difference between a Terraform `check` warning and a test assertion failure?
5. Why are `roles/owner` and `roles/editor` poor workload roles?
6. What security value do ownership and classification labels provide?
7. Why should production apply use the exact plan that passed policy?
8. What information belongs in a policy exception?
9. When might an advisory control be more appropriate than a blocking control?
10. What can a provider-specific scanner detect that this local model cannot?

## 18. Individual completion checklist

There is no submission requirement.

- [ ] Generated a Terraform plan in JSON format.
- [ ] Observed all intended failures in the insecure design.
- [ ] Passed the policy gate with the secure design.
- [ ] Applied only the reviewed secure plan.
- [ ] Ran native Terraform tests.
- [ ] Distinguished blocking policies from advisory checks.
- [ ] Designed a CI/CD quality-gate sequence.
- [ ] Destroyed all local practice objects.

## 19. References

- [Terraform plan command](https://developer.hashicorp.com/terraform/cli/commands/plan)
- [Terraform show JSON output](https://developer.hashicorp.com/terraform/cli/commands/show)
- [Terraform test command](https://developer.hashicorp.com/terraform/cli/commands/test)
- [Terraform validation mechanisms](https://developer.hashicorp.com/terraform/language/validate)
- [Checkov quick start](https://www.checkov.io/1.Welcome/Quick%20Start.html)
