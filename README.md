# 📚 Library Management System — SQL Project

A complete relational database project simulating a real-world **Library Management System**, built entirely in SQL. It covers database design, CRUD operations, CTAS (Create Table As Select), stored procedures, and advanced analytical queries.

---

## 🗂️ Project Overview

| | |
|---|---|
| **Project Title** | Library Management System |
| **Level** | Intermediate |
| **Database** | `library_db` (PostgreSQL) |
| **Tables** | `branch`, `employees`, `members`, `books`, `issued_status`, `return_status` |

This project models the day-to-day operations of a library — issuing and returning books, tracking members and employees across branches, and generating business insights — using only SQL.

---

## 🎯 Objectives

1. **Database Design** — Build a normalized schema with proper primary/foreign key relationships.
2. **CRUD Operations** — Insert, update, delete, and query records across all tables.
3. **CTAS (Create Table As Select)** — Generate derived summary tables from query results.
4. **Stored Procedures** — Automate business logic (issuing/returning books) with reusable PL/pgSQL procedures.
5. **Advanced Analysis** — Answer real business questions: overdue books, branch performance, top employees, high-risk members, and fines.

---

## 🧱 Database Schema

```sql
CREATE TABLE branch (
    branch_id       VARCHAR(10) PRIMARY KEY,
    manager_id      VARCHAR(10),
    branch_address  VARCHAR(30),
    contact_no      VARCHAR(15)
);

CREATE TABLE employees (
    emp_id      VARCHAR(10) PRIMARY KEY,
    emp_name    VARCHAR(30),
    position    VARCHAR(30),
    salary      DECIMAL(10,2),
    branch_id   VARCHAR(10) REFERENCES branch(branch_id)
);

CREATE TABLE members (
    member_id       VARCHAR(10) PRIMARY KEY,
    member_name     VARCHAR(30),
    member_address  VARCHAR(30),
    reg_date        DATE
);

CREATE TABLE books (
    isbn           VARCHAR(50) PRIMARY KEY,
    book_title     VARCHAR(80),
    category       VARCHAR(30),
    rental_price   DECIMAL(10,2),
    status         VARCHAR(10),
    author         VARCHAR(30),
    publisher      VARCHAR(30)
);

CREATE TABLE issued_status (
    issued_id          VARCHAR(10) PRIMARY KEY,
    issued_member_id   VARCHAR(30) REFERENCES members(member_id),
    issued_book_name   VARCHAR(80),
    issued_date        DATE,
    issued_book_isbn   VARCHAR(50) REFERENCES books(isbn),
    issued_emp_id      VARCHAR(10) REFERENCES employees(emp_id)
);

CREATE TABLE return_status (
    return_id         VARCHAR(10) PRIMARY KEY,
    issued_id         VARCHAR(30),
    return_book_name  VARCHAR(80),
    return_date       DATE,
    return_book_isbn  VARCHAR(50) REFERENCES books(isbn)
);
```

**Entity relationships:** `members` and `employees` are linked to `issued_status` (who borrowed what, and who processed it); `books` is linked to both `issued_status` and `return_status`; `employees` are linked to `branch` (which branch they work at, and who manages it).

---

## 📁 Repository Structure

| File | Description |
|---|---|
| `app_library.sql` | Schema definitions + full task list (project brief/prompts) |
| `insert_queries.sql` | Seed data — members, branches, employees, books, initial issued/returned records |
| `insert_queries2.sql` | Additional seed data (recent issues) + `book_quality` column added to `return_status` |
| `solution_day_1.sql` | Solutions to Tasks 1–12 (CRUD, CTAS, basic analysis) |
| `SQL_PROJECT_02.sql` | Alternate pass at Tasks 1–5 (CRUD basics) |
| `lms_project_advanced_solution_2.sql` | Solutions to Tasks 13–19 (overdue books, stored procedures, branch reports, active members, top employees) |
| `SQL_PROJECT_02_Part_02.sql` | Alternate pass at Tasks 13–19 (advanced operations) |

---

## 🔧 Task Breakdown & Sample Queries

### CRUD Operations

**Task 1 — Add a new book**
```sql
INSERT INTO books(isbn, book_title, category, rental_price, status, author, publisher)
VALUES ('978-1-60129-456-2', 'To Kill a Mockingbird', 'Classic', 6.00, 'yes', 'Harper Lee', 'J.B. Lippincott & Co.');
```

**Task 2 — Update a member's address**
```sql
UPDATE members
SET member_address = '125 Main St'
WHERE member_id = 'C101';
```

**Task 3 — Delete an issued record**
```sql
DELETE FROM issued_status
WHERE issued_id = 'IS121';
```

**Task 4 — Books issued by a specific employee**
```sql
SELECT * FROM issued_status
WHERE issued_emp_id = 'E101';
```

**Task 5 — Members who issued more than one book**
```sql
SELECT ist.issued_emp_id, e.emp_name
FROM issued_status ist
JOIN employees e ON e.emp_id = ist.issued_emp_id
GROUP BY 1, 2
HAVING COUNT(ist.issued_id) > 1;
```

### CTAS & Analysis

**Task 6 — Book issue-count summary table**
```sql
CREATE TABLE book_cnts AS
SELECT b.isbn, b.book_title, COUNT(ist.issued_id) AS no_issued
FROM books b
JOIN issued_status ist ON ist.issued_book_isbn = b.isbn
GROUP BY 1, 2;
```

**Task 8 — Rental income by category**
```sql
SELECT b.category, SUM(b.rental_price) AS total_income, COUNT(*) AS times_issued
FROM books b
JOIN issued_status ist ON ist.issued_book_isbn = b.isbn
GROUP BY 1;
```

**Task 12 — Books not yet returned**
```sql
SELECT DISTINCT ist.issued_book_name
FROM issued_status ist
LEFT JOIN return_status rs ON ist.issued_id = rs.issued_id
WHERE rs.return_id IS NULL;
```

### Advanced Operations

**Task 13 — Members with overdue books (> 30 days)**
```sql
SELECT ist.issued_member_id, m.member_name, bk.book_title,
       ist.issued_date, CURRENT_DATE - ist.issued_date AS overdue_days
FROM issued_status ist
JOIN members m ON m.member_id = ist.issued_member_id
JOIN books bk ON bk.isbn = ist.issued_book_isbn
LEFT JOIN return_status rs ON rs.issued_id = ist.issued_id
WHERE rs.return_date IS NULL
  AND (CURRENT_DATE - ist.issued_date) > 30
ORDER BY 1;
```

**Task 15 — Branch performance report**
```sql
CREATE TABLE branch_reports AS
SELECT b.branch_id, b.manager_id,
       COUNT(ist.issued_id) AS number_book_issued,
       COUNT(rs.return_id)  AS number_of_book_return,
       SUM(bk.rental_price) AS total_revenue
FROM issued_status ist
JOIN employees e ON e.emp_id = ist.issued_emp_id
JOIN branch b    ON e.branch_id = b.branch_id
LEFT JOIN return_status rs ON rs.issued_id = ist.issued_id
JOIN books bk ON ist.issued_book_isbn = bk.isbn
GROUP BY 1, 2;
```

**Task 16 — Active members (last 2 months)**
```sql
CREATE TABLE active_members AS
SELECT * FROM members
WHERE member_id IN (
    SELECT DISTINCT issued_member_id
    FROM issued_status
    WHERE issued_date >= CURRENT_DATE - INTERVAL '2 month'
);
```

**Task 17 — Top employees by books processed**
```sql
SELECT e.emp_name, b.*, COUNT(ist.issued_id) AS no_book_issued
FROM issued_status ist
JOIN employees e ON e.emp_id = ist.issued_emp_id
JOIN branch b    ON e.branch_id = b.branch_id
GROUP BY 1, 2
ORDER BY no_book_issued DESC
LIMIT 3;
```

### Stored Procedures

**Task 14/19 — Automate returns and issues**

A `add_return_records()` procedure logs a return and flips the book's status back to `'yes'`:
```sql
CALL add_return_records('RS138', 'IS135', 'Good');
```

An `issue_book()` procedure checks availability before issuing — and blocks the transaction (with a friendly message) if the book is already checked out:
```sql
CALL issue_book('IS155', 'C108', '978-0-553-29698-2', 'E104'); -- succeeds
CALL issue_book('IS156', 'C108', '978-0-375-41398-8', 'E104'); -- book unavailable
```

**Task 20 — Overdue fines (CTAS)**

Builds a table of each member's overdue book count and total fine, calculated at **$0.50/day**:
```sql
CREATE TABLE overdue_fines AS
SELECT
    ist.issued_member_id,
    COUNT(*) AS overdue_books,
    SUM((CURRENT_DATE - ist.issued_date) - 30) * 0.50 AS total_fine
FROM issued_status ist
LEFT JOIN return_status rs ON rs.issued_id = ist.issued_id
WHERE rs.return_id IS NULL
  AND (CURRENT_DATE - ist.issued_date) > 30
GROUP BY ist.issued_member_id;
```

---

## 🧠 Skills Demonstrated

- Relational schema design with primary & foreign keys
- `INSERT` / `UPDATE` / `DELETE` operations
- Multi-table `JOIN`s (inner, left) across a 6-table schema
- Aggregations with `GROUP BY` / `HAVING`
- `CREATE TABLE AS SELECT` (CTAS) for derived reporting tables
- Date arithmetic and interval filtering
- PL/pgSQL **stored procedures** with conditional logic and `RAISE NOTICE`
- Business-driven reporting (overdue fines, branch performance, employee rankings)

---

## 🚀 How to Use

1. **Clone the repository**
   ```sh
   git clone https://github.com/<your-username>/<your-repo>.git
   cd <your-repo>
   ```

2. **Create the database**
   ```sql
   CREATE DATABASE library_db;
   ```

3. **Build the schema** — run `app_library.sql` first to create all tables.

4. **Load the data** — run `insert_queries.sql`, then `insert_queries2.sql`.

5. **Run the solutions** — execute `solution_day_1.sql` and `lms_project_advanced_solution_2.sql` (or the `SQL_PROJECT_02*` alternates) to see all task solutions and generated reports.

> 💡 Tested on **PostgreSQL**. Minor syntax tweaks (interval/date functions) may be needed for MySQL or other engines.

---

## 📌 Notes

- `book_quality` (`Good` / `Damaged`) was added to `return_status` after the initial schema to support high-risk member analysis.
- Several files represent iterative/alternate passes at the same tasks (e.g. `SQL_PROJECT_02_Part_02.sql` vs. `lms_project_advanced_solution_2.sql`) — kept for reference on different query approaches.

---

## 📄 License

This project is open for learning purposes. Feel free to fork, adapt, and build on it.
