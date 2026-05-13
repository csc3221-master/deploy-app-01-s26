# Transactions API
## Node + Express + MongoDB + Dev Container + Heroku

A classroom-style project demonstrating how to build a simple REST API using:

- Node.js
- Express
- MongoDB
- Docker
- VS Code Dev Containers
- Heroku Deployment

---

# Project Goal

Create a Transactions API for storing credit card transactions.

Transactions follow an append-only design:

- Transactions are never edited
- Transactions are never deleted
- Corrections are done using amendment transactions

---

# Data Model

Each transaction contains:

```json
{
  "id": "...",
  "creditCardNickname": "Costco Visa",
  "cardType": "Visa",
  "date": "2026-05-12",
  "amount": 42.75,
  "amendment": false,
  "comment": "Gas"
}
```

---

# Supported Card Types

- Visa
- Master
- Amex
- Discover
- Other

---

# API Endpoints

## POST

Create a transaction:

```text
POST /transactions
```

---

## GET

Get all transactions:

```text
GET /transactions
```

Get transaction by id:

```text
GET /transactions/:id
```

Get transactions by date:

```text
GET /transactions?date=YYYY-MM-DD
```

Get transactions by date range:

```text
GET /transactions?startDate=YYYY-MM-DD&endDate=YYYY-MM-DD
```

Get transactions by credit card nickname:

```text
GET /transactions?creditCardNickname=Costco Visa
```

---

# Important Design Decision

This API intentionally DOES NOT support:

- PUT
- PATCH
- DELETE

Financial records should not be destructively modified.

Instead:

```text
Incorrect Transaction
        ↓
Create Amendment Transaction
```

---

# Phase 0 - Pre Requirements

We will check this part together.

## MongoDB Atlas
Make sure you have your MongoDB Atlas account set up and ready to create a database.

## Heroku
To deploy our application we will need to do it using a tool called **Heroku CLI** (CLI stands for Command Line Interface). This tool will be used on the command line, this will look different for Mac and for Windows users.

Before installing the Heroku CLI make sure you have already created an account at Heroku and that you are able to see the dashboard.

> NOTE: You should be using a discounted Heroku taking advantage of the GitHub Student Developer Pack. Also, you were requested to setup your Heroku account **weeks** ago.

Install the Heroku Tools in your computer. Follow the instructions:
1. [Windows](https://devcenter.heroku.com/articles/heroku-cli)
2. [Mac](https://devcenter.heroku.com/articles/heroku-cli)



# Phase 1 - Create your Repo
Go to GitHub and create a public repo named transactions-api. Remember to set `Node` as your template for `.gitignore` Also, tell GitHub you want it to create a `README` file. Clone to your computer.

# Phase 2 - Create the Project Directory and File Structure

```text
transactions-api/
├── .devcontainer/
│   └── devcontainer.json
├── .env
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── package.json
├── server.js
├── seed.js
└── README.md
```

# Phase 3 — Create `.env`

Type the following into the `.env` file.

```env
PORT=3000
MONGODB_URI=mongodb://db:27017/transactionsdb
```

---

# Phase 4 — Create `docker-compose.yml`

```yaml
services:
  app:
    build: .
    working_dir: /app
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    env_file:
      - .env
    depends_on:
      - db
    command: sleep infinity

  db:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
```

---

# Phase 5 — Create `Dockerfile`

```dockerfile
FROM node:20-bookworm

# Install mongosh
RUN apt-get update && \
    apt-get install -y wget gnupg && \
    wget -qO - https://pgp.mongodb.com/server-7.0.asc | \
    gpg --dearmor -o /usr/share/keyrings/mongodb-server-7.0.gpg && \
    echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/debian bookworm/mongodb-org/7.0 main" \
    > /etc/apt/sources.list.d/mongodb-org-7.0.list && \
    apt-get update && \
    apt-get install -y mongodb-mongosh && \
    apt-get clean

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["sleep", "infinity"]
```

---

# Phase 6 — Create `.devcontainer/devcontainer.json`

To create the folder, you can do it with VS Code, from the Windows Explore / Finder or from bash as shown below:

```bash
mkdir .devcontainer
```

Create file `.devcontainer/devcontainer.json`:

```json
{
  "name": "Transactions API",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "app",
  "workspaceFolder": "/app",
  "shutdownAction": "stopCompose",

  "customizations": {
    "vscode": {
      "extensions": [
        "ms-azuretools.vscode-docker",
        "mongodb.mongodb-vscode",
        "dbaeumer.vscode-eslint"
      ]
    }
  },

  "forwardPorts": [3000, 27017],
  "remoteUser": "root"
}
```

---

# Reopen in Container
Now Visual Studio Code will detect the container option and you can go ahead and reload it. Once this is done, you will have access to the terminal!


# Phase 7 - Initialize the Project

Open the terminal from within VS Code. Make sure you are opening a container terminal and not your system's terminal.

Initialize npm:

```bash
npm init -y
```

Install dependencies:

```bash
npm install express mongodb dotenv
```

Install development dependency:

```bash
npm install --save-dev nodemon
```



# Phase 8 — Update `package.json`

```json
{
  "name": "transactions-api",
  "version": "1.0.0",
  "description": "Simple Express and MongoDB transactions API",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon --legacy-watch server.js",
    "seed": "node seed.js"
  },
  "dependencies": {
    "dotenv": "^16.4.7",
    "express": "^4.18.3",
    "mongodb": "^6.8.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.4"
  }
}
```

What did we need to update in `package.json`?

---

# Phase 9 — Create `server.js`

```js
require("dotenv").config();

const express = require("express");
const { MongoClient, ObjectId } = require("mongodb");

const app = express();

app.use(express.json());

const PORT = process.env.PORT || 3000;
const MONGODB_URI = process.env.MONGODB_URI;

if (!MONGODB_URI) {
  console.error("Missing MONGODB_URI in environment.");
  process.exit(1);
}

let db;
let transactionsCollection;

const validCardTypes = ["Visa", "Master", "Amex", "Discover", "Other"];

function isValidTransaction(body) {
  if (!body.creditCardNickname) return "creditCardNickname is required.";
  if (!body.cardType) return "cardType is required.";

  if (!validCardTypes.includes(body.cardType)) {
    return "Invalid card type.";
  }

  if (!body.date) return "date is required.";

  if (Number.isNaN(Date.parse(body.date))) {
    return "Invalid date.";
  }

  if (body.amount === undefined) {
    return "amount is required.";
  }

  if (typeof body.amount !== "number") {
    return "amount must be numeric.";
  }

  return null;
}

app.get("/", (req, res) => {
  res.json({
    message: "Transactions API is running"
  });
});

app.post("/transactions", async (req, res) => {
  const error = isValidTransaction(req.body);

  if (error) {
    return res.status(400).json({ error });
  }

  const transaction = {
    creditCardNickname: req.body.creditCardNickname,
    cardType: req.body.cardType,
    date: new Date(req.body.date),
    amount: req.body.amount,
    amendment: req.body.amendment === true,
    comment: req.body.comment || null,
    createdAt: new Date()
  };

  const result = await transactionsCollection.insertOne(transaction);

  res.status(201).json({
    ...transaction,
    id: result.insertedId
  });
});

app.get("/transactions", async (req, res) => {
  const filter = {};

  const { date, startDate, endDate, creditCardNickname } = req.query;

  if (creditCardNickname) {
    filter.creditCardNickname = creditCardNickname;
  }

  if (date) {
    const start = new Date(date);
    const end = new Date(date);

    end.setDate(end.getDate() + 1);

    filter.date = {
      $gte: start,
      $lt: end
    };
  }

  if (startDate || endDate) {
    filter.date = {};

    if (startDate) {
      filter.date.$gte = new Date(startDate);
    }

    if (endDate) {
      const end = new Date(endDate);
      end.setDate(end.getDate() + 1);

      filter.date.$lt = end;
    }
  }

  const transactions = await transactionsCollection
    .find(filter)
    .sort({ date: -1 })
    .toArray();

  res.json(transactions);
});

app.get("/transactions/:id", async (req, res) => {
  const { id } = req.params;

  if (!ObjectId.isValid(id)) {
    return res.status(400).json({
      error: "Invalid id."
    });
  }

  const transaction = await transactionsCollection.findOne({
    _id: new ObjectId(id)
  });

  if (!transaction) {
    return res.status(404).json({
      error: "Transaction not found."
    });
  }

  res.json(transaction);
});

async function startServer() {
  const client = new MongoClient(MONGODB_URI);

  await client.connect();

  db = client.db();

  transactionsCollection = db.collection("transactions");

  console.log("Connected to MongoDB");

  app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
  });
}

startServer().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

---

# Phase 10 — Create `seed.js`

```js
require("dotenv").config();

const { MongoClient } = require("mongodb");

const MONGODB_URI = process.env.MONGODB_URI;

if (!MONGODB_URI) {
  console.error("Missing MONGODB_URI in .env");
  process.exit(1);
}

const cardNicknames = [
  "Costco Visa",
  "Amazon Visa",
  "Travel Master",
  "Blue Amex",
  "Discover Cashback",
  "Everyday Card"
];

const cardTypes = [
  "Visa",
  "Master",
  "Amex",
  "Discover"
];

const comments = [
  "Gas",
  "Groceries",
  "Restaurant",
  "Coffee",
  "Flight",
  "Hotel",
  "Books",
  "Online Purchase",
  "Utilities",
  "Streaming Service",
  "Pharmacy",
  "Electronics"
];

function randomElement(array) {
  return array[Math.floor(Math.random() * array.length)];
}

function randomAmount(min, max) {
  return Number((Math.random() * (max - min) + min).toFixed(2));
}

function randomDateWithinLast90Days() {
  const now = new Date();

  const daysAgo = Math.floor(Math.random() * 90);

  const date = new Date(now);

  date.setDate(now.getDate() - daysAgo);

  return date;
}

function generateTransaction(index) {
  const amendment = Math.random() < 0.1;

  const amount = amendment
    ? -randomAmount(5, 250)
    : randomAmount(5, 250);

  return {
    creditCardNickname: randomElement(cardNicknames),

    cardType: randomElement(cardTypes),

    date: randomDateWithinLast90Days(),

    amount,

    amendment,

    comment: amendment
      ? "Amendment Transaction"
      : randomElement(comments),

    createdAt: new Date()
  };
}

async function seed() {
  const client = new MongoClient(MONGODB_URI);

  try {
    await client.connect();

    console.log("Connected to MongoDB");

    const db = client.db();

    const transactionsCollection =
      db.collection("transactions");

    const transactions = [];

    for (let i = 0; i < 100; i++) {
      transactions.push(generateTransaction(i));
    }

    const result =
      await transactionsCollection.insertMany(transactions);

    console.log(
      `Inserted ${result.insertedCount} transactions`
    );

  } catch (error) {
    console.error(error);
  } finally {
    await client.close();

    console.log("Disconnected from MongoDB");
  }
}

seed();
```

---

# Phase 11 - Run the Server

Inside the container terminal:

```bash
npm install
npm run dev
```

---

# Phase 12 — Test the API

Create a transaction:

```bash
curl -X POST http://localhost:3000/transactions \
  -H "Content-Type: application/json" \
  -d '{
    "creditCardNickname": "Costco Visa",
    "cardType": "Visa",
    "date": "2026-05-12",
    "amount": 42.75,
    "comment": "Gas"
  }'
```

Get all transactions:

```bash
curl http://localhost:3000/transactions
```

Get by date:

```bash
curl "http://localhost:3000/transactions?date=2026-05-12"
```

Get by date range:

```bash
curl "http://localhost:3000/transactions?startDate=2026-05-01&endDate=2026-05-31"
```

Get by card nickname:

```bash
curl "http://localhost:3000/transactions?creditCardNickname=Costco%20Visa"
```

You should also run all `GET` request tests on a browser, and test the `POST` request on Postman.

> Note: But the database has no data. Can we seed some data into it?
```bash
npm run seed
```
Now try again to do different GETs.

---

# Phase 13 — Prepare for Heroku

Create a `Procfile`:

```text
web: node server.js
```

Commit to git:

```bash
git add .
git commit -m "Initial transactions API"
```

---

# Phase 14 — Create MongoDB Atlas Database

Create a MongoDB Atlas cluster.

Copy the connection string.

Example:

```text
mongodb+srv://USERNAME:PASSWORD@cluster.mongodb.net/transactionsdb
```

---

# Phase 15 — Deploy to Heroku

The following commands need to be run inside your system terminal, **NOT** the containers terminal.

Login:

```bash
heroku login
```

Create app:

```bash
heroku create transactions-api
```

Set environment variable:

```bash
heroku config:set MONGODB_URI="mongodb+srv://USERNAME:PASSWORD@cluster.mongodb.net/transactionsdb"
```

Deploy:

```bash
git push heroku main
```

Open app:

```bash
heroku open
```

View logs:

```bash
heroku logs --tail
```

**Question** how can we run the seed program on Heroku?

# Phase 16

Test all API routes using browser and Postman on your deployed application.

---

# Possible Future Phases

Later improvements could include:

- Express routers
- MongoDB models
- validation middleware
- authentication
- pagination
- React frontend
- Docker production deployment
- CI/CD
- automated API testing

---

# Recommended Teaching Discussion Topics

- Why append-only systems matter
- Financial record immutability
- Environment variables
- Docker containers
- MongoDB document model
- REST API design
- Query filtering
- Local vs production databases
- Cloud deployment


# What to submit?

You will submit a URL to the repo you created for this activity. On this repo you will write a README file that will contain:
1. What were the new things you learned in this activity?
2. What is the purpose of the `seed.js` program?
3. What was the most dificult thing to do in this activity?
4. How would you say you were prudent in this assignment?
5. How would you say you need to be prudent when developing this kind of web applications?
6. URL of your deployed application as a link.
7. Screenshots of Postman making requests to your **deployed** application
8. Screenshotds
