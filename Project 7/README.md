<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">DEVOPS TOOLING WEBSITE SOLUTION</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>July 11, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Red_Hat-EE0000?style=for-the-badge&logo=redhat&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/NFS-004E89?style=for-the-badge&logo=nfs&logoColor=white" />
  <img src="https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white" />
  <img src="https://img.shields.io/badge/PHP-777BB4?style=for-the-badge&logo=php&logoColor=white" />
  <img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" />
</p>

---

## Project Overview

In this project, I implemented a DevOps Tooling Website Solution — a shared web infrastructure that a DevOps team can use for day-to-day tools. The setup consists of an NFS server providing shared storage, a dedicated MySQL database server, and three stateless RedHat web servers all serving the same content and sharing the same database. This architecture demonstrates how multiple web servers can be added or removed without losing data integrity, since all shared state lives on the NFS server and the database — not on individual web servers.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Step 1 — Prepare the NFS Server](#step-1--prepare-the-nfs-server)
- [Step 2 — Configure the Database Server](#step-2--configure-the-database-server)
- [Step 3 — Prepare the Web Servers](#step-3--prepare-the-web-servers)
- [Result](#result)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

This solution uses **5 EC2 instances**:

| Server | OS | Role |
|---|---|---|
| NFS Server | RedHat | Shared storage for `/mnt/apps`, `/mnt/logs`, `/mnt/opt` |
| Database Server | Ubuntu 20.04 | MySQL — single shared `tooling` database |
| Web Server 1, 2, 3 | RedHat | Apache + PHP, all mounting the same NFS shares and connecting to the same database |

Because all three web servers read/write to the same NFS-mounted storage and the same MySQL database, they are **stateless** — any one of them can be taken down or replaced without any data loss.

---

## Step 1 — Prepare the NFS Server

### 1.1 — Inspecting the Disks

I launched a **RedHat EC2** instance for the NFS server and created **3 EBS volumes of 10 GiB each** in the same AZ, then attached them. After SSHing in, I inspected the attached disks.

```bash
lsblk
ls /dev/ | grep nvme
df -h
```

![Inspect Disks](./Images/1.1%20Inspect%20desks.png)

---

### 1.2 — Partitioning the Disks

I used `fdisk` to create a single partition on each of the 3 disks.

```bash
sudo fdisk /dev/nvme1n1
sudo fdisk /dev/nvme2n1
sudo fdisk /dev/nvme3n1
```

Inside each prompt: `n` → `p` → `1` → `Enter` → `Enter` → `w`

```bash
lsblk
```

![Disks Partitions](./Images/1.2%20Disks%20partitions.png)

---

### 1.3 — Setting Up LVM

I marked each partition as a Physical Volume, grouped them into a Volume Group, and created 3 Logical Volumes for apps, logs, and opt.

```bash
sudo pvcreate /dev/nvme1n1p1
sudo pvcreate /dev/nvme2n1p1
sudo pvcreate /dev/nvme3n1p1

sudo pvs

sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

sudo vgs

sudo lvcreate -n lv-apps -L 9G webdata-vg
sudo lvcreate -n lv-logs -L 9G webdata-vg
sudo lvcreate -n lv-opt  -L 9G webdata-vg

sudo lvs
```

![Setting Up LVM](./Images/1.3%20Setting%20up%20LVM.png)

---

### 1.4 — Formatting as XFS

Unlike Project 6 which used ext4, this project formats the logical volumes as **XFS** — better suited for NFS shared storage.

```bash
sudo mkfs -t xfs /dev/webdata-vg/lv-apps
sudo mkfs -t xfs /dev/webdata-vg/lv-logs
sudo mkfs -t xfs /dev/webdata-vg/lv-opt
```

![XFS Formatting](./Images/1.4%20XFS%20Formatting.png)

---

### 1.5 — Creating Mount Points and Mounting

```bash
sudo mkdir -p /mnt/apps
sudo mkdir -p /mnt/logs
sudo mkdir -p /mnt/opt

sudo mount /dev/webdata-vg/lv-apps /mnt/apps
sudo mount /dev/webdata-vg/lv-logs /mnt/logs
sudo mount /dev/webdata-vg/lv-opt  /mnt/opt

df -h
```

![Create Mount Points and Mount](./Images/1.5%20Create%20Mount%20Points%20and%20Mount%20.png)

---

### 1.6 — Making Mounts Persistent

I retrieved the UUIDs and updated `/etc/fstab`.

> **Replace the UUIDs below with your own.** Run `sudo blkid` and copy the UUID values for `lv-apps`, `lv-logs`, and `lv-opt`.

```bash
sudo blkid
sudo vi /etc/fstab
```

```
UUID=<your-lv-apps-uuid>  /mnt/apps  xfs  defaults  0 0
UUID=<your-lv-logs-uuid>  /mnt/logs  xfs  defaults  0 0
UUID=<your-lv-opt-uuid>   /mnt/opt   xfs  defaults  0 0
```

```bash
sudo mount -a
sudo systemctl daemon-reload
df -h
```

![Make Mounts Persistent](./Images/1.6%20Make%20Mounts%20Persistent.png)

---

### 1.7 — Installing and Starting the NFS Server

```bash
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
sudo systemctl status nfs-server
```

![Installing and Starting NFS Server](./Images/1.7%20Installing%20and%20startig%20NFS%20Servers.png)

---

### 1.8 — Setting Permissions and Exporting NFS Shares

I set ownership and permissions so web servers can read, write, and execute files on NFS, then configured exports for the VPC subnet and opened the required ports.

```bash
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server
```

To find the subnet CIDR: AWS Console → EC2 → NFS instance → Networking tab → open the Subnet link → copy the IPv4 CIDR.

```bash
sudo vi /etc/exports
```

```
/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt  <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
```

```bash
sudo exportfs -arv
```

Checked which port NFS uses, then opened it via Security Groups:

```bash
rpcinfo -p | grep nfs
```

**Required inbound rules on the NFS server's Security Group:**

| Port | Protocol | Source |
|---|---|---|
| 2049 | TCP | Subnet CIDR |
| 111 | TCP | Subnet CIDR |
| 111 | UDP | Subnet CIDR |

![Setting Permissions](./Images/1.8%20Setting%20permissions.png)

---

## Step 2 — Configure the Database Server

I launched an **Ubuntu 20.04 EC2** instance for the database server and SSHed in using the `ubuntu` user.

```bash
sudo apt update
sudo apt install mysql-server -y
sudo systemctl start mysql
sudo systemctl enable mysql
```

```bash
sudo mysql
```

```sql
CREATE DATABASE tooling;
CREATE USER 'webaccess'@'<Subnet-CIDR>' IDENTIFIED BY 'YourPassword1!';
GRANT ALL ON tooling.* TO 'webaccess'@'<Subnet-CIDR>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```

I allowed remote connections by updating the MySQL config:

```bash
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```

Changed `bind-address` from `127.0.0.1` to `0.0.0.0`, then restarted:

```bash
sudo systemctl restart mysql
```

Opened **port 3306** on the DB server's Security Group with source set to the Subnet CIDR.

![Configuring the Database Server](./Images/2.0%20Configuring%20the%20Database%20Server.png)

---

## Step 3 — Prepare the Web Servers

> Steps 3.1 through 3.9 were repeated identically across all 3 web servers, connecting to the same NFS server and the same MySQL database.

### 3.1 — Installing the NFS Client

```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```

![Install NFS Client](./Images/3.1%20S1%20-%20Install%20NFS%20Client.png)

---

### 3.2 — Mounting the NFS Share

```bash
sudo mkdir -p /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP>:/mnt/apps /var/www
df -h
```

![Mount NFS Share](./Images/3.2%20S1%20-%20Mount%20NFS%20Share.png)

---

### 3.3 — Making the Mount Persistent

I updated `/etc/fstab` so the NFS mount would persist after a reboot.

```bash
sudo vi /etc/fstab
```

```
<NFS-Server-Private-IP>:/mnt/apps /var/www nfs defaults 0 0
```

---

### 3.4 — Installing Apache and PHP 8.2

```bash
sudo yum install httpd -y

sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo yum install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
sudo yum module reset php -y
sudo yum module enable php:remi-8.2 -y
sudo yum install -y php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1
```

![Install Apache and PHP 8.2](./Images/3.4%20Install%20Apache%20and%20PHP%208.2.png)

---

### 3.5 — Mounting Apache Logs to NFS

I mounted the Apache logs directory to the NFS server's logs export, then made the change permanent in fstab. Since NFS mounts don't support SELinux file contexts, I used a boolean instead to allow Apache to write logs over NFS.

```bash
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP>:/mnt/logs /var/log/httpd
```

```bash
sudo vi /etc/fstab
```

```
<NFS-Server-Private-IP>:/mnt/logs /var/log/httpd nfs defaults 0 0
```

```bash
sudo setsebool -P httpd_use_nfs 1
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```

![Mounting Apache Logs](./Images/3.5%20Mounting%20Apache%20logs.png)

---

### 3.6 — Verifying NFS is Working

I created a test file on the web server and confirmed it appeared on the NFS server, proving the shared storage was working correctly.

```bash
sudo touch /var/www/test.txt
```

![Verify NFS is Working](./Images/3.6a%20Verify%20NFS%20is%20Working.png)

Checked on the NFS server:

```bash
ls /mnt/apps
```

![Check on the NFS Server](./Images/3.6b%20Check%20on%20the%20NFS%20server.png)

---

### 3.7 — Forking and Cloning the Tooling App

I forked the tooling source code to my own GitHub account, then cloned it onto the web server and deployed it to the web root.

![Forking from Darey-io](./Images/3.7%20Forking%20from%20Darey-io.png)

```bash
sudo yum install git -y
git clone https://github.com/Adewalesemiloore/tooling.git
sudo cp -R tooling/html/. /var/www/html/
```

Opened **port 80** on the web server's Security Group → Inbound → HTTP → `0.0.0.0/0`.

If a 403 error appears, disable SELinux:

```bash
sudo setenforce 0
```

To make it permanent:

```bash
sudo vi /etc/sysconfig/selinux
```

Set `SELINUX=disabled`, then restart Apache:

```bash
sudo systemctl restart httpd
```

![Fork and Deploy the Tooling App](./Images/3.7a%20Fork%20and%20Deploy%20the%20Tooling%20App.png)

---

### 3.8 — Connecting the App to the Database

I updated the database connection details inside the app's `functions.php` file, then applied the database schema from the tooling repo — only on the first web server, since all three share the same database.

```bash
sudo vi /var/www/html/functions.php
```

```php
$db = mysqli_connect('<DB-Server-Private-IP>', 'webaccess', 'YourPassword1!', 'tooling');
```

Installed the MySQL client to apply the schema:

```bash
sudo yum install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm
sudo yum install -y mysql-community-client --nogpgcheck

mysql -h <DB-Server-Private-IP> -u webaccess -p tooling < tooling/tooling-db.sql
```

> The `tooling-db.sql` script automatically creates the admin user — no need to create it manually.

![Configured functions.php](./Images/3.8%20Cofigured%20the%20functions-php.png)

---

### 3.9 — Final Check

I visited each web server's public IP in the browser and confirmed the tooling dashboard loaded and logged in successfully.

```
http://<Web-Server-Public-IP>/index.php
```

**Web Server 1:**

![Final Check - Server 1](./Images/3.9%20Final%20check.png)

**Web Server 2:**

![Final Check - Server 2](./Images/3.9b%20Server%202%20Final%20check.png)

**Web Server 3:**

![Final Check - Server 3](./Images/3.9c%20Server%203%20Final%20check.png)

---

## Result

All three web servers successfully mounted the shared NFS storage, connected to the same MySQL database, and served the DevOps Tooling Website independently — confirming the stateless, three-tier architecture works as intended.

🎥 **Watch the project walkthrough:**

<p align="center">
  <video src="" controls style="max-width: 100%; height: auto;">
  </video>
</p>

---

## Errors & Fixes

Key errors I ran into during this project and how I resolved them.

---

**Partitioned the wrong disk (`nvme0n1`) — `pvcreate` failed with "device is too small"**

`nvme0n1` is the root/OS disk. The 3 attached EBS volumes were `nvme1n1`, `nvme2n1`, `nvme3n1`. Always confirm with `lsblk` before partitioning, and never touch the root disk.

---

**`mount: can't find UUID` after adding it to fstab**

The volume was already mounted, so the UUID lookup didn't resolve the same way. Used the device path instead:

```
/dev/webdata-vg/lv-apps  /mnt/apps  xfs  defaults  0 0
```

---

**`mount.nfs: access denied by server`**

The web server's IP fell outside the exported subnet CIDR. Widened the export and Security Group rules to cover the full VPC range:

```bash
sudo vi /etc/exports
```
```
/mnt/apps 172.31.0.0/16(rw,sync,no_all_squash,no_root_squash)
```
```bash
sudo exportfs -arv
```

---

**`httpd.service: Failed with result 'exit-code'` after mounting logs to NFS**

Apache couldn't write to its log file because SELinux blocks NFS-mounted directories by default, and `chcon` doesn't work on NFS mounts. Fixed with a boolean instead of a file context:

```bash
sudo setsebool -P httpd_use_nfs 1
```

---

**`git clone` — "Invalid username or token. Password authentication is not supported"**

GitHub no longer accepts account passwords for git operations. Since the repo was public, no authentication was needed at all — the issue was actually that the fork hadn't completed yet on GitHub's side. Confirmed the fork existed by visiting the repo URL directly before retrying the clone.

---

**`mysql: command not found` on the web server**

The MySQL client wasn't installed. Installed it from MySQL's official repo before running any `mysql -h` commands:

```bash
sudo yum install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm
sudo yum install -y mysql-community-client --nogpgcheck
```

---

**MySQL user host set with literal `< >` brackets, and CIDR notation not supported**

Copied a placeholder directly into the `CREATE USER` command including the angle brackets, and MySQL doesn't support CIDR ranges for user hosts in the first place. Fixed by dropping the broken user and recreating it with a wildcard host pattern:

```sql
DROP USER 'webaccess'@'<172.31.32.0/20>';
CREATE USER 'webaccess'@'172.31.%.%' IDENTIFIED BY 'YourPassword1!';
GRANT ALL ON tooling.* TO 'webaccess'@'172.31.%.%';
FLUSH PRIVILEGES;
```

---

*Project completed by **Adewale Oluwasemiloore** — Brandbody Studio*
