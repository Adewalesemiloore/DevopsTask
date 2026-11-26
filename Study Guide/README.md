# DevOps Projects 1–20 — Study Guide

A learning companion to the 20 project READMEs. Read the READMEs for the full steps and screenshots; use this guide for the big picture, the order to read in, and what to remember from each.

---

## How the projects fit together

The 20 projects tell one story:

| Phase | Projects | Theme |
|---|---|---|
| 1. Web stacks | 1–4 | Install and wire up the pieces of a web application on a single server |
| 2. Servers talking to servers | 5–6 | Client/server databases, storage (LVM) and three-tier architecture |
| 3. A real multi-server website | 7–8, 10 | Shared storage (NFS), database server, web servers, load balancing, HTTPS |
| 4. Automation | 9, 11–14 | Jenkins CI, then Ansible, roles, and full CI/CD pipelines |
| 5. Cloud and infrastructure as code | 15–19 | Build the whole AWS setup by hand, then with Terraform |
| 6. Containers | 20 | Move the application into Docker |

The same idea repeats at every level: **do it by hand first, understand it, then automate it.** Projects 7–10 are the manual version of what Projects 11–14 automate with Ansible and Jenkins. Project 15 is the manual cloud build that Projects 16–19 redo in Terraform.

---

## Suggested reading order

**Read in numerical order the first time.** Each project assumes the previous ones. Then read these clusters again together, because they only make sense side by side:

1. **7 → 8 → 10 → 11** (the manual website, then Ansible automating it)
2. **11 → 12 → 13 → 14** (Ansible and Jenkins growing step by step)
3. **15 → 16 → 17 → 18 → 19** (cloud by hand, then the same cloud in Terraform, then better organised)

Suggested pace: one phase per sitting. For each project, read the Project Overview and Architecture sections first, then the Errors & Fixes, then the steps. Knowing what went wrong makes the steps easier to remember.

---

## Project-by-project notes

### Phase 1 — Web stacks

**Project 1 — LAMP stack (Linux, Apache, MySQL, PHP)**
- *Learn:* launching an EC2 instance, installing Apache/MySQL/PHP, security groups, Apache virtual hosts.
- *Remember:* a stack is just layers, with each one doing a job: OS, web server, database, language. The security group (AWS firewall) must allow port 80 or the site won't load even when Apache is running.

**Project 2 — LEMP stack (Nginx instead of Apache)**
- *Learn:* the same stack with Nginx, and how Nginx hands PHP requests to PHP-FPM.
- *Remember:* Nginx can't run PHP itself, so it needs a PHP processor and a config block that passes requests to it. Compare this directly with Project 1.

**Project 3 — MERN to-do app** and **Project 4 — MEAN app**
- *Learn:* Node.js, Express routes, MongoDB models, and a frontend (React in Project 3, AngularJS in Project 4).
- *Remember:* backend (routes + database model) first, frontend second. The two projects differ in the frontend framework.

### Phase 2 — Servers talking to servers

**Project 5 — Client-server with MySQL**
- *Learn:* two EC2 instances, one running the MySQL server and one the client; creating a database and user; allowing remote connections.
- *Remember:* three things must all be right for a remote connection: MySQL listening on the right address, a user allowed from the client's address, and port 3306 open in the security group.

**Project 6 — WordPress on a three-tier architecture**
- *Learn:* Linux storage (disks, partitions, LVM, mounts), then WordPress on a web server with MySQL on a separate DB server.
- *Remember:* LVM order is **physical volume → volume group → logical volume → format → mount → add to `/etc/fstab`**. The three tiers are client, web server, database server. Read its Errors & Fixes section.

### Phase 3 — The multi-server website

**Project 7 — DevOps tooling website with NFS and a database**
- *Learn:* an NFS server sharing storage to several web servers, a database server, and the Tooling app connecting to all of it.
- *Remember:* NFS gives every web server the same files and logs, so adding a server doesn't mean copying data around. SELinux and firewall rules are the usual blockers.

**Project 8 — Load balancer with Apache**
- *Learn:* an Apache `mod_proxy_balancer` setup that spreads traffic across the web servers; watching traffic distribute live.
- *Remember:* the load balancer is one more server in front of the web servers. Unmount the shared logs so each web server keeps its own and you can see the traffic split.

**Project 10 — Load balancer with Nginx and HTTPS**
- *Learn:* the same job with Nginx, plus a free domain, an Elastic IP, and a Let's Encrypt certificate via Certbot with auto-renewal.
- *Remember:* HTTPS needs a real domain name. An Elastic IP keeps the address stable across restarts.

### Phase 4 — Automation

**Project 9 — Jenkins CI**
- *Learn:* installing Jenkins, a GitHub webhook that triggers builds, archiving artifacts, and copying files to the NFS server over SSH.
- *Remember:* flow is push → webhook → build → artifacts → deploy. Jenkins listens on port 8080, so it must be open.

**Project 11 — Ansible configuration management**
- *Learn:* Ansible on a Jenkins-Ansible "jump server", an inventory file, a playbook, SSH agent forwarding.
- *Remember:* Ansible needs no agent on the targets, just SSH. Different OS families use different users (`ec2-user` for Red Hat, `ubuntu` for Ubuntu) and package managers (`dnf` vs `apt`). Seven of its Errors & Fixes are worth reading closely.

**Project 12 — Refactoring and static assignments (imports and roles)**
- *Learn:* a `save_artifacts` Jenkins job, `site.yml` that imports other playbooks, and a role for the UAT web servers.
- *Remember:* a **role** is a standard folder layout (tasks, handlers, templates and so on) that makes automation reusable. `import_playbook` pulls other playbooks into `site.yml`.

**Project 13 — Dynamic assignments and community roles**
- *Learn:* `include` for dynamic loading, environment variable files, a community MySQL role, and load-balancer roles chosen per environment.
- *Remember:* **import is static** (decided when the playbook loads) and **include is dynamic** (decided while it runs). Use dynamic when a variable decides what happens.

**Project 14 — CI/CD**
- *Learn:* Jenkins pipelines with a `Jenkinsfile`, Blue Ocean, running Ansible from Jenkins, parameterised deployments, Artifactory, and SonarQube quality gates.
- *Remember:* CI builds and tests on every change, CD deploys it. A quality gate blocks a bad build from moving on. This is the longest README, so go through it one part at a time.

### Phase 5 — Cloud and infrastructure as code

**Project 15 — AWS cloud solution (manual)**
- *Learn:* VPC, subnets, route tables, internet and NAT gateways, security groups, ACM certificates, EFS, RDS, load balancers, launch templates and Auto Scaling Groups.
- *Remember:* public subnets face the internet, private subnets don't, and the NAT gateway lets private servers reach out. Build in this order: network → security → storage/database → load balancers → servers. This is the blueprint Terraform recreates next.

**Project 16 — Terraform Part 1**
- *Learn:* an IAM user, an S3 bucket for state, a VPC and subnets in Terraform, then refactoring to variables, data sources, `count`, `cidrsubnet()` and `length()`.
- *Remember:* the cycle is `terraform init → plan → apply`. Hard-coded first, then refactor; see how the code gets shorter and reusable.

**Project 17 — Terraform Part 2**
- *Learn:* tags, gateways, routes, IAM roles, security groups, ACM, load balancers, launch templates, ASGs, EFS, KMS and RDS, all as code.
- *Remember:* `terraform validate` catches references to things you haven't declared. `terraform destroy` at the end avoids ongoing AWS bills.

**Project 18 — Terraform Part 3 (refactoring)**
- *Learn:* an S3 remote backend with DynamoDB locking, `dynamic` blocks, `map` and `lookup`, conditional expressions, and splitting code into modules.
- *Remember:* remote state lets a team share state safely and the lock stops two people applying at once. Modules are Terraform's version of Ansible roles: reusable building blocks.

**Project 19 — Terraform Part 4 (Terraform Cloud)**
- *Learn:* a Terraform Cloud workspace connected to GitHub, variables stored in the workspace, manual and automatic runs.
- *Remember:* once connected, a push to GitHub triggers a plan automatically. Credentials go in workspace variables, never in code.

### Phase 6 — Containers

**Project 20 — Docker and Docker Compose**
- *Learn:* running MySQL in a container, a custom Docker network, a second container as a client, and building and running the Tooling app as an image.
- *Remember:* containers on the same custom network reach each other by name. Pass secrets as environment variables. Read its Errors & Fixes, since the database connection took several attempts.

---

## Key concepts cheat sheet

| Concept | One-line meaning |
|---|---|
| Security group | AWS firewall for a server; the first thing to check when something won't connect |
| Elastic IP | A fixed public IP that survives stop/start |
| LVM | Flexible disk management: physical volume → volume group → logical volume |
| NFS | Shared network storage mounted by several servers |
| Load balancer | Spreads incoming traffic across several servers |
| Webhook | GitHub calling Jenkins when code is pushed |
| CI/CD | Automatically build, test and deploy every change |
| Inventory | Ansible's list of servers and how to reach them |
| Playbook / role | Ansible tasks / a reusable packaged set of tasks |
| Import vs include | Ansible: loaded up front vs loaded at run time |
| Terraform state | Terraform's record of what it has built |
| Remote backend | State kept in S3 with DynamoDB locking |
| Module | A reusable group of Terraform files |
| Container | An application plus its dependencies packaged to run anywhere |

---

## Notes and cautions

These come from how Projects 12–20 were put together. Use them to know where to double-check.

1. **Inferred explanations.** For Projects 12–20 the screenshots came from another student's run. Some captions and some "root cause" lines in Errors & Fixes were inferred from the screenshots rather than stated in the source. Treat them as likely, not proven. This applies most to Project 15 captions, Project 17 images 17.21–17.28, and the `.env` failure in Project 20.
2. **Project 18's Errors & Fixes are pitfalls**, not real incidents, because the source showed no failures. Its `dynamic` blocks section is explanation only, with no code.
3. **Project 19:** the bucket and table names end in `-002` and the source doesn't say why. Packer and Ansible are only installed there, never run.
4. **Corrections made to the source:**
   - Project 12 uses `site.yml` (the source had a `site.yaml` typo).
   - Project 14's job parameter is named `inventory` and the credential ID is `ansible-ssh-key`.
   - Project 17's tfvars names match `variables.tf`.
   - Project 20's network is `tooling_network`.
5. **Five screenshots are not on GitHub yet** (Project 15: 15.171, 15.172; Project 17: 17.8, 17.11, 17.14) while they await blurring. Those images show as broken until they are added back.
6. **Placeholders.** Values such as `<db-password>`, `<rds-endpoint>` and `<your-domain>` stand in for your own. Never commit real keys, passwords or tokens.

## Self-test ideas

- Draw the Project 7 architecture from memory, then Project 15's, and compare.
- For each project, say aloud: what problem did this solve, and what did the previous project leave painful?
- Pick any Ansible step from Projects 11–13 and find its Terraform equivalent in 16–19.
- Take one Errors & Fixes entry per project and explain how you'd diagnose it without the answer.
