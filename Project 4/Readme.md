

<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center"> MEAN STACK DEPLOYMENT TO UBUNTU IN AWS</h3> 
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd>  <b style="color: #fb6900">Date:</b> <kbd>May 15, 2026</kbd</p>
</div>




![Amazon AWS](https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Angular](https://img.shields.io/badge/Angular-DD0031?style=for-the-badge&logo=angular&logoColor=white)
![Express.js](https://img.shields.io/badge/Express.js-ff007f?style=for-the-badge&logo=express&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-0052cc?style=for-the-badge&logo=mongodb&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

## Project Overview

In this project, I was tasked to build a simple Book Register web form using the MEAN stack and deploy it on an Ubuntu Server in AWS. My goal was to uderstand how to work with MongoDB, Express, Angular, and Node.js and understand how they all work together in a real server setup.



![Final Result of my Project](Images/0.0%20MEAN%20Stack%20Deployment%20project.png)

<br>

## <span style="color: #339933">STEP 1 – INSTALLING NODE.JS</span> 
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)

My first steps include: Updating my ubuntu system, and installing Node.js. 

```bash
sudo apt update
```
![sudo update](./Images/1.0%20-%20Sudo%20update.png)

```bash
sudo apt upgrade
```
![sudo upgrade](./Images/1.0a%20-%20Sudo%20upgrade.png)

```bash
sudo apt -y install curl dirmngr apt-transport-https lsb-release ca-certificates
```
![install certs](./Images/1.1%20-%20Adding%20certificates.png)

<br> 

<blockquote style="border-left: 5px solid #d9534f;">
  <strong style="color: #d9534f;">⚠️ Error:</strong> 
  On runninng the command below, I encountered an error because of the version. as seen below.
</blockquote>

<br>

```bash
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
```
  
![nodesource setup](./Images/1.1a%20-%20Nod%20js%20x12%20innnstallation%20warning.png)



Using the code below to resolve the error. I found the command for the altest node setup. 

![nodesource setup](./Images/1.1b%20-%20Node%20js%20x20%20installation.png)

```bash
sudo apt install -y nodejs
```
<br>

## <span style="color: #0469ff">STEP 2  – INSTALLING MONDGO DB</span>
![MongoDB](https://img.shields.io/badge/MongoDB-0052cc?style=for-the-badge&logo=mongodb&logoColor=white)

Next, I set up MongoDB to store my book records. MongoDB saves data in a flexible, JSON-like format which works really well with Node.js.

First, I added `adv --keyserver` which is needed to handle security keys:

![adv --keyserver](./Images/2.0%20Installing%20MongoDB.png)

<blockquote style="border-left: 5px solid #d9534f;">
  <strong style="color: #d9534f;">⚠️ Error:</strong> 
  Again i was met with an error as a result of the code desparities. 
</blockquote>

<br>![MongoDB installation code](./Images/2.1%20Installing%20MongoDB%20contd.png)
<br>


I used an updated code command to eventually install MongoDB

![MongoDB updated code](./Images/2.2%20Installing%20MongoDB%20contd.png)

After that, I told my system where to find MongoDB packages: Then I updated the package list and installed MongoDB:

```bash
sudo apt-get update
sudo apt-get install -y mongodb-org
```


I checked that MongoDB was up and running:

```bash
sudo systemctl status mongod
```

![install body-parser](./Images/2.3%20Installing%20MongoDB%20contd.png)


Next, I installed `body-parser`, a package that helps the server read data sent from the browser:

```bash
sudo npm install body-parser
```
![install body-parser](./Images/2.4a%20Installing%20body-parser.png)

Then I created a project folder and set it up as an npm project:

```bash
mkdir Books && cd Books
npm init
```
![npm init](./Images/2.5%20Making%20directory.png)

<BR>

## <span style="color: #ff007F">STEP 3  – SETTING UP EXPRESS AND ROUTES</span> 
![Express.js](https://img.shields.io/badge/Express.js-ff007f?style=for-the-badge&logo=express&logoColor=white)


I installed Express (the web server framework) and Mongoose (which helps Node.js talk to MongoDB):

```bash
sudo npm install express mongoose
```
![SETTING UP EXPRESS](./Images/2.8%20Installing%20Express%20Mongoose.png)

Then I created a folder for my app logic and set up a routes file that handles GET, POST, and DELETE requests for books:

```bash
mkdir apps && cd apps
vi routes.js
```



Here's the code I added to `routes.js`:

```javascript
var Book = require('./models/book');

module.exports = function(app) { 
  app.get('/book', function(req, res) { 
    Book.find({}, function(err, result) { 
      if ( err ) throw err; 
      res.json(result); 
    }); 
  });  
  app.post('/book', function(req, res) { 
    var book = new Book( { 
      name:req.body.name, 
      isbn:req.body.isbn, 
      author:req.body.author, 
      pages:req.body.pages 
    }); 
    book.save(function(err, result) { 
      if ( err ) throw err; 
      res.json( { 
        message:"Successfully added book", 
        book:result 
      }); 
    }); 
  }); 
  app.delete("/book/:isbn", function(req, res) { 
    Book.findOneAndRemove({ isbn: req.params.isbn }, function(err, result) { 
      if ( err ) throw err; 
      res.json( { 
        message: "Successfully deleted the book", 
        book: result 
      }); 
    }); 
  }); 
  var path = require('path'); 
  app.get('*', function(req, res) { 
    res.sendfile(path.join(__dirname + '../public/index.html')); 
  }); 
}; 
```


I also created the database model in `models/book.js`. This file describes what a "book" looks like in the database.

<BR>

## <span style="color: #DD0031">STEP 4  – Access the routes with AngularJS </span> 
![Angular](https://img.shields.io/badge/Angular-DD0031?style=for-the-badge&logo=angular&logoColor=white)


<details>
<summary><b>About Angular</b></summary>
<br>


> * AngularJS provides a web framework for creating dynamic views in your web applications.
> * In this tutorial, we use AngularJS to connect our web page with Express and perform actions on our book register. 
</details>

<br>

In the `Books` directory I created a folder named `Public` and added a file to that folder `script.js` using "VIM"

```bash
mkdir public && cd public
vi script.js
```

![Creating Public Folder](./Images/4.0%20adding%20public%20directory%20and%20script.png)

Then I created the HTML file that the user actually sees:

```bash
vi index.html
```
![Creating the frontennd](./Images/4.1%20adding%20public%20directory%20and%20script.png)

The site was plain at first so i decided to give it a little redesign.  but here's my initial result. 

![Creating Public Folder](./Images/4.3%20Site%20Visible.png)

visible via my public ip:3300 on my local browser. 

<br>

># <span style="color: #fe5900">Now comes the redesign</span> 

To archive this i had to change the a couple things. 

1. <b>The Interface:</b> Changing the interface means that the index.html file had to be reworked to look better, given a sketch of what i wanted it to look like. 
2. <b>The Logic:</b> I added a couple line in my script.js to accomodate the html changes

<details>
<summary><b>Interface Redesign Code</b></summary>
<br> 

```html
<!doctype html>
<html ng-app="myApp" ng-controller="myCtrl">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Book Register</title>
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.6.4/angular.min.js"></script>
    <script src="script.js"></script>
    <style>
      * { box-sizing: border-box; margin: 0; padding: 0; }

      body {
        font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
        background: #f5f4f0;
        color: #1a1a1a;
        min-height: 100vh;
        padding: 2rem 1rem;
      }

      .page {
        max-width: 860px;
        margin: 0 auto;
      }

      /* Header */
      .header {
        display: flex;
        align-items: center;
        gap: 14px;
        margin-bottom: 2rem;
      }
      .header-icon {
        width: 48px;
        height: 48px;
        border-radius: 12px;
        background: #EEEDFE;
        display: flex;
        align-items: center;
        justify-content: center;
        font-size: 24px;
      }
      .header h1 {
        font-size: 22px;
        font-weight: 600;
        color: #1a1a1a;
      }
      .header p {
        font-size: 14px;
        color: #6b6b6b;
        margin-top: 2px;
      }

      /* Stat cards */
      .stats {
        display: grid;
        grid-template-columns: repeat(3, 1fr);
        gap: 12px;
        margin-bottom: 1.5rem;
      }
      .stat {
        background: #fff;
        border: 1px solid #e8e8e4;
        border-radius: 10px;
        padding: 1rem 1.25rem;
      }
      .stat-label {
        font-size: 12px;
        color: #6b6b6b;
        margin-bottom: 4px;
        text-transform: uppercase;
        letter-spacing: 0.04em;
      }
      .stat-value {
        font-size: 24px;
        font-weight: 600;
        color: #1a1a1a;
      }

      /* Cards */
      .card {
        background: #fff;
        border: 1px solid #e8e8e4;
        border-radius: 14px;
        padding: 1.5rem;
        margin-bottom: 1.5rem;
      }
      .card-title {
        font-size: 13px;
        font-weight: 600;
        color: #6b6b6b;
        text-transform: uppercase;
        letter-spacing: 0.05em;
        margin-bottom: 1.25rem;
        display: flex;
        align-items: center;
        gap: 8px;
      }

      /* Form */
      .form-grid {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 14px;
      }
      .form-group {
        display: flex;
        flex-direction: column;
        gap: 6px;
      }
      .form-group label {
        font-size: 13px;
        font-weight: 500;
        color: #444;
      }
      .form-group input {
        height: 40px;
        border: 1px solid #ddd;
        border-radius: 8px;
        padding: 0 12px;
        font-size: 14px;
        color: #1a1a1a;
        background: #fafafa;
        outline: none;
        transition: border-color 0.15s, box-shadow 0.15s;
      }
      .form-group input:focus {
        border-color: #7F77DD;
        background: #fff;
        box-shadow: 0 0 0 3px rgba(127, 119, 221, 0.15);
      }

      /* Add button */
      .btn-add {
        margin-top: 1.25rem;
        height: 40px;
        padding: 0 22px;
        background: #534AB7;
        color: #fff;
        border: none;
        border-radius: 8px;
        font-size: 14px;
        font-weight: 500;
        cursor: pointer;
        transition: background 0.15s;
      }
      .btn-add:hover { background: #3C3489; }
      .btn-add:active { transform: scale(0.98); }

      /* Book rows */
      .books-list {
        display: flex;
        flex-direction: column;
        gap: 10px;
      }
      .book-row {
        display: flex;
        align-items: center;
        gap: 14px;
        padding: 12px 14px;
        border: 1px solid #e8e8e4;
        border-radius: 10px;
        background: #fafafa;
        transition: border-color 0.15s;
      }
      .book-row:hover { border-color: #c9c7f0; }
      .book-spine {
        width: 38px;
        height: 38px;
        border-radius: 8px;
        background: #EEEDFE;
        display: flex;
        align-items: center;
        justify-content: center;
        font-size: 20px;
        flex-shrink: 0;
      }
      .book-info {
        flex: 1;
        min-width: 0;
      }
      .book-name {
        font-size: 14px;
        font-weight: 600;
        color: #1a1a1a;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
      }
      .book-meta {
        font-size: 12px;
        color: #6b6b6b;
        margin-top: 2px;
      }
      .book-badge {
        font-size: 12px;
        padding: 3px 10px;
        border-radius: 99px;
        background: #EEEDFE;
        color: #3C3489;
        font-weight: 500;
        flex-shrink: 0;
      }

      /* Delete button */
      .btn-del {
        height: 32px;
        padding: 0 12px;
        background: transparent;
        border: 1px solid #ddd;
        border-radius: 7px;
        font-size: 12px;
        color: #6b6b6b;
        cursor: pointer;
        flex-shrink: 0;
        transition: all 0.15s;
      }
      .btn-del:hover {
        background: #fff0f0;
        border-color: #f09595;
        color: #A32D2D;
      }

      /* Empty state */
      .empty {
        text-align: center;
        padding: 3rem 1rem;
        color: #9b9b9b;
        font-size: 14px;
      }
      .empty-icon {
        font-size: 36px;
        display: block;
        margin-bottom: 10px;
      }

      @media (max-width: 540px) {
        .form-grid { grid-template-columns: 1fr; }
        .stats { grid-template-columns: 1fr 1fr; }
      }
    </style>
  </head>
  <body>
    <div class="page">

      <!-- Header -->
      <div class="header">
        <div class="header-icon">📚</div>
        <div>
          <h1>Book Register</h1>
          <p>Add and manage your book collection</p>
        </div>
      </div>

      <!-- Stats -->
      <div class="stats">
        <div class="stat">
          <div class="stat-label">Total Books</div>
          <div class="stat-value">{{ books.length }}</div>
        </div>
        <div class="stat">
          <div class="stat-label">Authors</div>
          <div class="stat-value">{{ uniqueAuthors() }}</div>
        </div>
        <div class="stat">
          <div class="stat-label">Avg Pages</div>
          <div class="stat-value">{{ avgPages() }}</div>
        </div>
      </div>

      <!-- Add Book Form -->
      <div class="card">
        <div class="card-title">➕ Add a book</div>
        <div class="form-grid">
          <div class="form-group">
            <label>Title</label>
            <input type="text" ng-model="Name" placeholder="e.g. Atomic Habits">
          </div>
          <div class="form-group">
            <label>ISBN</label>
            <input type="text" ng-model="Isbn" placeholder="e.g. 978-0-525-57347-4">
          </div>
          <div class="form-group">
            <label>Author</label>
            <input type="text" ng-model="Author" placeholder="e.g. James Clear">
          </div>
          <div class="form-group">
            <label>Pages</label>
            <input type="number" ng-model="Pages" placeholder="e.g. 320">
          </div>
        </div>
        <button class="btn-add" ng-click="add_book()">+ Add Book</button>
      </div>

      <!-- Book List -->
      <div class="card">
        <div class="card-title">📖 Your collection</div>
        <div class="books-list">
          <div class="empty" ng-if="books.length === 0">
            <span class="empty-icon">📭</span>
            No books yet. Add your first one above.
          </div>
          <div class="book-row" ng-repeat="book in books">
            <div class="book-spine">📗</div>
            <div class="book-info">
              <div class="book-name">{{ book.name }}</div>
              <div class="book-meta">
                {{ book.author || 'Unknown author' }}
                <span ng-if="book.isbn"> · {{ book.isbn }}</span>
              </div>
            </div>
            <span class="book-badge" ng-if="book.pages">{{ book.pages }} pp</span>
            <button class="btn-del" data-ng-click="del_book(book)">🗑 Delete</button>
          </div>
        </div>
      </div>

    </div>
  </body>
</html>
```

</details>
<br>

> Following the redesign of the user interface, I encounted an issue were I was unable to add books to the register, so to solve this, I updated my routes file path to handle this by adding some functions to the existing route.

![Adding books error](./Images/5.0%20New%20interface%20error.png)


<details>
<summary><b>Updated Routes Function</b></summary>
<br>

```javascript

Updating the routes - to allow adding of books
cat > ~/Books/apps/routes.js
var Book = require('./models/book');
module.exports = function(app) {
  app.get('/book', function(req, res) {
    Book.find({}).then(function(result) {
      res.json(result);
    }).catch(function(err) {
      res.status(500).json({ message: err.message });
    });
  });

  app.post('/book', function(req, res) {
    var book = new Book({
      name: req.body.name,
      isbn: req.body.isbn,
      author: req.body.author,
      pages: req.body.pages
    });
    book.save().then(function(result) {
      res.json({
        message: "Successfully added book",
        book: result
      });
    }).catch(function(err) {
      res.status(500).json({ message: err.message });
    });
  });

  app.delete("/book/:isbn", function(req, res) {
    Book.findOneAndDelete(req.query).then(function(result) {
      res.json({
        message: "Successfully deleted the book",
        book: result
      });
    }).catch(function(err) {
      res.status(500).json({ message: err.message });
    });
  });

  var path = require('path');
  app.use(function(req, res) {
    res.sendFile(path.join(__dirname + '/public', 'index.html'));
  });
};
```
</details>
<br>

To make the UI feel more responsive and dynamic, i added a few lines to the `script.js` angular controller code. 

<details>
<summary><b>Script.js Update</b></summary>
<br>

```javascript
var app = angular.module('myApp', []);
app.controller('myCtrl', function($scope, $http) {
 $http( {
   method: 'GET',
   url: '/book'
 }).then(function successCallback(response) {
   $scope.books = response.data;
 }, function errorCallback(response) {
   console.log('Error: ' + response);
 });
 $scope.del_book = function(book) {
   $http( {
     method: 'DELETE',
     url: '/book/:isbn',
     params: {'isbn': book.isbn}
   }).then(function successCallback(response) {
     console.log(response);
   }, function errorCallback(response) {
     console.log('Error: ' + response);
   });
 };
 $scope.add_book = function() {
   var body = '{ "name": "' + $scope.Name +
   '", "isbn": "' + $scope.Isbn +
   '", "author": "' + $scope.Author +
   '", "pages": "' + $scope.Pages + '" }';
   $http({
     method: 'POST',
     url: '/book',
     data: body
   }).then(function successCallback(response) {
     console.log(response);
   }, function errorCallback(response) {
     console.log('Error: ' + response);
   });
 };
 $scope.uniqueAuthors = function() {
   if (!$scope.books) return 0;
   var authors = $scope.books.map(function(b) { return b.author; }).filter(Boolean);
   return new Set(authors).size;
 };
 $scope.avgPages = function() {
   if (!$scope.books) return '—';
   var withPages = $scope.books.filter(function(b) { return b.pages > 0; });
   if (!withPages.length) return '—';
   var total = withPages.reduce(function(s, b) { return s + parseInt(b.pages); }, 0);
   return Math.round(total / withPages.length);
 };
});
```

</details>

With all errors now fixed i was able to successfully add books to the register. 

<br>

<p align="center">
  <video src="Images/5.5 Book register video.mov" controls style="max-width: 100%; height: auto;">
  </video>
</p>

<blockquote style="border-left: 5px solid #d9534f;">
  <strong style="color: #d9534f;">⚠️ Important Notice:</strong> 
  Because i was having a bit of difficulty using VI to update. I tried my hands on Nano and CAT.  
</blockquote><br>

<b>I used the command below to update the routes & Script

```bash
cat ~/Books/apps/routes.js
cat > script.js
```

Fidings: cat overwrites previous file content. 
to test i ran 

```bash
cd ~/Books && node server.js
```

to get the site running with no errors. </b>