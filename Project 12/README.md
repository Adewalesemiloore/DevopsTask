<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">ANSIBLE REFACTORING AND STATIC ASSIGNMENTS (IMPORTS AND ROLES)</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>November 26, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Red_Hat-EE0000?style=for-the-badge&logo=redhat&logoColor=white" />
  <img src="https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white" />
  <img src="https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white" />
  <img src="https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white" />
  <img src="https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white" />
  <img src="https://img.shields.io/badge/YAML-CB171E?style=for-the-badge&logo=yaml&logoColor=white" />
  <img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" />
</p>

---

## Project Overview

In Project 11, I put every task into a single `common.yml` playbook. That is fine for two operating systems and a handful of servers, but it stops scaling the moment more tasks and more server types arrive. Anyone reading the playbook has to scan the whole file to work out which parts apply to which host, and the file slowly fills with commented-out experiments.

This project is about **refactoring** that code — changing how it is organised without changing what it does to the infrastructure. I cover three ideas:

- **A cleaner Jenkins pipeline.** Instead of every build creating a new directory under Jenkins, a second job copies the artifacts of each build into one fixed folder on the Jenkins-Ansible server.
- **Imports.** `site.yml` becomes the single entry point of the whole configuration and pulls in smaller, focused playbooks with `import_playbook`.
- **Roles.** I build a reusable `webserver` role (Apache plus the Tooling website) and use it to configure two brand new **UAT** web servers.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part 1 — Jenkins Job Enhancement](#part-1--jenkins-job-enhancement)
- [Part 2 — Refactor with Imports in site.yml](#part-2--refactor-with-imports-in-siteyml)
- [Part 3 — Create the Webserver Role for UAT](#part-3--create-the-webserver-role-for-uat)
- [Part 4 — Reference the Role from site.yml](#part-4--reference-the-role-from-siteyml)
- [Part 5 — Commit and Test Against UAT](#part-5--commit-and-test-against-uat)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

The Jenkins-Ansible server from Project 11 stays at the centre. Two new RHEL 8 instances are added as the UAT environment.

| Server | OS | Role |
|---|---|---|
| Jenkins-Ansible | Ubuntu | Jenkins + Ansible, jump server, holds the `ansible-config-mgt` repo |
| Web1-UAT, Web2-UAT | RHEL 8 | New UAT web servers configured through the `webserver` role |
| NFS, Web, DB servers | Red Hat / Ubuntu | Existing `dev` environment |
| Load Balancer | Ubuntu | Existing `dev` environment |

The repository layout after this project:

```
├── static-assignments
│   ├── common.yml
│   ├── common-del.yml
│   └── uat-webservers.yml
├── inventory
│   ├── dev.yml
│   ├── stage.yml
│   ├── uat.yml
│   └── prod.yml
├── playbooks
│   └── site.yml
└── roles
    └── webserver
```

---

## Part 1 — Jenkins Job Enhancement

Each push used to create a separate build directory on the Jenkins server, which wastes space and makes it awkward to run commands from one known place. I fixed this with a second Jenkins job that stores the latest artifacts in a single directory. It relies on the **Copy Artifact** plugin.

### 1.0 — Create the Artifact Directory

On the Jenkins-Ansible server I created the target directory and opened up its permissions so Jenkins can write to it:

```bash
sudo mkdir /home/ubuntu/ansible-config-artifact
sudo chmod -R 0777 /home/ubuntu/ansible-config-artifact
```

![Creating the artifact directory](./Images/Task%2012.1.png)

### 2.0 — Install the Copy Artifact Plugin

In Jenkins: **Manage Jenkins → Plugins → Available**, search for **Copy Artifact** and install it without restarting Jenkins.

![Installing Copy Artifact plugin](./Images/Task%2012.2.png)

### 3.0 — Create the save_artifacts Job

I created a new **Freestyle project** named `save_artifacts`.

![Creating the save_artifacts project](./Images/Task%2012.3.png)

This job should run when my existing `ansible` job finishes, so I configured it with a **Build after other projects are built** trigger pointing at `ansible`.

![Build trigger on the ansible project](./Images/Task%2012.4.png)

> **Note:** To save disk space, I can limit how many builds Jenkins keeps (for example the last 2 or 5). The same setting can be applied to the `ansible` job.

![Discarding old builds](./Images/Task%2012.5.png)

### 4.0 — Add the Copy Artifacts Build Step

The whole point of `save_artifacts` is to copy files into `/home/ubuntu/ansible-config-artifact`. I added a **Copy artifacts from another project** build step with `ansible` as the source project and that directory as the target.

![Copy artifacts build step](./Images/Task%2012.6.png)

### 5.0 — Test the Setup

I edited `README.md` in the `ansible-config-mgt` repo to trigger the chain. The first run of `save_artifacts` failed with an access denied error on the target directory (see [Errors & Fixes](#errors--fixes)).

![save_artifacts failing with AccessDeniedException](./Images/Task%2012.7.png)

After correcting the ownership and permissions of the directory:

```bash
sudo mkdir -p /home/ubuntu/ansible-config-artifact
sudo chown -R jenkins:jenkins /home/ubuntu/ansible-config-artifact
sudo chmod -R 755 /home/ubuntu/ansible-config-artifact
ls -ld /home/ubuntu/ansible-config-artifact
sudo chown -R jenkins:jenkins /home/ubuntu
```

![Fixing ownership of the artifact directory](./Images/Task%2012.9.png)

the job ran successfully and copied the artifacts from the `ansible` job.

![save_artifacts successful](./Images/Task%2012.8.png)

The files now sit inside `/home/ubuntu/ansible-config-artifact` and get refreshed with every commit to the main branch, which makes the pipeline much tidier.

![Artifacts in the target directory](./Images/Task%2012.10.png)

---

## Part 2 — Refactor with Imports in site.yml

### 6.0 — Create a Refactor Branch

Before changing any code I pulled the latest `main` and created a new branch called `refactor`.

![Creating the refactor branch](./Images/Task%2012.11.png)

### 7.0 — Why Split the Playbook?

Everything in `common.yml` works today, so why touch it? Because breaking tasks into separate files makes complex configuration far easier to read, reuse and hand over to colleagues. Reuse is the key idea: a small playbook can be imported wherever it is needed.

### 8.0 — Create site.yml and static-assignments

1. Inside `playbooks/`, create `site.yml`. This becomes the **entry point** of the entire infrastructure configuration and the parent of all other playbooks.
2. In the root of the repository create a folder called `static-assignments`. It holds all the child playbooks. The name is just my own organisation, not an Ansible rule. It is called "static" because imports are resolved when the playbook is parsed.
3. Move `common.yml` into `static-assignments`.

![Moving common.yml into static-assignments](./Images/Task%2012.12.png)

4. In `site.yml`, import `common.yml` using the built-in `import_playbook` directive:

```yaml
---
- hosts: all
- import_playbook: ../static-assignments/common.yml
```

![site.yml importing common.yml](./Images/Task%2012.13.png)

I then checked that the refactored playbook still parses correctly against the `dev` inventory:

```bash
ansible-playbook -i inventory/dev.yml --syntax-check playbooks/site.yml
```

![Syntax check of site.yml](./Images/Task%2012.14.png)

### 9.0 — Add a common-del Playbook

Wireshark was already installed on the `dev` servers from Project 11, so I created a second child playbook, `static-assignments/common-del.yml`, that removes it.

![Creating common-del.yml](./Images/Task%2012.15.png)

```yaml
---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    yum:
      name: wireshark
      state: removed

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    apt:
      name: wireshark
      state: absent
      autoremove: yes
      purge: yes
      autoclean: yes
```

> The package name on the Ubuntu play is `wireshark`. My first draft used `wireshark-qt`, which did not match and caused the load balancer to be skipped silently. See [Errors & Fixes](#errors--fixes).

Then I switched the import in `site.yml` to `common-del.yml` and ran it against `dev`:

```bash
cd /home/ubuntu/ansible-config-mgt/
ansible-playbook -i inventory/dev.yml playbooks/site.yml
```

![First run of common-del through site.yml](./Images/Task%2012.16.png)

After correcting the package name in the file:

![Corrected common-del.yml](./Images/Task%2012.17.png)

the rerun reached every host and removed Wireshark from the load balancer.

![Rerun after the fix](./Images/Task%2012.18.png)

I confirmed the removal on the servers with `wireshark --version`.

![Wireshark removed on the servers](./Images/Task%2012.19.png)

At this point I have a one-command way to install or remove a package across many servers just by changing which playbook `site.yml` imports.

---

## Part 3 — Create the Webserver Role for UAT

The `dev` environment is tidy, so I set it aside and prepared two fresh servers for **UAT**. Rather than stuffing more tasks into one playbook, I used a dedicated **role** so the configuration can be reused.

### 10.0 — Launch the UAT Servers

I launched two **RHEL 8** EC2 instances and named them **Web1-UAT** and **Web2-UAT**.

![Web1-UAT and Web2-UAT instances](./Images/Task%2012.20.png)

### 11.0 — Generate the Role Skeleton

Roles live in a `roles/` directory next to the playbooks. The role skeleton can be created with `ansible-galaxy`:

```bash
mkdir roles
cd roles
ansible-galaxy init webserver
```

![ansible-galaxy init webserver](./Images/Task%2012.21.png)

> Creating the structure manually is also fine, and since everything is stored in GitHub it is recommended to create the folders there rather than only on the Jenkins-Ansible server.

The generated structure contains `defaults`, `files`, `handlers`, `meta`, `tasks`, `templates`, `tests` and `vars`.

![Generated role structure](./Images/Task%2012.22.png)

I removed the directories I do not need (`files`, `tests`, `vars`) to keep the role lean.

![Role structure after cleanup](./Images/Task%2012.22.5.png)

```
└── webserver
    ├── README.md
    ├── defaults
    │   └── main.yml
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    └── templates
```

### 12.0 — Update the UAT Inventory

`inventory/uat.yml` holds the two UAT servers. My first draft only listed the two private IPs under a `uat-webservers` group, with no user or key:

![First draft of the UAT inventory](./Images/Task%2012.23.png)

I then used the YAML inventory format and set the user and key for each host explicitly (this is the corrected version, see [Errors & Fixes](#errors--fixes)):

```yaml
uat-webservers:
  hosts:
    web1:
      ansible_host: <Web1-UAT-Private-IP>
      ansible_user: ec2-user
      ansible_ssh_private_key_file: "/home/ubuntu/.ssh/<your-key>.pem"

    web2:
      ansible_host: <Web2-UAT-Private-IP>
      ansible_user: ec2-user
      ansible_ssh_private_key_file: "/home/ubuntu/.ssh/<your-key>.pem"
```

![UAT inventory](./Images/Task%2012.24.png)

### 13.0 — Point Ansible at the Roles Directory

In `/etc/ansible/ansible.cfg` I uncommented `roles_path` and gave it the full path to my roles directory so Ansible knows where to find them:

```
roles_path = /home/ubuntu/ansible-config-mgt/roles
```

![roles_path in ansible.cfg](./Images/Task%2012.25.png)

### 14.0 — Write the Role Tasks

In `roles/webserver/tasks/main.yml` I described what a UAT web server needs: Apache installed, the Tooling site cloned from GitHub into `/var/www/html`, and `httpd` running.

```yaml
---
- name: install apache
  become: true
  ansible.builtin.yum:
    name: "httpd"
    state: present

- name: install git
  become: true
  ansible.builtin.yum:
    name: "git"
    state: present

- name: clone a repo
  become: true
  ansible.builtin.git:
    repo: https://github.com/<your-github-username>/tooling.git
    dest: /var/www/html
    force: yes

- name: copy html content to one level up
  become: true
  command: cp -r /var/www/html/html/ /var/www/

- name: Start service httpd, if not started
  become: true
  ansible.builtin.service:
    name: httpd
    state: started

- name: recursively remove /var/www/html/html/ directory
  become: true
  ansible.builtin.file:
    path: /var/www/html/html
    state: absent
```

![Webserver role tasks](./Images/Task%2012.26.png)

---

## Part 4 — Reference the Role from site.yml

Inside `static-assignments` I created `uat-webservers.yml`, the assignment that applies the role to the UAT group:

```yaml
---
- hosts: uat-webservers
  roles:
     - webserver
```

Since `site.yml` is the entry point, it must import this assignment as well:

```yaml
---
- hosts: all
- import_playbook: ../static-assignments/common.yml

- hosts: uat-webservers
- import_playbook: ../static-assignments/uat-webservers.yml
```

![site.yml importing the UAT assignment](./Images/Task%2012.27.png)

---

## Part 5 — Commit and Test Against UAT

I committed the changes, opened a Pull Request and merged it into `main`. The webhook triggered both Jenkins jobs, and the files were copied into `/home/ubuntu/ansible-config-mgt/` on the Jenkins-Ansible server.

Then I ran `site.yml` against the UAT inventory:

```bash
ansible-playbook -i /home/ubuntu/ansible-config-mgt/inventory/uat.yml /home/ubuntu/ansible-config-mgt/playbooks/site.yml
```

My first attempt failed to authenticate to the UAT hosts.

![First UAT run failing](./Images/Task%2012.28.png)

I fixed the inventory so each host carried its own user and key (the version shown in section 12.0):

![Reconfigured UAT inventory](./Images/Task%2012.29.png)

and the rerun configured both servers.

![UAT run output - part 1](./Images/Task%2012.30.png)

![UAT run output - part 2](./Images/Task%2012.31.png)

With TCP port 80 open in the UAT security groups, both servers serve the Tooling website:

`http://<Web1-UAT-Public-IP-or-DNS>/index.php`

![Web1-UAT serving the site](./Images/Task%2012.32.png)

`http://<Web2-UAT-Public-IP-or-DNS>/index.php`

![Web2-UAT serving the site](./Images/Task%2012.33.png)

The Ansible architecture now looks like this:

![Final Ansible architecture](./Images/Task%2012.34.png)

---

## Errors & Fixes

Issues I hit during this project and how they were resolved.

---

### save_artifacts Job Fails with AccessDeniedException

**Symptom:** The first `save_artifacts` build ended in `FAILURE` with `java.nio.file.AccessDeniedException: /home/ubuntu/ansible-config-artifact`.

**Root cause:** The directory was created with `sudo`, so Jenkins (running as the `jenkins` user) did not have permission to create files in it.

**Fix:** Give the `jenkins` user ownership and sensible permissions, then rebuild:

```bash
sudo mkdir -p /home/ubuntu/ansible-config-artifact
sudo chown -R jenkins:jenkins /home/ubuntu/ansible-config-artifact
sudo chmod -R 755 /home/ubuntu/ansible-config-artifact
sudo chown -R jenkins:jenkins /home/ubuntu
```

---

### common-del.yml Did Not Remove Wireshark from the Load Balancer

**Symptom:** After importing `common-del.yml`, the run reported `ok` and no change for the load balancer, so Wireshark was still installed there.

**Root cause:** The Ubuntu task used the package name `wireshark-qt`, which did not match the package that had been installed in Project 11 (`wireshark`).

**Fix:** Change the package name to `wireshark` in the `apt` task. On the rerun the load balancer showed `changed=1` and every host was reached.

---

### UAT Playbook Run Fails with Permission Denied

**Symptom:** Running `sudo ansible-playbook -i inventory/uat.yml playbooks/site.yml` printed warnings about invalid characters in group names and unmatched host patterns, then failed with `UNREACHABLE! ... Permission denied (publickey,gssapi-keyex,gssapi-with-mic)` on the UAT servers.

**Root cause:** The UAT inventory did not tell Ansible which user and private key to use for the new RHEL servers. The warnings about `webservers`, `nfs`, `lb` and `db` are expected here, because the UAT inventory only contains the `uat-webservers` group.

**Fix:** Rewrite `inventory/uat.yml` in YAML form with `ansible_host`, `ansible_user: ec2-user` and `ansible_ssh_private_key_file` for each host (see section 12.0), then rerun the playbook.

---

*Project completed by **Adewale Oluwasemiloore***
