## SIMPLE TO-DO APPLICATION ON MERN WEB STACK

In this project, you are tasked to implement a web solution based on MERN stack in AWS Cloud.
MERN Web stack consists of following components:
- MongoDB: A document-based, No-SQL database used to store application data in a form of documents.
- ExpressJS: A server-side Web Application framework for Node.js.
- ReactJS: A frontend framework developed by Facebook. It is based on JavaScript, used to build User Interface (UI) components.
- Node.js: A JavaScript runtime environment. It is used to run JavaScript on a machine rather than in a browser.


## <span style="color:lightgreen">Step 0 – Preparing prerequisites</span>

In order to complete this project, you will need an AWS account and a virtual server with Ubuntu Server OS. If you do not have an AWS account – go back to Project 1 Step 0 to sign in to AWS free tier account and create a new EC2 Instance of t2.nano family with Ubuntu Server 20.04 LTS (HVM) image. Remember, you can have multiple EC2 instances, but make sure you STOP the ones you are not working with at the moment to save available free hours.

![alt text](Images/0.0%20Creating%20EC2%20Instance.png)
![alt text](Images/0.1%20Connecting%20to%20EC2%20Instance.png)
![alt text](Images/0.2%20Connecting%20to%20EC2%20Instance.png)


## <span style="color:lightgreen">STEP 1 – BACKEND CONFIGURATION</span>

Update ubuntu

`sudo apt update` <br>

![alt text](Images/1.0%20Backend%20Configuration.png)

Upgrade ubuntu

`sudo apt upgrade`

Let’s get the location of Node.js software from Ubuntu repositories.

`curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -` <BR>

![alt text](Images/1.1%20Upgrading%20ubuntu.png)

![alt text](Images/1.2%20Getting%20Node%20on%20ubuntu.png)

Install Node.js on the server

Install Node.js with the command below

`sudo apt-get install -y nodejs`


![alt text](Images/1.3%20Installing%20Node.js.png)

Note: The command above installs both `nodejs` and `npm`. NPM is a package manager for Node like `apt` for Ubuntu, it is used to install Node modules & packages and to manage dependency conflicts.

Verify the node installation with the command below

`node -v` 

Verify the node installation with the command below

Application Code Setup

`npm -v`  

Create a new directory for your To-Do project:

`mkdir Todo`

Run the command below to verify that the Todo directory ls created with ls command

`ls`

TIP: In order to see some more useful information about files and directories, you can use following combination of keys ls -lih – it will show you different properties and size in human readable format. You can learn more about different useful keys for ls command with ls --help. Now change your current directory to the newly created one:

`cd Todo`

Next, you will use the command npm init to initialise your project, so that a new file named package.json will be created. This file will normally contain information about your application and the dependencies that it needs to run. Follow the prompts after running the command. You can press Enter several times to accept default values, then accept to write out the package.json file by typing yes.

![alt text](Images/1.4%20Creating%20a%20package_json.png)


## <span style="color:lightgreen">INSTALL EXPRESSJS</span>

Remember that Express is a framework for Node.js, therefore a lot of things developers would have programmed is already taken care of out of the box. Therefore, it simplifies development, and abstracts a lot of low-level details. For example, Express helps to define routes of your application based on HTTP methods and URLs. To use express, install it using npm:

`npm install express`

Now create a file index.js with the command below

`touch index.js`

Run ls to confirm that your index.js file is successfully created Install the dotenv module

![alt text](Images/1.5%20installing%20expressjs.png)

Open the index.js file with the command below

`vim index.js`

Type the code below into it and save. Do not get overwhelmed by the code you see. For now, simply paste the code into the file.

![alt text](Images/1.6%20Vim%20Index_js%20Code%20edit.png)

Notice that we have specified to use port 5000 in the code. This will be required later when we go on the browser.

Use: w to save in vim and use: qa to exit vim Now it is time to start our server to see if it works. Open your terminal in the same directory as your index.js file and type:

`node index.js`

If everything goes well, you should see Server running on port 5000 in your terminal.

![alt text](Images/1.7%20Confirmation%20of%20server.png)

Accessing the public site. 

`http://<PublicIP-or-PublicDNS>:5000`

![alt text](Images/1.8%20Public%20domain.png)

### Routes
There are three actions that our To-Do application needs to be able to do:
- Create a new task
- Display list of all tasks
- Delete a completed task

Each task will be associated with some particular endpoint and will use different standard HTTP request methods: POST, GET, DELETE.

For each task, we need to create routes that will define various endpoints that the To-do app will depend on. So let us create a folder routes

`mkdir routes`

Tip: You can open multiple shells in Putty or Linux/Mac to connect to the same EC2 Change directory to routes folder.

`cd routes`

Now, create a file api.js with the command below

`touch api.js`

Open the file with the command below

`vim api.js`

![alt text](Images/1.9%20Creating%20a%20directory.png)


### MODELS

Now comes the interesting part, since the app is going to make use of Mongodb which is a NoSQL database, we need to create a model.

A model is at the heart of JavaScript based applications, and it is what makes it interactive.

We will also use models to define the database schema . This is important so that we will be able to define the fields stored in each Mongodb document. (Seems like a lot of information, but not to worry, everything will become clear to you over time. I promise!!!) In essence, the Schema is a blueprint of how the database will be constructed, including other data fields that may not be required to be stored in the database. These are known as virtual properties

To create a Schema and a model, install mongoose which is a Node.js package that makes working with mongodb easier. Change directory back Todo folder with cd .. and install Mongoose


`npm install mongoose`
Create a new folder models :

`mkdir models`

Change directory into the newly created ‘models’ folder with

`cd models`

Inside the models folder, create a file and name it todo.js

`touch todo.js`

Tip: All three commands above can be defined in one line to be executed consequently with help of && operator, like this:

`mkdir models && cd models && touch todo.js`

![alt text](Images/2.0%20Creating%20the%20Model%20directory.png)

Open the file created with `vim todo.js` then paste the code below in the file:

![alt text](Images/2.1%20Definind%20the%20models.png)

Now we need to update our routes from the file api.js in ‘routes’ directory to make use of the new model.

In Routes directory, open api.js with vim api.js, delete the code inside with :%d command and paste there code below into it then save and exit

![alt text](Images/2.2%20Updating%20the%20routes%20api.png)


### MONGODB DATABASE

We need a database where we will store our data. For this we will make use of mLab. mLab provides MongoDB database as a service solution (DBaaS), so to make life easy, you will need to sign up for a shared clusters free account, which is ideal for our use case. Sign up here MONGODB. Follow the sign up process, select AWS as the cloud provider, and choose a region near you.

![alt text](Images/3.0%20Creating%20mongo%20DB.png)

Allow access to the MongoDB database from anywhere (Not secure, but it is ideal for testing)

#### IMPORTANT NOTE

In the image below, make sure you change the time of deleting the entry from 6 Hours to 1 Week

![alt text](Images/3.2%20Adding%20IP%20address.png)

Create a MongoDB database and collection inside mLab

![alt text](Images/3.1%20Connecting%20to%20a%20cluster.png)

![alt text](Images/3.1a%20Cluster%20created.png)

![alt text](Images/3.3%20Creating%20a%20database.png)

![alt text](Images/3.3a%20Creating%20a%20database.png)

Ensure to update <username>, <password>, <network-address> and <database> according to your setup. Here is how to get your connection string

![alt text](Images/3.4%20adding%20the%20string%20to%20VI.png)


Now we need to update the `index.js` to reflect the use of .env so that Node.js can connect to the database. Simply delete existing content in the file, and update it with the entire code below. To do that using vim, follow below steps

Open the file with `vim index.js`
Press esc
Type :
Type %d
Hit ‘Enter’ The entire content will be deleted, then,
Press i to enter the insert mode in vim
Now, paste the entire code below in the file.

![alt text](Images/3.5%20Index.png)

Using environment variables to store information is considered more secure and best practice to separate configuration and secret data from the application, instead of writing connection strings directly inside the index.js application file.
Start your server using the command:

`node index.js`

You shall see a message ‘Database connected successfully’, if so – we have our backend configured. Now we are going to test it.
Testing Backend Code without Frontend using RESTful API

![alt text](Images/3.6%20Database%20connected%20successfully.png)

So far we have written backend part of our To-Do application, and configured a database, but we do not have a frontend UI yet. We need ReactJS code to achieve that. But during development, we will need a way to test our code using RESTfulL API. Therefore, we will need to make use of some API development client to test our code.

In this project, we will use Postman to test our API. Click Install Postman to download and install postman on your machine.

Click HERE to learn how perform CRUD operartions on Postman You should test all the API endpoints and make sure they are working. For the endpoints that require body, you should send JSON back with the necessary fields since it’s what we setup in our code.

Now open your Postman, create a POST request to the API `http://<PublicIP-or-PublicDNS>:5000/api/todos`. This request sends a new task to our To-Do list so the application could store it in the database.

Note: make sure your set header key Content-Type as application/json

![alt text](Images/3.7%20Post%20request%20to%20API.png)

![alt text](Images/3.8%20Post%20request%20to%20API-Body.png)

Create a GET request to your API on `http://<PublicIP-or-PublicDNS>:5000/api/todos.`

![alt text](Images/3.9%20Post%20request%20to%20API-Response.png)

This request retrieves all existing records from out To-do application (backend requests these records from the database and sends it us back as a response to GET request).

Optional task: Try to figure out how to send a DELETE request to delete a task from out To-Do list.

**Hint: **To delete a task – you need to send its ID as a part of DELETE request. By now you have tested backend part of our To-Do application and have made sure that it supports all three operations we wanted:

- Display a list of tasks – HTTP GET request
- Add a new task to the list – HTTP POST request
- Delete an existing task from the list – HTTP DELETE request

We have successfully created our Backend, now let go create the Frontend.

<br>

## STEP 2 – FRONTEND CREATION

Since we are done with the functionality we want from our backend and API, it is time to create a user interface for a Web client (browser) to interact with the application via API. To start out with the frontend of the To-do app, we will use the create-react-app command to scaffold our app.

In the same root directory as your backend code, which is the Todo directory, run:

`npx create-react-app client`

![alt text](Images/4.0%20Creating%20react%20app%20client.png)

This will create a new folder in your Todo directory called client, where you will add all the react code.

`Running a React App`

Before testing the react app, there are some dependencies that need to be installed.

Install concurrently. It is used to run more than one command simultaneously from the same terminal window.

`npm install concurrently --save-dev`

Install nodemon. It is used to run and monitor the server. If there is any change in the server code, nodemon will restart it automatically and load the new changes.

`npm install nodemon --save-dev`

In Todo folder open the package.json file. Change the highlighted part of the below screenshot and replace with the code below.

![alt text](Images/4.1%20Installig%20concurrently.png)

![alt text](Images/4.2%20add%20script.png)

![alt text](Images/4.2%20Installing%20Nodemonpng.png)

Configure Proxy in package.json
Change directory to ‘client’

`cd client`

Open the package.json file

`vi package.json`

Add the key value pair in the package.json file "proxy": "http://localhost:5000".
The whole purpose of adding the proxy configuration in number 3 above is to make it possible to access the application directly from the browser by simply calling the server url like http://localhost:5000 rather than always including the entire path like http://localhost:5000/api/todos

Now, ensure you are inside the Todo directory, and simply do:

`npm run dev`

![alt text](Images/4.5%20Run%20dev.png)

Creating your React Components
One of the advantages of react is that it makes use of components, which are reusable and also makes code modular. For our Todo app, there will be two stateful components and one stateless component.

From your Todo directory run

`cd client`

move to the src directory

`cd src`

Inside your src folder create another folder called components

`mkdir components`

Move into the components directory with

`cd components`

Inside ‘components’ directory create three files Input.js, ListTodo.js and Todo.js.

`touch Input.js ListTodo.js Todo.js`

Open Input.js file

`vi Input.js`

![alt text](Images/4.6%20Componnent.png)

![alt text](Images/4.7%20Import%20axios.png)

To make use of Axios, which is a Promise based HTTP client for the browser and node.js, you need to cd into your client from your terminal and run yarn add axios or npm install axios.

Move to the src folder
`cd ..`
Move to clients folder
`cd ..`
Install Axios
`npm install axios`
Sip a coffee, click on the next button and let finish this up.

![alt text](Images/4.8%20Installing%20axios.png)

Go to ‘components’ directory

`cd src/components`

After that open your ListTodo.js

`vi ListTodo.js`

in the `ListTodo.js` copy and paste the following code

![alt text](Images/4.8a%20Listing%20todo.png)

Then in your `Todo.j`s file you write the following code

![alt text](Images/4.8b%20Get%20todo%20in%20vi%20todo-js.png)

We need to make little adjustment to our react code. Delete the logo and adjust our App.js to look like this.

Move to the src folder
`cd ..`
Make sure that you are in the src folder and run
`vi App.js`

![alt text](Images/4.8c%20Changing%20the%20App-js%20to%20import%20from%20react.png)

After pasting, exit the editor.
In the src directory open the `App.css`

`vi App.css`

Then paste the following code into `App.css:`


`npm run dev`

Assuming no errors when saving all these files, our To-Do app should be ready and fully functional with the functionality discussed earlier: creating a task, deleting a task and viewing all your tasks.

<br>

<p align="center">
  <video src="https://github.com/user-attachments/assets/2e3bbf75-27be-4f5e-8d02-d9c5b1d2af39" controls style="max-width: 100%; height: auto;">
  </video>
</p>
