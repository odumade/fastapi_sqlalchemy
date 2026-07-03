# FastAPI + SQLAlchemy Product API — Complete Build Tutorial

This guide walks you through rebuilding this project from scratch. By the end, you will have a working REST API that stores product data in a SQLite database using FastAPI, SQLAlchemy, and Pydantic.

---

## Table of Contents

1. [What You Are Building](#1-what-you-are-building)
2. [Architecture Overview](#2-architecture-overview)
3. [Prerequisites](#3-prerequisites)
4. [Step 1 — Create the Project Folder](#4-step-1--create-the-project-folder)
5. [Step 2 — Set Up a Virtual Environment](#5-step-2--set-up-a-virtual-environment)
6. [Step 3 — Install Dependencies](#6-step-3--install-dependencies)
7. [Step 4 — Create `database.py`](#7-step-4--create-databasepy)
8. [Step 5 — Create `models.py`](#8-step-5--create-modelspy)
9. [Step 6 — Create `schemas.py`](#9-step-6--create-schemaspy)
10. [Step 7 — Create `crud.py`](#10-step-7--create-crudpy)
11. [Step 8 — Create `main.py`](#11-step-8--create-mainpy)
12. [Step 9 — Run the Application](#12-step-9--run-the-application)
13. [Step 10 — Test the API](#13-step-10--test-the-api)
14. [How Data Flows Through the Project](#14-how-data-flows-through-the-project)
15. [Final Project Structure](#15-final-project-structure)
16. [Common Mistakes to Avoid](#16-common-mistakes-to-avoid)
17. [Next Steps and Extensions](#17-next-steps-and-extensions)

---

## 1. What You Are Building

This project is a **Product Management REST API**. Clients can:

| Method   | Endpoint                  | Action              |
|----------|---------------------------|---------------------|
| `POST`   | `/products`               | Create a product    |
| `GET`    | `/products`               | List all products   |
| `GET`    | `/products/{product_id}`  | Get one product     |
| `PUT`    | `/products/{product_id}`  | Update a product    |
| `DELETE` | `/products/{product_id}`  | Delete a product    |

Each product has:

- `id` — auto-generated integer primary key
- `name` — product name
- `description` — product description
- `price` — price as a float
- `stock` — available quantity as an integer

Data is persisted in a local **SQLite** file named `products.db`, created automatically on first run.

---

## 2. Architecture Overview

The project uses a **layered architecture**. Each file has a single responsibility:

```
HTTP Request
     │
     ▼
┌─────────┐     Validates input/output with Pydantic
│ main.py │     Defines routes, HTTP status codes
└────┬────┘
     │ calls
     ▼
┌─────────┐     Business logic & database operations
│ crud.py │
└────┬────┘
     │ reads/writes
     ▼
┌──────────┐    ┌───────────┐
│ models.py│◄───│database.py│  Connection & session management
└──────────┘    └───────────┘
     ▲
     │ shape of API data
┌──────────┐
│schemas.py│
└──────────┘
```

**Why separate files?**

- **`database.py`** — Database connection is configured once and reused everywhere.
- **`models.py`** — Defines how data is stored in the database (tables and columns).
- **`schemas.py`** — Defines how data looks in API requests and responses (validation).
- **`crud.py`** — Keeps database logic out of route handlers (Create, Read, Update, Delete).
- **`main.py`** — Only handles HTTP: routing, status codes, and wiring dependencies.

This separation makes the code easier to test, maintain, and extend.

---

## 3. Prerequisites

Before starting, make sure you have:

- **Python 3.10+** installed ([python.org](https://www.python.org/downloads/))
- A terminal (PowerShell, Command Prompt, or bash)
- A text editor or IDE (VS Code, Cursor, PyCharm, etc.)

Verify Python is installed:

```powershell
python --version
```

You should see something like `Python 3.13.x`.

---

## 4. Step 1 — Create the Project Folder

**Why:** Every project needs a dedicated directory to keep source code, dependencies, and generated files organized.

**What to do:**

```powershell
mkdir fastapi-sqlalchemy
cd fastapi-sqlalchemy
```

At this point your folder is empty. You will add files in a specific order based on what each file depends on.

**Final layout (for reference):**

```
fastapi-sqlalchemy/
├── .venv/              # Virtual environment (created in Step 2)
├── products.db         # SQLite database (auto-created on first run)
├── database.py         # Database connection
├── models.py           # SQLAlchemy table definitions
├── schemas.py          # Pydantic request/response models
├── crud.py             # Database operations
└── main.py             # FastAPI app and routes
```

> **Note:** This project does not use a `.env` file or external API integrations. The database URL is defined directly in `database.py`. For production apps, you would typically move secrets and configuration into environment variables.

---

## 5. Step 2 — Set Up a Virtual Environment

**Why:** A virtual environment isolates this project's packages from other Python projects on your machine. Without it, installing FastAPI here could conflict with packages elsewhere.

**What to do:**

```powershell
python -m venv .venv
```

Activate it:

**Windows (PowerShell):**
```powershell
.\.venv\Scripts\Activate.ps1
```

**Windows (Command Prompt):**
```cmd
.\.venv\Scripts\activate.bat
```

**macOS / Linux:**
```bash
source .venv/bin/activate
```

When active, your terminal prompt usually shows `(.venv)`.

**How it connects:** All `pip install` commands from this point install packages only inside `.venv`. When you run the app, you use the Python interpreter inside `.venv`.

---

## 6. Step 3 — Install Dependencies

**Why:** This project relies on four third-party libraries. You need them before writing any application code.

| Package      | Purpose                                              |
|--------------|------------------------------------------------------|
| `fastapi`    | Web framework for building the REST API              |
| `uvicorn`    | ASGI server that runs the FastAPI application        |
| `sqlalchemy` | ORM (Object-Relational Mapper) for database access   |
| `pydantic`   | Data validation (included with FastAPI, but explicit)|

**Install:**

```powershell
pip install fastapi uvicorn sqlalchemy pydantic
```

**Optional but recommended — save dependencies to a file:**

```powershell
pip freeze > requirements.txt
```

This creates a `requirements.txt` so others can recreate your environment with:

```powershell
pip install -r requirements.txt
```

**Dependencies between packages:**

- FastAPI uses **Pydantic** internally for request/response validation.
- FastAPI uses **Starlette** under the hood for HTTP handling (installed automatically).
- **Uvicorn** is the server; it imports your `main.py` and serves it.
- **SQLAlchemy** is independent; it only talks to the database.

---

## 7. Step 4 — Create `database.py`

**Why this file comes first:** Every other database-related file depends on the engine, session factory, and `Base` class defined here. No other project files are imported.

**Role:** Configures the SQLite database connection and provides a `get_db()` function that gives each HTTP request its own database session.

**Create `database.py` with the following content:**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

DATABASE_URL = "sqlite:///./products.db"

engine = create_engine(
    DATABASE_URL, connect_args={"check_same_thread": False}
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Line-by-line explanation

| Code | Explanation |
|------|-------------|
| `DATABASE_URL = "sqlite:///./products.db"` | Connection string. `sqlite:///` means SQLite. `./products.db` creates a file in the project root. |
| `create_engine(...)` | Creates the database engine — the core interface to the database. |
| `connect_args={"check_same_thread": False}` | SQLite normally allows only one thread per connection. FastAPI handles requests in different threads, so this disables that restriction. **Only needed for SQLite.** |
| `sessionmaker(...)` | Factory that creates database sessions. `autocommit=False` means you must call `commit()` to save changes. `autoflush=False` means changes are not sent to the DB until you commit or query. |
| `Base = declarative_base()` | Base class for all SQLAlchemy models. Every table class in `models.py` will inherit from this. |
| `get_db()` | A **generator** used by FastAPI's dependency injection. It opens a session, yields it to the route, then closes it when the request finishes — even if an error occurs. |

### How it connects to other files

- **`models.py`** imports `Base` to define table classes.
- **`main.py`** imports `engine`, `Base`, and `get_db`.
- **`crud.py`** receives a `Session` object created by `get_db()`.

---

## 8. Step 5 — Create `models.py`

**Why:** Models define the **database schema** — what tables exist and what columns they have. SQLAlchemy maps Python classes to SQL tables.

**Role:** Defines the `Product` table and its columns.

**Depends on:** `database.py` (for `Base`).

**Create `models.py`:**

```python
from sqlalchemy import Column, Integer, String, Float
from database import Base


class Product(Base):
    __tablename__ = "products"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    description = Column(String)
    price = Column(Float)
    stock = Column(Integer)
```

### Line-by-line explanation

| Code | Explanation |
|------|-------------|
| `class Product(Base)` | Inherits from `Base`, registering this class with SQLAlchemy's metadata. |
| `__tablename__ = "products"` | The actual SQL table name. |
| `id = Column(Integer, primary_key=True, index=True)` | Auto-incrementing primary key. `index=True` speeds up lookups by `id`. |
| `name = Column(String, index=True)` | Product name. Indexed because you may search or filter by name. |
| `description = Column(String)` | Free-text description. |
| `price = Column(Float)` | Decimal price stored as a floating-point number. |
| `stock = Column(Integer)` | Inventory count. |

### How it connects

- **`database.py`** provides `Base`.
- **`crud.py`** creates, reads, updates, and deletes `Product` instances.
- **`main.py`** calls `Base.metadata.create_all(bind=engine)` at startup, which reads all models that inherit from `Base` and creates the `products` table if it does not exist.

> **Important:** The table is not created when you write `models.py`. It is created when the app starts and `create_all()` runs in `main.py`. That is why `models.py` must be imported (directly or indirectly) before `create_all()` is called.

---

## 9. Step 6 — Create `schemas.py`

**Why:** API data and database data serve different purposes. You do not want clients to send an `id` when creating a product (the database generates it), but you do want to return `id` in responses. **Pydantic schemas** define and validate the shape of JSON in requests and responses.

**Role:** Defines three schema classes for input validation and output serialization.

**Depends on:** Only `pydantic` (no project files).

**Create `schemas.py`:**

```python
from pydantic import BaseModel


class ProductBase(BaseModel):
    name: str
    description: str
    price: float
    stock: int


class ProductCreate(ProductBase):
    pass


class ProductResponse(ProductBase):
    id: int

    class Config:
        from_attributes = True
```

### Line-by-line explanation

| Code | Explanation |
|------|-------------|
| `ProductBase` | Shared fields for both creating and returning products. |
| `ProductCreate(ProductBase)` | Schema for **incoming** POST/PUT requests. Inherits all fields from `ProductBase`. No `id` — clients must not set it. |
| `ProductResponse(ProductBase)` | Schema for **outgoing** responses. Adds `id` because the database assigns it after insert. |
| `class Config: from_attributes = True` | Tells Pydantic it can build this model from a SQLAlchemy object (e.g. `models.Product`) by reading its attributes, not just from a dict. Required for `response_model=ProductResponse` to work. |

### Models vs. Schemas — why both?

| | `models.Product` (SQLAlchemy) | `schemas.ProductCreate` (Pydantic) |
|---|---|---|
| Purpose | Database table mapping | API request validation |
| Used when | Reading/writing to DB | Parsing incoming JSON |
| Has `id` on create? | Yes (after insert) | No |

This separation is a deliberate design choice. It prevents clients from sending fields they should not control and keeps validation logic out of your database layer.

### How it connects

- **`main.py`** uses `ProductCreate` for request bodies and `ProductResponse` for responses.
- **`crud.py`** accepts `ProductCreate` and returns `models.Product` objects, which FastAPI converts to `ProductResponse`.

---

## 10. Step 7 — Create `crud.py`

**Why:** CRUD stands for **Create, Read, Update, Delete** — the four basic database operations. Putting them in a separate file keeps `main.py` thin and focused on HTTP concerns.

**Role:** All database logic for products lives here.

**Depends on:** `models.py`, `schemas.py`, and SQLAlchemy's `Session`.

**Create `crud.py`:**

```python
from sqlalchemy.orm import Session
import models
import schemas


def create_product(db: Session, product: schemas.ProductCreate):
    db_product = models.Product(**product.model_dump())
    db.add(db_product)
    db.commit()
    db.refresh(db_product)
    return db_product


def get_products(db: Session):
    return db.query(models.Product).all()


def get_product(db: Session, product_id: int):
    return db.query(models.Product).filter(models.Product.id == product_id).first()


def update_product(db: Session, product_id: int, updated: schemas.ProductCreate):
    product = get_product(db, product_id)
    if product:
        for key, value in updated.model_dump().items():
            setattr(product, key, value)
        db.commit()
        db.refresh(product)
    return product


def delete_product(db: Session, product_id: int):
    product = get_product(db, product_id)
    if product:
        db.delete(product)
        db.commit()
    return product
```

### Function-by-function explanation

#### `create_product`

```python
db_product = models.Product(**product.model_dump())
```
Converts the Pydantic schema to a dict (`model_dump()`) and unpacks it into a SQLAlchemy `Product` instance.

```python
db.add(db_product)    # Stage the new row
db.commit()           # Write to the database
db.refresh(db_product) # Reload to get the auto-generated id
```

Returns the saved product with its new `id`.

#### `get_products`

Returns every row in the `products` table as a list of `Product` objects.

#### `get_product`

Filters by `id` and returns the first match, or `None` if not found. `main.py` turns `None` into a 404 response.

#### `update_product`

1. Fetches the existing product.
2. If found, loops over the updated fields and applies them with `setattr`.
3. Commits and refreshes.
4. Returns the updated product, or `None` if the id did not exist.

#### `delete_product`

1. Fetches the product.
2. If found, deletes and commits.
3. Returns the deleted product object (or `None`). `main.py` checks the return value to decide whether to send 404.

### How it connects

- Receives `db: Session` from `get_db()` in `main.py`.
- Uses `models.Product` for database rows.
- Uses `schemas.ProductCreate` for validated input data.

---

## 11. Step 8 — Create `main.py`

**Why:** This is the **entry point** of the application. It creates the FastAPI app, registers routes, wires up the database, and starts serving HTTP requests.

**Role:** HTTP layer only — no raw SQL or direct table manipulation.

**Depends on:** All other project files.

**Create `main.py`:**

```python
from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy.orm import Session

import crud
import schemas
from database import Base, engine, get_db

Base.metadata.create_all(bind=engine)

app = FastAPI(title="Product API with SQLAlchemy")


@app.post("/products", response_model=schemas.ProductResponse)
def create_product(product: schemas.ProductCreate, db: Session = Depends(get_db)):
    return crud.create_product(db, product)


@app.get("/products", response_model=list[schemas.ProductResponse])
def get_products(db: Session = Depends(get_db)):
    return crud.get_products(db)


@app.get("/products/{product_id}", response_model=schemas.ProductResponse)
def get_product(product_id: int, db: Session = Depends(get_db)):
    product = crud.get_product(db, product_id)
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return product


@app.put("/products/{product_id}", response_model=schemas.ProductResponse)
def update_product(product_id: int, product: schemas.ProductCreate, db: Session = Depends(get_db)):
    updated = crud.update_product(db, product_id, product)
    if not updated:
        raise HTTPException(status_code=404, detail="Product not found")
    return updated


@app.delete("/products/{product_id}")
def delete_product(product_id: int, db: Session = Depends(get_db)):
    deleted = crud.delete_product(db, product_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="Product not found")
    return {"message": "Product deleted successfully"}
```

### Key concepts explained

#### Imports

```python
from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy.orm import Session
```

- `FastAPI` — creates the application instance.
- `Depends` — FastAPI's dependency injection system.
- `HTTPException` — raises proper HTTP error responses (e.g. 404).
- `Session` — type hint for the database session.

#### Table creation at startup

```python
Base.metadata.create_all(bind=engine)
```

Runs once when the module loads. SQLAlchemy inspects all models registered with `Base` and creates missing tables. Importing `crud` (which imports `models`) ensures the `Product` model is registered before this line runs.

#### The FastAPI app

```python
app = FastAPI(title="Product API with SQLAlchemy")
```

`app` is what Uvicorn serves. The `title` appears in the auto-generated docs at `/docs`.

#### Route decorators

Each `@app.get(...)`, `@app.post(...)`, etc. registers a URL path and HTTP method.

#### Dependency injection with `Depends(get_db)`

```python
def create_product(product: schemas.ProductCreate, db: Session = Depends(get_db)):
```

For every request to this endpoint, FastAPI:

1. Calls `get_db()` to open a database session.
2. Passes the session as `db`.
3. Closes the session when the request completes.

You never call `get_db()` manually — FastAPI handles it.

#### Request body validation

```python
product: schemas.ProductCreate
```

FastAPI reads the JSON body, validates it against `ProductCreate`, and returns `422 Unprocessable Entity` automatically if validation fails (e.g. missing `name` or wrong type).

#### Response serialization

```python
response_model=schemas.ProductResponse
```

FastAPI converts the returned SQLAlchemy object into JSON matching `ProductResponse`, excluding any extra fields.

#### Error handling

```python
if not product:
    raise HTTPException(status_code=404, detail="Product not found")
```

Returns a proper JSON error response instead of `null`.

### Import style — use absolute imports

This project uses a **flat layout** (all `.py` files in one folder). Use:

```python
import crud
from database import Base, engine, get_db
```

**Do not** use relative imports like `from . import crud` unless you wrap the project in a Python package with an `__init__.py` and run it as a module. With `uvicorn main:app`, relative imports will fail.

---

## 12. Step 9 — Run the Application

**Why:** Writing the code is not enough — you need an ASGI server to handle incoming HTTP connections. Uvicorn fills that role.

**From the project root, with the virtual environment activated:**

```powershell
uvicorn main:app --reload
```

| Part | Meaning |
|------|---------|
| `main` | The Python module (`main.py`) |
| `app` | The FastAPI instance inside that module |
| `--reload` | Auto-restart when you edit files (development only) |

**Expected output:**

```
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [...]
INFO:     Started server process [...]
INFO:     Application startup complete.
```

**What happens on first run:**

1. Python imports `main.py`.
2. `crud` is imported, which imports `models`, registering the `Product` table.
3. `Base.metadata.create_all()` creates `products.db` and the `products` table.
4. Uvicorn starts listening on port 8000.

**Interactive API documentation:**

Open your browser to:

- **Swagger UI:** [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)
- **ReDoc:** [http://127.0.0.1:8000/redoc](http://127.0.0.1:8000/redoc)

FastAPI generates these automatically from your route definitions and schemas.

---

## 13. Step 10 — Test the API

### Using Swagger UI (`/docs`)

1. Go to [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs).
2. Expand **POST /products**.
3. Click **Try it out**.
4. Enter a JSON body:

```json
{
  "name": "Wireless Mouse",
  "description": "Ergonomic wireless mouse",
  "price": 29.99,
  "stock": 150
}
```

5. Click **Execute**. You should get `200` with the created product including an `id`.

6. Try **GET /products** to list all products.
7. Try **GET /products/1** to fetch by id.
8. Try **PUT /products/1** to update.
9. Try **DELETE /products/1** to remove.

### Using PowerShell

**Create a product:**
```powershell
Invoke-RestMethod -Uri "http://127.0.0.1:8000/products" -Method POST -ContentType "application/json" -Body '{"name":"Keyboard","description":"Mechanical keyboard","price":89.99,"stock":50}'
```

**List products:**
```powershell
Invoke-RestMethod -Uri "http://127.0.0.1:8000/products" -Method GET
```

**Get one product:**
```powershell
Invoke-RestMethod -Uri "http://127.0.0.1:8000/products/1" -Method GET
```

### Using curl (Git Bash / macOS / Linux)

```bash
curl -X POST http://127.0.0.1:8000/products \
  -H "Content-Type: application/json" \
  -d '{"name":"USB Cable","description":"Type-C cable","price":12.99,"stock":200}'
```

---

## 14. How Data Flows Through the Project

### Example: Creating a product (`POST /products`)

```
Client sends JSON
       │
       ▼
┌──────────────────────────────────────────────────┐
│ main.py                                          │
│  1. FastAPI parses JSON → ProductCreate (Pydantic)│
│  2. Depends(get_db) → opens Session              │
│  3. Calls crud.create_product(db, product)     │
└──────────────────────┬───────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────┐
│ crud.py                                          │
│  4. Product(**product.model_dump())              │
│  5. db.add → db.commit → db.refresh              │
│  6. Returns models.Product instance              │
└──────────────────────┬───────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────┐
│ main.py                                          │
│  7. FastAPI serializes → ProductResponse (JSON) │
│  8. get_db() closes Session                      │
└──────────────────────┬───────────────────────────┘
                       ▼
              Client receives JSON
```

### Example: Product not found (`GET /products/999`)

```
main.py → crud.get_product(db, 999) → returns None
main.py → raise HTTPException(404)
Client receives: {"detail": "Product not found"}
```

---

## 15. Final Project Structure

```
fastapi-sqlalchemy/
│
├── .venv/                  # Virtual environment (do not commit)
│
├── products.db             # SQLite database (auto-generated)
│
├── database.py             # Step 4 — DB engine, session, get_db()
├── models.py               # Step 5 — Product table definition
├── schemas.py              # Step 6 — Pydantic validation models
├── crud.py                 # Step 7 — Create, Read, Update, Delete logic
├── main.py                 # Step 8 — FastAPI routes (entry point)
│
├── requirements.txt        # Optional — pinned dependencies
└── TUTORIAL.md             # This guide
```

### File creation order (summary)

| Order | File         | Depends on                          |
|-------|--------------|-------------------------------------|
| 1     | Folder       | —                                   |
| 2     | `.venv`      | —                                   |
| 3     | Dependencies | `.venv`                             |
| 4     | `database.py`| SQLAlchemy only                     |
| 5     | `models.py`  | `database.py`                       |
| 6     | `schemas.py` | Pydantic only                       |
| 7     | `crud.py`    | `models.py`, `schemas.py`           |
| 8     | `main.py`    | All of the above                    |

---

## 16. Common Mistakes to Avoid

### 1. Relative imports in a flat project

**Wrong:**
```python
from . import crud
from .database import engine
```

**Right:**
```python
import crud
from database import engine
```

Relative imports require a package structure. This project is flat and is run with `uvicorn main:app`.

### 2. Forgetting to import models before `create_all()`

If `models.py` is never imported, `create_all()` will not know about the `Product` table and no table will be created. Importing `crud` in `main.py` solves this because `crud` imports `models`.

### 3. Missing FastAPI imports

Using `FastAPI`, `Depends`, or `HTTPException` without importing them causes `NameError` at startup.

### 4. Not activating the virtual environment

If you run `uvicorn` without activating `.venv`, you may get `ModuleNotFoundError: No module named 'fastapi'`.

### 5. Using `id` in create requests

Clients should not send `id` when creating a product. The database auto-generates it. That is why `ProductCreate` does not include `id`.

### 6. Forgetting `from_attributes = True`

Without it, FastAPI cannot convert SQLAlchemy objects to Pydantic response models and you will get a serialization error.

---

## 17. Next Steps and Extensions

Once the basic API works, consider these improvements:

| Enhancement | What to change |
|-------------|----------------|
| Environment variables | Move `DATABASE_URL` to a `.env` file using `python-dotenv` |
| PostgreSQL | Change `DATABASE_URL` to `postgresql://user:pass@localhost/dbname` |
| Migrations | Add **Alembic** instead of `create_all()` for production schema changes |
| Pagination | Add `skip` and `limit` query params to `GET /products` |
| Authentication | Add JWT tokens with `python-jose` and `passlib` |
| Tests | Add `pytest` and `httpx` to test endpoints |
| Docker | Containerize with a `Dockerfile` for deployment |

---

## Quick Reference — Rebuild Checklist

- [ ] Create `fastapi-sqlalchemy` folder
- [ ] Create and activate virtual environment
- [ ] Install `fastapi`, `uvicorn`, `sqlalchemy`, `pydantic`
- [ ] Create `database.py`
- [ ] Create `models.py`
- [ ] Create `schemas.py`
- [ ] Create `crud.py`
- [ ] Create `main.py`
- [ ] Run `uvicorn main:app --reload`
- [ ] Open [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)
- [ ] Create, read, update, and delete a product

You now have a complete, working FastAPI + SQLAlchemy REST API and understand how every piece fits together.
