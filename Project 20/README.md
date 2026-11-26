<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">MIGRATION TO THE CLOUD WITH CONTAINERIZATION PART 1 — DOCKER & DOCKER COMPOSE</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>November 26, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" />
  <img src="https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white" />
  <img src="https://img.shields.io/badge/PHP-777BB4?style=for-the-badge&logo=php&logoColor=white" />
  <img src="https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white" />
  <img src="https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white" />
</p>

---

## Project Overview

Until now I have deployed everything on virtual machines — EC2 instances inside a VPC, set up by hand or with Terraform and Ansible. That scales to a point, but imagine dozens or hundreds of small applications, each needing a different OS, runtime version or conflicting dependencies. I would end up spinning up a server for every group of applications, which becomes tedious and expensive to maintain, especially with a microservice architecture.

**Containers** solve this, and **Docker** is the standard way of building and running them. Instead of a whole guest operating system, a container packs only an isolated, minimal slice of it with everything the application needs. It is lighter and faster than a VM, and the same image behaves identically on any machine that runs the Docker engine. That ends the classic "it works on my machine" problem.

In this project I built a small **proof of concept (POC)** by migrating the **Tooling web application** — a PHP front end backed by a MySQL database that I already know from the earlier projects — from a VM-based setup to a containerized one:

- installed Docker on an Ubuntu EC2 instance,
- ran MySQL from a pre-built image as a container,
- created a custom Docker network so the app and the database can talk to each other by hostname,
- created a non-root database user and loaded the Tooling schema,
- built the Tooling application image from its Dockerfile and ran it as a container,
- fixed a database connection problem and logged in to the running application.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part 1 — Install Docker](#part-1--install-docker)
- [Part 2 — MySQL in a Container](#part-2--mysql-in-a-container)
- [Part 3 — Connect to MySQL and Create a User](#part-3--connect-to-mysql-and-create-a-user)
- [Part 4 — Prepare the Database Schema](#part-4--prepare-the-database-schema)
- [Part 5 — Build and Run the Tooling App](#part-5--build-and-run-the-tooling-app)
- [Result](#result)
- [Errors & Fixes](#errors--fixes)

---

## Architecture Overview

Both containers run on one Docker host (an Ubuntu EC2 instance) and share a user-defined bridge network:

| Component | Image / Source | Role |
|---|---|---|
| Docker host | Ubuntu EC2 instance with Docker Engine | Runs all containers |
| `tooling_network` | Bridge network, subnet `172.18.0.0/24` | Private network shared by the app and the database |
| `mysql-server` | `mysql/mysql-server:latest` | Database container, hostname `mysqlserverhost` |
| `mysql-client` | `mysql` (temporary, `--rm`) | Disposable client used to test the DB connection |
| `tooling-app` | `tooling:0.0.1` (built from the Tooling repo Dockerfile) | Apache + PHP web app |

Traffic flows: **Browser -> host port 8085 -> container port 80 (Apache/PHP) -> `mysqlserverhost:3306` over `tooling_network`.**

> Port 8085 on the host is mapped to port 80 inside the container, because the Apache server inside listens on 80 and I chose a free host port. Make sure the EC2 security group allows inbound traffic on that port.

---

## Part 1 — Install Docker

### 1.0 — Docker Engine

Docker Engine is a client-server application: a long-running daemon (`dockerd`), an API for programs to talk to the daemon, and the `docker` CLI client. On my Windows workstation I could install Docker Desktop with `winget`:

```powershell
winget install -e --id Docker.DockerDesktop
```

![Installing Docker Desktop with winget](./Images/Task%2020.1.png)

For the project itself I used an Ubuntu EC2 instance as the Docker host:

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
```

![Installing Docker on the Ubuntu instance](./Images/Task%2020.2.png)

I confirmed the installation:

```bash
docker --version
```

![Docker version](./Images/Task%2020.3.png)

To run Docker commands without typing `sudo` every time, I added my user to the `docker` group and logged out and back in for it to take effect:

```bash
sudo usermod -aG docker ubuntu
exit
```

![Adding the user to the docker group](./Images/Task%2020.4.png)

---

## Part 2 — MySQL in a Container

### 2.0 — Pulling the MySQL Image

I pulled the MySQL image from Docker Hub:

```bash
docker pull mysql/mysql-server:latest
```

![Pulling the MySQL image](./Images/Task%2020.5.png)

and listed the local images to check it downloaded:

```bash
docker image ls
```

![Listing images](./Images/Task%2020.6.png)

---

### 3.0 — Running the First MySQL Container

I deployed a MySQL container in the background (`-d`) with a root password and the MySQL port published:

```bash
docker run -d \
  --name mysql-server \
  -e MYSQL_ROOT_PASSWORD=<my-secret-pw> \
  -p 3306:3306 \
  mysql/mysql-server:latest
```

Then I checked that it was running:

```bash
docker ps -a
```

The status changes from `health: starting` to `healthy` once the setup is complete.

![First MySQL container running and healthy](./Images/Task%2020.7.png)

---

### 4.0 — Connecting Directly to the Container (Approach 1)

The first way to reach the database is to run the MySQL client **inside** the server container with `docker exec`:

```bash
docker exec -it mysql-server mysql -u root -p
```

Flags used:

- `exec` — run a command inside a running container,
- `-it` — interactive, with a pseudo-TTY,
- `mysql -u root -p` — the client command, user `root`, prompting for the password.

![Connecting directly to MySQL inside the container](./Images/Task%2020.8.png)

I also checked the container list and its logs:

```bash
docker ps
docker logs mysql-server
```

![docker ps and docker logs](./Images/Task%2020.9.png)

---

### 5.0 — Removing the First Container

The next approach needs a custom network, so I stopped and removed this first container and verified that the list was empty:

```bash
docker ps -a
docker stop mysql-server
docker rm mysql-server
docker ps -a
```

![Stopping and removing the MySQL container](./Images/Task%2020.10.png)

---

### 6.0 — Creating a Custom Network

A custom network is not strictly required, since Docker uses the default bridge network otherwise. But it is useful when I need to control the CIDR range of the containers in the stack, so I created a dedicated network with its own subnet and used it for both the database and the application:

```bash
docker network create --subnet=172.18.0.0/24 tooling_network
docker network ls
```

![Creating the tooling_network network](./Images/Task%2020.11.png)

---

### 7.0 — Running MySQL on the Custom Network

I stored the root password in an environment variable so it is not typed into every command, and verified it:

```bash
export MYSQL_PW=<my-secret-pw>
echo $MYSQL_PW
```

![Exporting the MYSQL_PW variable](./Images/Task%2020.12.png)

Then I started the MySQL server container on the new network, with a fixed hostname the app can use to find it:

```bash
docker run --network tooling_network -h mysqlserverhost --name=mysql-server -e MYSQL_ROOT_PASSWORD=$MYSQL_PW -d mysql/mysql-server:latest
```

Flags used:

- `-d` — detached (background) mode,
- `--network` — attach the container to a network,
- `-h` — set the container hostname.

```bash
docker ps
```

![MySQL container running on the custom network](./Images/Task%2020.13.png)

---

## Part 3 — Connect to MySQL and Create a User

It is best practice not to connect to MySQL remotely as `root`, so I created a dedicated user with an SQL script.

### 8.0 — The `create_user.sql` Script

```sql
CREATE USER '<db-user>'@'%' IDENTIFIED BY '<db-password>';
GRANT ALL PRIVILEGES ON *.* TO '<db-user>'@'%';
```

![Writing create_user.sql](./Images/Task%2020.14.png)

![Verifying the script contents](./Images/Task%2020.15.png)

I ran it against the MySQL container, from the directory where the file lives:

```bash
docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < create_user.sql
```

> The `mysql: [Warning] Using a password on the command line interface can be insecure.` message is normal here. It is only a warning.

![Running create_user.sql](./Images/Task%2020.16.png)

---

### 9.0 — Using a Second Container as the MySQL Client

A nice advantage of containers is that I did not need to install any MySQL client on the host. I ran a throw-away second container as the client on the same network:

```bash
docker run --network tooling_network --name mysql-client -it --rm mysql mysql -h mysqlserverhost -u <db-user> -p$MYSQL_PW
```

Flags used:

- `--name` — name for the container,
- `-it` — interactive, with a pseudo-TTY,
- `--rm` — delete the container automatically when it exits,
- `--network` — attach to the same network as the server,
- `-h` — the MySQL server's hostname (`mysqlserverhost`),
- `-u` / `-p` — the user from `create_user.sql` and its password.

On the first run, Docker did not find the `mysql` image locally and pulled it.

![Client container pulling the mysql image](./Images/Task%2020.17.png)

After that, the client connected to the server container as my new user and I got the `mysql>` prompt.

![MySQL client connected to mysqlserverhost](./Images/Task%2020.18.png)

---

## Part 4 — Prepare the Database Schema

The Tooling application needs its database and tables, so I loaded the schema that ships with the application.

### 10.0 — Cloning the Tooling Repository

```bash
git clone https://github.com/darey-devops/tooling.git
```

![Cloning the tooling repository](./Images/Task%2020.19.png)

### 11.0 — Exporting the Schema Location

The schema file is `tooling/html/tooling_db_schema.sql`. I changed into that directory, exported its location and verified it:

```bash
cd tooling/html
export tooling_db_schema=tooling_db_schema.sql
echo $tooling_db_schema
```

![Exporting the schema file path](./Images/Task%2020.20.png)

### 12.0 — Loading the Schema

I used `docker exec` to feed the SQL file to the database in the running container:

```bash
docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < $tooling_db_schema
```

My first attempt was rejected with `ERROR 1045 (28000): Access denied for user 'root'@'localhost'`:

![Access denied error when loading the schema](./Images/Task%2020.21.png)

The `$MYSQL_PW` variable turned out to be empty in my shell session, so I recreated it:

```bash
echo $MYSQL_PW
export MYSQL_PW="<my-secret-pw>"
echo $MYSQL_PW
```

![Recreating the MYSQL_PW variable](./Images/Task%2020.22.png)

I confirmed I could log in as root again:

```bash
docker exec -it mysql-server mysql -uroot -p$MYSQL_PW
```

![Root login working after recreating the variable](./Images/Task%2020.23.png)

and then the schema import went through cleanly (just the usual password warning):

```bash
docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < $tooling_db_schema
```

![Schema loaded successfully](./Images/Task%2020.24.png)

---

### 13.0 — Updating the `.env` File

The application reads its database connection details from `tooling/html/.env`. It is a hidden file, so I opened it with `vim`:

```bash
sudo vim .env
```

![Opening the .env file](./Images/Task%2020.26.1.png)

and set the connection details:

```
MYSQL_IP=mysqlserverhost
MYSQL_USER=<db-user>
MYSQL_PASS=<db-password>
MYSQL_DBNAME=toolingdb
```

- `MYSQL_IP` — left as `mysqlserverhost`, the hostname of the DB container,
- `MYSQL_USER` / `MYSQL_PASS` — the user created from `create_user.sql`,
- `MYSQL_DBNAME` — `toolingdb`.

![Contents of the .env file](./Images/Task%2020.25.png)

---

## Part 5 — Build and Run the Tooling App

### 14.0 — Building the Image

Containerizing an app starts with a **Dockerfile** — a recipe that tells Docker how to pack the application into an image. The Tooling repository already has one. From the `tooling` directory (where the Dockerfile lives) I built the image:

```bash
cd ~/tooling
docker build -t tooling:0.0.1 .
```

`-t` tags the image as `tooling:0.0.1`, and the trailing `.` tells Docker to look for the Dockerfile in the current directory.

![Starting the docker build](./Images/Task%2020.26.png)

The build starts from `php:7-apache`, installs Composer, copies the Apache config and the `html` folder into `/var/www`, and finishes with `CMD ["start-apache"]`. It ended with a successful tag:

![Build finished and image tagged tooling:0.0.1](./Images/Task%2020.27.png)

---

### 15.0 — Running the Container

```bash
docker run --network tooling_network -p 8085:80 -it tooling:0.0.1
```

- `--network` puts the app on the same virtual network as the database so they can reach each other,
- `-p 8085:80` maps host port 8085 to the container's Apache on port 80. The host's port 80 may already be in use, so I used the free port 8085.

The container started and Apache served my first request, though it also printed the `AH00558` warning about the server's fully qualified domain name.

![Running the tooling container in the foreground](./Images/Task%2020.28.png)

> `AH00558: apache2: Could not reliably determine the server's fully qualified domain name` is only a warning. It can be silenced by setting the `ServerName` directive globally in the Apache config.

I later re-ran it detached, with a name, so it stays in the background:

```bash
docker run -d --network tooling_network -p 8085:80 --name tooling-app tooling:0.0.1
```

![Running the tooling container detached with a name](./Images/Task%2020.288.png)

---

### 16.0 — Fixing the Database Connection

Opening `http://<EC2-Public-IP>:8085` showed a database connection error instead of the login page:

![mysqli connection failed warning in the browser](./Images/Task%2020.29.png)

To fix it, I stopped and removed the running container and opened `db_conn.php`:

```bash
docker stop tooling-app
docker rm tooling-app
cd ~/tooling/html
sudo vi db_conn.php
```

![Stopping the container and opening db_conn.php](./Images/Task%2020.30.png)

In `db_conn.php` I commented out the four lines that read the connection values from the environment (`$_ENV[...]`) and passed the values to `mysqli` directly, using the DB container's hostname:

```php
//$servername = $_ENV["MYSQL_IP"]; // input servername
//$username = $_ENV["MYSQL_USER"]; // input username
//$password = $_ENV["MYSQL_PASS"]; //input password
//$dbname = $_ENV["MYSQL_DBNAME"]; // input dbname

// Create connection
$conn = new mysqli("mysqlserverhost", "<db-user>", "<db-password>", "toolingdb");
```

![db_conn.php with hard-coded connection values](./Images/Task%2020.31.png)

Because the application files are copied into the image at build time, I had to **rebuild** the image for the change to take effect:

```bash
cd ~/tooling
docker build -t tooling:0.0.1 .
```

![Rebuilding the image](./Images/Task%2020.32.png)

Then I ran it again, and this time the login/register page of the Tooling website loaded at `http://<EC2-Public-IP>:8085`.

![Tooling website register page served from the container](./Images/Task%2020.33.png)

---

### 17.0 — Logging In

The application's default login is the email `test@gmail.com` with password `12345` (users are also stored in the `toolingdb.user` table). After logging in I reached the dashboard.

![Tooling website dashboard](./Images/Task%2020.34.png)

and the Devops Tools page.

![Devops Tools page](./Images/Task%2020.35.png)

---

## Result

The Tooling web application is now fully containerized. A MySQL container and a PHP/Apache container run on the same user-defined Docker network, the app reaches the database by hostname, and the site is reachable in the browser through the mapped host port. This is the proof of concept for moving larger, microservice-style workloads from VMs to Docker.

---

## Errors & Fixes

Errors I ran into during this project and how I resolved them.

---

**`ERROR 1045 (28000): Access denied for user 'root'@'localhost'` when loading the schema**

**Symptom:** `docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < $tooling_db_schema` returned `Access denied for user 'root'@'localhost' (using password: YES)`.

**Root cause:** The `MYSQL_PW` environment variable was not set to the real root password in my current shell session (`echo $MYSQL_PW` printed nothing), so the wrong password was being sent.

**Fix:** Re-exported the variable, checked it with `echo`, tested a root login, then re-ran the schema import:

```bash
export MYSQL_PW="<my-secret-pw>"
echo $MYSQL_PW
docker exec -it mysql-server mysql -uroot -p$MYSQL_PW
docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < $tooling_db_schema
```

---

**`Warning: mysqli::__construct(): (HY000/2002): No such file or directory in /var/www/db_conn.php` / `Connection failed: No such file or directory`**

**Symptom:** The site at `http://<EC2-Public-IP>:8085` displayed a PHP mysqli connection warning instead of the login page.

**Root cause:** The application was not picking up the database host from the environment/`.env` values, so `mysqli` had no valid host to connect to.

**Fix:** Stop and remove the container, edit `db_conn.php` to comment out the `$_ENV[...]` lines and call `new mysqli("mysqlserverhost", "<db-user>", "<db-password>", "toolingdb")` directly, **rebuild** the image (`docker build -t tooling:0.0.1 .`) and run it again on `tooling_network`.

---

**`AH00558: Could not reliably determine the server's fully qualified domain name`**

**Symptom:** Apache printed this message when the container started.

**Root cause:** No `ServerName` directive is set in the Apache configuration inside the image.

**Fix:** It is a harmless warning. To silence it, set `ServerName` globally in the Apache config and rebuild the image.

---

**`mysql: [Warning] Using a password on the command line interface can be insecure.`**

**Symptom:** The warning appears every time `-p$MYSQL_PW` is used.

**Root cause:** The password is passed as a command-line argument.

**Fix:** None needed for this lab; the commands succeeded. In real environments, prefer prompting for the password or using a secrets mechanism.

---

*Project completed by **Adewale Oluwasemiloore***
