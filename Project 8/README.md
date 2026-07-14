<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">LOAD BALANCER SOLUTION WITH APACHE</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>July 14, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white" />
  <img src="https://img.shields.io/badge/Red_Hat-EE0000?style=for-the-badge&logo=redhat&logoColor=white" />
  <img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" />
</p>

---

## Project Overview

In Project 7, the Tooling Website was served by 3 separate web servers, each with its own IP address — meaning a user would need to know and choose between 3 different URLs to access the same website. This project solves that problem by introducing an **Apache Load Balancer** on a dedicated Ubuntu EC2 instance, sitting in front of the 3 web servers.

The load balancer gives users a single point of access with one public IP address, while quietly distributing incoming requests across all 3 web servers behind the scenes. This is called **horizontal scaling** — instead of making one server more powerful (vertical scaling), the load is spread across multiple servers, which is the same principle large-scale platforms like Google use to serve millions of users. If one web server goes down, the load balancer simply routes traffic to the remaining healthy servers, and users never notice the difference.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Step 1 — Launch the Load Balancer Instance](#step-1--launch-the-load-balancer-instance)
- [Step 2 — Install Apache and Required Modules](#step-2--install-apache-and-required-modules)
- [Step 3 — Configure Load Balancing](#step-3--configure-load-balancing)
- [Step 4 — Verify the Load Balancer Works](#step-4--verify-the-load-balancer-works)
- [Step 5 — Unmount Shared Logs](#step-5--unmount-shared-logs)
- [Step 6 — Watch Traffic Distribution Live](#step-6--watch-traffic-distribution-live)
- [Step 7 — Local DNS Name Resolution (Optional)](#step-7--local-dns-name-resolution-optional)
- [Result](#result)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

This solution builds directly on top of Project 7's infrastructure by adding **one new instance**:

| Server | OS | Role |
|---|---|---|
| Load Balancer | Ubuntu 20.04 | Apache configured with `mod_proxy_balancer`, distributing traffic to all 3 web servers |
| Web Server 1, 2, 3 | RedHat | Same as Project 7 — Apache + PHP, serving the Tooling Website |
| Database Server | Ubuntu 20.04 | Same MySQL server from Project 7 |
| NFS Server | RedHat | Same shared storage from Project 7 |

Users now access the website through **one single public IP** — the load balancer's — instead of needing to know and choose between 3 separate web server addresses.

---

## Prerequisites

Before starting, all of Project 7's infrastructure must be up and running:

- 3 RHEL8 Web Servers
- 1 MySQL DB Server (Ubuntu 20.04)
- 1 RHEL8 NFS Server

I confirmed all of these were healthy and running before starting this project.

![Prerequisites Check](./Images/1.0%20%20Project%208%20Load%20balancer%20Instance.png)

---

## Step 1 — Launch the Load Balancer Instance

I launched a new **Ubuntu 20.04 EC2** instance and named it `Project-8-apache-lb`. I then opened **TCP port 80** in its Security Group's Inbound Rules so it could accept incoming web traffic, and SSHed in — this became my dedicated Load Balancer terminal for the rest of the project.

---

## Step 2 — Install Apache and Required Modules

**→ Ran on the Load Balancer terminal**

```bash
sudo apt update
sudo apt install apache2 -y
sudo apt install libxml2-dev -y
```

![Installing Apache](./Images/2.1%20Installing%20Apache.png)

I then enabled the modules Apache needs to act as a load balancer:

```bash
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic
```

![Enabling the Modules](./Images/2.2%20Enabling%20the%20modules%20for%20the%20load%20balancer.png)

Restarted Apache to apply the new modules and confirmed it was running:

```bash
sudo systemctl restart apache2
sudo systemctl status apache2
```

![Restarting Apache](./Images/2.3%20Restarting%20%20Apache%20to%20apply%20modules.png)

---

## Step 3 — Configure Load Balancing

**→ Ran on the Load Balancer terminal**

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```

Inside the `<VirtualHost *:80>` ... `</VirtualHost>` block, I added:

```
<Proxy "balancer://mycluster">
    BalancerMember http://172.31.39.169:80 loadfactor=5 timeout=1
    BalancerMember http://172.31.43.142:80 loadfactor=5 timeout=1
    BalancerMember http://172.31.19.104:80 loadfactor=5 timeout=1
    ProxySet lbmethod=bytraffic
    # ProxySet lbmethod=byrequests
</Proxy>

ProxyPreserveHost On
ProxyPass / balancer://mycluster/
ProxyPassReverse / balancer://mycluster/
```

> Replace the IPs above with your own 3 web servers' private IPs, found in AWS Console → EC2 → each instance → Private IPv4 address.

Saved and restarted Apache:

```bash
sudo systemctl restart apache2
```

![Configuring the Load Balancer](./Images/3.0%20Configuring%20the%20load%20balancer.png)

---

## Step 4 — Verify the Load Balancer Works

I visited the load balancer's public IP in my browser:

```
http://<Load-Balancer-Public-IP>/index.php
```

The Tooling Website loaded successfully — confirming requests were being correctly forwarded to the web servers behind the load balancer.

![Load Balancer Serving the Site](./Images/4.0%20display.png)

---

## Step 5 — Unmount Shared Logs

In Project 7, I mounted `/var/log/httpd/` on each web server to the NFS server's `/mnt/logs`, meaning all 3 servers were writing to one shared log file. To properly verify that the load balancer distributes traffic across independent servers, each web server needs its **own local log directory**.

**→ Repeated on Web Server 1, Web Server 2, and Web Server 3's terminals**

```bash
sudo vi /etc/fstab
```

Removed the line referencing `/var/log/httpd`, keeping the `/var/www` (apps) mount intact. Then unmounted and recreated the log directory locally:

```bash
sudo umount /var/log/httpd
sudo mkdir -p /var/log/httpd
sudo chown -R apache:apache /var/log/httpd
sudo restorecon -Rv /var/log/httpd
sudo systemctl restart httpd
```

![Unmounting Shared Logs](./Images/5.0%20Unmounting%20Shared%20Logs.png)

---

## Step 6 — Watch Traffic Distribution Live

**→ Opened 3 separate terminal tabs — one per web server**

On each web server, I ran:

```bash
sudo tail -f /var/log/httpd/access_log
```

Then refreshed the load balancer's URL several times in my browser:

```
http://<Load-Balancer-Public-IP>/index.php
```

New log entries appeared across all 3 web servers roughly evenly, since `loadfactor` was set equally (5) for each — confirming the load balancer distributes traffic correctly and users are being served by more than one server without noticing.

![Watching the Access Logs](./Images/6.0%20watching%20the%20access%20logs.png)

---

## Step 7 — Local DNS Name Resolution (Optional)

To make the load balancer's config easier to read, I set up local name resolution using `/etc/hosts`, so the config file uses friendly names instead of raw IP addresses.

**→ Ran on the Load Balancer terminal**

```bash
sudo vi /etc/hosts
```

Added:
```
<WebServer1-Private-IP> Web1
<WebServer2-Private-IP> Web2
<WebServer3-Private-IP> Web3
```

Updated the load balancer config to use these names:

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```

```
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
BalancerMember http://Web3:80 loadfactor=5 timeout=1
```

Restarted Apache and tested the resolution locally:

```bash
sudo systemctl restart apache2
curl http://Web1
curl http://Web2
curl http://Web3
```

Each returned the Tooling Website's HTML correctly.

> **Note:** these names only resolve locally on the load balancer instance — they are not accessible from other servers or the internet.

---

## Result

Users can now access the Tooling Website through a single URL — the load balancer's public IP — without needing to know or choose between the 3 individual web servers behind it. Traffic is distributed evenly across all 3 servers, and the setup tolerates individual server failures gracefully since the load balancer simply routes around any server that becomes unavailable.

---

## Errors & Fixes

Key errors I ran into during this project and how I resolved them.

---

**HTTP 500 error when accessing the site through the Load Balancer**

The load balancer's error log showed `Connection refused` when trying to reach the web servers — all 3 had been stopped and restarted, and Apache wasn't running on any of them.

```bash
sudo systemctl status httpd
sudo systemctl start httpd
```

---

**`PHP Fatal error: Uncaught mysqli_sql_exception: Permission denied` in `functions.php`**

Even though the database credentials were correct and worked fine via the MySQL CLI, PHP itself couldn't connect. This was an SELinux restriction blocking Apache/PHP-FPM from making outbound network connections to the database — a setting that had reset after the servers were stopped and restarted.

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo setsebool -P httpd_can_network_connect_db 1
sudo systemctl restart php-fpm
sudo systemctl restart httpd
```

---

**Unable to SSH into a web server — terminal hangs with no response**

The instance had been stopped in AWS and had no active public IP to connect to. Confirmed the instance state in the AWS Console, started it, and waited for it to fully boot before retrying SSH.

---

**`curl` from the Load Balancer returned nothing at all**

This indicated a stalled connection rather than a clean error. Running `curl -v` (verbose mode) revealed the actual HTTP status code and response, which pointed to the real underlying issue (the 500 error and PHP misconfiguration above) instead of a vague blank response.

```bash
curl -v http://<Web-Server-Private-IP>
```

---

Project completed by **Adewale Oluwasemiloore** 
