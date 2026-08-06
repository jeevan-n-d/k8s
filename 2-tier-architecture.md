If an interviewer says:

> **"Design a 2-tier architecture from scratch and explain everything. Assume we are building a web application."**

They are expecting you to explain it from **0 to 100%**, not just draw boxes.

Let's build a **real-world 2-tier architecture**.

---

# Scenario

Let's build a simple Employee Management System.

Users can:

* Login
* Add employees
* View employees
* Delete employees

We'll use:

| Component          | Technology               |
| ------------------ | ------------------------ |
| Frontend + Backend | Node.js (or Spring Boot) |
| Database           | MySQL                    |
| Server             | EC2                      |
| Cloud              | AWS                      |

---

# What is a 2-Tier Architecture?

A 2-tier architecture has only **two layers**:

```
Tier 1
Application Layer

Tier 2
Database Layer
```

The application directly communicates with the database.

There is **no separate application server layer**, no load balancer, and no cache.

---

# Overall Architecture

```text
                 USER
                  │
          Browser (Chrome)
                  │
                  ▼
          Internet (HTTPS)
                  │
                  ▼
        AWS Security Group
                  │
                  ▼
      EC2 (Node.js Application)
                  │
      MySQL Port 3306
                  │
                  ▼
           RDS MySQL
```

---

# Step 1: User Opens Website

Suppose the user types:

```
http://13.234.xx.xx
```

or

```
https://employee.com
```

Browser sends

```
GET /
```

to your EC2 instance.

---

# Step 2: DNS (Optional)

If using a domain

```
employee.com
```

DNS resolves

```
employee.com

↓

13.234.xx.xx
```

Now browser knows where to send request.

---

# Step 3: Internet

Request travels through

```
Laptop

↓

ISP

↓

Internet

↓

AWS Network

↓

EC2
```

---

# Step 4: Security Group

Security Group acts like a firewall.

Example

Inbound Rules

```
HTTP 80

HTTPS 443

SSH 22
```

Only these ports are allowed.

Everything else is blocked.

---

# Step 5: EC2 Instance

Inside EC2

```
Ubuntu
```

Installed software

```
Node.js

Git

Nginx (optional)

Application
```

Folder

```
/home/ubuntu/app
```

Application files

```
app.js

package.json

.env

routes/

controllers/

models/

public/
```

---

# Example Folder

```
employee-app/

│

├── app.js

├── package.json

├── .env

├── routes/

│     employee.js

├── controllers/

│     employeeController.js

├── models/

│     employeeModel.js

├── views/

└── public/
```

---

# app.js

```javascript
const express = require("express");

const mysql = require("mysql2");

const app = express();

const db = mysql.createConnection({
 host:"database.amazonaws.com",
 user:"admin",
 password:"password",
 database:"employee"
});

db.connect();

app.get("/",(req,res)=>{

db.query("SELECT * FROM employee",(err,result)=>{

res.send(result);

});

});

app.listen(3000);
```

---

# package.json

```json
{
"name":"employee",

"dependencies":{

"express":"latest",

"mysql2":"latest"

}

}
```

---

# .env

```
DB_HOST=database.amazonaws.com

DB_USER=admin

DB_PASSWORD=password

DB_NAME=employee
```

---

# Database Layer

Instead of installing MySQL on EC2

we use

```
Amazon RDS
```

Engine

```
MySQL
```

---

# Database Table

Employee

```
ID

NAME

EMAIL

SALARY
```

SQL

```sql
CREATE TABLE employee(

id INT PRIMARY KEY,

name VARCHAR(100),

email VARCHAR(100),

salary INT

);
```

---

# User clicks

```
View Employees
```

Browser sends

```
GET /employees
```

---

# Node.js receives

```
GET /employees
```

Controller

```javascript
SELECT * FROM employee;
```

---

# SQL executed

Database returns

```
1 John

2 Alice

3 Bob
```

---

# Application converts

```
JSON
```

Browser receives

```json
[
{

"id":1,

"name":"John"

}
]
```

Browser displays table.

---

# Complete Request Flow

```
User

↓

Browser

↓

Internet

↓

Security Group

↓

EC2

↓

Node.js

↓

MySQL Driver

↓

Amazon RDS

↓

SQL Execution

↓

Result

↓

Node.js

↓

Browser

↓

User
```

---

# Required AWS Resources

### 1. VPC

```
Default VPC
```

or custom VPC.

---

### 2. Public Subnet

EC2 lives here.

---

### 3. Private Subnet

Database should ideally live here.

---

### 4. Internet Gateway

Allows internet access.

---

### 5. Route Table

Routes

```
0.0.0.0/0

↓

Internet Gateway
```

---

### 6. Security Groups

EC2

Allow

```
22

80

443
```

RDS

Allow

```
3306

Source

EC2 Security Group
```

Never allow

```
3306

0.0.0.0/0
```

---

# Deployment Steps

Launch EC2

↓

Install Git

↓

Clone Repository

↓

Install Node

↓

npm install

↓

Configure .env

↓

Start App

↓

Connect to RDS

↓

Done

---

# Advantages

* Easy to develop.
* Low cost.
* Good for small applications.
* Simple deployment.

---

# Disadvantages

* Single EC2 instance is a single point of failure.
* Difficult to scale.
* No load balancing.
* Application downtime if EC2 fails.
* Database can become a bottleneck.

---

# If the interviewer asks, "What files are needed?"

For a Node.js application:

```
employee-app/
├── app.js
├── package.json
├── package-lock.json
├── .env
├── routes/
├── controllers/
├── models/
├── public/
├── views/
└── README.md
```

If you're using Java Spring Boot instead:

```
employee-app/
├── src/main/java/
├── src/main/resources/
│   ├── application.properties
│   └── static/
├── pom.xml
└── mvnw
```

---

## Interview Tip

When explaining a design, don't just list AWS services. Walk through the **request flow**:

> "The user enters the website URL in the browser. DNS resolves the domain to the EC2 instance. The request passes through the Security Group and reaches the Node.js application running on EC2. The application processes the request, connects securely to the MySQL database on Amazon RDS, executes SQL queries, receives the results, and returns an HTTP response to the browser."

That kind of end-to-end explanation is exactly what interviewers are looking for because it demonstrates that you understand **how all the components work together**, not just what each service does.
