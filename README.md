# CI/CD Kubernetes Policy Pipeline

A production-grade GitHub Actions CI/CD pipeline demonstrating **Policy-as-Code** enforcement for Terraform infrastructure on Kubernetes using OPA/Rego with Conftest, running in ephemeral Kind clusters.

Built by **Vurumu Mahesh (VM)** — Platform Engineering / SRE portfolio project.

## Project Goal

Show an end-to-end CI/CD workflow that treats **infrastructure as code and policy as code** — validating Terraform changes, enforcing security/compliance rules with OPA/Rego inside a disposable Kubernetes cluster, and aggregating deployment reports — all before anything reaches a real environment.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          GITHUB ACTIONS WORKFLOW                             │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐   ┌────────────┐   ┌─────────────────┐   ┌───────────┐         │
│  │ version  │──▶│   report   │──▶│ terraform_check │──▶│  combine  │         │
│  │ (SHA+ts) │   │ dev/stg/prd│   │  (dev/stg/prd)  │   │  reports  │         │
│  └──────────┘   └────────────┘   └───────┬─────────┘   └───────────┘         │
│                                          │                                   │
│                                          ▼                                   │
│                                ┌────────────────────┐                        │
│                                │   k8s-policy-check  │                       │
│                                │  (Conftest + Kind)  │                       │
│                                └─────────┬──────────┘                        │
│                                          │                                   │
│                               ┌──────────┼──────────┐                        │
│                               ▼          ▼          ▼                        │
│                          ┌─────────┐ ┌────────┐ ┌────────┐                   │
│                          │  Kind   │ │ Apply  │ │ Wait & │                   │
│                          │ Cluster │▶│ Job    │▶│ Debug  │                   │
│                          └─────────┘ └────────┘ └────────┘                   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Repository Layout

```
.
├── .github/
│   ├── workflows/pipeline.yaml     # Main CI/CD workflow
│   └── actions/generate-report/    # Composite action for deployment reports
├── ci-cd-k8s-policy-pipeline/      # The core project
│   ├── terraform/                  # Infrastructure as Code
│   ├── policy-enforcement/         # OPA/Rego + Conftest + Kind
│   ├── Taskfile.yaml               # Task runner scripts
│   ├── devbox.json                 # Reproducible dev environment
│   ├── README.md                   # Pipeline deep-dive
│   └── GITHUB_ACTIONS_README.md    # Workflow internals
├── k8s-practice/                   # K8s learning exercises (gitignored)
└── opa_practice/                   # OPA/Rego learning exercises (gitignored)
```

> A deep dive into the inner pipeline lives in [`ci-cd-k8s-policy-pipeline/README.md`](ci-cd-k8s-policy-pipeline/README.md).

## CI/CD Pipeline

| Job | Purpose | Matrix | Environment |
|-----|---------|--------|-------------|
| `version` | Generate build timestamp + short SHA | — | — |
| `report` | Generate deployment report | `dev`, `staging`, `prod` | GitHub Environments |
| `terraform_check` | `fmt` → `init` → `validate` → `apply` | `dev`, `staging`, `prod` | GitHub Environments |
| `combine` | Merge all report + output artifacts | — | — |
| `k8s-policy-check` | Policy validation in an ephemeral Kind cluster | — | — |

### Triggers & Concurrency
Runs on push/PR to `main` and manual dispatch. Concurrency control cancels stale runs:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### Environment Protection
`dev`, `staging`, and `prod` are protected GitHub Environments — each requires manual approval before `terraform_check` applies changes.

## Policy Examples (`policy-enforcement/policies/terraform.rego`)

Policies are written in OPA/Rego and evaluated by Conftest against the Terraform plan, using Rego v1 syntax:

```rego
package main

import rego.v1

deny contains msg if {
  resource := input.resource_changes[_]
  resource.type == "local_file"
  not resource.change.after.filename
  msg := "local_file resources must specify a filename"
}
```

### ❌ Example Failed Policy Check

A `local_file` resource missing its required `filename` is rejected:

```bash
$ conftest test ci-cd-k8s-policy-pipeline/terraform/*.tf \
    --policy ci-cd-k8s-policy-pipeline/policy-enforcement/policies

FAIL - main - local_file resources must specify a filename

1 test, 0 passed, 1 failed
```

### ✅ Example Successful Check

A valid resource with `filename` set passes cleanly:

```bash
$ conftest test ci-cd-k8s-policy-pipeline/terraform/*.tf \
    --policy ci-cd-k8s-policy-pipeline/policy-enforcement/policies

PASS - main - ci-cd-k8s-policy-pipeline/terraform/modules/deployment_record/main.tf

1 test, 1 passed, 0 failed
```

## How the Policy Check Runs in CI

1. **`helm/kind-action`** boots an ephemeral Kind cluster using `kind-config.yaml`.
2. The repo is **hostPath-mounted** at `/workspace` inside the cluster.
3. A Kubernetes **Job** (`tf-check`) runs `openpolicyagent/conftest` against the Terraform files.
4. CI **waits** for the Job to complete; on failure it dumps pod status and logs.
5. The ephemeral cluster is **destroyed** after the run.

```yaml
# k8s-policy-check job step
- name: Apply Terraform check Job
  run: kubectl apply -f ci-cd-k8s-policy-pipeline/policy-enforcement/job.yaml

- name: Wait for Job to complete
  run: kubectl wait --for=condition=complete job/tf-check -n dev --timeout=60s

- name: Debug - show pod status if failed
  if: failure()
  run: |
    kubectl get pods -n dev
    kubectl describe pod -n dev -l job-name=tf-check
    kubectl logs -n dev -l job-name=tf-check --all-containers=true || true
```

## Terraform Infrastructure (`ci-cd-k8s-policy-pipeline/terraform/`)

Provisioned through a reusable `deployment_record` module that writes an environment-tagged deployment record file:

```
terraform/
├── main.tf                      # Root module -> deployment_record
├── variables.tf                 # environment input
├── outputs.tf                   # created_filename, environment_used
├── modules/deployment_record/   # Reusable module
└── .terraform.lock.hcl          # Provider lock file
```

```hcl
resource "local_file" "deployment_record" {
  filename = "deployment-record-${var.environment}.txt"
  content  = "Deployed to: ${var.environment}"
}
```

## Local Development

### Prerequisites
- Terraform ≥ 1.7
- Docker (for Kind)
- `kubectl` ≥ 1.28
- `conftest` ≥ 0.48
- `devbox` (optional, for a reproducible env)

### Quick Start

```bash
# 1. Enter the reproducible dev environment
devbox shell

# 2. Format / validate Terraform
task tf:fmt
task tf:validate

# 3. Boot a local Kind cluster
task kind:create

# 4. Run policy checks locally
kubectl apply -f ci-cd-k8s-policy-pipeline/policy-enforcement/job.yaml
kubectl wait --for=condition=complete job/tf-check -n dev --timeout=60s
kubectl logs job/tf-check -n dev

# 5. Cleanup
task kind:delete
```

## Security Considerations

- No secrets in the repo — GCP auth via GitHub Environments + OIDC
- Provider lock file committed (`.terraform.lock.hcl`)
- Policy-as-Code catches misconfigurations at CI time
- Ephemeral Kind clusters destroyed after each run

## Experience

**Software Engineer Intern — Divami Design Labs** (May–June 2026)
Built AI-NIMS, a PWA for NIMS Hyderabad's Cardiothoracic Surgery department — React/TypeScript, FastAPI/Python with Gemini API and Pydantic AI.

## Contact

**Vurumu Mahesh (VM)**
📧 [vgsvpmahesh@gmail.com](mailto:vgsvpmahesh@gmail.com)
🔗 [GitHub](https://github.com/VurumuMahesh15)
