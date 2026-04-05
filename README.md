# Finance Tracking System

A clean, production-structured REST API built with **FastAPI** for managing personal and organizational financial records with role-based access control.

---

## Table of Contents

- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Setup & Running](#setup--running)
- [User Roles](#user-roles)
- [API Endpoints](#api-endpoints)
- [Analytics & Summaries](#analytics--summaries)
- [Export](#export)
- [Testing](#testing)
- [Design Decisions & Assumptions](#design-decisions--assumptions)

---

## Tech Stack

| Layer        | Choice                                   |
|--------------|------------------------------------------|
| Framework    | FastAPI                                  |
| ORM          | SQLAlchemy 2.0                           |
| Database     | SQLite (swappable to PostgreSQL via env) |
| Auth         | JWT (python-jose) + bcrypt (passlib)     |
| Validation   | Pydantic v2                              |
| Tests        | pytest                                   |

---

## Project Structure

```
finance_system/
├── app/
│   ├── main.py                  # FastAPI app factory & lifespan
│   ├── api/
│   │   ├── deps.py              # Auth dependencies & role guards
│   │   └── v1/
│   │       ├── router.py        # Top-level v1 router
│   │       └── endpoints/
│   │           ├── auth.py      # Register, login, /me
│   │           ├── users.py     # User management (admin)
│   │           ├── transactions.py  # CRUD + filters + pagination
│   │           └── analytics.py     # Summaries, breakdowns, export
│   ├── core/
│   │   ├── config.py            # Settings via pydantic-settings
│   │   ├── security.py          # Password hashing, JWT
│   │   └── exceptions.py        # Typed HTTP exceptions
│   ├── db/
│   │   ├── base.py              # SQLAlchemy Base + model imports
│   │   └── session.py           # Engine, SessionLocal, get_db()
│   ├── models/
│   │   ├── user.py              # User model + UserRole enum
│   │   └── transaction.py       # Transaction model + enums
│   ├── schemas/
│   │   ├── user.py              # Pydantic schemas for users
│   │   ├── transaction.py       # Pydantic schemas for transactions
│   │   └── analytics.py         # Analytics response schemas
│   └── services/
│       ├── user_service.py      # User business logic
│       ├── transaction_service.py  # Transaction business logic
│       ├── analytics_service.py    # Summary & analytics logic
│       └── export_service.py       # CSV / JSON export
├── scripts/
│   └── seed.py                  # Seed DB with demo users & data
├── tests/
│   └── test_core.py             # Unit tests for logic & validation
├── requirements.txt
├── .env.example
└── README.md
```

---

## Setup & Running

### 1. Clone & create virtual environment

```bash
git clone <repo-url>
cd finance_system
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Configure environment

```bash
cp .env.example .env
# Edit .env if needed (defaults work out of the box with SQLite)
```

### 4. Seed the database (optional but recommended)

```bash
python scripts/seed.py
```

This creates three demo users and ~6 months of sample transactions:

| Username | Password     | Role     |
|----------|--------------|----------|
| admin    | Admin1234    | admin    |
| analyst  | Analyst1234  | analyst  |
| viewer   | Viewer1234   | viewer   |

### 5. Run the server

```bash
uvicorn app.main:app --reload
```

API is now live at **http://localhost:8000**  
Interactive docs: **http://localhost:8000/docs**  
OpenAPI JSON: **http://localhost:8000/openapi.json**

---

## User Roles

| Permission                          | Viewer | Analyst | Admin |
|-------------------------------------|:------:|:-------:|:-----:|
| View own transactions               | ✅     | ✅      | ✅    |
| View ALL transactions               | ❌     | ✅      | ✅    |
| Filter & paginate transactions      | ✅     | ✅      | ✅    |
| Create transactions                 | ❌     | ✅      | ✅    |
| Update own transactions             | ❌     | ✅      | ✅    |
| Update any transaction              | ❌     | ❌      | ✅    |
| Delete own transactions             | ❌     | ✅      | ✅    |
| Delete any transaction              | ❌     | ❌      | ✅    |
| View financial summary              | ✅     | ✅      | ✅    |
| View category breakdown             | ✅     | ✅      | ✅    |
| View monthly totals (detailed)      | ❌     | ✅      | ✅    |
| Full analytics report               | ❌     | ✅      | ✅    |
| Export CSV / JSON                   | ❌     | ✅      | ✅    |
| Manage users (list/update/delete)   | ❌     | ❌      | ✅    |

---

## API Endpoints

All endpoints are prefixed with `/api/v1`. Protected endpoints require:
```
Authorization: Bearer <token>
```

### Auth

| Method | Path                    | Description                    | Auth |
|--------|-------------------------|--------------------------------|------|
| POST   | `/auth/register`        | Register a new user            | No   |
| POST   | `/auth/login`           | Login and receive JWT token    | No   |
| GET    | `/auth/me`              | Get current user profile       | Yes  |

### Transactions

| Method | Path                          | Description                          | Min Role |
|--------|-------------------------------|--------------------------------------|----------|
| GET    | `/transactions/`              | List transactions (filtered, paged)  | Viewer   |
| POST   | `/transactions/`              | Create a new transaction             | Analyst  |
| GET    | `/transactions/{id}`          | Get a single transaction             | Viewer   |
| PATCH  | `/transactions/{id}`          | Update a transaction                 | Analyst  |
| DELETE | `/transactions/{id}`          | Delete a transaction                 | Analyst  |

**Available query filters for `GET /transactions/`:**

| Parameter   | Type                     | Example                  |
|-------------|--------------------------|--------------------------|
| `type`      | `income` / `expense`     | `?type=expense`          |
| `category`  | See categories below     | `?category=food`         |
| `date_from` | `YYYY-MM-DD`             | `?date_from=2024-01-01`  |
| `date_to`   | `YYYY-MM-DD`             | `?date_to=2024-12-31`    |
| `min_amount`| Decimal                  | `?min_amount=100`        |
| `max_amount`| Decimal                  | `?max_amount=5000`       |
| `page`      | Integer (default: 1)     | `?page=2`                |
| `page_size` | Integer 1–100 (def: 20)  | `?page_size=50`          |

**Transaction categories:**

Income: `salary`, `freelance`, `investment`, `gift`  
Expense: `food`, `transport`, `utilities`, `entertainment`, `healthcare`, `education`, `shopping`, `rent`, `other`

### Analytics

| Method | Path                              | Description                                 | Min Role |
|--------|-----------------------------------|---------------------------------------------|----------|
| GET    | `/analytics/summary`              | Total income, expenses, balance             | Viewer   |
| GET    | `/analytics/category-breakdown`   | Per-category totals & percentages           | Viewer   |
| GET    | `/analytics/monthly-totals`       | Monthly income/expense for last N months    | Analyst  |
| GET    | `/analytics/detailed`             | Full report (summary + categories + months) | Analyst  |
| GET    | `/analytics/export/csv`           | Download transactions as CSV                | Analyst  |
| GET    | `/analytics/export/json`          | Download transactions as JSON               | Analyst  |

### Users (Admin Only)

| Method | Path             | Description             |
|--------|------------------|-------------------------|
| GET    | `/users/`        | List all users          |
| GET    | `/users/{id}`    | Get a user              |
| PATCH  | `/users/{id}`    | Update a user           |
| DELETE | `/users/{id}`    | Delete a user           |

---

## Analytics & Summaries

### `/analytics/summary`
```json
{
  "total_income": "15420.00",
  "total_expenses": "8930.50",
  "balance": "6489.50",
  "transaction_count": 142,
  "income_count": 24,
  "expense_count": 118,
  "average_income": "642.50",
  "average_expense": "75.68"
}
```

### `/analytics/category-breakdown?transaction_type=expense`
```json
[
  { "category": "food", "total": "2100.00", "count": 45, "percentage": 23.5 },
  { "category": "rent", "total": "3600.00", "count": 6, "percentage": 40.3 }
]
```

### `/analytics/monthly-totals`
```json
[
  {
    "year": 2024, "month": 11, "month_name": "November",
    "income": "5000.00", "expenses": "2800.00", "net": "2200.00"
  }
]
```

---

## Export

Analysts and Admins can export filtered transaction data:

```bash
# CSV export
GET /api/v1/analytics/export/csv?type=expense&date_from=2024-01-01

# JSON export
GET /api/v1/analytics/export/json?category=food
```

---

## Testing

```bash
pytest tests/ -v
```

Tests cover:
- Password hashing and JWT token roundtrip
- Schema validation (valid inputs, invalid inputs, edge cases)
- Role enforcement (viewer cannot write, analyst cannot edit others', admin can do all)

---

## Design Decisions & Assumptions

1. **SQLite by default** — Easily swappable to PostgreSQL by changing `DATABASE_URL` in `.env`. SQLAlchemy abstracts the difference.

2. **JWT over sessions** — Stateless auth scales better and is the modern standard for REST APIs.

3. **Viewer data scoping** — Viewers only ever see their own transactions. Analysts and Admins see everyone's. This is enforced in the service layer, not the route layer, so it's consistent regardless of how the service is called.

4. **Analyst can create/edit/delete their own** — The assumption is that an Analyst is a power-user who enters and manages records, while a Viewer is a stakeholder who reviews reports. Admins have unrestricted access.

5. **Amount as `Numeric(12,2)`** — Stored as a fixed-precision decimal in the DB and as Python `Decimal` throughout to avoid floating-point rounding issues in financial calculations.

6. **Soft deletes not used** — Simple hard deletes are used. A production system might prefer soft deletes; this is noted as a potential extension.

7. **`updated_at` auto-managed by SQLAlchemy** — The `onupdate` lambda ensures the field is always accurate without manual intervention.

8. **Pagination defaults to 20 per page, max 100** — A sensible default to prevent accidental large queries.

9. **Register endpoint allows any role** — In a real system, the register endpoint would always create `viewer` role users, and only an admin could elevate roles. For demo/testing purposes, the role can be passed at registration.
