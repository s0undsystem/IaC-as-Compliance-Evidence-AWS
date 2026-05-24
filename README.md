# compliant-s3

Purpose: A reviewed, signed, immutably-stored Terraform commit is stronger evidence than a screenshot. This lab builds the vault that holds your evidence and the script that puts it there.

Architecture:
   Lab 2.3 workspace                capture-evidence.sh                 Object Lock vault
   ─────────────────                ───────────────────                 ─────────────────
   tfplan, terraform/      ──▶      collect plan.json,        ──▶      s3://VAULT/runs/RUN_ID/
   .tf files, git log               state.json, commit.txt             bundle.tar.gz
                                    version.txt; SHA-256                Retention: GOVERNANCE
                                    each; tar; aws put-object           or COMPLIANCE
                                                  │
                                                  ▼
                                       single-line JSON receipt
                                       (run_id, key, version_id)

## What this repository contains

- `terraform/`: Terraform configuration that creates an AWS S3 evidence vault.
- `scripts/capture-evidence.sh`: Shell script that packages Terraform plan/state and git metadata, then uploads it to the vault.

## Behavior

- `terraform/` creates an S3 bucket with:
  - object lock enabled
  - versioning enabled
  - server-side encryption using AES-256
  - a bucket policy that denies bucket deletion to non-root principals
- `capture-evidence.sh` collects evidence from a workspace:
  - converts `tfplan` to `plan.json` if `tfplan` exists
  - pulls Terraform state into `state.json`
  - records the latest git commit in `commit.txt`
  - records Terraform version in `version.txt`
  - builds `manifest.json` with filename, sha256, and size
  - tars the bundle and uploads it to S3 under `runs/<RUN_ID>/bundle.tar.gz`
  - prints a JSON receipt with `run_id`, `vault`, `key`, and `version_id`

## Requirements

- Terraform 1.6 or later
- AWS CLI configured with credentials and permissions to create S3 resources and put objects
- `git` installed and available in the workspace if you want commit metadata
- `sha256sum` or `shasum -a 256` available on the system

## Terraform usage

```sh
cd terraform
terraform init
terraform apply
```

After apply, use the output `vault_name` as the `--vault` value for the capture script.

## Evidence capture usage

```sh
cd ..
./scripts/capture-evidence.sh --workspace <path> --run-id <id> --vault <bucket> [--profile <p>]
```

- `--workspace`: path to the Terraform workspace containing `tfplan`, state data, and git history
- `--run-id`: unique identifier for this evidence capture
- `--vault`: S3 bucket name from Terraform output
- `--profile`: optional AWS CLI profile

## Configuration

- `project_name` defaults to `cgep-lab`
- `lock_mode` defaults to `GOVERNANCE`; set to `COMPLIANCE` for production-grade evidence
- `retention_days` defaults to `1`

## Notes

- `terraform/.terraform.lock.hcl` is expected and should be kept for reproducible provider versions.
- Do not commit local state files, plan files, or generated evidence files.
- The script creates a temporary workspace bundle and removes it on exit.
