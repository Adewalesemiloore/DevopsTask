<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">AWS CLOUD SOLUTION FOR 2 COMPANY WEBSITES USING A REVERSE PROXY</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>November 26, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Amazon_VPC-8C4FFF?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Amazon_RDS-527FFF?style=for-the-badge&logo=amazonrds&logoColor=white" />
  <img src="https://img.shields.io/badge/Route_53-8C4FFF?style=for-the-badge&logo=amazonroute53&logoColor=white" />
  <img src="https://img.shields.io/badge/NGINX-009639?style=for-the-badge&logo=nginx&logoColor=white" />
  <img src="https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white" />
  <img src="https://img.shields.io/badge/Red_Hat-EE0000?style=for-the-badge&logo=redhat&logoColor=white" />
  <img src="https://img.shields.io/badge/WordPress-21759B?style=for-the-badge&logo=wordpress&logoColor=white" />
  <img src="https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white" />
</p>

---

## Project Overview

Until now I have used AWS mostly as a place to rent virtual machines (EC2, EBS, key pairs and basic security groups). The real power of the cloud goes well beyond that, and in this project I build a complete, secure network inside an **AWS VPC** for a fictitious company that runs two websites:

- a **WordPress** site for its main business, and
- a **Tooling website** for its DevOps team.

For better security and performance, both sites sit behind an **NGINX reverse proxy**. Cost, security and scalability are the key requirements, so the design must survive web server failures, absorb extra traffic, and stay reasonably cheap.

This project is done **manually through the AWS console** on purpose. Before automating infrastructure (which I start doing in Project 16 with Terraform), I need to understand each building block by hand, otherwise the automation becomes frustrating very quickly.

![Architecture](./Images/architecture.png)

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part 1 — AWS Account, Organization and Domain](#part-1--aws-account-organization-and-domain)
- [Part 2 — Set Up the VPC](#part-2--set-up-the-vpc)
- [Part 3 — Security Groups](#part-3--security-groups)
- [Part 4 — TLS Certificates with ACM](#part-4--tls-certificates-with-acm)
- [Part 5 — Configure EFS](#part-5--configure-efs)
- [Part 6 — Configure RDS](#part-6--configure-rds)
- [Part 7 — Target Groups and Load Balancers](#part-7--target-groups-and-load-balancers)
- [Part 8 — Build the AMIs](#part-8--build-the-amis)
- [Part 9 — Launch Templates](#part-9--launch-templates)
- [Part 10 — Auto Scaling Groups](#part-10--auto-scaling-groups)
- [Part 11 — Route 53 Records](#part-11--route-53-records)
- [Part 12 — Test the Websites](#part-12--test-the-websites)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

| Component | Purpose |
|---|---|
| Route 53 hosted zone | DNS for my registered domain |
| Internet Gateway + NAT Gateway | Public ingress and private subnet egress |
| Public ALB | Internet-facing entry point (HTTPS with an ACM certificate) |
| Nginx servers (ASG) | Reverse proxy in the public subnets |
| Bastion servers (ASG) | Single, restricted SSH entry point |
| Internal ALB | Receives traffic from Nginx and routes by host header |
| Webserver ASGs | WordPress and Tooling servers in private subnets |
| EFS | Shared file storage with one access point per website |
| RDS (Multi-AZ MySQL) | Highly available database in the data layer |
| KMS key | Encrypts the database storage |

Traffic flow: **Client → Route 53 → public ALB → Nginx → internal ALB → WordPress / Tooling webservers → RDS / EFS.**

---

## Part 1 — AWS Account, Organization and Domain

### 1.1 — Root account, sub-account and Organization Unit

I created an AWS root (master) account, then, inside it, a sub-account named **DevOps**.

![Creating the DevOps sub-account (1 of 5)](./Images/Task%2015.1.png)
![Creating the DevOps sub-account (2 of 5)](./Images/Task%2015.2.png)
![Creating the DevOps sub-account (3 of 5)](./Images/Task%2015.3.png)
![Creating the DevOps sub-account (4 of 5)](./Images/Task%2015.4.png)
![Creating the DevOps sub-account (5 of 5)](./Images/Task%2015.5.png)

Next I created an **Organization Unit (OU)** named **Dev** to hold development resources, and moved the DevOps account into it.

![Creating the Dev OU (1 of 2)](./Images/Task%2015.6.png)
![Creating the Dev OU (2 of 2)](./Images/Task%2015.7.png)
![Moving the DevOps account into the Dev OU (1 of 3)](./Images/Task%2015.8.png)
![Moving the DevOps account into the Dev OU (2 of 3)](./Images/Task%2015.9.png)
![Moving the DevOps account into the Dev OU (3 of 3)](./Images/Task%2015.10.png)
![DevOps account inside the Dev OU](./Images/Task%2015.11.png)

### 1.2 — Domain name and hosted zone

I registered an inexpensive domain at Namecheap, then created a **Route 53 hosted zone** for it and pointed the domain's name servers at the hosted zone.

![Creating the hosted zone and mapping the domain (1 of 4)](./Images/Task%2015.12.png)
![Creating the hosted zone and mapping the domain (2 of 4)](./Images/Task%2015.13.png)
![Creating the hosted zone and mapping the domain (3 of 4)](./Images/Task%2015.14.png)
![Creating the hosted zone and mapping the domain (4 of 4)](./Images/Task%2015.15.png)

---

## Part 2 — Set Up the VPC

### 2.1 — Create the VPC

![Creating the VPC (1 of 4)](./Images/Task%2015.16.png)
![Creating the VPC (2 of 4)](./Images/Task%2015.17.png)
![Creating the VPC (3 of 4)](./Images/Task%2015.18.png)
![Creating the VPC (4 of 4)](./Images/Task%2015.19.png)

### 2.2 — Enable DNS hostnames

![Enabling DNS hostnames (1 of 2)](./Images/Task%2015.20.png)
![Enabling DNS hostnames (2 of 2)](./Images/Task%2015.21.png)

### 2.3 — Internet Gateway

I created an internet gateway and attached it to the VPC.

![Creating and attaching the Internet Gateway (1 of 5)](./Images/Task%2015.22.png)
![Creating and attaching the Internet Gateway (2 of 5)](./Images/Task%2015.23.png)
![Creating and attaching the Internet Gateway (3 of 5)](./Images/Task%2015.24.png)
![Creating and attaching the Internet Gateway (4 of 5)](./Images/Task%2015.25.png)
![Creating and attaching the Internet Gateway (5 of 5)](./Images/Task%2015.26.png)

### 2.4 — Subnets

Following the architecture, I created the subnets across two Availability Zones: public subnets, private subnets for the webservers, and private subnets for the data layer. A CIDR calculator helped me split the VPC range cleanly.

![Creating the subnets (1 of 6)](./Images/Task%2015.27.png)
![Creating the subnets (2 of 6)](./Images/Task%2015.28.png)
![Creating the subnets (3 of 6)](./Images/Task%2015.29.png)
![Creating the subnets (4 of 6)](./Images/Task%2015.30.png)
![Creating the subnets (5 of 6)](./Images/Task%2015.31.png)
![Creating the subnets (6 of 6)](./Images/Task%2015.32.png)
![Using a CIDR calculator to plan subnet ranges (1 of 3)](./Images/Task%2015.33.png)
![Using a CIDR calculator to plan subnet ranges (2 of 3)](./Images/Task%2015.34.png)
![Using a CIDR calculator to plan subnet ranges (3 of 3)](./Images/Task%2015.35.png)

### 2.5 — Route tables

I created a public and a private route table and associated them with the matching subnets.

![Creating and associating route tables (1 of 6)](./Images/Task%2015.36.png)
![Creating and associating route tables (2 of 6)](./Images/Task%2015.37.png)
![Creating and associating route tables (3 of 6)](./Images/Task%2015.38.png)
![Creating and associating route tables (4 of 6)](./Images/TAsk%2015.39.png)
![Creating and associating route tables (5 of 6)](./Images/Task%2015.40.png)
![Creating and associating route tables (6 of 6)](./Images/Task%2015.41.png)

I then edited the public route table to send `0.0.0.0/0` to the Internet Gateway. This is what makes a public subnet reachable from the internet.

![Adding the Internet Gateway route to the public route table (1 of 2)](./Images/Task%2015.42.png)
![Adding the Internet Gateway route to the public route table (2 of 2)](./Images/Task%2015.43.png)

### 2.6 — Elastic IPs

I allocated 3 Elastic IPs: one for the NAT Gateway and two for the bastion hosts.

![Allocating Elastic IPs (1 of 4)](./Images/Task%2015.44.png)
![Allocating Elastic IPs (2 of 4)](./Images/Task%2015.45.png)
![Allocating Elastic IPs (3 of 4)](./Images/Task%2015.46.png)
![Allocating Elastic IPs (4 of 4)](./Images/Task%2015.47.png)

### 2.7 — NAT Gateway

I created the NAT Gateway in a public subnet and assigned one of the Elastic IPs to it, so private instances can reach the internet outbound.

![Creating the NAT Gateway (1 of 4)](./Images/Task%2015.48.png)
![Creating the NAT Gateway (2 of 4)](./Images/Task%2015.49.png)
![Creating the NAT Gateway (3 of 4)](./Images/Task%2015.50.png)
![Creating the NAT Gateway (4 of 4)](./Images/Task%2015.51.png)

---

## Part 3 — Security Groups

I created a separate security group for each tier, each allowing only the traffic that tier needs.

![Creating security groups](./Images/Task%2015.52.png)

- **Application Load Balancer (public):** open to the internet.

![Public ALB security group (1 of 2)](./Images/Task%2015.53.png)
![Public ALB security group (2 of 2)](./Images/Task%2015.54.png)

- **Bastion servers:** SSH allowed only from my workstation's public IP.

![Bastion security group (1 of 2)](./Images/Task%2015.55.png)
![Bastion security group (2 of 2)](./Images/Task%2015.56.png)

- **Nginx servers:** access only from the public ALB.

![Nginx security group (1 of 3)](./Images/Task%2015.57.png)
![Nginx security group (2 of 3)](./Images/Task%2015.58.png)
![Nginx security group (3 of 3)](./Images/Task%2015.59.png)

- **Internal ALB:** access only from the Nginx servers.

![Internal ALB security group](./Images/Task%2015.60.png)

- **Webservers:** access only from the internal ALB.

![Webserver security group](./Images/Task%2015.61.png)

- **Data layer (RDS and EFS):** only webservers may connect to RDS, while both Nginx and webservers may reach the EFS mount points.

![Data layer security group](./Images/Task%2015.62.png)

All the security groups together:

![All security groups](./Images/Task%2015.63.png)

---

## Part 4 — TLS Certificates with ACM

The load balancers need TLS certificates for secure connectivity. In **AWS Certificate Manager (ACM)** I requested a public **wildcard certificate** for my domain.

![Opening ACM](./Images/Task%2015.64.png)
![Requesting a wildcard certificate (1 of 3)](./Images/Task%2015.65.png)
![Requesting a wildcard certificate (2 of 3)](./Images/Task%2015.66.png)
![Requesting a wildcard certificate (3 of 3)](./Images/Task%2015.67.png)

I validated the domain with **DNS validation**, creating the CNAME record in Route 53.

![DNS validation of the certificate (1 of 6)](./Images/Task%2015.68.png)
![DNS validation of the certificate (2 of 6)](./Images/Task%2015.69.png)
![DNS validation of the certificate (3 of 6)](./Images/Task%2015.70.png)
![DNS validation of the certificate (4 of 6)](./Images/Task%2015.71.png)
![DNS validation of the certificate (5 of 6)](./Images/Task%2015.72.png)
![DNS validation of the certificate (6 of 6)](./Images/Task%2015.73.png)

---

## Part 5 — Configure EFS

I created an EFS file system with one mount target per AZ, placed in the two data-layer subnets and using the data-layer security group.

![Creating the EFS file system and mount targets (1 of 8)](./Images/Task%2015.74.png)
![Creating the EFS file system and mount targets (2 of 8)](./Images/Task%2015.75.png)
![Creating the EFS file system and mount targets (3 of 8)](./Images/Task%2015.76.png)
![Creating the EFS file system and mount targets (4 of 8)](./Images/Task%2015.77.png)
![Creating the EFS file system and mount targets (5 of 8)](./Images/Task%2015.78.png)
![Creating the EFS file system and mount targets (6 of 8)](./Images/Task%2015.79.png)
![Creating the EFS file system and mount targets (7 of 8)](./Images/Task%2015.80.png)
![Creating the EFS file system and mount targets (8 of 8)](./Images/Task%2015.81.png)

I then created **two access points**, one for WordPress and one for Tooling, so the two websites' files never overwrite each other when mounted.

![Creating the EFS access points (1 of 6)](./Images/Task%2015.82.png)
![Creating the EFS access points (2 of 6)](./Images/Task%2015.83.png)
![Creating the EFS access points (3 of 6)](./Images/Task%2015.84.png)
![Creating the EFS access points (4 of 6)](./Images/Task%2015.85.png)
![Creating the EFS access points (5 of 6)](./Images/Task%2015.86.png)
![Creating the EFS access points (6 of 6)](./Images/Task%2015.87.png)

---

## Part 6 — Configure RDS

### 6.1 — Create a KMS key

A KMS key is needed to encrypt the database.

![Creating the KMS key (1 of 7)](./Images/Task%2015.88.png)
![Creating the KMS key (2 of 7)](./Images/Task%2015.89.png)
![Creating the KMS key (3 of 7)](./Images/Task%2015.90.png)
![Creating the KMS key (4 of 7)](./Images/Task%2015.91.png)
![Creating the KMS key (5 of 7)](./Images/Task%2015.92.png)
![Creating the KMS key (6 of 7)](./Images/Task%2015.93.png)
![Creating the KMS key (7 of 7)](./Images/Task%2015.94.png)

For high availability and failover I used a **Multi-AZ RDS MySQL** deployment. With only 2 AZs I can fail over to one standby, but the concept is the same with 3.

### 6.2 — Create the DB subnet group

I created a subnet group containing the two private data-layer subnets.

![Creating the DB subnet group (1 of 5)](./Images/Task%2015.95.png)
![Creating the DB subnet group (2 of 5)](./Images/Task%2015.96.png)
![Creating the DB subnet group (3 of 5)](./Images/Task%2015.97.png)
![Creating the DB subnet group (4 of 5)](./Images/Task%2015.98.png)
![Creating the DB subnet group (5 of 5)](./Images/Task%2015.99.png)

### 6.3 — Create the database

![Creating the RDS database (1 of 2)](./Images/Task%2015.100.png)
![Creating the RDS database (2 of 2)](./Images/Task%2015.101.png)
![Choosing Multi-AZ DB instance deployment](./Images/Task%2015.102.png)
![Completing the RDS configuration (1 of 5)](./Images/Task%2015.103.png)
![Completing the RDS configuration (2 of 5)](./Images/Task%2015.104.png)
![Completing the RDS configuration (3 of 5)](./Images/Task%2015.105.png)
![Completing the RDS configuration (4 of 5)](./Images/Task%2015.106.png)
![Completing the RDS configuration (5 of 5)](./Images/Task%2015.107.png)

---

## Part 7 — Target Groups and Load Balancers

### 7.1 — Target groups

I created target groups for Nginx, Tooling and WordPress. No instances are registered yet because they have not been launched. The Auto Scaling Groups will register them later.

![Creating the target groups (1 of 6)](./Images/Task%2015.108.png)
![Creating the target groups (2 of 6)](./Images/Task%2015.109.png)
![Creating the target groups (3 of 6)](./Images/Task%2015.110.png)
![Creating the target groups (4 of 6)](./Images/Task%2015.111.png)
![Creating the target groups (5 of 6)](./Images/Task%2015.112.png)
![Creating the target groups (6 of 6)](./Images/Task%2015.113.png)

### 7.2 — Public-facing and internal load balancers

First the public-facing ALB, with HTTPS using the ACM certificate and Nginx as the default target.

![Creating the public Application Load Balancer (1 of 8)](./Images/Task%2015.114.png)
![Creating the public Application Load Balancer (2 of 8)](./Images/Task%2015.115.png)
![Creating the public Application Load Balancer (3 of 8)](./Images/Task%2015.116.png)
![Creating the public Application Load Balancer (4 of 8)](./Images/Task%2015.117.png)
![Creating the public Application Load Balancer (5 of 8)](./Images/Task%2015.118.png)
![Creating the public Application Load Balancer (6 of 8)](./Images/Task%2015.119.png)
![Creating the public Application Load Balancer (7 of 8)](./Images/Task%2015.120.png)
![Creating the public Application Load Balancer (8 of 8)](./Images/Task%2015.121.png)

Then the internal ALB, with WordPress as the default target.

![Creating the internal Application Load Balancer](./Images/Task%2015.122.png)

I added a listener rule that forwards traffic to the Tooling target group when the host header matches the tooling hostname.

![Host-header rule on the internal ALB (1 of 8)](./Images/Task%2015.123.png)
![Host-header rule on the internal ALB (2 of 8)](./Images/Task%2015.124.png)
![Host-header rule on the internal ALB (3 of 8)](./Images/Task%2015.125.png)
![Host-header rule on the internal ALB (4 of 8)](./Images/Task%2015.126.png)
![Host-header rule on the internal ALB (5 of 8)](./Images/Task%2015.127.png)
![Host-header rule on the internal ALB (6 of 8)](./Images/Task%2015.128.png)
![Host-header rule on the internal ALB (7 of 8)](./Images/Task%2015.129.png)
![Host-header rule on the internal ALB (8 of 8)](./Images/Task%2015.130.png)

---

## Part 8 — Build the AMIs

### 8.1 — Launch 4 RHEL 8 instances

I launched four RHEL 8 instances to become the Bastion, Nginx, WordPress and Tooling templates.

![Launching the RHEL 8 instances](./Images/Task%2015.131.png)

### 8.2 — Install base packages

Run on the bastion, Nginx, WordPress and Tooling servers:

```bash
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum install -y wget vim python3 telnet htop git mysql net-tools chrony
sudo systemctl start chronyd
sudo systemctl enable chronyd
```

### 8.3 — SELinux policies

For the webservers (Tooling, WordPress) and Nginx servers:

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo setsebool -P httpd_can_network_connect_db 1
sudo setsebool -P httpd_execmem 1
sudo setsebool -P httpd_use_nfs 1
```

### 8.4 — Install Amazon EFS utils

```bash
sudo yum install -y git
git clone https://github.com/aws/efs-utils
cd efs-utils
sudo yum install -y make
sudo yum install -y go cmake rust cargo openssl-devel make rpm-build
make rpm
sudo yum install -y ./build/amazon-efs-utils*rpm
```

### 8.5 — Self-signed certificate for the Nginx instance

```bash
sudo mkdir -p /etc/ssl/private
sudo chmod 700 /etc/ssl/private
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/private/<company>.key \
-out /etc/ssl/certs/<company>.crt
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

![Nginx self-signed certificate](./Images/Task%2015.132.png)

### 8.6 — Self-signed certificate for the Apache (Tooling) instance

```bash
sudo openssl req -newkey rsa:2048 -nodes \
-keyout /etc/pki/tls/private/<company>.key \
-x509 -days 365 \
-out /etc/pki/tls/certs/<company>.crt
sudo yum install -y httpd
sudo systemctl restart httpd
sudo systemctl enable httpd
sudo systemctl status httpd
sudo yum reinstall -y mod_ssl
sudo vim /etc/httpd/conf.d/ssl.conf
```

In `ssl.conf` I pointed the certificate and key directives at the files created above.

![Apache SSL configuration](./Images/Task%2015.322.png)

### 8.7 — Create AMIs from the instances

![Creating AMIs from the configured instances (1 of 3)](./Images/Task%2015.133.png)
![Creating AMIs from the configured instances (2 of 3)](./Images/Task%2015.134.png)
![Creating AMIs from the configured instances (3 of 3)](./Images/Task%2015.135.png)

---

## Part 9 — Launch Templates

From the custom AMIs I created a launch template for each instance type.

![Creating the launch templates (1 of 6)](./Images/Task%2015.136.png)
![Creating the launch templates (2 of 6)](./Images/Task%2015.137.png)
![Creating the launch templates (3 of 6)](./Images/Task%2015.138.png)
![Creating the launch templates (4 of 6)](./Images/Task%2015.139.png)
![Creating the launch templates (5 of 6)](./Images/Task%2015.140.png)
![Creating the launch templates (6 of 6)](./Images/Task%2015.141.png)

The user data for each template comes from my configuration repo and is edited with my own details. For the database name, I connected from an instance to the RDS endpoint and created databases of my choice.

![Launch template user data](./Images/Task%2015.144.png)

I tested connectivity to the RDS endpoint on port 3306, then created the databases:

```bash
nc -zv <rds-endpoint> 3306
mysql -h <rds-endpoint> -u admin -p
```

```sql
CREATE DATABASE toolingdb;
CREATE DATABASE wordpressdb;
SHOW DATABASES;
```

![Testing RDS connectivity with netcat](./Images/Task%2015.145.png)
![Creating the application databases](./Images/Task%2015.146.png)

With those credentials in hand, I finished the launch template user data:

![Completing launch template user data (1 of 2)](./Images/Task%2015.142.png)
![Completing launch template user data (2 of 2)](./Images/Task%2015.143.png)

---

## Part 10 — Auto Scaling Groups

### 10.1 — Bastion

![Bastion Auto Scaling Group (1 of 6)](./Images/Task%2015.147.png)
![Bastion Auto Scaling Group (2 of 6)](./Images/Task%2015.148.png)
![Bastion Auto Scaling Group (3 of 6)](./Images/Task%2015.149.png)
![Bastion Auto Scaling Group (4 of 6)](./Images/Task%2015.150.png)
![Bastion Auto Scaling Group (5 of 6)](./Images/Task%2015.151.png)
![Bastion Auto Scaling Group (6 of 6)](./Images/Task%2015.152.png)

### 10.2 — Nginx

![Nginx Auto Scaling Group (1 of 5)](./Images/Task%2015.153.png)
![Nginx Auto Scaling Group (2 of 5)](./Images/Task%2015.154.png)
![Nginx Auto Scaling Group (3 of 5)](./Images/Task%2015.155.png)
![Nginx Auto Scaling Group (4 of 5)](./Images/Task%2015.156.png)
![Nginx Auto Scaling Group (5 of 5)](./Images/Task%2015.157.png)

### 10.3 — WordPress and Tooling

I repeated the same process for WordPress and Tooling, selecting their respective target groups.

![WordPress and Tooling Auto Scaling Groups](./Images/Task%2015.158.png)

---

## Part 11 — Route 53 Records

I added alias records for `tooling.<your-domain>` and `wordpress.<your-domain>` pointing at the internet-facing load balancer.

![Creating the Route 53 alias records (1 of 5)](./Images/Task%2015.159.png)
![Creating the Route 53 alias records (2 of 5)](./Images/Task%2015.160.png)
![Creating the Route 53 alias records (3 of 5)](./Images/Task%2015.161.png)
![Creating the Route 53 alias records (4 of 5)](./Images/Task%2015.162.png)
![Creating the Route 53 alias records (5 of 5)](./Images/Task%2015.163.png)

> **Note:** While creating the records I pointed them at the public ALB instead of the internal ALB where I had already added the Tooling host-header rule. Rather than recreate the records, I chose to continue and learn from it: I created fresh target groups for the Tooling and WordPress pages on the public ALB.

![Creating new target groups for the public ALB (1 of 4)](./Images/Task%2015.164.png)
![Creating new target groups for the public ALB (2 of 4)](./Images/Task%2015.165.png)
![Creating new target groups for the public ALB (3 of 4)](./Images/Task%2015.166.png)
![Creating new target groups for the public ALB (4 of 4)](./Images/Task%2015.167.png)

Then I added host-header rules to the public ALB so each hostname is forwarded to the right target group.

![Adding host-header rules to the public ALB (1 of 3)](./Images/Task%2015.168.png)
![Adding host-header rules to the public ALB (2 of 3)](./Images/Task%2015.169.png)
![Adding host-header rules to the public ALB (3 of 3)](./Images/Task%2015.170.png)

---

## Part 12 — Test the Websites

Opening the Tooling site at first returned an error (see [Errors & Fixes](#errors--fixes) below for the full troubleshooting).

![Initial test of the websites (1 of 2)](./Images/Task%2015.171.png)
![Initial test of the websites (2 of 2)](./Images/Task%2015.172.png)

After the fixes, the site root answered through the load balancer (it still shows the default RHEL test page, since the tooling app has no index page at the root), and the application's login page loaded at `/login.php`:

```
https://tooling.<your-domain>
https://tooling.<your-domain>/login.php
```

![Site root responding through the load balancer](./Images/Task%2015.180.png)
![Tooling login page](./Images/Task%2015.181.png)

---

## Errors & Fixes

### 504 Gateway Time-out on the Tooling and WordPress sites

**Symptom:** After the records and rules were in place, both `tooling.<your-domain>` and `wordpress.<your-domain>` returned **504 Gateway Time-out**. In the target group, the `tooling-tg` target showed as **Unhealthy**.

![Unhealthy target in the tooling target group](./Images/Task%2015.173.png)

**Root cause:** Two problems stacked on each other. The webserver security group did not allow HTTP from the ALB security group, so health checks could not reach the server. Once that was fixed, Apache was running but answering `403 Forbidden` on `/` and `404` on `/login.php`, because the document root did not contain the tooling application (only a placeholder `index.html`, and the app files were not at the web root).

**Fix:**

1. Add an **HTTP** inbound rule to the webserver security group with the **ALB security group as source**.

![Adding HTTP from the ALB security group](./Images/Task%2015.174.png)

2. SSH into the Tooling server and confirm Apache is active:

```bash
sudo systemctl status httpd
```

![Apache running on the Tooling server](./Images/Task%2015.175.png)

3. Check the response from the server itself. It returned 403 on `/` and 404 on `/login.php`:

```bash
curl -I http://localhost
curl -I http://localhost/login.php
```

![403 and 404 from curl](./Images/Task%2015.176.png)

4. Remove the placeholder `index.html`:

```bash
cd /var/www/html
ls -la
sudo rm -f /var/www/html/index.html
```

![Removing the placeholder index file](./Images/Task%2015.177.png)

5. Clone the tooling repository into the web root and restart Apache:

```bash
cd /var/www/html
sudo git clone <tooling-repo-url> .
sudo systemctl restart httpd
```

![Cloning the tooling repo](./Images/Task%2015.178.png)

If the clone nests the code inside an extra `html` folder, move its contents up and restart Apache:

```bash
sudo cp -r /var/www/html/html/* /var/www/html/
sudo cp -r /var/www/html/html/.[!.]* /var/www/html/ 2>/dev/null || true
sudo rm -rf /var/www/html/html
sudo systemctl restart httpd
curl -I http://localhost/
curl -I http://localhost/login.php
```

![HTTP 200 OK from login.php](./Images/Task%2015.179.png)

Once `login.php` returned `HTTP/1.1 200 OK`, I reloaded the Tooling site through the load balancer and the login page loaded.

### Records pointed at the wrong load balancer

**Symptom:** The Route 53 records targeted the public ALB while the Tooling host-header rule lived on the internal ALB.

**Root cause:** A selection mistake when creating the alias records.

**Fix:** Created new Tooling and WordPress target groups on the public ALB and added host-header rules there (Part 11).

---

*Project completed by **Adewale Oluwasemiloore***
