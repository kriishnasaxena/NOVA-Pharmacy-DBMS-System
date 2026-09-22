# NOVA Pharmacy DBMS System

A production-aware Oracle PL/SQL schema for a Pharmacy Database Management System. This repository contains the DDL and PL/SQL code that implements secure user authentication, audit logging, inventory management, prescription processing, insurance claims, and safety features such as drug-interaction checks, automatic reorders, concurrency protections, and data integrity constraints.

## Overview

Designed a relational database using ER modeling and normalization. Implemented core features using SQL and PL/SQL to manage patients, prescriptions, and contracts — with security, auditability, and operational safety built in from the schema level up.

## Key Features

- Secure user authentication with per-user salted, peppered password hashing
- Central audit logging for sensitive table changes
- Referential integrity across patients, doctors, drugs, pharmacies, contracts, and prescriptions
- Drug-interaction checks that block critical prescription combinations
- Automatic reorder requests when stock drops below the minimum level
- Row-level concurrency protection for refills and insurance claims

## Design Goals

### Security
- Per-user salt + system pepper model for password hashing using `DBMS_CRYPTO` (SHA-256)
- Password policy enforcement (minimum length and basic complexity)
- Mapping from DB session to application user via a helper in `sec_pkg` (adaptable to application context)
- Avoids storing plaintext credentials

### Auditability
- Central `Audit_Log` table capturing operation type, record identifier, summaries for old/new values, user id, IP address, and timestamp
- Audit triggers for sensitive tables write `CHANGED_BY` as the mapped `System_Users.USER_ID` and capture `SYS_CONTEXT('USERENV','IP_ADDRESS')` where available
- Audit summaries are deliberately compact (limited `VARCHAR2` fields) to avoid unbounded CLOB growth

### Data Integrity
- Referential integrity across entities (patients, doctors, drugs, pharmacies, contracts, prescriptions)
- Canonical ordering and uniqueness for drug interactions to avoid symmetric duplicates
- Identity columns (`GENERATED ALWAYS AS IDENTITY`) for surrogate keys and `Prescription.PNO`
- Unique constraints for licenses and other business-critical fields

### Operational Safety
- Triggers and procedures do not issue `COMMIT` — the caller controls transaction boundaries to preserve atomicity and enable correct rollback behavior
- Reorder automation triggers on both `INSERT` and `UPDATE`, with a configurable reorder quantity per stock row
- Concurrency protections (row-level `SELECT ... FOR UPDATE`) for refill processing and claim handling
- Validations preventing critical drug interactions and over-dispensing

## Package Overview

**`sec_pkg`** — security helpers:

- `g_pepper` — application-level pepper constant (must be secured)
- `gen_salt()` — generates a 16-byte salt
- `hash_with_salt(pw, salt)` — returns SHA-256 of `salt || password || pepper`
- `validate_password_policy(password)` — enforces minimal password requirements
- `get_current_user_id()` — maps database session user to `System_Users.USER_ID` (adaptable to application context variable)

## Deployment

### Requirements
- Oracle Database supporting `DBMS_CRYPTO` and `IDENTITY` columns (Oracle 12c+ recommended)
- The executing schema needs rights to create tables, indexes, packages, triggers, and procedures

### How to Deploy
1. Open your SQL client connected to the target schema.
2. Run the script as a single file:
   - **SQL Developer:** paste into a worksheet and Run Script (F5), or run `@schema_v2_3.sql`
   - **SQL\*Plus / SQLcl:** `@/path/to/schema_v2_3.sql`
3. After deployment, compile packages/procedures if any are invalid:
   ```sql
   SELECT object_name, object_type, status FROM user_objects WHERE status = 'INVALID';
   ```
4. Create initial system users via `create_user(...)` and `COMMIT`.

## Operational Notes

### Transaction Handling
No `COMMIT` statements are present inside stored procedures or triggers. After invoking DML procedures, the application or DBA script must `COMMIT` or `ROLLBACK` explicitly.

### Pepper Management
`sec_pkg.g_pepper` is present in the package for convenience. For production you must:
- Replace the placeholder with a secure value
- Preferably store the pepper in an external secrets vault and inject it into the session, rather than keeping it in source control

### Session User Mapping
`sec_pkg.get_current_user_id()` maps `SYS_CONTEXT('USERENV','SESSION_USER')` to `System_Users.USERNAME`. In many application setups the DB session user will not equal the application username. For best practice:
- Set an application context with the current `USER_ID` at login (e.g., `DBMS_SESSION.SET_CONTEXT`), and modify `sec_pkg.get_current_user_id()` to read that context
- Alternatively, ensure the application uses a proxy or distinct DB session per application user and that the DB session username matches `System_Users.USERNAME`

## Usage Examples

**Create an application user** (remember to `COMMIT` after calling):
```sql
BEGIN
  create_user(
    p_user_id => 'user-0001',
    p_username => 'app_user',
    p_password => 'Secur3Pass',
    p_role => 'PHARMACIST',
    p_created_by => NULL
  );
END;
/
COMMIT;
```

**Authenticate a user:**
```sql
DECLARE
  v_result NUMBER;
  v_user_id VARCHAR2(36);
  v_role VARCHAR2(20);
BEGIN
  authenticate_user('app_user', 'Secur3Pass', v_result, v_user_id, v_role);
  DBMS_OUTPUT.PUT_LINE('Result=' || v_result || ' User=' || v_user_id || ' Role=' || v_role);
END;
/
```

**Add a prescription** (returns generated `PNO` in an OUT parameter; caller must `COMMIT`):
```sql
DECLARE
  v_pno NUMBER;
BEGIN
  add_prescription(
    p_pno => v_pno,
    p_p_date => SYSDATE,
    p_quantity => 10,
    p_d_aadhar_id => 'D12345678901',
    p_p_aadhar_id => 'P12345678901',
    p_trade_name => 'Paracetamol',
    p_dno => 1,
    p_dosage => '1 tablet twice daily',
    p_duration => 5,
    p_refills => 1
  );
  DBMS_OUTPUT.PUT_LINE('New PNO: ' || v_pno);
END;
/
COMMIT;
```

## Testing

Always deploy first to a development environment. Use the provided `test_data.sql` (or request one) to validate:

- User creation and authentication
- Audit entries include `CHANGED_BY` and `IP_ADDRESS`
- Drug interaction blocking for severe/critical interactions
- Auto-reorder request creation on `INSERT` and `UPDATE`
- Refill processing with concurrency protection

### Testing Checklist
- [ ] Create one `System_Users` account and verify `authenticate_user`
- [ ] Insert two `Drug` rows and create a critical `Drug_Interactions` entry; try creating a prescription that would trigger the interaction — it should be blocked
- [ ] Insert `Pharmacy_Stock` with quantity below `MIN_STOCK_LEVEL`; verify `Reorder_Requests` is created (audit entry logged) on `INSERT`
- [ ] Simulate concurrent `process_refill` calls (two sessions) and confirm row locks prevent over-dispensing
- [ ] Submit and process an insurance claim; ensure approval and rejection rules are enforced

## Backup and Rollback

- Keep a DDL export and a `drop_all.sql` cleanup script for clean redeployment cycles
- If the schema already contains objects from previous versions, drop or rename conflicting objects before applying the script

## Extensibility & Production Recommendations

- Move `g_pepper` out of the package and use a secure secrets manager
- Use `DBMS_SESSION`/application context to reliably set the current `USER_ID` in each session
- Consider moving detailed audit payloads to an append-only log storage (external log system) if you need full row-level snapshots
- Harden password policy (length, symbol requirements, expiration) and implement account lockout after multiple failed attempts
- Add role-based database proxies or application-level authorization to reduce reliance on session-user mapping

## Tech Stack

| Component | Technology |
|---|---|
| Database | Oracle Database (12c+) |
| Logic | PL/SQL (packages, triggers, procedures) |
| Security | `DBMS_CRYPTO` (SHA-256 salted + peppered hashing) |

## Authors

- Krishna Saxena — BITS Pilani, Hyderabad Campus
- Aryaman Narula — BITS Pilani, Hyderabad Campus
