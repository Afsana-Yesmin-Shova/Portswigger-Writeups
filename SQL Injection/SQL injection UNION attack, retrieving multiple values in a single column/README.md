# SQL Injection: UNION Attack — Retrieving Data from Other Tables

## PortSwigger Web Security Academy

**Category:** SQL Injection  
**Lab:** SQL injection UNION attack  
**Status:** ✅ Solved

---

## Lab Description

This lab contains a SQL injection vulnerability that can be exploited using a `UNION` attack.

The application displays products based on a selected product category. The objective is to use a SQL injection UNION attack to retrieve additional data from the database.

The lab demonstrates how a `UNION` query can be used to combine the results of the original SQL query with the results of another `SELECT` statement.

---

## Objective

Use a SQL injection UNION attack to retrieve data from another table in the application's database.

---

## Vulnerability

The application uses user-controlled input when constructing a SQL query.

A query may be conceptually similar to:

```sql
SELECT name, description
FROM products
WHERE category = 'Gifts'

Because the category parameter is not safely handled, an attacker can inject additional SQL syntax.

A UNION operator can then be used to append the results of another query to the original query.

Solution
Step 1: Identify the Vulnerable Parameter

First, open the PortSwigger lab and browse the product categories.

Select a category and intercept the request using Burp Suite.

The request contains a parameter similar to:

category=Gifts

The category parameter is potentially vulnerable to SQL injection.

Step 2: Send the Request to Burp Repeater

Send the intercepted request to Burp Suite Repeater.

Repeater allows us to modify the request and test different SQL injection payloads.

The original request contains the category parameter:

category=Gifts
Step 3: Determine the Number of Columns

Before using a UNION SELECT statement, we need to determine how many columns are returned by the original query.

One way to do this is by using:

' ORDER BY 1--

Then increase the column number:

' ORDER BY 2--
' ORDER BY 3--

Continue increasing the number until the application returns an error.

The highest valid column number indicates the number of columns returned by the original query.

Step 4: Determine Compatible Data Types

After determining the number of columns, test which columns can contain string values.

For example:

' UNION SELECT 'a',NULL--

If necessary, test different column positions:

' UNION SELECT NULL,'a'--

The response helps identify which columns accept string data.

This is important because the data retrieved from the database must be compatible with the corresponding column data types.

Step 5: Use UNION SELECT

Once the number of columns and compatible data types are known, construct a UNION SELECT payload.

For example:

' UNION SELECT NULL,'test'--

The exact payload depends on the number of columns returned by the original query.

The purpose of the payload is to append another SELECT statement to the original SQL query.

Conceptually, the query becomes:

SELECT name, description
FROM products
WHERE category = 'Gifts'

UNION

SELECT NULL, 'test'

If the injected value appears in the application response, the UNION attack is working.

Step 6: Retrieve Database Information

After confirming that the UNION injection works, the database can be queried for additional information.

Depending on the database engine, metadata tables can be used to identify tables and columns.

For example, in databases supporting information_schema, table names can be retrieved using a query similar to:

SELECT table_name
FROM information_schema.tables

The exact query syntax depends on the database management system used by the application.

Step 7: Retrieve Data from Another Table

After identifying an interesting table, its columns can be determined and queried using the UNION injection.

Conceptually:

SELECT column1, column2
FROM target_table

The resulting data is then returned as part of the application's normal response.

This demonstrates that the SQL injection vulnerability allows data from other database tables to be retrieved.

Understanding the UNION Attack

The SQL UNION operator combines the results of two or more SELECT statements.

For example:

SELECT name FROM products
UNION
SELECT username FROM users

If the queries are compatible, the database combines their results.

In a vulnerable web application, an attacker can exploit this behavior by injecting a UNION SELECT statement into a parameter controlled by the user.

Example Injection Structure

A typical UNION injection has the following structure:

' UNION SELECT column1,column2--

The first part:

'

closes the existing SQL string.

The:

UNION SELECT

adds another query to the original query.

The:

--

comments out the remaining portion of the original SQL statement.

Burp Suite Workflow

The general workflow used in this lab was:

Browser
   ↓
Intercept request
   ↓
Burp Suite Proxy
   ↓
Send to Repeater
   ↓
Identify vulnerable parameter
   ↓
Determine number of columns
   ↓
Determine compatible data types
   ↓
Test UNION SELECT
   ↓
Identify database information
   ↓
Retrieve data
Tools Used
Burp Suite
Burp Suite Repeater
Web Browser
PortSwigger Web Security Academy
Key Takeaways
SQL injection can allow an attacker to manipulate backend database queries.
A UNION attack can combine the results of the original query with another SELECT query.
The number of columns returned by the original query must be determined before constructing a compatible UNION query.
The data types of the selected columns must also be compatible.
Database metadata can sometimes be queried to identify tables and columns.
Parameterized queries and prepared statements are effective defenses against SQL injection.
Lab Status

Status: ✅ Solved

Vulnerability: SQL Injection

Technique: UNION-based SQL Injection

Impact: Retrieval of data from other database tables

Disclaimer

This write-up was created for educational and authorized security testing purposes using the PortSwigger Web Security Academy lab environment.

Do not use these techniques against systems without explicit authorization.
