# Top 20 Easy Terraform Interview Questions & Answers

**Topic:** Terraform fundamentals — for screening and entry-level interviews.

---

## 1. What is Terraform?

Terraform is an **open-source Infrastructure as Code (IaC)** tool from HashiCorp that lets you define cloud and on-prem infrastructure in declarative configuration files (HCL — HashiCorp Configuration Language) and provision it automatically.

You describe the **desired state**; Terraform figures out the steps to reach it.

---

## 2. What language does Terraform use?

**HCL (HashiCorp Configuration Language)** — a human-readable, declarative language.

Terraform also accepts **JSON** for machine-generated configs.

```hcl
resource "google_storage_bucket" "logs" {
  name     = "my-logs-bucket"
  location = "US"
}
```

---

## 3. What is Infrastructure as Code (IaC)?

IaC is the practice of managing and provisioning infrastructure (servers, networks, databases, etc.) through **machine-readable files** instead of manual processes or GUI clicks.

Benefits:
- **Version control** (Git)
- **Repeatability** — same config produces same infra
- **Automation** — CI/CD pipelines
- **Documentation** — code itself describes the infra

---

## 4. What is a Terraform provider?

A **provider** is a plugin that lets Terraform talk to a specific platform's API (AWS, GCP, Azure, Kubernetes, GitHub, etc.).

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = "my-gcp-project"
  region  = "us-central1"
}
```

---

## 5. What is a Terraform resource?

A **resource** is a single piece of infrastructure — a VM, bucket, DNS record, IAM policy, etc.

```hcl
resource "google_compute_instance" "web" {
  name         = "web-server"
  machine_type = "e2-medium"
  zone         = "us-central1-a"
}
```

Syntax: `resource "<TYPE>" "<NAME>" { ... }`

---

## 6. What are the core Terraform commands?

| Command | Purpose |
|---|---|
| `terraform init` | Initialize working directory, download providers/modules |
| `terraform plan` | Show what changes will be made (dry run) |
| `terraform apply` | Apply the changes to reach desired state |
| `terraform destroy` | Delete all resources managed by the config |
| `terraform validate` | Check syntax |
| `terraform fmt` | Auto-format `.tf` files |
| `terraform show` | Display current state |

---

## 7. What is `terraform init` and when do you run it?

`terraform init` initializes a working directory by:
- Downloading required **providers**
- Downloading **modules**
- Configuring the **backend** (where state is stored)

Run it:
- Once when you first clone the repo
- After adding/changing providers, modules, or backend config

---

## 8. What is the difference between `terraform plan` and `terraform apply`?

- **`terraform plan`** → previews changes; tells you what will be **created, updated, or destroyed**. It does *not* change anything.
- **`terraform apply`** → executes those changes against the real infrastructure.

Always run `plan` first in production to avoid surprises.

---

## 9. What is Terraform state?

The **state file** (`terraform.tfstate`) is a JSON file that maps your config to the **real-world resources** Terraform manages.

It stores:
- Resource IDs
- Attribute values
- Dependencies

Without state, Terraform wouldn't know which real resource your config refers to.

**Never edit it manually** — use `terraform state` commands.

---

## 10. Where should you store the Terraform state file?

**Locally** (`terraform.tfstate`) for solo learning only.

**Remotely** for any team or production use — using a **backend** like:
- **GCS** (Google Cloud Storage)
- **S3** + DynamoDB (locking)
- **Terraform Cloud / Enterprise**
- **Azure Blob Storage**

Remote backend provides:
- Shared access
- **State locking** (prevents concurrent runs)
- Encryption at rest

```hcl
terraform {
  backend "gcs" {
    bucket = "my-tf-state"
    prefix = "prod"
  }
}
```

---

## 11. What are Terraform variables?

Variables let you **parameterize** configurations — no hardcoded values.

```hcl
variable "region" {
  type        = string
  default     = "us-central1"
  description = "GCP region"
}

resource "google_storage_bucket" "b" {
  name     = "my-bucket"
  location = var.region
}
```

Set values via:
- `default` in declaration
- `terraform.tfvars` file
- `-var` CLI flag
- Environment variables (`TF_VAR_region`)

---

## 12. What are Terraform outputs?

**Outputs** expose values from your config after `apply` — useful for displaying info or passing data between modules.

```hcl
output "bucket_url" {
  value = google_storage_bucket.logs.url
}
```

View them: `terraform output` or `terraform output bucket_url`.

---

## 13. What is a Terraform module?

A **module** is a reusable, packaged collection of `.tf` files — like a function for infrastructure.

```hcl
module "vpc" {
  source  = "terraform-google-modules/network/google"
  version = "9.0.0"

  project_id   = "my-project"
  network_name = "prod-vpc"
}
```

Every Terraform config is technically a module (the **root module**). Modules can call other modules.

---

## 14. What is the difference between `count` and `for_each`?

Both create multiple instances of a resource.

**`count`** — uses an integer; resources indexed by number:
```hcl
resource "google_storage_bucket" "b" {
  count = 3
  name  = "bucket-${count.index}"
}
```

**`for_each`** — uses a map or set; resources indexed by key:
```hcl
resource "google_storage_bucket" "b" {
  for_each = toset(["logs", "backup", "archive"])
  name     = "${each.key}-bucket"
}
```

`for_each` is preferred — adding/removing items doesn't reindex other resources.

---

## 15. What are data sources in Terraform?

A **data source** lets you **read** existing infrastructure (not managed by your config) and use its attributes.

```hcl
data "google_compute_network" "default" {
  name = "default"
}

resource "google_compute_subnetwork" "sub" {
  network = data.google_compute_network.default.id
  # ...
}
```

Use case: reference resources created elsewhere (manually, by another team, or in a different state).

---

## 16. What is the `depends_on` argument?

Terraform usually infers dependencies automatically (from attribute references). `depends_on` is used when there's an **implicit dependency** Terraform can't detect.

```hcl
resource "google_storage_bucket" "b" {
  name     = "data"
  location = "US"
}

resource "google_storage_bucket_object" "obj" {
  bucket  = google_storage_bucket.b.name
  name    = "file.txt"
  content = "hi"

  depends_on = [google_project_iam_member.writer]
}
```

Use sparingly — explicit references are cleaner.

---

## 17. What is `terraform destroy`?

`terraform destroy` removes **all resources** managed by the current configuration.

```bash
terraform destroy            # destroys everything
terraform destroy -target=google_storage_bucket.b   # destroys one resource
```

Used to tear down dev/test environments. **Be very careful in prod** — it deletes infrastructure.

---

## 18. What is a Terraform workspace?

A **workspace** is a way to manage multiple state files within one configuration — typically used for environments (dev, staging, prod).

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
terraform workspace list
```

Each workspace has its own state. **However**, for serious multi-env setups, separate directories or repos are usually cleaner than workspaces.

---

## 19. How do you handle secrets in Terraform?

**Don't hardcode them.** Options:

- **Environment variables** with `TF_VAR_` prefix:
  ```bash
  export TF_VAR_db_password="s3cret"
  ```
- **Secret managers** — read at runtime via data sources (Google Secret Manager, AWS Secrets Manager, Vault).
- **`sensitive = true`** on variables/outputs — hides values from CLI output:
  ```hcl
  variable "db_password" {
    type      = string
    sensitive = true
  }
  ```
- **Remote state encryption** (GCS/S3 with KMS).

⚠️ Even with `sensitive`, secrets land in the state file — protect the state file like a secret.

---

## 20. What's the typical Terraform workflow?

1. **Write** — author `.tf` files describing desired infra.
2. **`terraform init`** — download providers/modules, set up backend.
3. **`terraform fmt` / `validate`** — format and syntax-check.
4. **`terraform plan`** — preview changes.
5. **Review** — confirm plan looks right (especially destructive changes).
6. **`terraform apply`** — provision the infra.
7. **Commit** — push `.tf` files to Git (never commit `terraform.tfstate` or `.tfvars` with secrets).
8. **Iterate** — repeat 1–7 for updates.
9. **`terraform destroy`** — tear down when no longer needed.

In CI/CD: `init → fmt -check → validate → plan → apply` on merge to main.
