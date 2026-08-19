# ANSIBLE CONFIGURATION MANAGEMENT — AUTOMATE INFRASTRUCTURE WITH ANSIBLE

<br />
<div align="center">

<h3 align="center">ANSIBLE CONFIGURATION MANAGEMENT — AUTOMATE INFRASTRUCTURE SETUP</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>August 19, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Red_Hat-EE0000?style=for-the-badge&logo=redhat&logoColor=white" />
  <img src="https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white" />
  <img src="https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white" />
  <img src="https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white" />
  <img src="https://img.shields.io/badge/YAML-CB171E?style=for-the-badge&logo=yaml&logoColor=white" />
  <img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" />
</p>

---

## Project Overview

In Projects 7 to 10, every server configuration was done manually — SSH in, run commands, repeat for each machine. That works with 3 servers. It breaks down fast with 10, 20, or 50.

This project solves that with **Ansible**, an open-source automation tool that lets you describe your infrastructure in simple YAML files called **playbooks**, then apply those configurations to any number of servers simultaneously with a single command. No agents needed on the target servers — Ansible connects over SSH.

The Jenkins-Ansible server acts as a **Jump Server (Bastion Host)** — a single secure entry point that can reach all other servers in the private network. When you push code to GitHub, Jenkins pulls it automatically via webhook, and Ansible runs the playbook against all target servers. The result: one push updates your entire infrastructure.

![Architecture Overview](./Images/0%20-%20ansible%20Jump%20server%20image.png)

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part 1 — Install and Configure Ansible](#part-1--install-and-configure-ansible)
- [Part 2 — Configure Jenkins to Trigger on GitHub Push](#part-2--configure-jenkins-to-trigger-on-github-push)
- [Part 3 — Set Up VS Code Remote Development](#part-3--set-up-vs-code-remote-development)
- [Part 4 — Build the Ansible Directory Structure](#part-4--build-the-ansible-directory-structure)
- [Part 5 — Set Up the Ansible Inventory](#part-5--set-up-the-ansible-inventory)
- [Part 6 — Write the Common Playbook](#part-6--write-the-common-playbook)
- [Part 7 — Push to GitHub and Trigger Jenkins](#part-7--push-to-github-and-trigger-jenkins)
- [Part 8 — Run the Ansible Playbook](#part-8--run-the-ansible-playbook)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

This project extends the existing infrastructure by turning the Jenkins server into a **Jenkins-Ansible** combo — it both triggers on code changes and runs Ansible playbooks against all other servers.

| Server | OS | Role |
|---|---|---|
| Jenkins-Ansible | Ubuntu | Runs Jenkins + Ansible. Acts as the Jump Server / Bastion Host |
| NFS Server | Red Hat | Shared storage for web servers |
| Web Server 1 & 2 | Red Hat | Serve the tooling website |
| Database Server | Ubuntu | MySQL database |
| Load Balancer | Ubuntu | Distributes traffic across web servers |

The flow: **Push to GitHub → GitHub webhook notifies Jenkins → Jenkins pulls the latest code → Ansible playbook runs against all servers simultaneously.**

---

## Part 1 — Install and Configure Ansible

### 1.0 — Install Ansible on the Jenkins-Ansible Server

First, update the Jenkins EC2 instance name tag to **Jenkins-Ansible** in the AWS console — this server will now serve a dual purpose.

Then SSH in and install Ansible:

```bash
sudo apt update
sudo apt install ansible
```

![Installing Ansible](./Images/1.0%20Installing%20Ansible.png)

Confirm it installed correctly:

```bash
ansible --version
```

![Ansible Version](./Images/1.1%20Ansible%20Version.png)

### 2.0 — Create the GitHub Repository

In your GitHub account, create a new repository named **ansible-config-mgt**. This is where all your Ansible code will live.

![Creating Repo](./Images/2.0%20-%20Creating%20ansible-config-mgt%20repo.png)
![Creating Repo](./Images/2.0a%20-%20Creating%20ansible-config-mgt%20repo.png)

---

## Part 2 — Configure Jenkins to Trigger on GitHub Push

Jenkins needs to watch the `ansible-config-mgt` repo and kick off a build every time you push code.

### 3.0 — Create a New Jenkins Freestyle Project

In Jenkins, click **New Item**, name it `ansible`, and select **Freestyle project**.

![Creating Jenkins Item](./Images/3.0%20Creating%20a%20new%20item.png)

Under **Source Code Management**, select **Git** and paste your `ansible-config-mgt` repo URL.

![Git URL](./Images/3.1%20Git%20URL.png)

### 3.2 — Add a Post-Build Action to Archive Artifacts

Under **Post-build Actions**, add **Archive the artifacts** and set the file pattern to `**` — this saves everything from every build.

![Post Build Action](./Images/3.3%20Post%20build%20action.png)

### 3.3 — Add the GitHub Webhook

Go to your `ansible-config-mgt` repo on GitHub → **Settings → Webhooks → Add webhook**.

Set the Payload URL to:
```
http://<Jenkins-Ansible-Public-IP>:8080/github-webhook/
```

![Adding Webhook](./Images/3.4%20-%20adding%20webhook.png)

Back in Jenkins, under **Build Triggers**, enable **"GitHub hook trigger for GITScm polling"**.

### 3.4 — Verify the Setup

Make a small change to `README.md` in your repo and push it. Jenkins should trigger a new build automatically.

![Checking Update](./Images/3.5%20Checking%20update.png)

Your setup now looks like this:

![Setup Overview](./Images/3.6%20Our%20set%20up.png)

> **Important:** Every time you stop and start the Jenkins-Ansible server, its public IP changes and you'd have to update the webhook. Assign an **Elastic IP** to avoid this.

![Elastic IP](./Images/3.7%20elastic%20IP.png)
![Updating Webhook](./Images/3.8%20updating%20Git%20webhook.png)

---

## Part 3 — Set Up VS Code Remote Development

Rather than editing files directly on the server, connect VS Code to Jenkins-Ansible so you can write and manage files locally with full IDE support.

### Set Up SSH Agent Forwarding

Ansible needs to SSH from Jenkins-Ansible into all the other servers. To make this work, your local key needs to be **forwarded through** to Jenkins-Ansible — so it can use it without storing a copy there.

On your local machine:

```bash
eval `ssh-agent -s`
ssh-add ~/.ssh/Semiloore-ec2.pem
ssh-add -l   # confirm the key fingerprint appears
```

Then connect to Jenkins-Ansible with agent forwarding:

```bash
ssh -A ubuntu@<Jenkins-Ansible-Public-IP>
```

![SSH Agent Setup](./Images/5%20—%20SSH%20agent%20setup%20%28for%20Ansible%20to%20reach%20your%20other%20servers%29.png)

> **Note:** Run `ssh-add` in plain Terminal *before* opening VS Code. If VS Code connects first, the agent won't forward.

---

## Part 4 — Build the Ansible Directory Structure

Clone the repo onto Jenkins-Ansible and set up the folder structure Ansible expects.

```bash
git clone https://github.com/Adewalesemiloore/ansible-config-mgt.git
cd ansible-config-mgt
```

Create a feature branch to work on (never edit `main` directly):

```bash
git checkout -b feature/prj-11-ansible-setup
```

Create the two core directories:

```bash
mkdir playbooks inventory
```

Then create the files:

```bash
touch playbooks/common.yml
touch inventory/dev.yml inventory/staging.yml inventory/uat.yml inventory/prod.yml
```

![Directory Structure](./Images/6%20%26%207%20—%20Create%20the%20directory%20structure.png)

---

## Part 5 — Set Up the Ansible Inventory

The inventory file tells Ansible which servers to connect to and with which user. Your infrastructure has a mix of Red Hat (uses `ec2-user`) and Ubuntu (uses `ubuntu`) servers.

Edit `inventory/dev.yml`:

```ini
[nfs]
172.31.42.61 ansible_ssh_user='ec2-user'

[webservers]
172.31.39.169 ansible_ssh_user='ec2-user'
172.31.43.142 ansible_ssh_user='ec2-user'

[db]
172.31.30.33 ansible_ssh_user='ubuntu'

[lb]
172.31.41.8 ansible_ssh_user='ubuntu'
```

> **Why different users?** Red Hat/Amazon Linux AMIs use `ec2-user`. Ubuntu AMIs use `ubuntu`. Check the AMI name in the AWS console if you're unsure which is which.

![Dev Inventory](./Images/8%20—%20Fill%20in%20your%20dev%20inventory.png)

Before running any playbook, SSH manually into each server once to accept its host key — Ansible will hang otherwise:

```bash
ssh -A ec2-user@172.31.39.169   # Web server 1
ssh -A ec2-user@172.31.43.142   # Web server 2
ssh -A ec2-user@172.31.42.61    # NFS server
ssh -A ubuntu@172.31.30.33      # DB server
ssh -A ubuntu@172.31.41.8       # Load balancer
```

Type `yes` at each prompt, then `exit` to return to Jenkins-Ansible.

---

## Part 6 — Write the Common Playbook

The `common.yml` playbook installs Wireshark on all servers. Because the servers use different package managers (`dnf` for Red Hat, `apt` for Ubuntu), the playbook is split into separate plays per group.

Edit `playbooks/common.yml`:

```yaml
- name: update web and nfs servers
  hosts: webservers, nfs
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
    - name: ensure wireshark is at the latest version
      dnf:
        name: wireshark
        state: latest

- name: update db server
  hosts: db
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
    - name: Update apt repo
      apt:
        update_cache: yes

    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
    - name: Update apt repo
      apt:
        update_cache: yes

    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest
```

![Common Playbook](./Images/9%20—%20Write%20the%20common%20playbook.png)

---

## Part 7 — Push to GitHub and Trigger Jenkins

Commit your work on the feature branch and push it:

```bash
git add .
git commit -m "Add inventory and common playbook"
git push origin feature/prj-11-ansible-setup
```

![Commit and Push](./Images/10%20-%20commit%20and%20push.png)

On GitHub, open a **Pull Request** from `feature/prj-11-ansible-setup` into `main`. Review the changes, then merge.

![Pull Request](./Images/11%20—%20Open%20a%20Pull%20Request%20and%20merge.png)

Back on Jenkins-Ansible, pull the merged changes:

```bash
git checkout main
git pull origin main
```

Check Jenkins — a new build should have triggered automatically from the merge. The build artifacts (your inventory and playbook files) will be saved to:

```
/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```

![Jenkins Build Artifacts](./Images/12%20-%20Jenkins%20Build%20artifacts%20update.png)

---

## Part 8 — Run the Ansible Playbook

Now run the playbook from the `ansible-config-mgt` directory:

```bash
cd ~/ansible-config-mgt
ansible-playbook -i inventory/dev.yml playbooks/common.yml
```

A successful run looks like this — all servers show `ok` or `changed`, with `unreachable=0` and `failed=0`:

```
PLAY RECAP
172.31.30.33    : ok=3    changed=2    unreachable=0    failed=0
172.31.39.169   : ok=2    changed=1    unreachable=0    failed=0
172.31.41.8     : ok=3    changed=2    unreachable=0    failed=0
172.31.42.61    : ok=2    changed=1    unreachable=0    failed=0
172.31.43.142   : ok=2    changed=1    unreachable=0    failed=0
```

Verify Wireshark installed on any server:

```bash
ssh -A ec2-user@172.31.39.169
wireshark --version
```

Your updated architecture now looks like this — Jenkins-Ansible sitting at the centre, reaching all servers through the VPC:

![Final Architecture](./Images/3.6%20Our%20set%20up.png)

---

---

## Errors & Fixes

Real issues hit during this project and exactly how they were resolved.

---

### SSH Agent Forwarding Not Working in VS Code

**Symptom:** Running `ssh-add -l` inside the VS Code terminal showed `The agent has no identities` — meaning the key wasn't forwarding from the local machine through to Jenkins-Ansible.

**Root cause:** VS Code's Remote-SSH only forwards the SSH agent if the key is already loaded in the local agent *before* VS Code establishes the connection. Opening VS Code first and then adding the key does nothing.

**Fix:**
1. Kill the VS Code remote session: `Cmd+Shift+P` → **Remote-SSH: Kill Current VS Code Server**
2. In plain Mac Terminal (not VS Code):
```bash
eval `ssh-agent -s`
ssh-add ~/.ssh/Semiloore-ec2.pem
ssh-add -l   # confirm fingerprint appears
```
3. Only then reconnect VS Code to jenkins-ansible
4. Verify forwarding worked: `ssh-add -l` inside VS Code terminal should now show the key

Also add `AddKeysToAgent yes` to `~/.ssh/config` so the key auto-loads on future connections:

```
Host jenkins-ansible
    HostName 16.170.19.202
    User ubuntu
    IdentityFile ~/.ssh/Semiloore-ec2.pem
    ForwardAgent yes
    AddKeysToAgent yes
```

---

### VS Code Remote-SSH Connection Timing Out

**Symptom:** VS Code showed `OfflineError: The connection timed out` when trying to connect to jenkins-ansible, even with good internet.

**Root cause:** The `~/.ssh/config` had two conflicting entries pointing to the same IP (`Host 16.170.19.202` and `Host jenkins-ansible`). This caused VS Code to get confused about which config to use.

**Fix:** Remove the duplicate entry and keep only one clean block:

```
Host jenkins-ansible
    HostName 16.170.19.202
    User ubuntu
    IdentityFile ~/.ssh/Semiloore-ec2.pem
    ForwardAgent yes
    AddKeysToAgent yes
```

---

### Permission Denied on Webservers — Wrong Username

**Symptom:** `ssh -A ubuntu@172.31.39.169` returned `Permission denied (publickey)`.

**Root cause:** The web servers and NFS server run **Red Hat Linux**, not Ubuntu. Red Hat AMIs use `ec2-user`, not `ubuntu`.

**How it was identified:** Switching to `ec2-user` connected immediately and showed the Red Hat Lightspeed registration banner, confirming the OS.

**Fix:** Use `ec2-user` for all Red Hat servers:
```bash
ssh -A ec2-user@172.31.39.169
```

And update `inventory/dev.yml` accordingly — `ec2-user` for webservers and NFS, `ubuntu` for db and lb.

---

### Database Server SSH Hanging — Port 22 Not Open

**Symptom:** `ssh -A ubuntu@172.31.30.33` connected at the TCP level (`nc -zv` succeeded) but the SSH handshake never completed — the terminal just hung.

**Root cause:** The Database Server's security group had no inbound rule for **port 22**. It had rules for 3306, 8080, 443 etc., but SSH was missing entirely.

**Fix:** AWS Console → EC2 → Database Server → Security tab → Edit inbound rules → Add:

| Type | Port | Source |
|---|---|---|
| SSH | 22 | 172.31.0.0/16 |

After saving the rule, the existing frozen connection still needed to be killed (`Ctrl+C`) and retried — security group changes don't apply to already-established (or hanging) connections.

---

### Database Server SSH Still Hanging After Opening Port 22

**Symptom:** Even after adding the port 22 rule, SSH to the DB server continued to hang at the handshake.

**Root cause:** The SSH daemon on the database server had gotten into a broken state — it was accepting TCP connections but not responding to the SSH protocol handshake.

**Fix:** Reboot the instance from the AWS console:

AWS Console → EC2 → Database Server → **Instance state → Reboot instance**

After ~60 seconds, SSH connected immediately.

---

### Ansible Playbook Failing on DB Server — Wrong Package Manager

**Symptom:** The initial `common.yml` used `dnf` for `webservers, nfs, db` — but the database server is Ubuntu, which uses `apt`. Ansible would either fail or skip tasks on that host.

**Fix:** Split the db server into its own play with `apt` and `remote_user: ubuntu`:

```yaml
- name: update db server
  hosts: db
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
    - name: Update apt repo
      apt:
        update_cache: yes
    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest
```

---

### LB Server — Host Key Verification Failed

**Symptom:** Ansible couldn't connect to the Load Balancer (`172.31.41.8`) and returned `Host key verification failed`.

**Root cause:** Ansible's strict host key checking requires the server's fingerprint to already be in `~/.ssh/known_hosts`. The LB server had never been SSH'd into manually, so no fingerprint existed.

**Fix:** SSH into the LB server once manually before running the playbook:

```bash
ssh -A ubuntu@172.31.41.8
```

Type `yes` when prompted, then `exit`. Ansible can now connect without issue.

---

### Jenkins Not Triggering on GitHub Push

**Symptom:** Pushing to `main` didn't trigger a new Jenkins build. The GitHub webhook showed `failed to connect to host`.

**Root cause:** The webhook Payload URL was pointing to the wrong server IP — it was hitting `13.53.44.138` (the Load Balancer) instead of `16.170.19.202` (Jenkins-Ansible).

**Fix:** GitHub → ansible-config-mgt repo → Settings → Webhooks → Edit → update the Payload URL to:
```
http://16.170.19.202:8080/github-webhook/
```

> **Prevention:** Assign an Elastic IP to Jenkins-Ansible so the IP never changes when the instance is stopped and started. The Elastic IP (`16.170.19.202`) is already assigned in this setup.

---

*Project completed by **Adewale Oluwasemiloore***
