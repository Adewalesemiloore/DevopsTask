<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">ANSIBLE DYNAMIC ASSIGNMENTS (INCLUDE) AND COMMUNITY ROLES</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>November 26, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Red_Hat-EE0000?style=for-the-badge&logo=redhat&logoColor=white" />
  <img src="https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white" />
  <img src="https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white" />
  <img src="https://img.shields.io/badge/NGINX-009639?style=for-the-badge&logo=nginx&logoColor=white" />
  <img src="https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white" />
  <img src="https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white" />
  <img src="https://img.shields.io/badge/YAML-CB171E?style=for-the-badge&logo=yaml&logoColor=white" />
  <img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" />
</p>

---

## Project Overview

Projects 11 and 12 gave me the foundations of Ansible: playbooks, static imports and roles. In this project I keep configuring the UAT environment, but with two new ideas.

The first is the difference between **static** and **dynamic** assignments:

```
import  = Static
include = Dynamic
```

With `import`, Ansible pre-processes every referenced file when the playbook is **parsed**, so anything that changes afterwards is ignored. With `include`, the content is processed only when the playbook **executes**, so it can react to what is happening at run time. Static assignments are usually more reliable and easier to debug, so I keep them as the default and reserve dynamic assignments for what they do best: **environment-specific variables**.

The second idea is not reinventing the wheel. For the database I download a production-ready **community role** (`geerlingguy.mysql`) from Ansible Galaxy instead of writing my own, and I build `nginx` and `apache` roles for the load balancer, where a variable decides which one is active in each environment.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part 1 — Introduce Dynamic Assignments](#part-1--introduce-dynamic-assignments)
- [Part 2 — Update site.yml](#part-2--update-siteyml)
- [Part 3 — Community Role: MySQL](#part-3--community-role-mysql)
- [Part 4 — Load Balancer Roles](#part-4--load-balancer-roles)
- [Part 5 — Test Against UAT](#part-5--test-against-uat)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

The work happens in the same `ansible-config-mgt` repository, driven from the Jenkins-Ansible server, and targets the UAT web servers built in Project 12.

| Component | Purpose |
|---|---|
| `dynamic-assignments/env-vars.yml` | Includes variables from the file that matches the current environment |
| `env-vars/*.yml` | One variables file per environment (`dev`, `stage`, `uat`, `prod`) |
| `static-assignments/` | Child playbooks (`common.yml`, `webservers.yml`, `loadbalancers.yml`) |
| `roles/` | `webserver`, `mysql` (community), `nginx`, `apache` |

The final repository layout:

```
├── dynamic-assignments
│   └── env-vars.yml
├── env-vars
│   ├── dev.yml
│   ├── stage.yml
│   ├── uat.yml
│   └── prod.yml
├── inventory
│   ├── dev.yml
│   ├── stage.yml
│   ├── uat.yml
│   └── prod.yml
├── playbooks
│   └── site.yml
├── roles
│   ├── apache
│   ├── mysql
│   ├── nginx
│   └── webserver
└── static-assignments
    ├── common.yml
    ├── webservers.yml
    └── loadbalancers.yml
```

---

## Part 1 — Introduce Dynamic Assignments

### 1.0 — Create the Branch and Folders

In the `ansible-config-mgt` repository I started a new branch called `dynamic-assignments`. Inside it I created a `dynamic-assignments` folder with an empty `env-vars.yml` file. `site.yml` will include this file later.

![Dynamic assignments folder structure](./Images/Task%2013.1.png)

> Depending on how the roles were created in Project 12, the `roles` folder may only exist on the Jenkins-Ansible server (when `ansible-galaxy` was used). Since I want everything tracked in GitHub, I made sure the `roles` structure was committed to the repository too.

### 2.0 — Create Environment Variable Files

Each environment has its own unique values (server names, IP addresses, credentials), so I created an `env-vars` folder at the root of the repository with one YAML file per environment: `dev.yml`, `stage.yml`, `uat.yml` and `prod.yml`.

![env-vars folder](./Images/Task%2013.2.png)

![env-vars files](./Images/Task%2013.3.png)

### 3.0 — Write env-vars.yml

`dynamic-assignments/env-vars.yml` finds the first matching environment file and loads its variables:

```yaml
---
- name: collate variables from env specific file, if it exists
  hosts: all
  tasks:
    - name: looping through list of available files
      include_vars: "{{ item }}"
      with_first_found:
        - files:
            - dev.yml
            - stage.yml
            - prod.yml
            - uat.yml
          paths:
            - "{{ playbook_dir }}/../env-vars"
      tags:
        - always
```

![env-vars.yml](./Images/Task%2013.4.png)

Three things are worth noting here:

1. **`include_vars` instead of `include`.** From Ansible 2.8 the generic `include` is deprecated and split into `include_role`, `include_tasks` and `include_vars`. Imports were split the same way (`import_role`, `import_tasks`).
2. **Special variables.** `playbook_dir` lets Ansible work out where the running playbook lives and navigate from there to `env-vars`. `inventory_file` resolves to the name of the inventory in use.
3. **`with_first_found`.** Ansible loops through the list and uses the first file it finds, which makes it easy to fall back to default values when an environment file does not exist. The order of that list matters, as I found out later.

---

## Part 2 — Update site.yml

I updated `site.yml` to use the dynamic assignment. The first attempt is covered in [Errors & Fixes](#errors--fixes); the version that finally worked imports the environment variables first, then the static assignments:

```yaml
---
- import_playbook: ../dynamic-assignments/env-vars.yml

- import_playbook: ../static-assignments/common.yml

- import_playbook: ../static-assignments/webservers.yml

- import_playbook: ../static-assignments/loadbalancers.yml
```

![Working site.yml](./Images/Task%2013.17.png)

---

## Part 3 — Community Role: MySQL

Writing a MySQL role from scratch (install the package, create a database, configure users) would be re-inventing the wheel. Community roles on Ansible Galaxy are production-ready and handle many Linux flavours, so I used the MySQL role by **geerlingguy**.

### 4.0 — Prepare Git on Jenkins-Ansible

To keep GitHub in step with the server after installing a role, I made sure `git` was installed (`git --version`), then inside `ansible-config-mgt` ran:

```bash
git init
git pull https://github.com/<your-github-username>/ansible-config-mgt.git
git remote add origin https://github.com/<your-github-username>/ansible-config-mgt.git
git branch roles-feature
git switch roles-feature
```

### 5.0 — Install and Rename the Role

Inside the `roles` directory I installed the role from Galaxy and renamed its folder to `mysql`:

```bash
ansible-galaxy install geerlingguy.mysql
mv geerlingguy.mysql/ mysql
```

![Installing the MySQL role](./Images/Task%2013.5.png)

![Renamed mysql role](./Images/Task%2013.6.png)

### 6.0 — Configure Credentials

I read the role's `README.md` and set the database name and user the Tooling website needs. The values live in the environment file `env-vars/uat.yml` (the credentials are hidden here):

```yaml
---
env: uat

mysql_root_password: <root-password>

mysql_databases:
  - name: <tooling-database-name>

mysql_users:
  - name: <tooling-db-user>
    host: "<allowed-host>"
    password: <db-user-password>
    priv: "*.*:ALL"
```

![UAT environment variables for MySQL](./Images/Task%2013.7.png)

### 7.0 — Push and Merge

```bash
git add .
git commit -m "Commit new role files into GitHub"
git push --set-upstream origin roles-feature
```

I opened a Pull Request from `roles-feature` and merged it into `main`.

![Merged pull request](./Images/Task%2013.8.png)

Back on Jenkins-Ansible I pulled the merged changes:

```bash
git checkout main
git pull origin main
```

![Pulling main on Jenkins-Ansible](./Images/Task%2013.9.png)

---

## Part 4 — Load Balancer Roles

I want to choose between an **Nginx** and an **Apache** load balancer per environment. Only one can run at a time, so a variable acts as the switch.

### 8.0 — Create the Roles

```bash
ansible-galaxy init roles/nginx
ansible-galaxy init roles/apache
```

![Creating the nginx and apache roles](./Images/Task%2013.10.png)

### 9.0 — Declare the Switch Variables

In each role's `defaults/main.yml` I declared a flag for that load balancer plus a shared flag saying whether a load balancer is needed at all. Everything defaults to `false`.

`roles/nginx/defaults/main.yml`:

```yaml
---
enable_nginx_lb: false
load_balancer_is_required: false
```

`roles/apache/defaults/main.yml`:

```yaml
---
enable_apache_lb: false
load_balancer_is_required: false
```

![Defaults of the nginx and apache roles](./Images/Task%2013.11.png)

### 10.0 — Assignment and site.yml

`static-assignments/loadbalancers.yml` applies whichever role is enabled:

```yaml
---
- hosts: lb
  roles:
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }
```

and `site.yml` carries a block that imports it for the `lb` group:

```yaml
- name: Loadbalancers assignment
  hosts: lb
  import_playbook: ../static-assignments/loadbalancers.yml
  when: load_balancer_is_required
```

![loadbalancers.yml and site.yml](./Images/Task%2013.12.png)

### 11.0 — Choose a Load Balancer per Environment

To activate Nginx in UAT, I set the matching variables in `env-vars/uat.yml`:

```yaml
enable_nginx_lb: true
load_balancer_is_required: true
```

Switching to Apache is just a matter of flipping the two enable flags in the environment's file.

---

## Part 5 — Test Against UAT

My first test run failed straight away.

![First failed run](./Images/Task%2013.13.png)

I worked through it step by step (details in [Errors & Fixes](#errors--fixes)). First I reworked `playbooks/site.yml`:

![Reworked site.yml](./Images/Task%2013.14.png)

That exposed another problem, an empty `webservers.yml`:

![Empty playbook error](./Images/Task%2013.15.png)

so I gave the assignment its role:

![webservers.yml with the webserver role](./Images/Task%2013.16.png)

and reworked `site.yml` again (see Part 2). The next run still did not configure the servers:

![Run still loading the wrong environment file](./Images/Task%2013.18.png)

so I corrected `dynamic-assignments/env-vars.yml` and `inventory/uat.yml`:

![Corrected env-vars.yml](./Images/Task%2013.19.png)

![Corrected UAT inventory](./Images/Task%2013.20.png)

The playbook now loads `uat.yml`, applies `common`, and configures the web servers through the `webserver` role:

![Successful run - variables and common](./Images/Task%2013.21.png)

![Successful run - webserver role](./Images/Task%2013.22.png)

![Successful run - recap](./Images/Task%2013.23.png)

The same flow works with the Apache load balancer: set its flag to `true`, the other to `false`, update the inventory for the environment and run Ansible against it.

---

## Errors & Fixes

Issues I hit during this project and how they were resolved.

---

### import_playbook Rejected with "Extra Params"

**Symptom:** The first test run (run with `-i ansible/host.ini`, which could not be parsed as an inventory and also produced "Unable to parse ... as an inventory source" warnings) ended with `ERROR! this task 'import_playbook' has extra params, which is only allowed in the following modules: include, include_role, ...`.

**Root cause:** The first draft of `site.yml` mixed a `tasks:` block, `import_playbook` and `include` together in one play. `import_playbook` is a top-level directive and cannot be nested under `tasks:` or carry extra parameters.

**Fix:** Rewrite `site.yml` so each directive stands on its own at the top level, and point `-i` at a real inventory file (for example `inventory/uat.yml`).

---

### "Empty playbook, nothing to do" for webservers.yml

**Symptom:** `ERROR! Empty playbook, nothing to do: .../static-assignments/webservers.yml`

**Root cause:** `static-assignments/webservers.yml` had been created but had no play in it.

**Fix:** Add the play that applies the role:

```yaml
---
- hosts: webservers
  roles:
    - webserver
```

---

### UAT Run Loaded dev.yml and Matched No Web Servers

**Symptom:** The run loaded `env-vars/dev.yml` for the UAT servers (`item=.../env-vars/dev.yml` in the output), printed `Could not match supplied host pattern, ignoring: webservers`, and the web server plays showed `skipping: no hosts matched`.

**Root cause:** Two separate causes. `with_first_found` takes the first file it finds, and `dev.yml` was first in the list of candidates, so it won even in UAT. And the UAT inventory did not define a `webservers` group, so the play targeting `webservers` had no hosts.

**Fix:** Put `uat.yml` first in the `files` list of `dynamic-assignments/env-vars.yml`, and define the hosts under a `webservers` group in `inventory/uat.yml`, with `ansible_host`, `ansible_user` and `ansible_ssh_private_key_file` set for each host so Ansible can connect directly.

---

*Project completed by **Adewale Oluwasemiloore***
