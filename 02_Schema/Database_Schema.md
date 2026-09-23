# ParaBank Database Schema

## Document Information

| Field | Details |
|---|---|
| Project | ParaBank Database Testing & SQL Validation |
| Doc ID | DIA-PARABANK-003 |
| Source Documents | 01_Requirement_Understanding_Document.md (v1.1) and UI_DB_Impact_Matrix.xlsx (v2.0 Final) |
| Author | Boopathi T |
| Database | HSQLDB |
| Database Name | parabank |
| Connection (default, per RUD) | `jdbc:hsqldb:hsql://localhost:9001/parabank` |
| Schema Source | Application behavior + tables/columns referenced in UI → DB Impact Matrix |
| Status | Phase 2 — Database Impact Analysis |

> **Note on connection port:** The RUD documents 9001 as the verified default HSQLDB port per the official ParaBank repo, with 9002 as the documented fallback only if a local conflict occurs (e.g. Intel Graphics Command Center on Windows). Use 9001 unless a conflict is actually hit.

---

## 1. Purpose

This document describes the database structure relevant to the ParaBank UI-to-database validation project. It is built directly from two sources:

1. **01_Requirement_Understanding_Document.md** — business rules and module scope (REQ-01 to REQ-07).
2. **UI_DB_Impact_Matrix.xlsx** — the actual table/column mapping and captured before/after evidence for functions that have already been tested.

Per the RUD, no official Parasoft schema/DDL was available, so table and column names here reflect what has been **referenced in the UI → DB Impact Matrix and observed in testing**, not a full `INFORMATION_SCHEMA` dump. Full column-level metadata (data types, nullability, PK/constraint names) is explicitly flagged as **outstanding** — to be captured by running the `INFORMATION_SCHEMA` queries in Section 6 against the live database.

---

## 2. Database

| Item | Value |
|---|---|
| Application | ParaBank |
| Database engine | HSQLDB |
| Database name | parabank |
| Connection (default) | `jdbc:hsqldb:hsql://localhost:9001/parabank` |
| Fallback connection (only if port conflict) | `jdbc:hsqldb:hsql://localhost:9002/parabank` |

---

## 3. Tables Referenced So Far

Sourced from the **UI_DB_Impact_Matrix** and **Table_Relationships** sheets of `UI_DB_Impact_Matrix.xlsx`.

### Core banking tables (in scope for this project)

- CUSTOMER
- ACCOUNT
- TRANSACTION

### Other tables referenced via verified relationships (outside core testing scope)

- COMPANY
- STOCK
- POSITIONS

The core UI-to-database validation in this project focuses on:

```
CUSTOMER → ACCOUNT → TRANSACTION
```

> Other tables that may exist in the running database (e.g. session/config tables) have not yet been confirmed against `INFORMATION_SCHEMA` and are intentionally **not listed here** to avoid guessing. They should be added only once confirmed against the live schema.

---

## 4. CUSTOMER

### Purpose
Stores customer information used by registration, login, and profile-related functions (REQ-01, REQ-02).

### Columns referenced in testing (from DB-001, DB-002, DB-015 in the Impact Matrix)

| Column | Purpose |
|---|---|
| ID | Unique customer identifier |
| FIRST_NAME | Customer first name |
| LAST_NAME | Customer last name |
| USERNAME | Login username |
| PASSWORD | Login credential |

*(Additional profile fields referenced generically by DB-015 "Update Contact Information" — not yet itemized; to be confirmed via `INFORMATION_SCHEMA`.)*

### Actual test data (from Actual_DB_Evidence sheet)

| Field | Value |
|---|---|
| ID | 12434 |
| USERNAME | sandyqa |

Customer record was created through the ParaBank application (Registration) and subsequently verified in the database, consistent with REQ-01.

---

## 5. ACCOUNT

### Purpose
Stores customer bank accounts and account balances (REQ-03, REQ-04, REQ-05).

### Columns referenced in testing (DB-003 through DB-008)

| Column | Purpose |
|---|---|
| ID | Unique account identifier |
| CUSTOMER_ID | Links account to CUSTOMER |
| TYPE | Account type (Checking/Savings) |
| BALANCE | Current account balance |

### Relationship

`ACCOUNT.CUSTOMER_ID` references `CUSTOMER.ID` (Verified — Table_Relationships sheet).

```
CUSTOMER.ID
     |
     v
ACCOUNT.CUSTOMER_ID
```

### Actual test accounts (from Actual_DB_Evidence sheet — current state)

| Account ID | Customer ID | Current Balance |
|---:|---:|---:|
| 13566 | 12434 | $265.50 |
| 13677 | 12434 | $200.00 |

### Captured balance movement (Transfer Funds + Bill Pay)

| Account | Before | After | Cause |
|---|---:|---:|---|
| 13566 | $415.50 | $315.50 | Funds Transfer Sent – $100 |
| 13677 | $100.00 | $200.00 | Funds Transfer Received – $100 |
| 13566 | $315.50 | $265.50 | Bill Payment to Bank of America – $50 |

This chain (415.50 → 315.50 via transfer → 265.50 via bill pay) matches the current balance shown for account 13566 above, and supports REQ-04 (exact-amount transfer between own accounts) and REQ-05 (bill pay debits the account).

---

## 6. TRANSACTION

### Purpose
Stores financial transaction records associated with customer accounts (REQ-07).

### Columns referenced in testing (DB-007, DB-009, DB-010 through DB-014)

| Column | Purpose |
|---|---|
| ID | Unique transaction identifier |
| ACCOUNT_ID | Account associated with the transaction |
| TYPE | Transaction type |
| DATE | Transaction date |
| AMOUNT | Transaction amount |
| DESCRIPTION | Transaction description |

### Relationship

`TRANSACTION.ACCOUNT_ID` references `ACCOUNT.ID` (Verified — Table_Relationships sheet).

```
ACCOUNT.ID
     |
     v
TRANSACTION.ACCOUNT_ID
```

### Actual transactions validated (from Actual_DB_Evidence sheet)

**Transfer Funds**

| Account | Amount | Description |
|---|---:|---|
| 13566 | $100 | Funds Transfer Sent |
| 13677 | $100 | Funds Transfer Received |

**Bill Payment**

| Account | Amount | Description |
|---|---:|---|
| 13566 | $50 | Bill Payment to Bank of America |

Both sets of records confirm REQ-07 (transaction logged and linked to the correct account) for the functions already tested.

---

## 7. COMPANY

### Purpose
Referenced by the STOCK table via a verified relationship. Not part of the core banking flows in scope for this project (see RUD — "Not testing" list has no stock-trading exclusion, but the Impact Matrix does not currently map any DB-### item to COMPANY, so no columns have been captured yet).

---

## 8. STOCK

### Purpose
Referenced via a verified relationship to COMPANY.

### Relationship

`STOCK.SYMBOL` references `COMPANY.SYMBOL` (Verified — Table_Relationships sheet).

```
COMPANY.SYMBOL
      |
      v
STOCK.SYMBOL
```

No column list has been captured for this table yet — it has not been exercised by any tested UI function in the Impact Matrix.

---

## 9. POSITIONS

### Purpose
Stores customer stock-position information.

### Relationship

`POSITIONS.CUSTOMER_ID` references `CUSTOMER.ID` (Verified — Table_Relationships sheet).

```
CUSTOMER.ID
     |
     v
POSITIONS.CUSTOMER_ID
```

Like COMPANY/STOCK, this table is outside the current banking UI-to-database testing scope defined in the RUD.

---

## 10. Core Database Structure

```
CUSTOMER
   |
   | CUSTOMER_ID
   v
ACCOUNT
   |
   | ACCOUNT_ID
   v
TRANSACTION
```

This supports the module scope defined in the RUD:

- Registration
- Login
- Account Overview
- Open New Account
- Transfer Funds
- Bill Pay
- Find Transactions
- Loan Request (persistence path not yet confirmed — see DB-019, Section 11)

---

## 11. Functions Mapped But Not Yet Executed

Per the Impact Matrix legend, the following are documented from ParaBank's official functionality but **not yet executed/verified** by hands-on testing (Status = "Remaining"):

| ID | Function | Table(s) | Expected Impact |
|---|---|---|---|
| DB-010 | Find Transactions | TRANSACTION | Existing records retrieved, no modification |
| DB-011 | Transaction Search by Amount | TRANSACTION | Matching transactions retrieved |
| DB-012 | Transaction Search by Date | TRANSACTION | Transactions for date retrieved |
| DB-013 | Transaction Search by Date Range | TRANSACTION | Transactions in range retrieved |
| DB-014 | Transaction Search by Month/Type | TRANSACTION | Matching transactions retrieved |
| DB-015 | Update Contact Information | CUSTOMER | Existing customer info updated |
| DB-016 | Account Details | ACCOUNT | Existing account info retrieved |
| DB-017 | Deposit Funds | ACCOUNT, TRANSACTION | Balance increases; transaction recorded |
| DB-018 | Withdraw Funds | ACCOUNT, TRANSACTION | Balance decreases; transaction recorded |
| DB-019 | Request Loan | Loan/application persistence — table not yet identified | Loan request processed; application state persisted |
| DB-020 | Logout | Session state | Session terminated; business tables unchanged |

DB-019 in particular is flagged in the RUD (REQ-06) as one the author is "less sure about" — the table(s) backing loan persistence have not yet been confirmed and should not be guessed at here.

---

## 12. Schema Validation Approach

Per the RUD, no official schema/DDL is published by Parasoft, so structure should be confirmed against the live database rather than assumed. Recommended query:

```sql
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    DATA_TYPE,
    IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'PUBLIC'
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

Outstanding items to fill in once this is run:

- Full column list (including any not yet touched by a tested UI function)
- Data types and nullability
- Primary key and constraint names
- Confirmation of any additional tables not yet referenced (e.g. NEWS, PARAMETER, SEQUENCE, or a loan-specific table) — none of these are asserted here because they have not appeared in the RUD or the Impact Matrix.

---

## 13. Project Status

| Item | Status |
|---|---|
| Database Impact Analysis (UI → DB mapping) | Complete for tested functions; remaining functions documented but not executed |
| Core schema identification (CUSTOMER, ACCOUNT, TRANSACTION) | Identified from tested functions |
| Foreign-key relationships | Verified (see `Foreign_Key_Relationships.md`) |
| Detailed column metadata (types, PK/constraints, full column list) | Outstanding — to be captured via `INFORMATION_SCHEMA` |
| Loan persistence table (REQ-06 / DB-019) | Not yet identified |
