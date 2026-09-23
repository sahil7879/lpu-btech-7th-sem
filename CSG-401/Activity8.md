# Individual Activity 8: Terraform State File Failure and Recovery

**Course:** CSG 401 - Getting Started with Terraform in GCP  
**Activity type:** Controlled failure-and-recovery practical  
**Suggested duration:** 2-3 hours  
**Cloud or billing account required:** No  
**External provider required:** No

## 1. Purpose

This activity demonstrates what happens when someone manually changes, corrupts, or deletes a Terraform state file. You will deliberately cause failures and recover from verified backups.

The activity uses only Terraform's built-in `terraform_data` resource. Never perform these experiments against the state used by the GCP capstone or any shared environment.

## 2. Safety boundary

Create a new disposable directory for this activity. Before every experiment, confirm that the current directory contains the practice configuration shown here.

Do not perform the activity when:

- A cloud provider is configured
- The directory uses a production or shared backend
- Another Terraform process is running
- The state belongs to another activity
- You do not have a verified backup

Direct state editing is demonstrated only to show why it is unsafe. Terraform documentation instructs users to use supported state commands instead of editing `terraform.tfstate` directly.

## 3. Learning outcomes

After completing the activity, you should be able to:

1. Explain the relationship between configuration, state, and real infrastructure.
2. Create and verify a state backup.
3. Observe how valid but incorrect manual state changes affect outputs and plans.
4. Recognize errors caused by malformed state JSON.
5. Predict what Terraform plans after state is deleted.
6. Recover local state using a verified snapshot.
7. Explain why applying after state loss can duplicate real cloud resources.
8. Describe safer recovery options for a versioned remote backend.

## 4. Failure model

```text
Configuration (.tf)
       |
       | compared during plan
       v
Terraform state ----------------> Object identity and cached attributes
       |
       `-- If lost, Terraform no longer knows which real objects it owns
```

State loss does not normally delete remote infrastructure. It removes Terraform's mapping to that infrastructure. A subsequent plan can therefore propose creating replacements while the original objects still exist.

## 5. Create the disposable project

Create:

```text
terraform-state-failure-lab/
|-- main.tf
|-- variables.tf
|-- outputs.tf
`-- .gitignore
```

Create `variables.tf`:

```hcl
variable "environment" {
  description = "Name stored in the local practice state."
  type        = string
  default     = "dev"
}
```

Create `main.tf`:

```hcl
terraform {
  required_version = ">= 1.7.0"
}

resource "terraform_data" "environment" {
  input = {
    name  = var.environment
    owner = "student"
  }
}

resource "terraform_data" "service" {
  input = {
    name           = "web"
    environment_id = terraform_data.environment.id
    port           = 80
  }
}
```

Create `outputs.tf`:

```hcl
output "environment_name" {
  value = var.environment
}

output "environment_id" {
  value = terraform_data.environment.id
}

output "service_record" {
  value = terraform_data.service.output
}
```

Create `.gitignore`:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
state-backup*.json
lost-state*.json
corrupt-state*.json
baseline-*.json
```

## 6. Establish and record the baseline

Run:

```powershell
terraform init
terraform fmt
terraform validate
terraform plan -out=baseline.tfplan
terraform apply baseline.tfplan
terraform state list
terraform output
```

Expected addresses:

```text
terraform_data.environment
terraform_data.service
```

Create a machine-readable baseline record:

```powershell
terraform show -json | Set-Content -Encoding utf8 baseline-show.json
terraform output -json | Set-Content -Encoding utf8 baseline-outputs.json
```

On Bash, replace the PowerShell pipes with:

```bash
terraform show -json > baseline-show.json
terraform output -json > baseline-outputs.json
```

## 7. Create and verify backups

Create two independent copies:

```powershell
terraform state pull | Set-Content -Encoding utf8 state-backup-01.json
Copy-Item terraform.tfstate state-backup-file-copy.json
```

Bash equivalent:

```bash
terraform state pull > state-backup-01.json
cp terraform.tfstate state-backup-file-copy.json
```

Verify that the pulled snapshot is readable without replacing active state:

```powershell
terraform show -json state-backup-01.json | ConvertFrom-Json | Select-Object -ExpandProperty format_version
```

Bash equivalent:

```bash
terraform show -json state-backup-01.json | jq -r .format_version
```

Record the lineage and serial values:

```powershell
$snapshot = Get-Content state-backup-01.json -Raw | ConvertFrom-Json
$snapshot | Select-Object lineage, serial, terraform_version
```

Bash equivalent:

```bash
jq '{lineage, serial, terraform_version}' state-backup-01.json
```

Do not continue unless the backup parses successfully and contains both resources.

## 8. Experiment A - Make a valid but incorrect manual change

This is the only experiment where you intentionally edit valid state data.

1. Close every Terraform process.
2. Open `terraform.tfstate` in a plain-text editor.
3. Locate the root output named `environment_name`.
4. Change only its stored value from `dev` to `tampered-manually`.
5. Preserve valid JSON and save the file as UTF-8 without a byte-order mark.

Run:

```powershell
terraform output environment_name
terraform plan
```

Expected observations:

- `terraform output` initially trusts the changed state and displays `tampered-manually`.
- The configuration still defines `dev`.
- `terraform plan` proposes correcting the root output.
- Terraform cannot know why the stored value changed.

Repair through the normal workflow:

```powershell
terraform apply
terraform output environment_name
```

Expected final value: `dev`.

Question: What could happen if the edited value were an object ID rather than a simple output?

## 9. Experiment B - Corrupt the JSON structure

Create another recovery point:

```powershell
terraform state pull | Set-Content -Encoding utf8 state-backup-02.json
```

Open `terraform.tfstate`, delete its final closing brace, and save it. The file is now malformed JSON.

Run:

```powershell
terraform state list
terraform plan
```

Terraform should refuse to load the state. Do not try random edits until the error disappears; restore the verified snapshot.

PowerShell recovery:

```powershell
Copy-Item state-backup-02.json terraform.tfstate -Force
terraform state list
terraform plan
```

Bash recovery:

```bash
cp state-backup-02.json terraform.tfstate
terraform state list
terraform plan
```

The plan should return to no changes.

## 10. Experiment C - Delete or lose the state file

First create a final verified snapshot:

```powershell
terraform state pull | Set-Content -Encoding utf8 state-backup-03.json
terraform show state-backup-03.json
```

Simulate accidental loss without permanently discarding the file:

```powershell
Move-Item terraform.tfstate lost-state-original.json
```

Bash equivalent:

```bash
mv terraform.tfstate lost-state-original.json
```

Inspect the directory and run:

```powershell
terraform state list
terraform plan
```

Expected observations:

- Terraform behaves as though no objects are managed.
- `terraform state list` reports that no state file was found (or shows no managed objects if an empty state snapshot exists).
- The plan proposes creating both configured objects.
- The configuration did not change; only Terraform's memory of ownership disappeared.

**Do not apply this plan.** With real GCP resources, applying after state loss could:

- Create duplicate resources with different names or IDs
- Fail because globally unique names already exist
- Allocate additional billable VMs, disks, addresses, or databases
- Overwrite or conflict with existing policies
- Leave the original resources orphaned and still billable

## 11. Recover deleted state

Use one of the following methods, not both.

### Method A - Restore the verified file

PowerShell:

```powershell
Copy-Item state-backup-03.json terraform.tfstate -Force
terraform state list
terraform plan
```

Bash:

```bash
cp state-backup-03.json terraform.tfstate
terraform state list
terraform plan
```

### Method B - Push a verified snapshot through Terraform

Remove the newly created empty state file first if Terraform generated one while planning. Preserve it rather than deleting it permanently:

```powershell
if (Test-Path terraform.tfstate) {
  Move-Item terraform.tfstate empty-state-after-loss.json
}

terraform state push state-backup-03.json
terraform state list
terraform plan
```

Bash equivalent:

```bash
if [ -f terraform.tfstate ]; then
  mv terraform.tfstate empty-state-after-loss.json
fi

terraform state push state-backup-03.json
terraform state list
terraform plan
```

`terraform state push` performs lineage and serial safety checks. Do not use `-force` merely to bypass a rejection. Investigate why the destination is newer or belongs to a different lineage.

## 12. Compare direct editing with supported operations

| Goal | Unsafe approach | Supported approach |
|---|---|---|
| Inspect an object | Read and interpret internal JSON | `terraform state show ADDRESS` |
| List managed objects | Search the JSON manually | `terraform state list` |
| Capture a snapshot | Copy an unknown temporary file | `terraform state pull` |
| Rename an address | Search-and-replace JSON | A declarative `moved` block or `terraform state mv` |
| Stop managing an object | Delete its JSON object | `terraform state rm` after review and backup |
| Restore a snapshot | Paste JSON fragments | Restore a verified backend version or use `state push` carefully |
| Recover an existing remote object | Recreate it blindly | Add configuration and use an `import` block or `terraform import` |

## 13. Real GCP recovery decision exercise

Assume a GCP state file is lost but the VPC, VM, and bucket still exist.

Use this recovery order:

1. Stop all Terraform applies.
2. Preserve the current directory, logs, plans, and any remaining state artifacts.
3. Check the remote backend and its object-version history.
4. Identify the latest trustworthy snapshot and verify its lineage and age.
5. Compare its managed addresses with real GCP resources.
6. Restore the reviewed snapshot through the backend's recovery process.
7. Run `terraform plan` and investigate every proposed action.
8. If no usable state exists, reconstruct configuration and import existing objects one at a time.
9. Do not apply until unexpected creation and destruction actions have been resolved.

State recovery is an incident-management task, not a routine apply.

## 14. Local versus remote state

| Capability | Local state | Versioned GCS backend |
|---|---|---|
| Location | Workstation file | Cloud Storage object |
| Accidental deletion protection | Manual copies and `.backup` | Object version history when enabled |
| Team access | Unsafe file sharing | Centralized IAM-controlled access |
| Locking | Local process/filesystem behavior | Backend-supported locking behavior |
| Auditability | Operating-system dependent | Cloud audit and object history options |
| Recovery | Restore trusted file | Restore reviewed object version or push snapshot |

Remote state reduces some risks but does not make state indestructible. Apply least privilege, versioning, encryption, audit logging, and protected retention appropriate to the environment.

## 15. Cleanup

After the active state has been restored and `terraform plan` shows no unexpected change:

```powershell
terraform destroy -auto-approve
terraform state list
```

The final state list should be empty.

Keep the activity directory only if you want to review the backup files. Every backup contains state data and must be protected like the active state.

## 16. Review questions

1. Why did changing only the stored output affect `terraform output`?
2. Why could Terraform repair the output using configuration?
3. What caused Terraform to reject malformed JSON?
4. Why did deleting state make Terraform propose new objects?
5. Would deleting state delete an existing Compute Engine VM?
6. What risks arise from applying immediately after state loss?
7. What are state lineage and serial used to protect?
8. Why should `terraform state push -force` be exceptional?
9. When is importing safer than applying?
10. How does GCS object versioning improve recovery?
11. Why must state backups be protected as sensitive data?
12. Why should direct state editing never be part of a normal workflow?

## 17. Individual completion checklist

There is no submission requirement.

- [ ] Created and verified three state snapshots.
- [ ] Observed a manually altered output value.
- [ ] Repaired the altered value through normal Terraform operations.
- [ ] Observed Terraform reject malformed state JSON.
- [ ] Restored valid state from a verified backup.
- [ ] Simulated complete local-state loss.
- [ ] Confirmed that Terraform proposed recreating all configured objects.
- [ ] Recovered the state without applying the recreation plan.
- [ ] Confirmed a no-change plan after recovery.
- [ ] Destroyed the disposable local objects.

## 18. References

- [Terraform state overview](https://developer.hashicorp.com/terraform/language/state)
- [Purpose of Terraform state](https://developer.hashicorp.com/terraform/language/state/purpose)
- [Terraform state commands](https://developer.hashicorp.com/terraform/cli/commands/state)
- [`terraform state pull`](https://developer.hashicorp.com/terraform/cli/commands/state/pull)
- [Recover state from backup](https://developer.hashicorp.com/terraform/cli/state/recover)
- [State storage and locking](https://developer.hashicorp.com/terraform/language/state/backends)
- [Manage sensitive data](https://developer.hashicorp.com/terraform/language/manage-sensitive-data)
