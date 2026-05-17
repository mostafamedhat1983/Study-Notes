tags:
  - Terraform
---

Pre-commit hooks let you automatically run checks **before each git commit**. This helps you:

- keep formatting and style consistent  
- catch syntax and logic mistakes early  
- generate up-to-date documentation for modules  
- detect security issues before code even leaves your machine  

A typical Terraform pre-commit stack:

- `terraform fmt` – format code  
- `terraform validate` – syntax & internal consistency  
- `tflint` – lint + provider/best‑practice checks  
- `terraform-docs` – generate module documentation  
- `tfsec` – security scanning for Terraform

---

## 1. Install the pre-commit framework

Install once on your machine:

```bash
pip install pre-commit
```

(or use your preferred way to install the `pre-commit` CLI.)

---

## 2. Create `.pre-commit-config.yaml`

In the **root** of your Terraform repo, add a file named:

```text
.pre-commit-config.yaml
```

Example configuration that runs fmt, validate, tflint, terraform-docs, and tfsec:

```yaml
repos:
  # Terraform fmt / validate / tflint / docs
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0   # pin to a specific released version
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_docs

  # tfsec security scanner
  - repo: https://github.com/aquasecurity/tfsec
    rev: v1.28.1   # example version; pin to a stable tag
    hooks:
      - id: tfsec
```

What each hook does:

- **`terraform_fmt`**  
  Runs `terraform fmt` (usually with `-recursive`) on staged Terraform files to enforce standard formatting.

- **`terraform_validate`**  
  Runs `terraform validate` to check that configuration is syntactically valid and internally consistent. It does not talk to the real cloud.

- **`terraform_tflint`**  
  Runs TFLint to lint Terraform syntax and provider usage (invalid instance types, regions, some security/best‑practice rules).

- **`terraform_docs`**  
  Runs terraform-docs to generate or update documentation for your modules (typically README tables for inputs and outputs).

- **`tfsec`**  
  Runs tfsec to statically scan Terraform for security misconfigurations (public S3 buckets, open security groups, missing encryption, etc.).

---

## 3. Install the Git hook in the repo

From the repo root:

```bash
pre-commit install
```

This writes a `.git/hooks/pre-commit` script that runs `pre-commit` automatically whenever you run:

```bash
git commit
```

New workflow:

1. Edit Terraform files (for example in VS Code).  
2. `git add ...`  
3. `git commit`  
4. Pre-commit runs all hooks on staged files.  
   - If they succeed, the commit goes through.  
   - If any fail, the commit is blocked; you fix issues and commit again.

---

## 4. Run all hooks once on the whole repo

When you first enable pre-commit (or after cloning on a new machine), it is useful to fix everything in one shot:

```bash
pre-commit run --all-files
```

This will:

- format all Terraform files  
- validate configuration  
- lint with TFLint  
- regenerate docs with terraform-docs  
- run a full tfsec scan  

After this initial cleanup, future commits will only check changed files.

---

## 5. Why run security checks at pre-commit time?

Running `tfsec` (and TFLint) in pre-commit means:

- you see security issues immediately while coding  
- you avoid pushing obviously insecure changes to the remote repo  
- CI pipelines become a second safety net, not the first place you learn about misconfigurations  

A good pattern is:

- **pre-commit (local):** `fmt`, `validate`, `tflint`, `terraform-docs`, `tfsec`  
- **CI (remote):** possibly a heavier tfsec or Checkov run over the entire repo, plus any org-specific policies

---

## 6. Tool selection: do you need more than one security tool?

At minimum:

- `tflint` + `tfsec` together already give you:
  - syntax and provider-usage linting  
  - many common security and best-practice checks  

In larger environments you might also add:

- **Checkov** in CI for broader multi-IaC security/compliance checks.

But for an individual Terraform-heavy project, this stack is practical and strong:

- pre-commit: `terraform_fmt`, `terraform_validate`, `terraform_tflint`, `terraform_docs`, `tfsec`  
- CI: optional additional `tfsec` / Checkov run across all modules

---