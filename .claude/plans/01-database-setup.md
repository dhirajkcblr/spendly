# Implementation Plan: Database Setup (Step 1)

## Context

Spendly currently has only a stub `database/db.py` and a `app.py` with placeholder routes. Every future feature (auth, profile, expense CRUD) depends on a working SQLite data layer existing first. This step implements that foundation exactly per `.claude/specs/01-database-setup.md`: a `get_db()` / `init_db()` / `seed_db()` trio in `database/db.py`, wired into `app.py` startup, with no ORM and parameterized SQL only.

## Files to Change

### `database/db.py` (replace stub)

```python
import sqlite3
from datetime import date
from werkzeug.security import generate_password_hash

DB_PATH = "spendly.db"

def get_db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    return conn

def init_db():
    conn = get_db()
    conn.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL,
            created_at TEXT DEFAULT (datetime('now'))
        )
    """)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS expenses (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            amount REAL NOT NULL,
            category TEXT NOT NULL,
            date TEXT NOT NULL,
            description TEXT,
            created_at TEXT DEFAULT (datetime('now')),
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    """)
    conn.commit()
    conn.close()

def seed_db():
    conn = get_db()
    existing = conn.execute("SELECT 1 FROM users LIMIT 1").fetchone()
    if existing:
        conn.close()
        return

    password_hash = generate_password_hash("demo123")
    cursor = conn.execute(
        "INSERT INTO users (name, email, password_hash) VALUES (?, ?, ?)",
        ("Demo User", "demo@spendly.com", password_hash),
    )
    user_id = cursor.lastrowid

    today = date.today()
    sample_expenses = [
        (user_id, 45.50, "Food", today.replace(day=1).isoformat(), "Groceries"),
        (user_id, 12.00, "Transport", today.replace(day=3).isoformat(), "Bus pass"),
        (user_id, 89.99, "Bills", today.replace(day=5).isoformat(), "Electricity"),
        (user_id, 25.00, "Health", today.replace(day=8).isoformat(), "Pharmacy"),
        (user_id, 15.00, "Entertainment", today.replace(day=10).isoformat(), "Movie ticket"),
        (user_id, 60.00, "Shopping", today.replace(day=14).isoformat(), "Clothes"),
        (user_id, 8.75, "Food", today.replace(day=18).isoformat(), "Coffee"),
        (user_id, 20.00, "Other", today.replace(day=20).isoformat(), "Misc"),
    ]
    conn.executemany(
        "INSERT INTO expenses (user_id, amount, category, date, description) VALUES (?, ?, ?, ?, ?)",
        sample_expenses,
    )
    conn.commit()
    conn.close()
```

Notes:
- DB file name: `spendly.db` in the project root (already covered by `.gitignore`'s `*.db` rule).
- `date.today().replace(day=N)` keeps all 8 sample dates within the current month, one per category as required, in `YYYY-MM-DD` via `.isoformat()`.
- Every query uses `?` placeholders — no string formatting.
- `seed_db()` checks for any existing user row before inserting, so re-running `init_db()`/`seed_db()` never duplicates data.

### `app.py` (small addition)

- Add import: `from database.db import get_db, init_db, seed_db`
- Right after `app = Flask(__name__)`, add:
  ```python
  with app.app_context():
      init_db()
      seed_db()
  ```
- No route logic changes; existing placeholder routes stay untouched.

## Verification

1. Delete any stray `spendly.db` if present, then run `python app.py` (or `flask run`) from the project root and confirm it starts without errors.
2. Inspect the created DB: `sqlite3 spendly.db ".schema"` to confirm both tables and constraints exist; `select * from users;` and `select * from expenses;` to confirm 1 user + 8 expenses covering all 7 categories.
3. Restart the app a second time and re-check row counts — confirm no duplicate user/expenses were inserted.
4. Sanity-check constraints in a `python` shell using `get_db()`:
   - Inserting a second user with the same email raises `sqlite3.IntegrityError` (UNIQUE).
   - Inserting an expense with a nonexistent `user_id` raises `sqlite3.IntegrityError` (FOREIGN KEY), confirming `PRAGMA foreign_keys = ON` is active.
