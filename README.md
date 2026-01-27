[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/dZ5tEP45)
# Week 1: Project Structure - Domain vs Infrastructure

## Learning Objectives
By the end of this lesson, you will:
- Understand the separation between Domain and Infrastructure layers
- Recognize why architectural boundaries matter in data engineering
- Implement business logic independently from I/O operations

---

## What is Clean Architecture?

Clean Architecture, introduced by Robert C. Martin (Uncle Bob), organizes code into layers with a critical rule: **dependencies point inward**.

> "The overriding rule that makes this architecture work is The Dependency Rule. This rule says that source code dependencies can only point inwards. Nothing in an inner circle can know anything at all about something in an outer circle."
> 
> — Robert C. Martin, *Clean Architecture*

### The Two Layers We're Focusing On

**Domain Layer (Inner Circle)**
- Pure business logic
- No knowledge of databases, files, APIs, or external systems
- Highly testable - no mocks needed
- Example: "A leak alert is always critical"

**Infrastructure Layer (Outer Circle)**
- Handles I/O operations (databases, file systems, network)
- Depends on Domain layer (can import from it)
- Example: "Save this alert to SQLite"

**The Golden Rule**: Domain code never imports from Infrastructure. Infrastructure can import from Domain.

---

## Why This Matters in Data Engineering

In DE, you're constantly dealing with external systems: databases, message queues, cloud storage, APIs. Without clear boundaries:
- Business logic gets tangled with I/O code
- Testing requires spinning up databases
- Switching from Kafka to Kinesis means rewriting validation logic
- Bug fixes in one area break unrelated features

Clean separation means:
- Test business rules without touching a database
- Swap data sources without changing processing logic
- Understand code faster - know exactly where to look

---

## Today's System: Oil Well Alert Monitoring

You're building a monitoring system for oil wells that processes sensor readings and stores alerts.

### Data Model

**Heartbeats Table** (already implemented as example)
```
heartbeats (site_id TEXT, timestamp TEXT)
```

**Alerts Table** (you'll implement)
```
alerts (timestamp TEXT, site_id TEXT, alert_type TEXT, severity TEXT, latitude REAL, longitude REAL)
```

### Alert Types & Classification Rules

| Alert Type | Severity |
|------------|----------|
| PRESSURE | MODERATE |
| TEMPERATURE | MODERATE |
| LEAK | CRITICAL |
| ACOUSTIC | MODERATE |
| BLOCKAGE | CRITICAL |

---

## Template Code Walkthrough

### Project Structure
```
oil-well-monitoring/
├── domain/
│   ├── __init__.py
│   ├── models.py          # Data classes
│   └── processors.py      # Business logic
├── infrastructure/
│   ├── __init__.py
│   ├── database.py        # DB connection
│   └── repositories.py    # Data access
├── main.py
└── tests/
    └── test_week1.py
```

### Heartbeat System (Reference Implementation)

**Domain Layer** (`domain/processors.py`):
```python
def validate_heartbeat(site_id: str, timestamp: str) -> bool:
    """Pure business logic - no I/O"""
    if not site_id or len(site_id) < 3:
        return False
    if not timestamp:
        return False
    return True
```

**Infrastructure Layer** (`infrastructure/repositories.py`):
```python
def insert_heartbeat(conn, site_id: str, timestamp: str):
    """Handles database I/O"""
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO heartbeats (site_id, timestamp) VALUES (?, ?)",
        (site_id, timestamp)
    )
    conn.commit()
```

**Integration** (`main.py`):
```python
def process_heartbeat(conn, site_id: str, timestamp: str):
    if validate_heartbeat(site_id, timestamp):  # Domain
        insert_heartbeat(conn, site_id, timestamp)  # Infrastructure
```

Notice the flow: validate (domain) → persist (infrastructure).

---

## Your Assignment

Implement the alert monitoring system following the same pattern.

### What You Need to Do

1. **In `domain/processors.py`**: Implement `classify_alert(alert_type: str) -> str`
   - Takes an alert type (PRESSURE, TEMPERATURE, LEAK, ACOUSTIC, BLOCKAGE)
   - Returns "CRITICAL" or "MODERATE" based on the classification table above
   - This is pure business logic with no I/O

2. **In `infrastructure/repositories.py`**: Implement `insert_alert(...)`
   - Takes connection, timestamp, site_id, alert_type, severity, latitude, longitude
   - Inserts into the alerts table
   - This handles database I/O

3. **In `main.py`**: Complete `process_alert_reading(...)`
   - First, call `classify_alert()` to get the severity (Domain)
   - Then, call `insert_alert()` with all parameters including severity (Infrastructure)
   - Wire domain and infrastructure together

### Key Insight

Before you pass data to `insert_alert()`, you must add the `severity` field by calling the classification function. The classification logic belongs in the Domain layer because it's a business rule. The database insertion belongs in Infrastructure because it's I/O.

---

## Setup Instructions

1. Clone this repository
2. Install dependencies: `pip install -r requirements.txt`
3. Complete the TODOs in the code
4. Run tests: `pytest tests/test_week1.py -v`
5. Run the application: `python main.py`

### Expected Output

When you run `main.py`, you should see:
```
Processing heartbeat for SITE_001...
Heartbeat recorded successfully.

Processing alert: LEAK at SITE_001...
Alert recorded with severity: CRITICAL

Processing alert: TEMPERATURE at SITE_002...
Alert recorded with severity: MODERATE
```

---

## Discussion Questions

1. **Dependency Direction**: Why can Infrastructure import from Domain, but not vice versa? What breaks if we reverse this?

2. **Testability**: How would you test `classify_alert()` vs `insert_alert()`? Which one is easier to test and why?

3. **Real-World Scenario**: Imagine your company switches from SQLite to PostgreSQL. Which files would you need to change? Which files should remain untouched?

4. **Beyond Week 1**: What other business logic might belong in the Domain layer for this system? (Hint: think about validation rules, alert deduplication, priority scoring)

5. **Code Smell**: If you see `import sqlite3` at the top of a file in the `domain/` folder, what's wrong? How would you fix it?

---

## Next Week Preview

Week 2 will introduce **type validation** using Pydantic. You'll learn how to enforce data contracts at the boundaries between layers, catching errors before they propagate through your system.

---

## Success Criteria

- ✅ All tests pass
- ✅ Domain layer doesn't import from infrastructure
- ✅ Alert classification follows the business rules
- ✅ Alerts are persisted with correct severity

**Remember**: The goal isn't just to make tests pass. The goal is to internalize why we separate concerns and recognize where different types of code belong. Take your time understanding the heartbeat example before implementing alerts.