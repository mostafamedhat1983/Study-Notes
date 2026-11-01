---
tags:
  - Terraform
  - Troubleshooting
---

Knowing these problems helps you diagnose Terraform configuration, state, provider, and plan failures safely.

---

## Main idea

Most Terraform problems involve syntax, provider versions, authentication, state drift, dependencies, or unsafe apply behavior.

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. Terraform cannot find a command

- **Why it happens:** Terraform is not installed or is outside PATH.
- **Quick sign:** The shell reports command not found.
- **Solution:** Install the approved version and verify its location.
- **Useful commands:** `command -v terraform`, `terraform version`

---

## 2. Configuration syntax error

- **Why it happens:** HCL blocks, braces, quotes, or expressions are malformed.
- **Quick sign:** `terraform validate` or plan reports a syntax error with a file and line.
- **Solution:** Fix the reported syntax and review nearby blocks; validate before planning.
- **Useful commands:** `terraform fmt -check`, `terraform validate`

---

## 3. Unsupported argument or block

- **Why it happens:** The argument does not exist in the selected provider/resource version.
- **Quick sign:** Plan reports an unsupported argument.
- **Solution:** Check the provider documentation for the exact version and adjust the configuration or provider constraint.
- **Useful commands:** `terraform providers`, `terraform version`

---

## 4. Provider version conflict

- **Why it happens:** Multiple modules or configurations require incompatible provider versions.
- **Quick sign:** Init fails while resolving provider versions.
- **Solution:** Align required_provider constraints and module expectations, then reinitialize.
- **Useful commands:** `terraform init -upgrade`, `terraform providers`

---

## 5. Provider authentication fails

- **Why it happens:** Cloud credentials are absent, expired, or associated with the wrong account/region.
- **Quick sign:** Plan or apply reports authentication or authorization failure.
- **Solution:** Verify the provider's supported credential chain and intended identity; prefer roles or workload identity over static keys.
- **Useful commands:** `aws sts get-caller-identity`, `terraform plan`

---

## 6. Backend initialization fails

- **Why it happens:** Backend credentials, bucket/container, region, or permissions are incorrect.
- **Quick sign:** `terraform init` fails while configuring the backend.
- **Solution:** Verify backend configuration and access permissions; never place state in a publicly accessible location.
- **Useful commands:** `terraform init -reconfigure`, `aws s3 ls s3://<state-bucket>`

---

## 7. State lock is held

- **Why it happens:** Another apply is running or a previous run left a stale lock.
- **Quick sign:** Terraform reports that state is locked.
- **Solution:** Confirm no operation is active, inspect lock metadata, and use the backend's documented unlock procedure only when safe.
- **Useful commands:** `terraform force-unlock <lock-id>`

---

## 8. Resource already exists

- **Why it happens:** Infrastructure was created outside Terraform or by another state/configuration.
- **Quick sign:** Apply reports that the resource already exists.
- **Solution:** Import the resource into the intended configuration and state, or change the resource address; do not delete production resources casually.
- **Useful commands:** `terraform import`, `terraform state list`

---

## 9. Resource not found during refresh

- **Why it happens:** The resource was deleted outside Terraform or the provider cannot access it.
- **Quick sign:** Plan reports the resource no longer exists.
- **Solution:** Confirm whether deletion was intentional; remove it from state or recreate it according to the desired configuration.
- **Useful commands:** `terraform plan`, `terraform state list`, `terraform state rm`

---

## 10. State file is missing or corrupted

- **Why it happens:** The backend object was deleted, moved, or modified outside Terraform.
- **Quick sign:** Terraform cannot read state or reports checksum/version errors.
- **Solution:** Restore from a verified state backup; do not recreate state manually.
- **Useful commands:** `terraform state list`, backend-specific backup/restore procedure

---

## 11. State contains resources from another environment

- **Why it happens:** Workspaces, directories, or backend keys were mixed up.
- **Quick sign:** Plan proposes changes to unexpected resources.
- **Solution:** Stop before applying, verify workspace/backend key and environment variables, and inspect the resource list.
- **Useful commands:** `terraform workspace show`, `terraform state list`, `terraform plan`

---

## 12. Plan shows unexpected destroy

- **Why it happens:** A required attribute changed, a resource was renamed, or configuration/state diverged.
- **Quick sign:** Plan includes `- destroy` for resources that should remain.
- **Solution:** Read the diff carefully and use targeted planning; prevent accidental replacement with lifecycle rules only when the semantics are understood.
- **Useful commands:** `terraform plan -out=tfplan`, `terraform show -no-color tfplan`, `terraform plan -target=<resource>`

---

## 13. Apply fails halfway

- **Why it happens:** A dependency failed, a quota was reached, or an external service rejected a change.
- **Quick sign:** Some resources are created while the apply reports an error.
- **Solution:** Save the plan/log evidence, correct the root cause, and re-run plan before applying; avoid repeated blind applies.
- **Useful commands:** `terraform plan -out=tfplan`, `terraform apply tfplan`

---

## 14. Dependency order is wrong

- **Why it happens:** Terraform cannot infer a dependency from configuration alone.
- **Quick sign:** A resource is created before its dependency exists.
- **Solution:** Reference the dependency's attribute so Terraform can infer order, or add an explicit dependency only when necessary.
- **Useful commands:** `terraform graph`, `terraform plan`

---

## 15. Circular dependency

- **Why it happens:** Resources reference each other directly or indirectly.
- **Quick sign:** Terraform reports a dependency cycle.
- **Solution:** Redesign references, split the resource relationship, or introduce an intermediate resource where appropriate.
- **Useful commands:** `terraform graph`

---

## 16. Variable type mismatch

- **Why it happens:** Input values do not match declared variable types or validation rules.
- **Quick sign:** Plan fails with a type-constraint or validation error.
- **Solution:** Inspect variable declarations and supplied values; use explicit types and validation for critical inputs.
- **Useful commands:** `terraform validate`, `terraform plan -var-file=<file>`

---

## 17. Required variable is missing

- **Why it happens:** A variable has no default and was not supplied.
- **Quick sign:** Plan asks for an unset variable.
- **Solution:** Supply the variable through the approved mechanism; avoid committing secrets to variable files.
- **Useful commands:** `terraform plan -var 'name=value'`, `terraform plan -var-file=<file>`

---

## 18. Sensitive value appears in output

- **Why it happens:** A secret-bearing value was marked or exposed incorrectly.
- **Quick sign:** A secret appears in plan, state, logs, or output.
- **Solution:** Rotate the exposed secret, mark sensitive values appropriately, and restrict output and state access.
- **Useful commands:** `terraform output`, `terraform state pull`

---

## 19. Drift between state and real infrastructure

- **Why it happens:** Manual console changes, automation outside Terraform, or provider-side changes altered infrastructure.
- **Quick sign:** Plan reports changes although the configuration did not change.
- **Solution:** Review the drift, decide whether to accept, import, update state, or change configuration; do not apply blindly.
- **Useful commands:** `terraform plan -detailed-exitcode`, `terraform state list`

---

## 20. CI/CD apply behaves differently from local runs

- **Why it happens:** Working directory, backend configuration, variables, provider versions, or credentials differ.
- **Quick sign:** The same code produces different plans locally and in CI.
- **Solution:** Compare version, backend, workspace, variables, and identity; make the pipeline deterministic.
- **Useful commands:** `terraform version`, `terraform workspace show`, `terraform providers`, `terraform plan -out=tfplan`

---

## Common mistakes

- applying without reviewing the full plan
- editing state directly without a backup
- storing state in an unsecured location
- using broad provider version constraints inconsistently
- committing secrets or sensitive plan output

---

## Good practices

- run fmt and validate before commit
- always review plan output before apply
- use remote state with locking and encryption
- keep one environment per backend key/workspace strategy
- pin provider versions deliberately
- use least-privilege deployment credentials

---

## Must memorize

```bash
terraform version
terraform init -upgrade
terraform fmt -check
terraform validate
terraform plan -out=tfplan
terraform show -no-color tfplan
terraform state list
terraform workspace show
terraform providers
```

---

## Key ideas

- Plan is the safety checkpoint; never skip reviewing it.
- State records Terraform's view of real infrastructure.
- Provider and backend configuration strongly affect behavior.
- Drift requires investigation, not automatic correction.
