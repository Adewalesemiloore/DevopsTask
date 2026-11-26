<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM PART 4 — TERRAFORM CLOUD</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>November 26, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white" />
  <img src="https://img.shields.io/badge/Terraform_Cloud-844FBA?style=for-the-badge&logo=terraform&logoColor=white" />
  <img src="https://img.shields.io/badge/Packer-02A8EF?style=for-the-badge&logo=packer&logoColor=white" />
  <img src="https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white" />
  <img src="https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
</p>

---

## Project Overview

Up to now I have installed and run Terraform myself, on a machine I had to set up, maintain and keep up to date. That works for one person, but a team needs a consistent remote environment with a shared state and a repeatable workflow. In the cloud world, the usual answer is a **managed** version of the tool: I do not install or maintain anything, I just sign up and use it as a service.

**Terraform Cloud** is exactly that. It runs Terraform commands on disposable virtual machines (called *remote operations*), keeps the state for me, stores sensitive variables safely, and can be wired to GitHub so that a `git push` triggers a plan automatically.

In this project I migrated my existing Terraform code to Terraform Cloud and used it to manage my AWS infrastructure:

- created an account and an organization,
- created a workspace using the **version control workflow**, connected to a new `terraform-cloud` GitHub repository,
- configured AWS credentials and the Terraform variables as workspace variables,
- ran a plan and apply from the web console (and fixed a failing plan),
- destroyed the resources, and then proved that a push to GitHub triggers a plan automatically.

I also prepared my local machine with **Packer** and **Ansible**, because the repository now carries an `AMI` folder (for building images with Packer) and an `Ansible` folder (for configuring the infrastructure) alongside the Terraform code from the previous project.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part 1 — Create the Terraform Cloud Account and Organization](#part-1--create-the-terraform-cloud-account-and-organization)
- [Part 2 — Create a Workspace and Connect GitHub](#part-2--create-a-workspace-and-connect-github)
- [Part 3 — Configure Workspace Variables](#part-3--configure-workspace-variables)
- [Part 4 — Prepare the Repository with Packer and Ansible](#part-4--prepare-the-repository-with-packer-and-ansible)
- [Part 5 — Run Plan and Apply from the Web Console](#part-5--run-plan-and-apply-from-the-web-console)
- [Part 6 — Test Automated Plans from GitHub](#part-6--test-automated-plans-from-github)
- [Result](#result)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

| Component | Role |
|---|---|
| GitHub repo `terraform-cloud` | Source of truth for all Terraform, Packer (`AMI`) and Ansible code |
| Terraform Cloud organization + workspace | Runs `plan`/`apply` remotely, stores state and variables |
| Workspace variables | AWS credentials (environment variables) and Terraform input variables |
| AWS account | Where the resources are actually created |
| Local machine | Has Packer and Ansible installed for building images and configuring servers |

The flow: **git push → GitHub → Terraform Cloud (VCS-triggered run) → automatic plan → manual confirm → apply to AWS.**

---

## Part 1 — Create the Terraform Cloud Account and Organization

### 1.0 — Creating the Account

I signed up at `app.terraform.io`, verified my email, and landed on the Organizations page. Most Terraform Cloud features are free, so the free tier was enough for this project.

![Terraform Cloud Organizations page](./Images/Task%2019.1.png)

---

### 2.0 — Creating the Organization

I clicked **Create organization**, chose the **Personal** type, and gave the organization a unique name.

![Create a new organization](./Images/Task%2019.2.png)

I confirmed the name and notification email and created it.

![Confirming the organization name and email](./Images/Task%2019.3.png)

---

## Part 2 — Create a Workspace and Connect GitHub

### 3.0 — Choosing the Workflow

Terraform Cloud offers three workflows: **version control**, **CLI-driven** and **API-driven**. I picked the **Version Control Workflow**, the most common and recommended one, because runs are triggered by changes in my Git repository.

Before this step I created a new GitHub repository named **terraform-cloud** and pushed the Terraform code from the previous projects into it.

![Choosing the Version Control Workflow](./Images/Task%2019.4.png)

---

### 4.0 — Connecting GitHub

On the "Connect to VCS" step I selected **GitHub.com** and authorized Terraform Cloud to access my GitHub account.

![Choosing GitHub.com as the VCS provider](./Images/Task%2019.5.png)

![Authorizing Terraform Cloud on GitHub](./Images/Task%2019.6.png)

I installed the app for **Only select repositories** and picked just the `terraform-cloud` repository, which keeps the access as narrow as possible.

![Selecting only the terraform-cloud repository](./Images/Task%2019.7.png)

In "Configure settings" I added a description, left every other setting on its default, and clicked **Create workspace**. The workspace overview confirmed the configuration was uploaded and offered **Configure variables** as the next step.

![Workspace created and configuration uploaded](./Images/Task%2019.7.1.png)

---

## Part 3 — Configure Workspace Variables

Terraform Cloud supports two kinds of variables: **environment variables** and **Terraform variables**. Either can be marked **Sensitive**, which makes it write-only so it is never displayed in the UI again.

### 5.0 — AWS Credentials as Environment Variables

I created the credentials Terraform Cloud needs to provision AWS, as **environment variables** marked Sensitive:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION`

```
AWS_ACCESS_KEY_ID      = <your-access-key-id>
AWS_SECRET_ACCESS_KEY  = <your-secret-access-key>
AWS_REGION             = <your-region>
```

![Adding the AWS_REGION environment variable as Sensitive](./Images/Task%2019.8.png)

> Never commit real credentials to the repository. Keeping them as sensitive workspace variables means they live only inside Terraform Cloud.

Once these are set, Terraform Cloud is ready to apply the code from GitHub and create the AWS resources.

---

## Part 4 — Prepare the Repository with Packer and Ansible

In Project 18 the idea was to use **Packer** to build images and **Ansible** to configure the infrastructure, so I added two new folders to the repository:

- **`AMI`** — Packer templates and scripts for building images,
- **`Ansible`** — playbooks, inventory and roles to configure the infrastructure.

Both tools must be installed locally first.

### 6.0 — Installing Packer and Ansible

On Windows I installed Packer with `winget`, then confirmed the version:

```powershell
winget install -e --id Hashicorp.Packer
packer --version
packer version
```

I then opened WSL and started installing Ansible. (The first `sudo` attempt was typed with the wrong password, so it asked again.)

```bash
wsl
sudo apt update
sudo apt install -y ansible
```

![Installing Packer on Windows and starting the Ansible install in WSL](./Images/Task%2019.9.png)

After entering the correct password, the package lists were refreshed.

![apt update completing in WSL](./Images/Task%2019.10.png)

I installed `ansible-core` and confirmed it with `ansible --version`:

```bash
sudo apt install ansible-core
ansible --version
```

![Installing ansible-core](./Images/Task%2019.11.png)

![ansible --version output](./Images/Task%2019.12.png)

---

### 7.0 — Updated Repository Layout

With the tools ready, I reorganised the repository to include the new folders. The `Ansible/playbooks/site.yml` file imports the individual playbooks (nginx, tooling, wordpress, jenkins, artifactory and sonarqube), with only the sonarqube play enabled for now, and there is a dynamic inventory file `aws_ec2.yml` under `Ansible/inventory`.

![Repository layout with AMI and Ansible folders and site.yml](./Images/Task%2019.13.png)

---

## Part 5 — Run Plan and Apply from the Web Console

### 8.0 — Starting a Manual Run

In the workspace I opened the **Runs** tab.

![The Runs tab in the workspace](./Images/Task%2019.14.png)

My first run failed during the plan, which is covered in [Errors & Fixes](#errors--fixes). Here is the failing plan:

![Plan errored with missing required variables](./Images/Task%2019.15.png)

---

### 9.0 — Fixing the Missing Variables

The plan complained that the required variables `keypair` and `account_no` had no value. I went to **Variables** and clicked **Add variable**.

![Variables page with the Add variable button](./Images/Task%2019.16.png)

I added `keypair` as a **Terraform variable** (not an environment variable).

![Adding the keypair Terraform variable](./Images/Task%2019.17.png)

Then I added `account_no` the same way, this time marked **Sensitive**.

![Adding the account_no Terraform variable as Sensitive](./Images/Task%2019.18.png)

---

### 10.0 — Re-running the Plan and Apply

With the variables in place, I went back to **Runs**, clicked **New run** and left the run type on **Plan and apply (standard)**.

![Starting a new run with the standard plan and apply type](./Images/Task%2019.19.png)

This time the plan finished successfully: 4 resources to create (the DynamoDB lock table, and the S3 bucket with its versioning and encryption configuration).

![Plan finished with 4 resources to create](./Images/Task%2019.20.png)

I confirmed and applied. The apply finished and all 4 resources were created. Note that the bucket and table names ended in `-002` in this run.

![Apply finished with 4 resources created](./Images/Task%2019.21.png)

---

### 11.0 — Confirming in the AWS Console

I confirmed the resources in AWS. The new S3 bucket appears next to the one from the previous project:

![New S3 bucket in the AWS console](./Images/Task%2019.22.png)

and so does the new DynamoDB lock table, active with `LockID` as the key:

![New DynamoDB lock table in the AWS console](./Images/Task%2019.23.png)

---

### 12.0 — Destroying the Resources

Before moving on to the next test, I queued a **destroy** run so I would not leave resources running. The plan showed 4 to destroy, and the apply finished with all 4 resources deleted.

![Destroy run finished with all 4 resources deleted](./Images/Task%2019.24.1.png)

---

## Part 6 — Test Automated Plans from GitHub

Because the workspace is connected to GitHub, I did not need to start runs by hand anymore. Any push to the repository should trigger a **plan** automatically, while the **apply** still waits for my manual approval. I deliberately leave "Auto apply" off: provisioning cloud resources costs money, so reviewing the plan first avoids "bill shock" from a misconfiguration.

### 13.0 — Making a Small Change

I made a tiny change to `backend.tf` by adding an `Env = "test"` tag to the S3 bucket.

![A small tag change in backend.tf](./Images/Task%2019.24.png)

### 14.0 — Pushing to GitHub

```bash
git add .
git commit -m "test auto plan trigger"
git push
```

![Pushing the change to GitHub](./Images/Task%2019.24.2.png)

### 15.0 — Plan Triggered Automatically

In Terraform Cloud, a new run named after my commit message appeared immediately, labelled as triggered via GitHub on the `main` branch. It reached the **Planned** state without me touching anything.

![New run triggered automatically from GitHub](./Images/Task%2019.25.png)

Opening the run showed the finished plan with 4 resources to create, and Terraform Cloud waiting for me to manually **Confirm & apply**.

![Plan finished and waiting for manual confirm and apply](./Images/Task%2019.26.png)

---

## Result

My Terraform code now runs from Terraform Cloud instead of my own machine. The state is managed remotely, AWS credentials and inputs are stored as sensitive workspace variables, and the workspace is wired to my `terraform-cloud` GitHub repository so that every push triggers a plan automatically, with a manual approval gate before anything is applied.

---

## Errors & Fixes

---

**Plan errored: `No value for required variable`**

**Symptom:** The first plan from the web console failed with two errors: `No value for required variable` for `keypair` (line 52 of `variables.tf`) and `account_no` (line 58). The apply step showed "Apply will not run".

**Root cause:** The root module declared input variables `keypair` and `account_no` with no default, and I had only set the AWS credential environment variables in the workspace. Terraform Cloud had no value to give them.

**Fix:** In the workspace go to **Variables -> Add variable**, choose the **Terraform variable** category (not environment), and add `keypair` and `account_no`, marking `account_no` as Sensitive. Then queue a new run; the plan succeeded.

---

*Project completed by **Adewale Oluwasemiloore***
