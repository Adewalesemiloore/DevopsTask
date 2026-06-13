<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">WEB SOLUTION WITH WORDPRESS ON AWS</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>June 13, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Red_Hat-EE0000?style=for-the-badge&logo=redhat&logoColor=white" />
  <img src="https://img.shields.io/badge/WordPress-21759B?style=for-the-badge&logo=wordpress&logoColor=white" />
  <img src="https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white" />
  <img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" />
</p>

---

## Project Overview

In this project, I configured storage infrastructure on two RedHat EC2 instances in AWS and deployed a full three-tier web solution using WordPress as the CMS and MySQL as the backend database. One server hosts WordPress via Apache, and the other runs a dedicated MySQL database server. Both servers use LVM (Logical Volume Manager) to manage storage across three EBS volumes each — giving real-world experience with disk partitioning, volume groups, and logical volumes on Linux.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part 1 — Web Server Storage Setup](#part-1--web-server-storage-setup)
- [Part 2 — Database Server Storage Setup](#part-2--database-server-storage-setup)
- [Part 3 — Installing WordPress](#part-3--installing-wordpress)
- [Part 4 — Installing MySQL on the DB Server](#part-4--installing-mysql-on-the-db-server)
- [Part 5 — Configuring the Database](#part-5--configuring-the-database)
- [Part 6 — Connecting WordPress to the Database](#part-6--connecting-wordpress-to-the-database)
- [Result](#result)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

This project follows a **Three-Tier Architecture**:

| Layer | Role | Server |
|---|---|---|
| Presentation | Client / Browser | Your local machine |
| Business | Web Server + WordPress | EC2 Instance 1 (RedHat) |
| Data | Database Server | EC2 Instance 2 (RedHat) |

Both EC2 instances use **LVM (Logical Volume Manager)** to manage storage across 3 EBS volumes each, giving flexibility to resize storage without downtime.

---

## Part 1 — Web Server Storage Setup

### 1.0 — Launching the EC2 Instance & Creating Volumes

I started by launching a **RedHat EC2** instance (`t3.micro`) to serve as the web server, then created **3 EBS volumes of 10 GiB each** in the same Availability Zone as the instance and attached them one by one.

![Creating EC2 Instance](./Images/1.0%20Creating%20EC2%20instance.png)

![Creating Volumes](./Images/1.0a%20Creating%20Volumes.png)

![Attaching Volume 1](./Images/1.1b%20attaching%20volume%20to%20instance.png)

![Attaching Volume 2](./Images/1.1c%20attaching%20volume%20to%20instance.png)

![Attaching Volume 3](./Images/1.1d%20attaching%20volume%20to%20instance.png)

---

### 1.2 — Connecting & Inspecting Disks

After SSHing into the instance using `ec2-user`, I verified the attached volumes:

```bash
lsblk
ls /dev/ | grep nvme
df -h
```

> **Note:** Because this is a `t3.micro` instance, AWS uses NVMe storage. The disks show up as `nvme1n1`, `nvme2n1`, `nvme3n1` — not `xvdb/xvdc/xvdd` as older guides may suggest.

![Connecting to EC2](./Images/1.2%20%20Connecting%20to%20the%20ec2%20instance.png)

![Checking Disks](./Images/1.2a%20checking%20disk.png)

![Checking Disk Space](./Images/1.2b%20used%20disk%20space.png)

---

### 1.3 — Partitioning the Disks

I used `fdisk` to create a single partition on each of the 3 disks:

```bash
sudo fdisk /dev/nvme1n1
sudo fdisk /dev/nvme2n1
sudo fdisk /dev/nvme3n1
```

Inside each `fdisk` prompt: `n` → `p` → `1` → `Enter` → `Enter` → `w`

Then verified with `lsblk` — partitions `nvme1n1p1`, `nvme2n1p1`, `nvme3n1p1` were created.

![Partitioning Disk 1](./Images/1.3%20Partitioning%20disk%201.png)

![Partitioning Disk 2](./Images/1.3b%20Partitioning%20disk%202.png)

![Partitioning Disk 3](./Images/1.3c%20Partitioning%20disk%203.png)

![Viewing Partitions](./Images/1.3dview%20the%20newly%20configured%20partition.png)

---

### 1.4 — Setting Up LVM

`lvm2` was already installed. I ran a disk scan, then set up Physical Volumes, a Volume Group, and two Logical Volumes:

```bash
sudo lvmdiskscan

sudo pvcreate /dev/nvme1n1p1
sudo pvcreate /dev/nvme2n1p1
sudo pvcreate /dev/nvme3n1p1

sudo pvs

sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

sudo vgs

sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg

sudo lvs
sudo vgdisplay -v
sudo lsblk
```

![LVM Disk Scan](./Images/1.4%20lvm%20disk%20scan.png)

![Creating Physical Volumes](./Images/1.4a%20Create%20Physical%20Volumes.png)

![Verifying Physical Volumes](./Images/1.4b%20Physical%20Volumes.png)

![Creating Logical Volumes](./Images/1.4c%20Creating%20logical%20volumes.png)

![Verifying with VGDisplay](./Images/1.4d%20Verifying%20everything%20VGdisplay.png)

![Verifying with lsblk](./Images/1.4e%20Verifying%20everything%20Sudo%20lsblk.png)

---

### 1.5 — Formatting, Mounting & Backing Up Logs

```bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv

sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs

sudo mount /dev/webdata-vg/apps-lv /var/www/html/

# Back up logs before mounting over /var/log
sudo rsync -av /var/log/. /home/recovery/logs/

sudo mount /dev/webdata-vg/logs-lv /var/log

# Restore logs
sudo rsync -av /home/recovery/logs/. /var/log
```

![Formatting Logical Volumes](./Images/1.5%20Format%20both%20logical%20volumes%20as%20ext4.png)

![Creating Directories](./Images/1.5a%20Creating%20directories%20to%20backing%20up.png)

![Mounting and Restoring Logs](./Images/1.5b%20Mouting%20and%20restoring%20logs.png)

---

### 1.6 — Making Mounts Persistent

I retrieved the UUIDs with `sudo blkid` and updated `/etc/fstab`.

> **Replace the UUIDs below with your own.** Run `sudo blkid` and copy the UUID values for `/dev/mapper/webdata--vg-apps--lv` and `/dev/mapper/webdata--vg-logs--lv`.

```bash
sudo vi /etc/fstab
```

```
UUID=<your-apps-lv-uuid>  /var/www/html  ext4  defaults  0 0
UUID=<your-logs-lv-uuid>  /var/log       ext4  defaults  0 0
```

Then tested and reloaded:

```bash
sudo mount -a
sudo systemctl daemon-reload
df -h
```

![Updating fstab](./Images/1.6%20Updating%20etc:fstab.png)

![Testing Configuration](./Images/1.6a%20Test%20the%20configuration%20and%20reload%20the%20daemon.png)

![Verifying with df -h](./Images/1.6b%20Verify%20your%20setup%20by%20running%20df%20-h.png)

---

## Part 2 — Database Server Storage Setup

I launched a **second RedHat EC2 instance** and repeated the same disk setup — with two differences: only one logical volume (`db-lv`) is created, and it mounts to `/db` instead of `/var/www/html`.

```bash
sudo lvcreate -n db-lv -L 20G webdata-vg
sudo mkfs -t ext4 /dev/webdata-vg/db-lv
sudo mkdir -p /db
sudo mount /dev/webdata-vg/db-lv /db
```

> **Replace the UUID below with your own.** Run `sudo blkid` and copy the UUID for `/dev/mapper/webdata--vg-db--lv`.

```bash
sudo vi /etc/fstab
```

```
UUID=<your-db-lv-uuid>  /db  ext4  defaults  0 0
```

```bash
sudo mount -a
sudo systemctl daemon-reload
df -h
```

![New DB EC2 Instance](./Images/2.0%20New%20Ec2%20instance.png)

---

## Part 3 — Installing WordPress

Back on the **web server**, I installed Apache, PHP 8.2, and WordPress.

### 3.1 — Apache & PHP

```bash
sudo yum -y update
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json

sudo systemctl enable httpd
sudo systemctl start httpd

sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo yum install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
sudo yum module reset php -y
sudo yum module enable php:remi-8.2 -y
sudo yum install -y php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1

sudo systemctl restart httpd
```

![Updating and Installing Apache](./Images/3.0%20Updating%20and%20installing%20apache.png)

![Apache and PHP Install](./Images/3.1a%20Update%20and%20Install%20Apache%20+%20PHP.png)

![Starting Apache](./Images/3.1b%20Start%20and%20enable%20Apache.png)

![Installing PHP 8.2](./Images/3.1c%20Install%20PHP%208.2%20via%20Remi%20repo.png)

---

### 3.2 — Downloading WordPress

```bash
mkdir wordpress && cd wordpress
sudo wget https://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
sudo cp -R wordpress /var/www/html/
```

![Downloading WordPress](./Images/3.2%20Download%20and%20Set%20Up%20WordPress.png)

---

### 3.3 — Configuring SELinux

```bash
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1
sudo setsebool -P httpd_can_network_connect_db 1
```

![SELinux Configuration](./Images/3.3%20Configure%20SELinux%20for%20WordPress.png)

---

## Part 4 — Installing MySQL on the DB Server

On the **DB server**, `mysql-server` isn't available in the default RHEL 9 repos, so I installed from MySQL's official community repo:

```bash
sudo yum install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm
sudo yum install -y mysql-community-server --nogpgcheck
sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo systemctl status mysqld
```

![Installing MySQL](./Images/4.0%20Install%20MySQL%20(DB%20Server).png)

![MySQL Running](./Images/4.0a%20SQL%20installed%2C%20active%20ad%20running.png)

---

## Part 5 — Configuring the Database

MySQL 8 generates a temporary root password on first install. I retrieved and reset it, then created the WordPress database and user.

```bash
sudo grep 'temporary password' /var/log/mysqld.log
sudo mysql -u root -p
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourNewRootPassword1!';

CREATE DATABASE wordpress;
CREATE USER 'myuser'@'<web-server-private-ip>' IDENTIFIED BY 'YourPassword1!';
GRANT ALL ON wordpress.* TO 'myuser'@'<web-server-private-ip>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```

> **Replace `<web-server-private-ip>`** with your web server's private IP address. You can find it in the AWS EC2 console under instance details.

> **Password policy:** MySQL 8 enforces password complexity. Your password must include uppercase, lowercase, a number, and a special character.

I also updated `/etc/my.cnf` to allow remote connections:

```bash
sudo vi /etc/my.cnf
```

Add at the end:

```
[mysqld]
bind-address=0.0.0.0
```

```bash
sudo systemctl restart mysqld
```

Then opened **port 3306** in the DB server's Security Group inbound rules, with source set to your web server's private IP with `/32`.

![Database Configuration](./Images/5.0%20Configure%20the%20Database.png)

---

## Part 6 — Connecting WordPress to the Database

Back on the **web server**, I installed the MySQL client and tested the connection:

```bash
sudo yum install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm
sudo yum install -y mysql-community-client --nogpgcheck
```

```bash
sudo mysql -u myuser -p -h <db-server-private-ip>
```

> **Replace `<db-server-private-ip>`** with your DB server's private IP address.

Then edited the WordPress config file:

```bash
sudo vi /var/www/html/wordpress/wp-config.php
```

Updated the database credentials:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'myuser' );
define( 'DB_PASSWORD', 'YourPassword1!' );
define( 'DB_HOST', '<db-server-private-ip>' );
```

Disabled Apache's default welcome page and restarted:

```bash
sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf_backup
sudo systemctl restart httpd
```

Opened **port 80** on the web server's Security Group inbound rules: `0.0.0.0/0`.

Then visited:

```
http://<web-server-public-ip>/wordpress/
```

> **Replace `<web-server-public-ip>`** with your web server's public IP from the AWS EC2 console.

![Installing MySQL Client on Web Server](./Images/6.0%20Installing%20sql%20on%20web%20server.png)

![Editing WordPress Config](./Images/6.0a%20Edit%20WordPress%20config.png)

![Testing Database Connection](./Images/6.0a%20Testing%20database%20connection.png)

---

## Result

WordPress successfully connected to the remote MySQL database and the setup page loaded correctly.

🎥 **Watch the project walkthrough:**

<p align="center">
  <video ![Result](./Images/Final%20result.mov) controls style="max-width: 100%; height: auto;">
  </video>

---

## Errors & Fixes

Key errors I ran into during this project and how I resolved them.

---

**`gdisk: command not found`**

`gdisk` isn't installed by default on RedHat.

```bash
sudo yum install -y gdisk
```

`fdisk` works just as well for disks under 2TB, which is what this project uses.

---

**`sudo mysql` → `Access denied for user 'root'@'localhost'`**

MySQL 8 no longer allows passwordless root login. A temporary password is generated on install.

```bash
sudo grep 'temporary password' /var/log/mysqld.log
sudo mysql -u root -p
ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourNewPassword1!';
```

---

**`No match for argument: mysql-server`**

`mysql-server` isn't in the default RHEL 9 repos. Install from MySQL's official community repo:

```bash
sudo yum install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm
sudo yum install -y mysql-community-server --nogpgcheck
```

---

**`Error: GPG check FAILED`**

MySQL's GPG key check failed during install. Fixed by adding `--nogpgcheck`:

```bash
sudo yum install -y mysql-community-server --nogpgcheck
```

---

**`Error establishing a database connection` (WordPress)**

Caused by a typo in `wp-config.php`. Always verify credentials after editing:

```bash
sudo cat /var/www/html/wordpress/wp-config.php | grep DB_
```

Make sure the password matches exactly what was set in MySQL — including special characters.

---

*Project completed by **Adewale Oluwasemiloore** — Brandbody Studio*