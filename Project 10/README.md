<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">LOAD BALANCER SOLUTION WITH NGINX AND SSL/TLS</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>July 25, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/NGINX-009639?style=for-the-badge&logo=nginx&logoColor=white" />
  <img src="https://img.shields.io/badge/Let's_Encrypt-003A70?style=for-the-badge&logo=letsencrypt&logoColor=white" />
  <img src="https://img.shields.io/badge/SSL%2FTLS-informational?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" />
</p>

---

## Project Overview

In Project 8, I built a load balancer using Apache to distribute traffic across 3 web servers. A good DevOps engineer needs to be comfortable with more than one tool for the same job — so this project rebuilds that same load balancing solution using **Nginx** instead, on a completely fresh Ubuntu instance.

Beyond just switching load balancers, this project tackles a more important problem: **securing the connection itself**. By default, traffic between a browser and a web server travels as plain, readable text — anyone with access to the network in between (a coffee shop Wi-Fi router, an ISP, etc.) can potentially intercept it. This is known as a **Man-In-The-Middle (MITM) attack**, and it's a real risk for anything involving passwords, personal data, or sensitive information.

The fix is **SSL/TLS** — the technology behind HTTPS — which encrypts the connection between browser and server so intercepted traffic is unreadable garbage to anyone without the right key. To get a valid SSL certificate, a website needs a **registered domain name** recognized by a Certificate Authority. In this project, I registered a free subdomain through DuckDNS, pointed it to my server using an AWS Elastic IP, and used **Certbot** (Let's Encrypt's official client) to issue and auto-renew a free SSL certificate — giving the Tooling Website a proper padlock-secured HTTPS connection.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part 1 — Configure Nginx as a Load Balancer](#part-1--configure-nginx-as-a-load-balancer)
- [Part 2 — Register a Domain and Secure the Connection](#part-2--register-a-domain-and-secure-the-connection)
- [Result](#result)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

This project reuses the 3 web servers, DB server, and NFS server from Projects 7 and 8, and replaces the Apache load balancer with a new Nginx-based one:

| Server | OS | Role |
|---|---|---|
| Nginx LB | Ubuntu 20.04 | Load balancer, HTTPS termination, SSL certificate |
| Web Server 1, 2, 3 | RedHat | Same as Project 7/8 — Apache + PHP, Tooling Website |
| Database Server | Ubuntu 20.04 | Same MySQL server from Project 7 |
| NFS Server | RedHat | Same shared storage from Project 7 |

Traffic now flows: **Browser → HTTPS (443) → Nginx LB → HTTP (80) → one of the 3 web servers**, with the encryption layer stopping at the load balancer.

---

## Part 1 — Configure Nginx as a Load Balancer

### 1.0 — Launching the Nginx LB Instance

I launched a new **Ubuntu 20.04 EC2** instance and named it `Nginx LB`. I opened both **TCP port 80** (HTTP) and **TCP port 443** (HTTPS) in its Security Group's Inbound Rules.

---

### 2.0 — Setting Up Local DNS Names for the Web Servers

I updated `/etc/hosts` with friendly names pointing to each of my 3 web servers' private IPs, so the Nginx config could reference them by name instead of raw IPs.

```bash
sudo vi /etc/hosts
```

```
<WebServer1-Private-IP> Web1
<WebServer2-Private-IP> Web2
<WebServer3-Private-IP> Web3
```

![Set Up Local DNS Names](./Images/2.0%20Set%20Up%20Local%20DNS%20Names%20for%20Your%20Web%20Servers.png)

I confirmed all 3 web servers from Project 7 were attached and reachable before continuing.

![Attached Web Servers from Project 7](./Images/2.1%20Attached%20my%203%20web%20servers%20from%20project%207.png)

---

### 3.0 — Installing Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

![Installing Nginx](./Images/3.0%20Installing%20Nginx.png)

---

### 4.0 — Configuring Nginx as a Load Balancer

```bash
sudo vi /etc/nginx/nginx.conf
```

Inside the `http { }` block, I added:

```
upstream myproject {
    server Web1 weight=5;
    server Web2 weight=5;
    server Web3 weight=5;
}

server {
    listen 80;
    server_name www.domain.com;
    location / {
        proxy_pass http://myproject;
    }
}
```

![Configuring Nginx as a Load Balancer](./Images/4.0%20Configuring%20Nginx%20as%20a%20load%20balancer.png)

![Nginx Upstream Block in Config](./Images/4.1%20I%20configured%20the%20Nginx%20upstream%20block%20in%20config.png)

Restarted Nginx and confirmed it was active:

```bash
sudo systemctl restart nginx
sudo systemctl status nginx
```

![Nginx Active and Running](./Images/4.2%20Restarted%20nginx%20and%20confirmed%20the%20status%20is%20active%20and%20running.png)

---

## Part 2 — Register a Domain and Secure the Connection

### 5.0 — Creating a Free Domain with DuckDNS

Since a real domain registrar requires payment, I used **DuckDNS** to get a free subdomain — `adewalesemiloore.duckdns.org` — as a zero-cost alternative that still supports full SSL certificate issuance.

![Created a Domain Using DuckDNS](./Images/5.0%20I%20created%20a%20domain%20using%20Duck%20DNS.png)

---

### 6.0 — Allocating an Elastic IP

Because EC2 public IPs change on every stop/start, I allocated an **Elastic IP** and associated it with the Nginx LB instance, giving it a permanent address for DNS to point to.

![Created an Elastic IP](./Images/6.0%20created%20an%20Elastic%20IP.png)

---

### 7.0 — Pointing the Domain to the Elastic IP

I updated the IP field on DuckDNS to point my domain to the new Elastic IP.

![Updated Elastic IP in DuckDNS](./Images/7.0%20Updated%20my%20elastic%20IP%20i%20this%20field.png)

I confirmed the DNS was resolving correctly:

```bash
nslookup adewalesemiloore.duckdns.org
```

![Confirmed DNS Was Visible](./Images/7.1%20Cofirmed%20that%20my%20DNS%20was%20visible.png)

---

### 8.0 — Updating Nginx with the Real Domain Name

```bash
sudo vi /etc/nginx/nginx.conf
```

Updated `server_name www.domain.com;` to my actual domain:

```
server_name adewalesemiloore.duckdns.org;
```

```bash
sudo systemctl restart nginx
```

![Updated Nginx Config with Real DNS](./Images/8.0%20Updated%20Nginx%20config%20to%20have%20my%20real%20DNS.png)

---

### 9.0 — Confirming Snapd and Installing Certbot

```bash
sudo systemctl status snapd
```

![Confirming Snapd is Running](./Images/9.0%20Confirming%20snapd%20is%20running.png)

```bash
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

![Installing Certbot](./Images/9.1%20Installing%20Certbot.png)

---

### 10.0 — Requesting the SSL/TLS Certificate

```bash
sudo certbot --nginx
```

Certbot picked up my domain automatically from `nginx.conf`, and after confirming my email and agreeing to the terms, it issued and installed the certificate directly into my Nginx configuration.

![Requesting SSL/TLS Certificate](./Images/10.0%20Requesting%20my%20SSL:TLS%20Certificate.png)

---

### 11.0 — Confirming the Certificate

I visited `https://adewalesemiloore.duckdns.org` and confirmed the padlock icon appeared, then clicked it to verify the certificate details — issued by Let's Encrypt.

![Confirmed the Certificate](./Images/11.0%20Confirmed%20the%20certificate.png)

---

### 12.0 — Testing Certificate Auto-Renewal

Since Let's Encrypt certificates expire every 90 days, I tested the renewal process in dry-run mode to confirm it would work automatically before it's actually needed:

```bash
sudo certbot renew --dry-run
```

![Tested Certificate Auto-Renewal](./Images/12.0%20Tested%20Certificate%20Auto-Renewal.png)

I then scheduled a cronjob to run the renewal check twice daily:

```bash
sudo crontab -e
```

```
0 */12 * * *   root /usr/bin/certbot renew > /dev/null 2>&1
```

This ensures the certificate renews itself automatically well before it ever expires, with no manual intervention needed.

---

## Result

The Tooling Website is now served through Nginx acting as a load balancer, secured with a free, auto-renewing SSL/TLS certificate from Let's Encrypt — accessible over HTTPS at a custom domain name rather than a raw IP address.

---

## Errors & Fixes

Key errors I ran into during this project and how I resolved them.

---

**`OpenPGP signature verification failed` / `Package 'jenkins' has no installation candidate`**

*(Carried over from setting up related infrastructure alongside this project.)* The GPG key downloaded via `wget -O` saved in a format the repo couldn't verify. Fixed by piping the key directly into the keyring file instead:

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /etc/apt/keyrings/jenkins-keyring.asc > /dev/null
```

---

**Domain showed Nginx's default welcome page instead of the Tooling Website**

Two issues stacked together: `server_name` was set with a full URL (`http://domain/`) instead of just the domain, and Nginx's default site config was still enabled, overriding my custom config.

```bash
sudo vi /etc/nginx/nginx.conf   # fixed server_name to just the domain
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

---

**`HTTP 500 Internal Server Error` when accessing the site**

The same SELinux restriction from Project 8 resurfaced after the web servers were stopped and restarted, blocking PHP from reaching the database.

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo setsebool -P httpd_can_network_connect_db 1
sudo systemctl restart php-fpm
sudo systemctl restart httpd
```

---

**Site briefly showed the default RedHat Apache test page**

The `/var/www` NFS mount had dropped on the web server, so requests weren't reaching the actual application files. Remounted it:

```bash
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP>:/mnt/apps /var/www
```

---

**Forgot the MySQL `webaccess` password**

Reset it directly from the DB server and updated the matching value in each web server's `functions.php`:

```sql
ALTER USER 'webaccess'@'172.31.%.%' IDENTIFIED BY 'NewPassword1!';
FLUSH PRIVILEGES;
```

---

*Project completed by **Adewale Oluwasemiloore**