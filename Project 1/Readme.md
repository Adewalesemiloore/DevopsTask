## MY LAMP STACK IMPLEMENTATION IN AWS (Linux, Apache, MySQL, PHP)

<span style="color:orange">**After successful completion of PBL projects, i've**</span>

1. Become very confident on the Linux Terminal
2. Deepened my understanding of Web Stacks and familiarity between the differences between the different Web Technology stacks such as LAMP, LEMP, MEAN, and MERN stacks
3. Developed solid Linux administration skills in Storage management, NFS, troubleshooting, and basic networking.
4. Gainned basic knowledge of AWS platform and components used to host a website of various Web stacks.

<br>

**This document details the provisioning of a LAMP stack (Linux, Apache, MySQL and PHP) on an AWS EC2 instance. The setup process was executed as part of environment configuration, and the screenshots below serve as deployment artifacts, presented in the order they were captured during the stack initialization.**

## <span style="color:lightgreen">Step 0 – Preparing prerequisites</span> 
---
In order to complete this project, I needed an AWS account and a virtual server with Ubuntu Server OS.

- I Registered an AWS accout (I had to create a new one here)
- Launched a new EC2 instance of t3.micro family with Ubuntu Server 24.04 LTS (HVM) on AWS. 

<br>

![alt text](/Project%201/Images/0.1%20-%20EC2%20Launch.png)

<br>

- Selected my preferred region (the closest to you) and launch a new EC2 instance of t3.micro family with Ubuntu Server 24.04 LTS (HVM)

<br>


![alt text](/Project%201/Images/0.3%20-%20EC2%20Launch.png)


- I sucessfully connected EC2 instance and got it running in my terminal. as shown in the images below. 

![alt text](/Project%201/Images/0.4%20-%20EC2%20Launch.png) <br>

![alt text](/Project%201/Images/0.5%20-%20EC2%20Launch.png) <br>

![alt text](/Project%201/Images/0.6%20-%20EC2%20Success.png)


The next step of the process after running my EC2 instance was setting up/ installing APache. 


## <span style="color:Pink">STEP 1 — INSTALLING APACHE AND UPDATING THE FIREWALL</span> 

I installed Apache using Ubuntu’s package manager **‘apt’**

- Update a list of packages in package manager by running `sudo apt update`

![alt text](/Project%201/Images/1.1%20-%20installing%20Apache.png)

- Run apache2 package installation `sudo apt install apache2`

![alt text](/Project%201/Images/1.2%20-%20installing%20Apache.png)

- To verify that apache2 is running as a Service in our OS, use following command `sudo systemctl status apache2`

<br>

<span style="color:lightgreen">If it is green and running, then you did everything correctly – you have just launched your first Web Server in the Clouds!</span>


![alt text](/Project%201/Images/1.5%20-%20Verifying%20Apache.png)

- Before we can receive any traffic by our Web Server, we need to open TCP port 80 which is the default port that web browsers use to access web pages on the Internet As we know, we have TCP port 22 open by default on our EC2 machine to access it via SSH, so we need to add a rule to EC2 configuration to open inbound connection through port 80:

I changed the inbound rules of the EC2 security group to all all tracfics. 


![alt text](/Project%201/Images/1.3%20-%20Security%20groups.png)

To access it locally in our Ubuntu shell, I used `run: curl http://localhost:80`  an optionally `curl http://<your-ip-address>`

> **One thing i took note of, on "every start and stop" of the EC2 instance, the IP address changes. so i had to be cautious of that.**

![alt text](/Project%201/Images/1.4%20Connecting%20locally.png)


><span style="color:lightgreen">on running the IP publically, i encountered an error where the site kept showing an error. i discovered the error was a result of the protocol in used. **HTTPS** instead of **HTTP**</span>

![alt text](/Project%201/Images/1.%205a%20-%20Not%20connecting.jpeg)<br>
![alt text](/Project%201/Images/1.4%20Connecting%20Public%20link%20.jpeg)


<br>

## <span style="color:Orange">STEP 2 — INSTALLING MYSQL</span>

Again, I used apt to acquire and install this software:`sudo apt install mysql-server`

- When prompted, confirm installation by typing Y, and then ENTER.

![alt text](/Project%201/Images/2.0%20-%20Installing%20MYSQL.png)

- When the installation is finished, log in to the MySQL console by typing: `sudo mysql`

![alt text](/Project%201/Images/2.1%20-%20Installing%20MYSQL.png)


## The other steps incuded, installing <span style="color:lightgreen">PHP, CREATING A VIRTUAL HOST FOR YOUR WEBSITE USING APACHE and ENABLE PHP ON THE WEBSITE</span>

![alt text](/Project%201/Images/3.0%20-%20installing%20PHP.png)

<br>


![alt text](/Project%201/Images/4.0%20%20-%20Everything%20else.png)  

<br>

**At this point: My LAMP stack is completely installed and fully operational.**

● Linux (Ubuntu)

● Apache HTTP Server

● MySQL

● PHP


I used VIM to create a PHP index and make it visible publically. 

![alt text](/Project%201/Images/4.0a%20PHP%20on%20site%20.png)