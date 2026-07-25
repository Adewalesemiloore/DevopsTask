<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">TOOLING WEBSITE DEPLOYMENT AUTOMATION WITH CONTINUOUS INTEGRATION — INTRODUCTION TO JENKINS</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>July 20, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white" />
  <img src="https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white" />
  <img src="https://img.shields.io/badge/SSH-black?style=for-the-badge&logo=gnu-bash&logoColor=white" />
  <img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" />
</p>

---

## Project Overview

In Project 8, I introduced horizontal scaling — spreading traffic across multiple web servers behind a load balancer. That solves the traffic problem, but it introduces a new one: every time source code changes, those changes need to be manually copied to every server in the fleet. With 2 or 3 servers, that's tedious. With dozens or hundreds, it's simply not sustainable by hand.

This project solves that with **Continuous Integration (CI)** — automating the process of picking up code changes and deploying them, using **Jenkins**, one of the most widely used open-source automation servers in the industry. I configured a Jenkins server that listens for changes pushed to my GitHub Tooling repository via a **webhook**, automatically pulls the latest code, archives it as a build artifact, and then pushes those files over **SSH** to the NFS server — the same shared storage all 3 web servers read from. The end result: pushing a change to GitHub is enough to update the live website, with zero manual file copying.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part 1 — Install and Configure Jenkins](#part-1--install-and-configure-jenkins)
- [Part 2 — Configure Jenkins to Retrieve Code via Webhooks](#part-2--configure-jenkins-to-retrieve-code-via-webhooks)
- [Part 3 — Configure Jenkins to Copy Files to the NFS Server via SSH](#part-3--configure-jenkins-to-copy-files-to-the-nfs-server-via-ssh)
- [Result](#result)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

This project adds **one new instance** to the existing Project 7/8 infrastructure:

| Server | OS | Role |
|---|---|---|
| Jenkins Server | Ubuntu 20.04 | Runs Jenkins, pulls code from GitHub, pushes files to NFS over SSH |
| NFS Server | RedHat | Same shared storage from Project 7 — receives deployed files at `/mnt/apps` |
| Web Server 1, 2, 3 | RedHat | Same as Project 7/8 — serve whatever files currently sit in the NFS share |

The flow: **Developer pushes to GitHub → GitHub webhook notifies Jenkins → Jenkins pulls the code and archives it → Jenkins pushes the files to the NFS server over SSH → Web servers instantly serve the updated files**, since they're all reading from the same NFS mount.

---

## Part 1 — Install and Configure Jenkins

### 2.0 — Launching and Preparing the Jenkins Server

I launched a new **Ubuntu Server 20.04 EC2** instance and named it `Jenkins`. Since Jenkins is Java-based, I installed the JDK first:

```bash
sudo apt update
sudo apt install default-jdk-headless -y
```

![Launching and Preparing Jenkins Server](./Images/2.0%20-%20Lunching%20and%20preparing%20Jenkins%20Server.png)

---

### 3.0 — Installing Jenkins

```bash
sudo rm -f /etc/apt/sources.list.d/jenkins.list
sudo rm -f /etc/apt/keyrings/jenkins-keyring.asc

sudo apt update
sudo apt install -y fontconfig openjdk-17-jre

sudo mkdir -p /etc/apt/keyrings
```

![Installing Jenkins](./Images/3.0%20Installing%20Jenkins.png)

```bash
sudo curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key \
  -o /etc/apt/keyrings/jenkins-keyring.asc
sudo chmod a+r /etc/apt/keyrings/jenkins-keyring.asc

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
```

![Installing Jenkins](./Images/3.1%20Installing%20Jenkins.png)

---

### 4.0 — Opening Jenkins' Port

Jenkins runs on port **8080** by default. I opened it in the instance's Security Group:

| Type | Protocol | Port | Source |
|---|---|---|---|
| Custom TCP | TCP | 8080 | 0.0.0.0/0 |

![Opening Jenkins Port](./Images/4.0%20Opening%20Jenkins%20Port.png)

---

### 5.0 — Initial Jenkins Setup

I visited `http://<Jenkins-Server-Public-IP>:8080` in my browser and retrieved the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

![Initial Jenkins Setup](./Images/5.0%20Initial%20Jenkins%20Setup.png)

I selected **"Install suggested plugins"** and let Jenkins install its default plugin set.

![Selected Install Suggested Plugins](./Images/5.1%20selected%20install%20suggested%20plugins.png)

Then created my admin user to complete the setup.

![Creating Admin User](./Images/5.2%20Creating%20and%20Admin%20User.png)

---

## Part 2 — Configure Jenkins to Retrieve Code via Webhooks

### 7.0 — Enabling a Webhook on the GitHub Repo

In my forked Tooling repository on GitHub, I went to **Settings → Webhooks → Add webhook**, and set the Payload URL to point to my Jenkins server:

```
http://<Jenkins-Server-Public-IP>:8080/github-webhook/
```

![Enabling a Webhook on GitHub](./Images/7.0%20Enabling%20a%20webhook%20on%20my%20Github%20Tooling.png)

---

### 7.1 — Confirming the Webhook Connected

GitHub sends a test ping when a webhook is created. I confirmed it was received successfully with a green checkmark under Recent Deliveries.

![Test Ping Received](./Images/7.1%20Test%20pinng%20received.png)

---

### 8.0 — Creating the Jenkins Freestyle Project

On the Jenkins dashboard, I clicked **New Item**, named it, and selected **Freestyle project**.

![Creating Jenkins Project](./Images/8.0%20Creating%20Jenkins%20Project.png)

---

### 9.0 — Connecting the GitHub Repo to Jenkins

Under **Source Code Management**, I selected **Git**, pasted my repo URL, and added GitHub credentials (a Personal Access Token, since GitHub no longer accepts account passwords for this).

![Connecting GitHub Repo to Jenkins](./Images/9.0%20Connecting%20github%20repo%20to%20jenkins.png)

---

### 10.0 — Running the First Manual Build

I clicked **Build Now** and confirmed build `#1` completed successfully by checking its Console Output.

![Running First Manual Build](./Images/10.0%20%20Running%20my%20first%20Manual%20Build.png)

---

### 11.0 — Configuring Automatic Triggers and Archiving

Under **Configure**, I enabled **"GitHub hook trigger for GITScm polling"** under Build Triggers, and added an **"Archive the artifacts"** Post-build Action set to `**` (archive everything).

![Configuring Triggers](./Images/11.0%20Configuring%20Triggers.png)

---

### 12.0 — Testing the Automatic Trigger

I made a small change to `README.md` in my GitHub repo and pushed it. A new build launched automatically via the webhook — confirming the push-triggered pipeline was working end to end.

```bash
ls /var/lib/jenkins/jobs/<Your_Job_Name>/builds/<build_number>/archive/
```

![Testing the Automatic Trigger](./Images/12.0%20Testing%20the%20Automatic%20Trigger.png)

---

## Part 3 — Configure Jenkins to Copy Files to the NFS Server via SSH

### 13.0 — Installing the "Publish Over SSH" Plugin

Under **Manage Jenkins → Plugins → Available plugins**, I searched for and installed **Publish Over SSH**, which allows Jenkins to transfer build artifacts to a remote server.

![Installing Publish Over SSH Plugin](./Images/13.0%20Installing%20publish%20over%20SSH%20Plugin.png)

---

### 14.0 — Configuring Jenkins to Connect to the NFS Server via SSH

Under **Manage Jenkins → System**, in the Publish over SSH section, I provided the `.pem` key contents for the NFS server, then added an SSH Server entry:

- **Hostname:** NFS server's private IP
- **Username:** `ec2-user`
- **Remote Directory:** `/mnt/apps`

```bash
cat your-key-name.pem
```
*(run on local machine, not any EC2 instance — copy the full BEGIN/END block)*

![Configuring Jenkins to Connect to NFS via SSH](./Images/14%20Configure%20jenkins%20to%20c...NFS%20Server%20via%20SSH.png)

---

### 14.1 — Testing the SSH Configuration

I clicked **Test Configuration** and confirmed it returned **Success**.

![Testing the SSH Configuration](./Images/14.1%20Test%20configuration.png)

---

### 16.0 — Testing the Full Pipeline

In the job's Post-build Actions, I added **"Send build artifacts over SSH"**, selected the NFS server config, and set Source files to `**`. I then pushed another change to `README.md` on GitHub and confirmed the build completed with `Finished: SUCCESS` in Console Output.

![Testing the Full Pipeline with NFS Server](./Images/16%20Testing%20the%20full%20pipeline%20with%20NFS%20Server.png)

---

### 17.0 — Verifying the Files Landed on the NFS Server

I connected to the NFS server via SSH and confirmed my exact change was reflected in the deployed file:

```bash
cat /mnt/apps/README.md | grep "your exact changes"
```

![Verifying Files Deployed on NFS Server](./Images/17%20%20Verifying%20that%20the%20file...ed%20on%20my%20NFS%20server.png)

---

## Result

Pushing a change to the GitHub Tooling repository now automatically triggers a Jenkins build, which pulls the latest code and deploys it directly to the NFS server's `/mnt/apps` directory — the same shared storage all 3 web servers serve from. The Tooling Website updates itself with zero manual file copying, completing a real, working CI pipeline.

---

## Errors & Fixes

Key errors I ran into during this project and how I resolved them.

---

**`OpenPGP signature verification failed: NO_PUBKEY 7198F4B714ABFC68` → `Package 'jenkins' has no installation candidate`**

The Jenkins signing key at `/etc/apt/keyrings/jenkins-keyring.asc` was missing, empty, or stale — Jenkins rotates its repo signing key periodically, and an older cached key had expired. Fixed by pulling the current year's key directly and verifying its fingerprint:

```bash
sudo rm /etc/apt/keyrings/jenkins-keyring.asc

sudo curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key \
  -o /etc/apt/keyrings/jenkins-keyring.asc
sudo chmod a+r /etc/apt/keyrings/jenkins-keyring.asc

sudo apt update
sudo apt install jenkins
```

Verified with:
```bash
gpg --show-keys /etc/apt/keyrings/jenkins-keyring.asc
```

---

**GitHub webhook — "failed to connect to host"**

The Payload URL had a copy/paste error — a duplicated `http` and a missing colon:
```
http://http//13.51.44.78/:8080/git...
```

Fixed by correcting it to the proper format:
```
http://13.51.44.78:8080/github-webhook/
```

Before finding the real cause, I briefly ruled out a Security Group or firewall block by confirming Jenkins was actually listening on port 8080:
```bash
sudo ss -tlnp | grep 8080
```

---

**Jenkins build stuck on "Waiting for next available executor"**

The Built-In Node had gone offline automatically — Jenkins' disk space monitor flagged `/tmp` as below its default 1 GiB threshold, even though the small `tmpfs` partition wasn't actually a real capacity problem.

**Fix:** Manually brought the node back online via its status page (**"Bring this node back online"**), since it was a false-positive threshold rather than genuine low disk space.

**Also addressed as a precaution** — Free Swap Space showed `0 B`, so I added 1G of swap:
```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

*Project completed by **Adewale Oluwasemiloore** — Brandbody Studio*
