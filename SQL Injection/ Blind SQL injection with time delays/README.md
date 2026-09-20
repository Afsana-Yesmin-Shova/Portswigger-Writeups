# Time-Based Blind SQL Injection

## PortSwigger Web Security Academy

**Category:** SQL Injection  
**Lab:** Time-based blind SQL injection  
**Status:** ✅ Solved

---

## Lab Description

This lab demonstrates a **Time-Based Blind SQL Injection** vulnerability.

The application processes user-controlled input without properly parameterizing or sanitizing it.

Unlike normal SQL injection, the application does not directly return useful database information in the response.

Instead, a time-delay function can be injected into the SQL query.

If the SQL payload is successfully executed, the server response is delayed by a noticeable amount of time.

This difference in response time can be used as a signal to confirm SQL injection and, in more advanced scenarios, extract information from the database.

---

## Objective

Identify and exploit the time-based blind SQL injection vulnerability by injecting a database sleep function and observing the resulting delay in the server response.

---

# Vulnerability Details

The vulnerability occurs because user-controlled input is incorporated into a backend SQL query without proper parameterization.

A vulnerable query can be conceptually represented as:

```sql
SELECT *
FROM users
WHERE tracking_id = '<user_input>'

If the input is injectable, an attacker can modify the query and introduce a database time-delay function.

For example, a SLEEP() function can force the database to pause execution.

Solution
Step 1: Capture the Request

First, open the lab and interact with the application normally.

Use Burp Suite to intercept the request.

Identify the parameter or cookie containing user-controlled input.

Send the request to:

Burp Suite → Repeater

This allows the request to be modified and tested repeatedly.

Step 2: Establish the Normal Response Time

Before testing the injection, send the original request several times.

The normal request should return relatively quickly.

For example:

Normal request → Fast response

This provides a baseline for comparison.

Step 3: Inject a Time Delay

Modify the vulnerable parameter and inject a SQL time-delay payload.

The payload used in the write-up is conceptually based on:

' || (SELECT FROM (SELECT(SLEEP(20)))a) || '

The purpose of the payload is to execute:

SLEEP(20)

which instructs the database to pause execution for approximately 20 seconds.

The exact syntax can vary depending on the database engine and the context in which the parameter is inserted.

Step 4: Observe the Response

Send the modified request through Burp Suite Repeater.

Compare the response time with the original request.

Normal Request
Fast response
Injected Request
Approximately 20-second delay

The significant delay indicates that the database executed the injected time-delay function.

Behavior Observed

The behavior can be summarized as:

Normal input
     ↓
Fast response

Injected SQL payload
     ↓
Database executes SLEEP(20)
     ↓
Server waits approximately 20 seconds
     ↓
Delayed response

This confirms the presence of a Time-Based Blind SQL Injection vulnerability.

Payload

The payload used for testing was:

' || (SELECT FROM (SELECT(SLEEP(20)))a) || '

The important component is:

SLEEP(20)

which introduces an approximately 20-second delay when executed.

Why This Works

Time-based blind SQL injection relies on the difference between two response times.

The application does not need to display database results.

Instead, the attacker uses execution time as a signal.

For example:

Condition is FALSE
        ↓
No delay
        ↓
Fast response

while:

Condition is TRUE
        ↓
SLEEP(20) executes
        ↓
Approximately 20-second delay

Therefore, the attacker can determine whether a SQL condition is true by measuring the server's response time.

Time-Based Blind SQLi Concept

A more advanced attack can use conditional logic.

Conceptually:

IF(condition, SLEEP(5), 0)

If the condition is true:

SLEEP(5)
    ↓
5-second delay

If the condition is false:

No sleep
    ↓
Normal response

This can theoretically be used to extract information one bit or character at a time.

For example:

Is the first character of the password 'a'?

If the answer is TRUE:

Delay occurs

If the answer is FALSE:

No delay

Repeating this process can allow database information to be extracted even when the application does not display query results.

Burp Suite Workflow
Browser
   ↓
Capture HTTP Request
   ↓
Identify User-Controlled Parameter
   ↓
Send Request to Burp Repeater
   ↓
Measure Normal Response Time
   ↓
Inject Time-Delay Payload
   ↓
Send Request
   ↓
Observe ~20 Second Delay
   ↓
Confirm Time-Based Blind SQL Injection
Impact

A successful time-based blind SQL injection vulnerability can allow an attacker to:

Extract information from the database.
Determine database values through timing differences.
Infer usernames and other sensitive information.
Potentially extract passwords.
Interact with or modify backend data depending on the SQL injection context.
Potentially escalate the attack toward broader database compromise.

The actual impact depends on the database privileges and the application's backend configuration.

Tools Used
Burp Suite
Burp Suite Repeater
Web Browser
PortSwigger Web Security Academy
Key Takeaways
Blind SQL injection does not always return database results directly.
Response timing can be used as a side channel to infer SQL query results.
Database functions such as SLEEP() can be used to introduce measurable delays.
Establishing a baseline response time is important when testing time-based vulnerabilities.
Time-based SQL injection can potentially be automated to extract information character by character.
Parameterized queries and prepared statements are the primary defenses against SQL injection.
Remediation

The primary defense is to use parameterized queries / prepared statements.

Instead of constructing SQL statements using string concatenation:

SELECT *
FROM users
WHERE id = '<user_input>'

the application should use a parameterized query:

SELECT *
FROM users
WHERE id = ?

Additional defensive measures include:

Properly parameterize all database queries.
Avoid concatenating user input into SQL statements.
Validate user-controlled input where appropriate.
Use least-privilege database accounts.
Avoid exposing detailed database errors.
Monitor unusual database response-time patterns.
Consider additional server-side protections against SQL injection.
Lab Status

Status: ✅ Solved

Vulnerability: Time-Based Blind SQL Injection

Technique: SQL Time Delay

Payload: SLEEP(20)

Observed Effect: Approximately 20-second server response delay

Disclaimer

This write-up was created for educational and authorized security testing purposes using the PortSwigger Web Security Academy lab environment.
