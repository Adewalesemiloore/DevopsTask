## MY WEB STACK IMPLEMENTATION (LEMP STACK) (Linux, Ngingx, MySQL, PHP)

In this project, I will be implementing a similar stack, but with an alternative Web Server – NGINX, which is also very popular and widely used by many websites on the Internet.

## <span style="color:lightgreen">Step 0 – Preparing prerequisites</span>

In order to complete this project, I needed to Create, connect and Launch a new EC2 instance of t3.micro family with Ubuntu Server 24.04 LTS (HVM) on AWS. - just as in project 1. and SSH into the instance. 

<br>

![alt text](Images/1.0%20-%20New%20EC2%20Instance.png)

- Connecting to the EC2 Instance Via my terminal

![alt text](Images/1.2%20-%20Conecting%20to%20EC2%20Instance.png)

## <span style="color:Pink">Step 1 – Installing the Nginx Web Server</span>

In order to display web pages to our site visitors, we are going to employ Nginx, a high-performance web server. We’ll use the apt package manager to install this package.
Since this is our first time using apt for this session, start off by updating your server’s package index. Following that, you can use apt install to get Nginx installed:

1. `sudo apt update`

![alt text](Images/1.2%20-%20Conecting%20to%20EC2%20Instance.png)

2. sudo apt install nginx

![alt text](Images/1.3%20-%20Installing%20Nginx.png)



After verifying my Ngingx install with
 `sudo systemctl status nginx` I encountered an error. although it showed that the status was active i could not find a way to quit out of the command. on a little research i found out that q was a way to quit and start over on my terminal

![alt text](Images/1.4%20Nginx%20status%20error.png)

To acess the site publicly on my browser i used my public address gotten from the EC2 instance created. 

![alt text](Images/1.5%20-%20Viewing%20nginx%20via%20public%20port.png)


## <span style="color:lightgreen">STEP 2 — INSTALLING MYSQL</span>

Now that I had a web server up and running, I needed to install a **DBMS (database managemment system)**

I used the ‘apt’ to acquire and install this software: `sudo apt install mysql-server`

![alt text](Images/2.0%20Installing%20SQL.png)

When prompted, I confirmed installation by typing Y, and then ENTER.

When the installation was finished, I logged in to the MySQL console by typing: `sudo mysql`

![alt text](Images/2.1%20Accessing%20mySQL.png)


This Basically connected to the MySQL server as the administrative database user root, which is inferred by the use of sudo when running this command.


I set a password for the root user, using mysql_native_password as default authentication method. `ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'PassWord.1';`

![alt text](Images/2.2%20Password%20change%20and%20access%20to%20mySQL.png)

and ran a security script `sudo mysql_secure_installation`


## <span style="color:Pink">STEP 3 – INSTALLING PHP</span>


You have Nginx installed to serve your content and MySQL installed to store and manage your data. Now you can install PHP to process code and generate dynamic content for the web server.

**Quick note:** While Apache embeds the PHP interpreter in each request, Nginx requires an external program to handle PHP processing and act as a bridge between the PHP interpreter itself and the web server. This allows for a better overall performance in most PHP-based websites, but it requires additional configuration. You’ll need to install php-fpm, which stands for “PHP fastCGI process manager”, and tell Nginx to pass PHP requests to this software for processing. Additionally, you’ll need php-mysql, a PHP module that allows PHP to communicate with MySQL-based databases. Core PHP packages will automatically be installed as dependencies.

To install these 2 packages at once, I ran: `sudo apt install php-fpm php-mysql`







## <span style="color:lightgreen">STEP 4 — CONFIGURING NGINX TO USE PHP PROCESSOR</span>

When using the Nginx web server, we can create server blocks (similar to virtual hosts in Apache) to encapsulate configuration details and host more than one domain on a single server. 

On Ubuntu 20.04, Nginx has one server block enabled by default and is configured to serve documents out of a directory at `/var/www/html.` While this works well for a single site, it can become difficult to manage if you are hosting multiple sites. Instead of modifying `/var/www/html`, we’ll create a directory structure within `/var/www` for your domain website, leaving `/var/www/html` in place as the default directory to be served if a client request does not match any other sites.


To Create the root web directory for my domain I used: `sudo mkdir /var/www/projectLEMP`

![alt text](/Images/4.0%20-%20%20CONFIGURING%20NGINX%20TO%20USE%20PHP%20PROCESSOR.png)

The Next steps involved, assigning ownership of the directory with the $USER environment variable, which will reference your current system user:
`sudo chown -R $USER:$USER /var/www/projectLEMP`
Then, open a new configuration file in Nginx’s sites-available directory using your preferred command-line editor. Here, we’ll use nano:
`sudo nano /etc/nginx/sites-available/projectLEMP`

**This will create a new blank file. and i pasted the instruction below within**

    #/etc/nginx/sites-available/projectLEMP
 
    server {
	
    listen 80;
	server_name projectLEMP www.projectLEMP;
	root /var/www/projectLEMP;
 
	index index.html index.htm index.php;
 
	location / {
    	try_files $uri $uri/ =404;
	}
 
	location ~ \.php$ {
    	include snippets/fastcgi-php.conf;
    	fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
 	}
 
	location ~ /\.ht {
    	deny all;
	}
 
    }


Activate your configuration by linking to the config file from Nginx’s sites-enabled directory:
sudo ln -s /etc/nginx/sites-available/projectLEMP /etc/nginx/sites-enabled/
This will tell Nginx to use the configuration next time it is reloaded. You can test your configuration for syntax errors by typing:
`sudo nginx -t`

You shall see following message:
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
If any errors are reported, go back to your configuration file to review its contents before continuing.

We also need to disable default Nginx host that is currently configured to listen on port 80, for this run:
`sudo unlink /etc/nginx/sites-enabled/default`

When you are ready, reload Nginx to apply the changes:
`sudo systemctl reload nginx`
Your new website is now active, but the web root `/var/www/projectLEMP` is still empty. Create an index.html file in that location so that we can test that your new server block works as expected:

![alt text](Images/4.1%20-%20%20CONFIGURING%20NGINX%20TO%20USE%20PHP%20PROCESSOR.png)

## <span style="color:Pink">STEP 5 – TESTING PHP WITH NGINX</span>

Your LEMP stack should now be completely set up.
At this point, your LAMP stack is completely installed and fully operational.
You can test it to validate that Nginx can correctly hand .php files off to your PHP processor.
You can do this by creating a test PHP file in your document root. Open a new file called info.php within your document root in your text editor:

`sudo nano /var/www/projectLEMP/info.php`

![alt text](Images/5.0%20-%20Testing%20PHP%20on%20nginx.png)

After this, i tried accessing thr page in my browser and was hit with a bad gateway error. 

![alt text](Images/5.1%20-%20Testing%20PHP%20on%20nginx%20-%20bad%20gateway.png)


To solve this error i spiraled round a bunch of different fixes. 

1. I edited the Ngingx default to reach my PHP file but this didn't seem to explicitly solve the issue as i was still hit with bad gateway. 

![alt text](Images/5.2%20-%20Editing%20nginx%20Defaut%20.png)

2. I figured the issue was that i had installed the latest PHP version copied from the doc the script with version 8.1. so the page version was not visible as a result. 

I caught the error by using `grep -r "php8.1" /etc/nginx/` this pulled what version of PHP was being checked and used. 

![alt text](Images/5.4%20-%20Php%20version%20visible.png)

I went back into - `#/etc/nginx/sites-available/projectLEMP` and found the version there and changed it to 8.3 which is my current PHP version. 

![alt text](Images/5.3%20-%20Project%20Lemp%20wrong%20version.png)

with this i has the page visible publically now. 

![alt text](Images/5.5%20-%20Php%20version%20visible.png)


## <span style="color:lightgreen">Step 6 — Retrieving data from MySQL database with PHP</span>

In this step I will create a test database (DB) with simple "To do list" and configure access to it, so the Nginx website would be able to query data from the DB and display it.

I created a database named example_database and a user named example_user, but I replaced these names with different values.

First, I connect to the MySQL console using the root account with: 
 `sudo mysql`

 ![alt text](Images/6.0%20Sudo%20Error%20and%20access.png)

 Eventually using `sudo mysql -u root -p` to gain access by putting the password set initially. 

 I created a new database with `mysql> CREATE DATABASE 'example_database';` 
 
 and also created user access with, 
 
 `mysql> CREATE USER 'example_user'@'%' IDENTIFIED WITH mysql_native_password BY 'password';`

after that i gave the user permission over the example_database database with:

`mysql> GRANT ALL ON example_database.* TO 'example_user'@'%';`

Image as seen below. 

![alt text](Images/6.1%20Creating%20a%20database%20with%20mysql.png)

You can test if the new user has the proper permissions by logging in to the MySQL console again, this time using the custom user credentials:

`mysql -u example_user -p`

Notice the -p flag in this command, which will prompt you for the password used when creating the example_user user. After logging in to the MySQL console, confirm that you have access to the example_database database:

`mysql> SHOW DATABASES;`

Next, we’ll create a test table named todo_list. From the MySQL console, run the following statement:

`CREATE TABLE example_database.todo_list (
mysql> 	item_id INT AUTO_INCREMENT,
mysql> 	content VARCHAR(255),
mysql> 	PRIMARY KEY(item_id)
mysql> );`

Insert a few rows of content in the test table. You might want to repeat the next command a few times, using different VALUES:

`mysql> INSERT INTO example_database.todo_list (content) VALUES ("My first important item");`

To confirm that the data was successfully saved to your table, run:
`mysql> SELECT * FROM example_database.todo_list;`

The above gave me the output below. 


![alt text](Images/6.2%20Creating%20a%20database%20with%20mysql.png)


The last step in this process was Creating a PHP script that will connect to MySQL and query for your content

I used `nano /var/www/projectLEMP/todo_list.php` to access the code editor and inputed the command below. 

    <?php
    $user = "example_user";
    $password = "password";
    $database = "example_database";
    $table = "todo_list";
 
    try {
    $db = new PDO("mysql:host=localhost;dbname=$database", $user, $password);
    echo "<h2>TODO</h2><ol>";
    foreach($db->query("SELECT content FROM $table") as $row) {
	echo "<li>" . $row['content'] . "</li>";
    }
    echo "</ol>";
    } catch (PDOException $e) {
	print "Error!: " . $e->getMessage() . "<br/>";
	die();
    }

and as a result

![alt text](Images/6.4%20viewing%20PHP%20script%20on%20public%20domain.png)