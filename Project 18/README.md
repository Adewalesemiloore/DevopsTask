<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM PART 3 — REFACTORING</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>November 26, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white" />
  <img src="https://img.shields.io/badge/Amazon_S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white" />
  <img src="https://img.shields.io/badge/DynamoDB-4053D6?style=for-the-badge&logo=amazondynamodb&logoColor=white" />
  <img src="https://img.shields.io/badge/VS_Code-007ACC?style=for-the-badge&logo=visualstudiocode&logoColor=white" />
  <img src="https://img.shields.io/badge/Infrastructure_as_Code-informational?style=for-the-badge" />
</p>

---

## Project Overview

In the earlier Terraform projects I wrote all my infrastructure in one long list of `.tf` files and let Terraform keep its state in a local file. That is fine for learning, but it breaks down as soon as the work becomes real. The state file is stored on one laptop, so no teammate can see it, and it contains secrets (passwords and keys processed by resources end up in it in plain JSON), so it should not be lying around unprotected.

This project is about fixing those problems and cleaning up the codebase. I did three things:

- **Moved the state to a remote backend.** I stored `terraform.tfstate` in an **S3 bucket** (versioned and encrypted) and used a **DynamoDB table** for **state locking**, so two people cannot run Terraform against the same infrastructure at the same time and corrupt the state.
- **Made the code more dynamic.** I looked at `dynamic` blocks for repetitive security group rules, `map` + `lookup` for picking region-specific AMI IDs, and conditional expressions for creating resources only when a flag says so.
- **Refactored into modules.** I broke the single big configuration from Project 17 into logical child modules (VPC, ALB, Auto Scaling, EFS, RDS, Security) that the root module calls, which makes the code easier to read, reuse and share between team members.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part 1 — Introduce an S3 Remote Backend with DynamoDB Locking](#part-1--introduce-an-s3-remote-backend-with-dynamodb-locking)
- [Part 2 — Make the Code Dynamic](#part-2--make-the-code-dynamic)
- [Part 3 — Refactor into Modules](#part-3--refactor-into-modules)
- [Result](#result)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

The plan I followed to move from the local backend to S3 was:

1. Add the S3 and DynamoDB resource blocks **before** deleting the local state file.
2. Update the `terraform` block to introduce the backend and locking.
3. Re-initialize Terraform.
4. Check that the state now lives in the S3 bucket.
5. Add `output.tf`.
6. Run `terraform apply`.

After that, the root configuration is split like this:

| Component | Purpose |
|---|---|
| S3 bucket (versioned, AES256 encrypted) | Remote storage for `terraform.tfstate` |
| DynamoDB table `terraform-locks` | State locking and consistency checking (`LockID` key) |
| `modules/VPC` | VPC, subnets, route tables, gateways and roles |
| `modules/ALB` | Application load balancer and certificate |
| `modules/AutoScaling` | Launch templates, ASGs and their user-data scripts |
| `modules/EFS` | Elastic File System |
| `modules/RDS` | Database resources |
| `modules/Security` | Security groups |
| Root (`main.tf`, `provider.tf`, `backend.tf`, `variables.tf`, `terraform.tfvars`) | Calls the modules and holds provider/backend settings |

---

## Part 1 — Introduce an S3 Remote Backend with DynamoDB Locking

### 1.0 — Creating the Bucket and Lock Table

I created a file called `backend.tf`. It defines the S3 bucket with **versioning** turned on and **server-side encryption** (AES256), because Terraform state can contain secrets. The bucket name is the one from my earlier project.

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "<your-tfstate-bucket-name>"

  tags = {
    Name = "terraform-state"
  }
}

resource "aws_s3_bucket_versioning" "versioning" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "encryption" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

Next I added the DynamoDB table. Previously, locking was handled by a local file (`terraform.tfstate.lock.info`). Now that the state is shared, the lock has to be shared too, so anyone running Terraform against the same infrastructure checks one central place.

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

> Terraform expects the bucket and the table to **already exist** before the backend is configured, so I ran `terraform plan` and `terraform apply` first to provision them.

![Applying the S3 bucket and DynamoDB resources](./Images/Task%2018.1.png)

---

### 2.0 — Configuring the S3 Backend

With the resources in place, I added the `backend "s3"` block to the `terraform` block:

```hcl
terraform {
  backend "s3" {
    bucket         = "<your-tfstate-bucket-name>"
    key            = "project18/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

Then I re-initialized so Terraform would switch backends:

```bash
terraform init
```

Terraform detected the existing local state and asked whether to copy it to the new `s3` backend. I answered `yes`.

![terraform init migrating state to the S3 backend](./Images/Task%2018.2.png)

---

### 3.0 — Verifying in the AWS Console

I opened the AWS console to confirm the migration worked. The `terraform.tfstate` file is now inside the bucket under the `project18/` prefix.

![State file inside the project18 prefix of the bucket](./Images/Task%2018.3.png)

Opening the object shows its properties (key, ARN, S3 URI) and an **Open** button to view the file.

![Properties of the terraform.tfstate object](./Images/Task%2018.4.png)

The state is just JSON — every resource Terraform manages is recorded here along with its outputs, which is exactly why it must be encrypted and access-controlled.

![Contents of the state file](./Images/Task%2018.5.png)

The DynamoDB `terraform-locks` table is also active, using `LockID` as its partition key.

![DynamoDB terraform-locks table](./Images/Task%2018.6.png)

---

### 4.0 — Adding Outputs and Applying

Before running `apply`, I created `output.tf` so the S3 bucket ARN and DynamoDB table name are displayed after each run:

```hcl
output "s3_bucket_arn" {
  value       = aws_s3_bucket.terraform_state.arn
  description = "The ARN of the S3 bucket"
}

output "dynamodb_table_name" {
  value       = aws_dynamodb_table.terraform_locks.name
  description = "The name of the DynamoDB table"
}
```

```bash
terraform apply
```

Terraform now reads the latest state from S3 on every run, so even if another engineer has applied changes, my view of the infrastructure is always current. The outputs printed at the end of the run:

![terraform apply outputs showing the bucket ARN and table name](./Images/Task%2018.7.png)

> With S3 versioning enabled, every state change creates a new version of `terraform.tfstate` in the bucket, which gives a free history and a way to roll back.

---

## Part 2 — Make the Code Dynamic

A remote backend solves collaboration, but there is still the problem of **isolating environments** (dev, sit, uat, preprod, prod). There are two ways to do it: **Terraform workspaces**, or **directory-based separation** using `terraform.tfvars`. As a rule of thumb, use directories when environments differ significantly and workspaces when they are nearly identical, to avoid duplicating configuration.

### 5.0 — Security Groups with `dynamic` Blocks

Security groups contain many near-identical `ingress` blocks. Terraform's `dynamic` blocks let me generate those repeated blocks from a collection instead of copy-pasting them, which keeps the security group code short and easy to change.

---

### 6.0 — AMIs with `map` and `lookup`

An AMI is a **regional** resource, so a hard-coded AMI ID breaks the moment the region changes. To pick the AMI dynamically, I defined a `map(string)` variable keyed by region:

```hcl
variable "images" {
  type = map(string)

  default = {
    us-east-1 = "ami-0c02fb55956c7d316"
    us-west-2 = "ami-xxxxxxxx"
  }
}
```

![The images map variable in variables.tf](./Images/Task%2018.8.png)

Then I used `lookup(map, key, [default])` in the launch templates. The key is `var.region`, so the correct AMI is chosen automatically, and the third argument is a default so the lookup never fails if the region is missing from the map:

```hcl
resource "aws_launch_template" "wordpress_launch_template" {
  image_id      = lookup(var.images, var.region, "ami-0c02fb55956c7d316")
  instance_type = "t2.micro"
}
```

![lookup in the WordPress launch template](./Images/Task%2018.9.png)

I did the same for the bastion launch template:

![lookup in the bastion launch template](./Images/Task%2018.10.png)

---

### 7.0 — Conditional Expressions

When I want to create a resource only if a condition holds, I use a conditional expression, `condition ? true_val : false_val`, together with `count`. Here a read replica is created only when `create_read_replica` is `true`:

```hcl
resource "aws_db_instance" "read_replica" {
  count               = var.create_read_replica == true ? 1 : 0
  replicate_source_db = aws_db_instance.this.id
}
```

---

## Part 3 — Refactor into Modules

The Project 17 repository was one long list of files creating every resource. That is hard to navigate and makes future changes stressful. **Modules** solve this: a module is just a directory of `.tf` files that groups related resources, and a root module calls child modules like this:

```hcl
module "network" {
  source = "./modules/network"
}
```

Module outputs can then be consumed by other resources, for example `module.servers.instance_ids`, as long as the module exposes them in an `outputs.tf`.

### 8.0 — Creating the Module Directories

I created a `modules` directory and one folder per resource group:

```bash
mkdir modules
cd modules/
mkdir VPC ALB AutoScaling EFS RDS Security
cd ..
```

![Creating the modules directories](./Images/Task%2018.11.png)

---

### 9.0 — Moving the Files into Their Modules

Then I moved each resource file into the module it belongs to. Networking files went into `VPC`:

```bash
mv main.tf internet_gateway.tf natgateway.tf route_tables.tf roles.tf subnets.tf modules/VPC/
```

![Moving the networking files into modules/VPC](./Images/Task%2018.12.png)

The load balancer, auto scaling and storage files went into their own modules:

```bash
mv alb.tf cert.tf modules/ALB/ && \
mv asg-bastion-nginx.tf asg-wordpress-tooling.tf nginx.sh wordpress.sh tooling.sh bastion.sh modules/AutoScaling/ && \
mv efs.tf modules/EFS/ && \
mv rds.tf modules/RDS/
```

![Moving the ALB, AutoScaling, EFS and RDS files](./Images/Task%2018.13.png)

Every module needs its own `variables.tf`, so I copied the root `variables.tf` into each one (I then trimmed each copy to only the variables that module actually uses):

```bash
cp variables.tf modules/VPC/variables.tf
cp variables.tf modules/ALB/variables.tf
cp variables.tf modules/AutoScaling/variables.tf
cp variables.tf modules/EFS/variables.tf
cp variables.tf modules/RDS/variables.tf
cp variables.tf modules/Security/variables.tf
```

![Copying variables.tf into each module](./Images/Task%2018.14.png)

The resulting module layout:

![Resulting module structure in VS Code](./Images/Task%2018.15.png)

The target structure for the working directory is:

```
└── PBL
    ├── modules
    |   ├── ALB
    |   ├── AutoScaling
    |   ├── EFS
    |   ├── RDS
    |   ├── Security
    |   └── VPC          (each with main/resource files + variables.tf, outputs.tf where needed)
    ├── main.tf
    ├── backend.tf
    ├── provider.tf
    ├── outputs.tf
    ├── terraform.tfvars
    └── variables.tf
```

---

### 10.0 — Root `main.tf` and `provider.tf`

I created new `main.tf` and `provider.tf` files in the root directory.

![New main.tf and provider.tf in the root](./Images/Task%2018.16.png)

The `provider` block that used to live in `modules/VPC/main.tf` was cut out and pasted into the root. A provider should be configured once at the root, not inside a child module.

```hcl
provider "aws" {
  region = var.region
}
```

![The provider block in the root file](./Images/Task%2018.17.png)

Finally I pinned the Terraform and provider versions in the `terraform` block so everyone on the team uses the same versions:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 5.0.0"
    }

    random = {
      source  = "hashicorp/random"
      version = "= 3.8.1"
    }
  }
}
```

![Pinned required_providers in provider.tf](./Images/Task%2018.18.png)

> After moving code into modules, each module needs its variables declared in its own `variables.tf`, and values needed by other modules must be exposed through `outputs.tf` and passed in from the root. The code can be split even further into smaller files for clarity.

---

## Result

Terraform state now lives in a versioned, encrypted S3 bucket with DynamoDB locking, so my whole team can work against the same state safely. The code uses `map`/`lookup` and conditionals to stay region-agnostic and flexible, and the infrastructure is organised into reusable modules called from a clean root configuration with pinned provider versions.

---

## Errors & Fixes

The source material for this project did not show a failed-then-fixed step, so instead of inventing incidents, here are the pitfalls that follow directly from the steps above.

---

**Backend configured before the bucket and table exist**

**Symptom:** `terraform init` fails when it tries to use the S3 backend.
**Root cause:** Terraform expects the S3 bucket and DynamoDB table to already exist before the backend block is used.
**Fix:** Add the bucket and table resources first, run `terraform plan` and `terraform apply`, and only then add the `backend "s3"` block and re-run `terraform init`.

---

**Deleting the local state too early**

**Symptom:** Terraform thinks no infrastructure exists and wants to create everything again.
**Root cause:** The local `terraform.tfstate` was removed before the state was copied to the remote backend.
**Fix:** Re-initialize with `terraform init` and answer `yes` when asked to copy the existing state to the new backend; only delete the local file after confirming the object is in the bucket.

---

**Hard-coded AMI breaks when the region changes**

**Symptom:** Launch templates fail because the AMI ID does not exist in the selected region.
**Root cause:** AMIs are regional.
**Fix:** Keep AMI IDs in a `map(string)` keyed by region and use `lookup(var.images, var.region, "<default-ami>")`, with a default so a missing key does not fail the run.

---

**Provider and variables missing after splitting into modules**

**Symptom:** Modules complain about undeclared variables or a missing provider configuration.
**Root cause:** Each module only sees its own `variables.tf`, and the `provider` block was left inside `modules/VPC/main.tf`.
**Fix:** Declare the required variables in every module's `variables.tf`, and move the `provider "aws"` block and the `required_providers` block to the root `provider.tf`.

---

*Project completed by **Adewale Oluwasemiloore***
