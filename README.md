# FinVault

> A secure, full-stack personal finance ledger — track accounts, record transactions, and maintain a double-entry audit trail.

---

## Overview

FinVault is a full-stack web application for personal financial management. It combines a hardened REST API with a clean SvelteKit dashboard to give you complete visibility over your accounts and transactions. Every financial movement is recorded in a double-entry ledger, ensuring your books always balance.

---

## Tech Stack

| Layer      | Technology                          |
|------------|-------------------------------------|
| Frontend   | SvelteKit 2, TypeScript, Vite       |
| Backend    | Node.js, Express 5, MongoDB/Mongoose |
| Auth       | JWT (httpOnly cookies), bcrypt      |
| Infra      | Docker, Docker Compose              |
| Security   | Helmet, express-rate-limit, CORS    |
| Email      | Nodemailer                          |

---

## Features

- **Authentication** — Register, login, logout with JWT stored in secure httpOnly cookies
- **Account Management** — Create and manage multiple financial accounts
- **Transactions** — Record deposits, withdrawals, and transfers
- **Double-Entry Ledger** — Every transaction generates a corresponding ledger entry
- **Email Notifications** — Transactional emails via Nodemailer
- **Rate Limiting** — Separate, stricter limits on auth routes
- **Docker DB** — MongoDB + Mongo Express spun up with a single command

---

## Project Structure

```
finvault/
├── backend/                  # Express REST API
│   ├── src/
│   │   ├── config/           # Database connection
│   │   ├── controllers/      # Route handlers
│   │   ├── middleware/       # Auth, error, validation
│   │   ├── models/           # Mongoose schemas
│   │   ├── routes/           # Express routers
│   │   └── services/         # Email service
│   ├── server.js
│   └── .env.example
├── frontend/                 # SvelteKit dashboard
│   └── src/
│       ├── lib/              # API client, auth helpers, utils
│       └── routes/           # Pages: login, register, accounts, transactions
├── docker-compose.yml        # MongoDB + Mongo Express
└── README.md
```

---

## Getting Started

### Prerequisites

- Node.js 20+
- Docker & Docker Compose

### 1. Start the database

```bash
docker compose up -d
```

MongoDB runs on `localhost:27017`. Mongo Express (admin UI) at `http://localhost:8081`.

### 2. Configure the backend

```bash
cd backend
cp .env.example .env
# Fill in JWT_SECRET, MONGO_URI, SMTP settings
npm install
npm run dev
```

Backend runs on `http://localhost:3000`.

### 3. Start the frontend

```bash
cd frontend
npm install
npm run dev
```

Frontend runs on `http://localhost:5173`.

---

## API Reference

| Method | Endpoint                  | Description              | Auth |
|--------|---------------------------|--------------------------|------|
| POST   | `/api/auth/register`      | Create account           | —    |
| POST   | `/api/auth/login`         | Login, get JWT cookie    | —    |
| POST   | `/api/auth/logout`        | Clear auth cookie        | ✓    |
| GET    | `/api/accounts`           | List all accounts        | ✓    |
| POST   | `/api/accounts`           | Create account           | ✓    |
| GET    | `/api/accounts/:id`       | Get account detail       | ✓    |
| GET    | `/api/transactions`       | List transactions        | ✓    |
| POST   | `/api/transactions`       | Record transaction       | ✓    |
| GET    | `/api/transactions/:id`   | Get transaction detail   | ✓    |
| GET    | `/health`                 | Health check             | —    |

---

## Environment Variables

```env
PORT=3000
NODE_ENV=development
MONGO_URI=mongodb://admin:password123@localhost:27017/finvault?authSource=admin
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRES_IN=7d
CORS_ORIGIN=http://localhost:5173

# Nodemailer
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=you@example.com
SMTP_PASS=your_smtp_password
EMAIL_FROM=FinVault <no-reply@finvault.app>
```

---

## Low-Level Design

### Class Diagram

```mermaid
classDiagram
    class User {
        +ObjectId _id
        +String email
        +String name
        -String password
        -Boolean systemUser
        +Date createdAt
        +Date updatedAt
        +comparePassword(password) Boolean
        ~pre_save_hashPassword()
    }

    class Account {
        +ObjectId _id
        +ObjectId user
        +String status
        +String currency
        +Date createdAt
        +Date updatedAt
        +getBalance() Number
    }

    class Transaction {
        +ObjectId _id
        +String type
        +ObjectId fromAccount
        +ObjectId toAccount
        +String status
        +Number amount
        +String idempotencyKey
        +Date createdAt
        +Date updatedAt
    }

    class LedgerEntry {
        +ObjectId _id
        +ObjectId account
        +Number amount
        +ObjectId transaction
        +String type
        ~pre_update_prevent()
        ~pre_delete_prevent()
    }

    class AuthController {
        +userRegisterController(req, res)
        +userLoginController(req, res)
        +userLogoutController(req, res)
        +getMeController(req, res)
    }

    class AccountController {
        +createAccountController(req, res)
        +getAccountsController(req, res)
        +getAccountByIdController(req, res)
    }

    class TransactionController {
        +createTransactionController(req, res)
        +getTransactionsController(req, res)
        +getTransactionByIdController(req, res)
    }

    class AuthMiddleware {
        +protect(req, res, next)
    }

    class ValidationMiddleware {
        +validateRegister(req, res, next)
        +validateLogin(req, res, next)
        +validateTransaction(req, res, next)
    }

    class ErrorMiddleware {
        +globalErrorHandler(err, req, res, next)
    }

    class EmailService {
        +sendRegisterEmail(email, name)
        +sendTransactionEmail(email, name, amount, txnId)
        +sendTransactionFailureEmail(email, name, amount, key)
    }

    User "1" --> "many" Account : owns
    Account "1" --> "many" LedgerEntry : has
    Transaction "1" --> "2" LedgerEntry : generates
    Transaction "many" --> "1" Account : fromAccount
    Transaction "many" --> "1" Account : toAccount

    AuthController ..> User : creates / queries
    AuthController ..> EmailService : sends welcome email
    AccountController ..> Account : CRUD
    TransactionController ..> Transaction : CRUD
    TransactionController ..> LedgerEntry : creates entries
    TransactionController ..> Account : checks balance / status
    TransactionController ..> EmailService : sends notifications

    AuthMiddleware ..> User : verifies JWT
    AuthController ..> AuthMiddleware : protected routes
    AccountController ..> AuthMiddleware : protected routes
    TransactionController ..> AuthMiddleware : protected routes
```

---

### Sequence Diagrams

#### User Registration

```mermaid
sequenceDiagram
    participant C as Client
    participant VM as ValidationMiddleware
    participant AC as AuthController
    participant DB as MongoDB (User)
    participant ES as EmailService

    C->>VM: POST /api/auth/register {email, name, password}
    VM-->>C: 400 Validation Error (if invalid)
    VM->>AC: next()
    AC->>DB: findOne({email})
    DB-->>AC: null | User
    AC-->>C: 409 User already exists (if found)
    AC->>DB: create({email, name, password})
    Note over DB: pre-save hook hashes password (bcrypt)
    DB-->>AC: User document
    AC->>AC: jwt.sign({userId})
    AC-->>C: 201 {user, token} + Set-Cookie: token (httpOnly)
    AC--)ES: sendRegisterEmail() [fire & forget]
```

#### User Login

```mermaid
sequenceDiagram
    participant C as Client
    participant VM as ValidationMiddleware
    participant AC as AuthController
    participant DB as MongoDB (User)

    C->>VM: POST /api/auth/login {email, password}
    VM-->>C: 400 Validation Error (if invalid)
    VM->>AC: next()
    AC->>DB: findOne({email}).select("+password")
    DB-->>AC: null | User
    AC-->>C: 401 Incorrect email or password (if null)
    AC->>AC: user.comparePassword(password)
    AC-->>C: 401 Incorrect email or password (if mismatch)
    AC->>AC: jwt.sign({userId})
    AC-->>C: 200 {user, token} + Set-Cookie: token (httpOnly)
```

#### Create Transaction (Transfer)

```mermaid
sequenceDiagram
    participant C as Client
    participant AM as AuthMiddleware
    participant VM as ValidationMiddleware
    participant TC as TransactionController
    participant ADB as MongoDB-Account
    participant TDB as MongoDB-Transaction
    participant LDB as MongoDB-Ledger
    participant ES as EmailService

    C->>AM: POST /api/transactions - fromAccount, toAccount, amount, idempotencyKey
    AM->>ADB: findById userId from JWT
    ADB-->>AM: User document
    AM-->>C: 401 Unauthorized if no valid JWT
    AM->>TC: next()

    TC->>ADB: findById fromAccount and findById toAccount in parallel
    ADB-->>TC: fromUserAccount, toUserAccount
    TC-->>C: 404 Account not found if missing
    TC-->>C: 403 Forbidden if fromAccount not owned by user

    TC->>TDB: findOne by idempotencyKey
    TDB-->>TC: null or existingTransaction
    TC-->>C: 200/202/409 if duplicate key found

    TC-->>C: 400 Accounts must be ACTIVE if status check fails
    TC-->>C: 400 Cross-currency not supported if currency mismatch

    TC->>ADB: fromUserAccount.getBalance via ledger aggregate
    ADB-->>TC: balance
    TC-->>C: 400 Insufficient balance if balance less than amount

    TC->>TDB: create transaction with status PENDING
    TDB-->>TC: Transaction PENDING

    TC->>LDB: create DEBIT entry for fromAccount
    TC->>LDB: create CREDIT entry for toAccount

    alt Ledger writes succeed
        TC->>TDB: update transaction status to COMPLETED
        TC-->>C: 201 transaction COMPLETED
        TC--)ES: sendTransactionEmail fire and forget
    else Ledger write fails
        TC->>TDB: findOneAndUpdate status PENDING to FAILED
        TC--)ES: sendTransactionFailureEmail fire and forget
        TC-->>C: 500 Internal Server Error
    end
```

#### Get Account Balance

```mermaid
sequenceDiagram
    participant C as Client
    participant AM as AuthMiddleware
    participant AC as AccountController
    participant ADB as MongoDB (Account)
    participant LDB as MongoDB (Ledger)

    C->>AM: GET /api/accounts/:id
    AM-->>C: 401 Unauthorized (if no valid JWT)
    AM->>AC: next()
    AC->>ADB: findById(id)
    ADB-->>AC: Account | null
    AC-->>C: 404 Not found (if null)
    AC-->>C: 403 Forbidden (if account.user ≠ req.user._id)
    AC->>LDB: aggregate([match account, group DEBIT/CREDIT, project balance])
    LDB-->>AC: {balance}
    AC-->>C: 200 {account, balance}
```

---

## License

MIT
