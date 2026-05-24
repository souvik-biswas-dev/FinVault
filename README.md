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

## License

MIT
