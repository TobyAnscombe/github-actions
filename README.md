# github-actions

Centralised reusable GitHub Actions workflows for all TobyAnscombe projects.

All workflows use `on: workflow_call` and are versioned on `main`. Calling repos reference them as:

```yaml
uses: TobyAnscombe/github-actions/.github/workflows/<workflow>.yml@main
```

---

## Available workflows

### `terraform-ci.yml` — Terraform CI

Three-job pipeline for any Terraform project.

| Job | Steps |
|-----|-------|
| `terraform` | `fmt -check -recursive` → `init -backend=false` → `validate` |
| `tflint` | `tflint --init` → `tflint --recursive` |
| `security` | Install Trivy (apt-get) → `trivy config . --severity ... --exit-code 1` |

**Inputs**

| Input | Default | Description |
|-------|---------|-------------|
| `terraform_version` | `~> 1.5` | Version constraint for `hashicorp/setup-terraform` |
| `tflint_version` | `v0.55.0` | TFLint release to install |
| `working_directory` | `.` | Root module directory |
| `trivy_severity` | `HIGH,CRITICAL` | Severity levels that fail the build |

**Caller example** (`.github/workflows/ci.yml`):

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  ci:
    uses: TobyAnscombe/github-actions/.github/workflows/terraform-ci.yml@main
    with:
      terraform_version: "~> 1.5"
      tflint_version: "v0.55.0"
```

**Prerequisites in the calling repo:**
- `.tflint.hcl` — TFLint plugin config (e.g. `tflint-ruleset-azurerm` or `tflint-ruleset-aws`)
- `.trivyignore` — accepted-risk suppressions with justifications

---

### `gitleaks.yml` — Secret scanning

Scans the full git history for secrets using [gitleaks](https://github.com/gitleaks/gitleaks) and uploads SARIF results to the GitHub Security tab.

**Inputs**

| Input | Default | Description |
|-------|---------|-------------|
| `gitleaks_config` | `""` | Path to `.gitleaks.toml` — omit to use built-in rules |

**Caller example** (`.github/workflows/secret-scan.yml`):

```yaml
name: Secret scan
on:
  push:
    branches: ["**"]
  pull_request:
    branches: ["**"]
jobs:
  secret-scan:
    uses: TobyAnscombe/github-actions/.github/workflows/gitleaks.yml@main
    with:
      gitleaks_config: ".gitleaks.toml"  # omit if no custom config
```

**Required repo setting:** Go to **Settings → Actions → General → Workflow permissions** and ensure *Read and write permissions* or at minimum that `security-events: write` is allowed (required to upload SARIF).

---

### `ansible-ci.yml` — Ansible lint

Installs `ansible-lint`, optionally installs Ansible Galaxy collections, then lints the repository.

**Inputs**

| Input | Default | Description |
|-------|---------|-------------|
| `ansible_lint_version` | `26.1.0` | `pip install ansible-lint==<version>` |
| `python_version` | `3.12` | Python version for the runner |
| `working_directory` | `.` | Directory to lint |
| `requirements_file` | `requirements.yml` | Galaxy requirements file — set to `""` to skip |

**Caller example** (`.github/workflows/ansible-lint.yml`):

```yaml
name: Ansible Lint
on:
  push:
    branches: ["**"]
  pull_request:
    branches: ["**"]
jobs:
  ansible-lint:
    uses: TobyAnscombe/github-actions/.github/workflows/ansible-ci.yml@main
    with:
      requirements_file: "requirements.yml"
```

---

## Adding CI to a new Terraform project

1. Copy the appropriate caller workflow below into `.github/workflows/ci.yml`
2. Add `.tflint.hcl` with the correct provider plugin
3. Add `.trivyignore` (can be empty initially)
4. Run `terraform fmt -recursive` locally before the first push

**Azure project `.tflint.hcl`:**

```hcl
plugin "azurerm" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-azurerm"
}
config { format = "compact" }
```

**AWS project `.tflint.hcl`:**

```hcl
plugin "aws" {
  enabled = true
  version = "0.38.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
config { format = "compact" }
```
