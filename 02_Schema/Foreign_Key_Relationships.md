# ParaBank Foreign Key Relationships

## Document Information

| Field | Details |
|---|---|
| Project | ParaBank Database Testing & SQL Validation |
| Doc ID | DIA-PARABANK-004 |
| Source Documents | UI_DB_Impact_Matrix.xlsx (sheet: Table_Relationships, Actual_DB_Evidence) and 01_Requirement_Understanding_Document.md (v1.1) |
| Author | Boopathi T |
| Database | HSQLDB |
| Database Name | parabank |
| Validation Source | Table_Relationships sheet — "Verified from database investigation" |
| Status | Phase 2 — Verified Relationships |

---

## 1. Purpose

This document records the foreign-key relationships identified for the ParaBank HSQLDB database, taken directly from the `Table_Relationships` sheet of `UI_DB_Impact_Matrix.xlsx`. These relationships are used to validate referential integrity and confirm that application-generated records are correctly connected between related tables, per REQ-03, REQ-04, and REQ-07 in the RUD.

---

## 2. Verified Foreign-Key Relationships

| Child Table | Child Column | Parent Table | Parent Column | Status |
|---|---|---|---|---|
| ACCOUNT | CUSTOMER_ID | CUSTOMER | ID | Verified |
| POSITIONS | CUSTOMER_ID | CUSTOMER | ID | Verified |
| STOCK | SYMBOL | COMPANY | SYMBOL | Verified |
| TRANSACTION | ACCOUNT_ID | ACCOUNT | ID | Verified |

(Order and content taken exactly as listed in the Table_Relationships sheet.)

---

## 3. CUSTOMER → ACCOUNT

### Relationship

```
CUSTOMER.ID
     |
     | referenced by
     v
ACCOUNT.CUSTOMER_ID
```

### Business Meaning

An account belongs to a customer (REQ-03).

### Actual test data (Actual_DB_Evidence sheet)

```
CUSTOMER
ID = 12434
USERNAME = sandyqa
```

Associated accounts (current state):

```
ACCOUNT
13566 → CUSTOMER_ID 12434 → Balance $265.50
13677 → CUSTOMER_ID 12434 → Balance $200.00
```

### Validation Query

```sql
SELECT
    A.ID AS ACCOUNT_ID,
    A.CUSTOMER_ID,
    C.ID AS CUSTOMER_ID,
    C.USERNAME
FROM ACCOUNT A
JOIN CUSTOMER C
    ON A.CUSTOMER_ID = C.ID
WHERE C.ID = 12434
ORDER BY A.ID;
```

### Expected Result

Every returned account should reference an existing customer (no orphans).

---

## 4. ACCOUNT → TRANSACTION

### Relationship

```
ACCOUNT.ID
     |
     | referenced by
     v
TRANSACTION.ACCOUNT_ID
```

### Business Meaning

Each financial transaction is associated with an account (REQ-04, REQ-05, REQ-07).

### Actual transactions validated (Actual_DB_Evidence sheet)

```
ACCOUNT 13566
    |
    +-- $100 Funds Transfer Sent      (before $415.50 → after $315.50)
    |
    +-- $50  Bill Payment to Bank of America (before $315.50 → after $265.50)

ACCOUNT 13677
    |
    +-- $100 Funds Transfer Received  (before $100.00 → after $200.00)
```

### Validation Query

```sql
SELECT
    T.ID AS TRANSACTION_ID,
    T.ACCOUNT_ID,
    A.ID AS ACCOUNT_ID,
    T.AMOUNT,
    T.DESCRIPTION
FROM TRANSACTION T
JOIN ACCOUNT A
    ON T.ACCOUNT_ID = A.ID
WHERE T.ACCOUNT_ID IN (13566, 13677)
ORDER BY T.ID DESC;
```

### Expected Result

Every transaction should reference an existing account (no orphans).

---

## 5. CUSTOMER → POSITIONS

### Relationship

```
CUSTOMER.ID
     |
     | referenced by
     v
POSITIONS.CUSTOMER_ID
```

### Business Meaning

Stock-position records are associated with customers. Outside the core banking scope defined in the RUD; relationship is recorded here because it was verified during the database investigation, but no positions data has been captured/tested yet.

### Status

Verified (relationship only — no test data captured).

---

## 6. COMPANY → STOCK

### Relationship

```
COMPANY.SYMBOL
     |
     | referenced by
     v
STOCK.SYMBOL
```

### Business Meaning

Stock records are associated with companies using the company/stock symbol. Outside the core banking scope defined in the RUD.

### Status

Verified (relationship only — no test data captured).

---

## 7. Complete Relationship Map

```
                 CUSTOMER
                /        \
               /          \
              v            v
          ACCOUNT       POSITIONS
             |
             |
             v
        TRANSACTION


          COMPANY
             |
             v
           STOCK
```

---

## 8. Core Banking Relationship

For the main database-testing scope defined in the RUD, the relevant chain is:

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

This relationship has been used to validate, with real captured evidence:

- Customer → Account ownership (customer 12434 → accounts 13566, 13677)
- Account → Transaction ownership (transfer and bill-pay transactions on 13566/13677)
- Account balance changes across a transfer and a bill payment
- Transfer Funds transactions (sent/received pair)
- Bill Payment transactions

---

## 9. Referential Integrity Checks

### 9.1 Orphan Accounts

```sql
SELECT A.*
FROM ACCOUNT A
LEFT JOIN CUSTOMER C
    ON A.CUSTOMER_ID = C.ID
WHERE C.ID IS NULL;
```

**Expected:** No rows.

### 9.2 Orphan Transactions

```sql
SELECT T.*
FROM TRANSACTION T
LEFT JOIN ACCOUNT A
    ON T.ACCOUNT_ID = A.ID
WHERE A.ID IS NULL;
```

**Expected:** No rows.

### 9.3 Orphan Positions

```sql
SELECT P.*
FROM POSITIONS P
LEFT JOIN CUSTOMER C
    ON P.CUSTOMER_ID = C.ID
WHERE C.ID IS NULL;
```

**Expected:** No rows.

### 9.4 Orphan Stock Records

```sql
SELECT S.*
FROM STOCK S
LEFT JOIN COMPANY C
    ON S.SYMBOL = C.SYMBOL
WHERE C.SYMBOL IS NULL;
```

**Expected:** No rows.

---

## 10. Foreign-Key Validation Objective

These checks confirm that:

1. Every account references a valid customer.
2. Every transaction references a valid account.
3. Every position references a valid customer.
4. Every stock record references a valid company.
5. No orphan records exist.
6. Application-generated data maintains the expected relationships — in line with the RUD's principle of validating application-generated database changes rather than performing isolated SQL exercises.

---

## 11. Project Status

| Item | Status |
|---|---|
| Foreign-key relationships (all 4) | Verified |
| CUSTOMER → ACCOUNT | Verified with actual test data (customer 12434, accounts 13566/13677) |
| ACCOUNT → TRANSACTION | Verified with actual test data (transfer + bill pay) |
| CUSTOMER → POSITIONS | Relationship verified; data not tested |
| COMPANY → STOCK | Relationship verified; data not tested |
| Referential-integrity SQL checks (9.1–9.4) | Prepared for execution — not yet run against live HSQLDB, no results captured |

> Distinction kept deliberately: relationships confirmed via the database investigation are marked **Verified**. The four orphan-record queries in Section 9 have been written but not executed — they stay **Prepared** until actually run and their results captured, so nothing here claims a pass that hasn't happened yet.
