<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center"> MEAN STACK DEPLOYMENT TO UBUNTU IN AWS</h3> 
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd>  <b style="color: #fb6900">Date:</b> <kbd>May 24, 2026</kbd</p>
</div>


<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white" />
  <img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" />
</p>



## Project Overview

In this project, I was tasked to set up a Client-Server architecture using two Linux servers in AWS, with MySQL as the database management system. One server acts as the MySQL Server (it holds and manages the data) and the other acts as the MySQL Client (it connects to the server and makes requests).

<B> An example being, The "POS Agent"</B> 

The Client (Your App/Laptop): This is the customer standing at the shop counter. They don't have the cash (data) themselves; they have to ask for it. When they say, "I want to withdraw ₦10,000," they are sending a request.

The Server (Your Linux Server): This is the POS Agent. They act as the middleman. They receive the customer's request, check the network, talk to the bank's system, and decide if the transaction is valid.

The Database (MySQL): This is the Bank Vault/Ledger. This is where the actual money (data) is stored securely. The POS agent doesn't keep all the cash in their pocket—that would be dangerous and disorganized. Instead, they "query" the vault to verify if the account has enough balance.

![Client server model](./Images/0.1%20Client%20server%20Architecture.jpeg)


A quick way to see this in action is by running this command in a terminal:

```bash
curl -Iv www.bing.com
```


My terminal becomes the client and `www.bing.com` is the server. The response shows that the request is being handled by a computer at IP `23.12.147.174` on port 80.


![Client server model](./Images/0%20-%20Verified%20Connectivity.png)

## Step 1: Create Two EC2 Instances in AWS

I created two Linux-based virtual servers in AWS:

- `mysql-server` — this is where the database lives
- `mysql-client` — this is what connects to the database remotely


![Creating EC2 instannces](./Images/0.0%20-%20Creating%202%20EC2%20instance.png)


---

## Step 2: Set Up MySQL Server

I SSH'd into the `mysql-server` instance and ran the following:

```bash
sudo apt update
sudo apt install mysql-server
```

![Sudo Apt update](./Images/1.0%20-%20sudo%20apt%20update.png)

---

![Innstalling mysql-server](./Images/1.1%20-%20sudo%20apt%20install%20mysql-server.png)

Then I made sure the MySQL service was running:

```bash
sudo systemctl start mysql.service
sudo systemctl status mysql.service
```

![SQL Status](./Images/1.2%20-%20mysql%20status.png)

---

### 2.1 Secure the Root Account

I logged into MySQL and set a password for the root user:

```bash
sudo mysql
```

For the password I'll be using 'PassWord'

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'PassWord';
```

<blockquote style="border-left: 5px solid #d9534f;">
  <strong style="color: #d9534f;">⚠️ Error:</strong> 
  However i encounted a error as a result of the new sql updates, which doesnt allow the states syntax for defaults passwords.  
</blockquote>

<br> 

![SQL error](./Images/1.4%20-%20SQL%20error.png)

![SQL password](./Images/1.4a%20-%20SQL%20passwords.png)

Then I ran the secure installation script to lock things down:

```bash
sudo mysql_secure_installation
```

---

### Create a Database and User

I logged back in as root:

```bash
sudo mysql -p
```

Then I created a new database:

```sql
CREATE DATABASE example_database;
```

Next, I created a new user and gave them full access to that database:

```sql
CREATE USER 'example_user'@'%' IDENTIFIED WITH mysql_native_password BY 'PassWord';
GRANT ALL ON example_database.* TO 'example_user'@'%';
```

> The `'%'` means this user can connect from any IP address — which is what allows the client server to reach the database remotely.

![SQL Priviledges](./Images/1.4b%20-%20SQL%20priviledges.png)

I exited the MySQL shell and tested that the new user worked:

```bash
mysql -u example_user -p
```

Once logged in, I confirmed the database was visible:

```sql
SHOW DATABASES;
```

![SQL Database](./Images/1.5%20-%20Example%20user-Database.png)


### Allow Remote Connections

By default, MySQL only accepts connections from the same machine (`127.0.0.1`). I changed that so the client server could connect remotely:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

I changed the `bind-address` value from `127.0.0.1` to `0.0.0.0`.


![SQL Database](./Images/1.6%20-%20Allowing%20remote%20connections.png)

Then I restarted MySQL to apply the change:

```bash
sudo systemctl restart mysql
sudo systemctl status mysql.service
```

![MySQL Restart](./Images/1.7%20-%20Restarting%20my%20SQL.png)

### Open Port 3306 in AWS

MySQL uses TCP port 3306 by default. I added an inbound rule in the `mysql-server` security group to allow traffic on that port — but only from the **private IP of the mysql-client** instance, not from the whole internet. This keeps things more secure.

![Inbound Rules](./Images/1.8%20-%20Editing%20security%20group.png)

<br>

## Step 3: Setting Up MySQL Client

I SSH'd into the `mysql-client` instance and installed the MySQL client package. 

```bash
sudo apt update && sudo apt upgrade
sudo apt install mysql-client -y
```

![Remote connection](./Images/2.0%20Connecting%20remotely%20on%20client.png)

From the `mysql-client` instance, I connected to the `mysql-server` database using its private IP address:

```bash
sudo mysql -u example_user -h <mysql-server-private-ip> -p
```

Once connected, I ran:

```sql
SHOW DATABASES;
```

The database I created on the server showed up — meaning the two machines were successfully talking to each other over the network.

![Inbound Rules](./Images/2.1%20Client%20VIsible.png)


![Inbound Rules](./Images/2.2%20Client%20and%20Server.png)


## What I Learned

Through this project, I learned how real-world systems split responsibilities across multiple servers rather than running everything on one machine. I now understand how to store data on a dedicated MySQL server while a separate client machine requests that data over a network. I learned how to restrict access using private IP addresses, ensuring only authorized machines can connect to the database server. Most importantly, I understand that the client never touches the data directly, it simply asks the server for what it needs, and the server responds. This mirrors exactly how companies protect and manage their data in production environments.

