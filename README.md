<div id="top"></div>

<!-- PROJECT LOGO -->
<br />
<div align="center">

  <h1 align="center">Authentication server</h3>

  <p align="center">
    Meant as study material for P1 Lomé D-CLIC students
  </p>
</div>

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#database">Database</a></li>
        <li><a href="#server">Server</a></li>
      </ul>
    </li>
    <li><a href="#features">Features</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
  </ol>
</details>

<!-- ABOUT THE PROJECT -->
## About The Project

Minimal but production grade registration and authentication (login) backend in NodeJS and MySQL.

### Built With

* [![Node][Node.js]][Node-url]
* [![Express][Express.js]][Express-url]
* [![MySQL][MySQL]][MySQL-url]

<!-- GETTING STARTED -->
## Getting Started

If you haven't already, install mysql v8.0.29, node v16.13.2 and npm v8.1.2 or later versions.

### Database
1. Create a database called userDB using phpmyadmin or workbench:

  ```sql
  CREATE SCHEMA 'userDB';
  ```
2. Create a table called userTable with userId, user and password as columns:

  ```sql
  CREATE TABLE `userDB`.`userTable` (`userId` INT NOT NULL AUTO_INCREMENT , `user` VARCHAR(45) NOT NULL , `password` VARCHAR(100) NOT NULL , PRIMARY KEY (`userId`)) ENGINE = InnoDB;
  ```

3. Finally create a database user with CRUD privileges only. Credentials will be saved in .env file

  ```env
  DB_HOST = localhost
  DB_USER = ftf
  DB_DATABASE = userDB
  DB_PORT = 3306
  PORT = 3000
  DB_PASSWORD = "votremotdepasse"
  ```

### Server

1. Create a project folder and navigate to it before executing npm init command to initialize node project. --y stands for yes to all default setup:

  ```sh
  mkdir projectfoldername
  cd projectfoldername
  npm init --y
  ```
2. Next we install node packages for this project: "express", "mysql", "bcrypt" and "nodemon" and "dotenv" as devDependencies (Do some research on what they are used for)

```sh
npm i express mysql bcrypt
npm i nodemon dotenv --save-dev
```
3. Create a file called index.js

```js
//type cmd "npx nodemon index" to start server and connect to database
//ctrl + c to stop server
//node_modules and database credentials should NOT be tracked by git (use .gitignore and .env)
//database user shouldn't be root and should only have CRUD access protected by a password (good practice)

const express = require("express");
const app = express();
const mysql = require("mysql");
require("dotenv").config();

//database credentials hidden in .env
const DB_HOST = process.env.DB_HOST;
const DB_USER = process.env.DB_USER;
const DB_PASSWORD = process.env.DB_PASSWORD;
const DB_DATABASE = process.env.DB_DATABASE;
const DB_PORT = process.env.DB_PORT;

//connecting to database
const db = mysql.createPool({
  connectionLimit: 100,
  host: DB_HOST,
  user: DB_USER,
  password: DB_PASSWORD,
  database: DB_DATABASE,
  port: DB_PORT,
});

//information on connected database
db.getConnection((err, connection) => {
  if (err) throw err;
  console.log(
    `Base de données connectée:
    host: ${DB_HOST} - user: ${DB_USER}, 
    db: ${DB_DATABASE} - port: ${DB_PORT}, 
    attempt: ` + connection.threadId
  );
});

const port = process.env.PORT;
app.listen(port, () => console.log(`Serveur démarré sur le port ${port}...`));

//Signup (Adding a user)--------------------------------------------------------------
const bcrypt = require("bcrypt");
app.use(express.json());
//middleware to read req.body.<params>
//creating a new user + hashing their password (do some research on hashing)
app.post("/createUser", async (req, res) => {
  const user = req.body.name;
  const hashedPassword = await bcrypt.hash(req.body.password, 10);

  //connecting to db and inserting new user information
  db.getConnection(async (err, connection) => {
    if (err) throw err;
    const sqlSearch = "SELECT * FROM userTable WHERE user = ?";
    const search_query = mysql.format(sqlSearch, [user]);
    const sqlInsert = "INSERT INTO userTable VALUES (0,?,?)";
    const insert_query = mysql.format(sqlInsert, [user, hashedPassword]);

    //query results
    await connection.query(search_query, async (err, result) => {
      if (err) throw err;
      console.log("------> Résultats de la recherche");
      console.log(result.length);

      //if user already exists
      if (result.length != 0) {
        connection.release();
        console.log("------> Utilisateur existe déjà");
        res.status(409).send("Utilisateur existe déjà");
      } else {
        await connection.query(insert_query, (err, result) => {
          connection.release();
          if (err) throw err;
          console.log("--------> Nouvel utilisateur créé");
          console.log(result.insertId);
          res.status(201).send("Nouvel utilisateur créé");
        });
      }
    });
  });
});

//Login (Authenticate user)-----------------------------------------------------------
app.post("/login", (req, res) => {
  const user = req.body.name;
  const password = req.body.password;
  //connect to db and search for user
  db.getConnection(async (err, connection) => {
    if (err) throw err;
    const sqlSearch = "Select * from userTable where user = ?";
    const search_query = mysql.format(sqlSearch, [user]);
    await connection.query(search_query, async (err, result) => {
      connection.release();
      //if user does not exist in the database
      if (err) throw err;
      if (result.length == 0) {
        console.log("--------> Utilisateur n'existe pas");
        res.status(404).send("Utilisateur n'existe pas");
      } else {
        const hashedPassword = result[0].password;

        //get the hashedPassword from result and compare to user entry
        if (await bcrypt.compare(password, hashedPassword)) {
          console.log("---------> Connection réussie");
          res.send(`${user} est connecté(e)!`);
        } else {
          console.log("---------> Mot de passe incorrect");
          res.status(401).send("Mot de passe incorrect!");
        }
      }
    });
  });
});
```

4. You must also create a file called .gitignore (files and folders to be ignored by git) and another file called .env (for environment variables such as database credentials)

5. Run index.js

```sh
npx nodemon index
```
If all is correct you should see:
```sh
[nodemon] 2.0.19
[nodemon] to restart at any time, enter `rs`
[nodemon] watching path(s): *.*
[nodemon] watching extensions: js,mjs,json
[nodemon] starting `node index index.js`
Serveur démarré sur le port 3000...
Base de données connectée:
    host: localhost - user: ftf, 
    db: userDB - port: 3306, 
    attempt: 1
```

6. Test your backend application using postman

* To create a new user

```sh
localhost:3000/createUser
```

```json
{
    "name": "kofi",
    "password": "motdepassesolide"
}
```

* To login

```sh
localhost:3000/login
```

```json
{
    "name": "kofi",
    "password": "motdepassesolide"
}
```
<!-- FEATURES -->
## Features

- [x] C5: Créer une base de données
- [x] C6: Développer les composants d’accès aux données
- [x] C7: Développer la partie back-end d’une application web ou web mobile
- [x] Security
    - [x] Hashed password with bcrypt
    - [x] Use of environment variable and gitignore
    - [x] DB user with CRUD privileges only


<!-- LICENSE -->
## License

Distributed under the MIT License.


<!-- CONTACT -->
## Contact

Sethisbrothers - sodjisethkokou@gmail.com

<p align="right">(<a href="#top">back to top</a>)</p>


<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[Express.js]: https://img.shields.io/badge/Express-20232A?style=for-the-badge&logo=express&logoColor=61DAFB
[Express-url]: https://expressjs.com
[Node.js]: https://img.shields.io/badge/Node.js-35495E?style=for-the-badge&logo=nodedotjs&logoColor=4FC08D
[Node-url]: https://nodejs.org/en/
[MySQL]: https://img.shields.io/badge/MySQL-000000?style=for-the-badge&logo=mysql&logoColor=white
[MySQL-url]: https://www.mysql.com

