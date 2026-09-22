# Time-Based Blind SQL Injection

**Lab:** Time-based blind SQL injection
**Status:** Solved ✅

---

## What makes this one different

Most of the labs so far give you something to look at — an error message, an extra row in the results, some kind of visible feedback. This one gives you nothing. The app takes user input, drops it straight into a query without sanitizing it, but never actually shows you the results or any errors.

So instead of reading data back, we have to *time* the database. If we can make it pause for a few seconds on command, that pause itself becomes the signal — no output needed.

**Goal:** confirm the injection by making the server visibly hang for ~20 seconds.

## Where the bug lives

Somewhere on the backend, there's a query roughly like:

```sql
SELECT *
FROM users
WHERE tracking_id = '<user_input>'
```

Since that input isn't parameterized, we can break out of the string and add our own SQL — including a call to the database's sleep function.

## Working through it

**1. Grab a request**

Browse the app like a normal user, catch a request in Burp, and figure out which parameter or cookie is carrying our input. Send it to Repeater — this whole exploit is basically "change payload, resend, check the clock," so Repeater is where we'll live.

**2. Get a baseline first**

Before touching anything, fire off the unmodified request a couple of times and note how fast it comes back. You need this baseline — without it, a slow response could just be network noise instead of proof the injection worked.

**3. Inject the delay**

Swap the parameter for a payload built around `SLEEP()`:

```
' || (SELECT FROM (SELECT(SLEEP(20)))a) || '
```

The exact syntax shifts depending on the database engine and exactly where in the query your input lands, but the core of it is always the same: get `SLEEP(20)` to actually execute.

**4. Watch the clock**

Send it and compare against your baseline:

- **Normal request** → comes back fast, like always
- **Injected request** → hangs for roughly 20 seconds before responding

That gap is all the confirmation you need. The database executed our injected code — we just can't see any output from it, only feel the delay.

## The payload

```
' || (SELECT FROM (SELECT(SLEEP(20)))a) || '
```

The part that actually matters is `SLEEP(20)` — everything else is just scaffolding to get that function call to execute inside the existing query.

## Why timing works as a signal at all

The whole point of blind SQLi is that you don't get to see query results directly, so you need some other channel to smuggle information out through. Time is a perfectly good one:

- Condition is false → no delay → fast response
- Condition is true → `SLEEP()` fires → noticeable delay

Once you can reliably turn a true/false condition into "fast" or "slow," you've got a working oracle. That's really all blind SQLi is — repeatedly asking yes/no questions and reading the answer off a stopwatch instead of the page content.

## Taking it further: conditional sleeps

The lab itself just confirms the vulnerability exists, but the same idea scales up into full data extraction using conditional logic:

```
IF(condition, SLEEP(5), 0)
```

True → 5-second delay. False → nothing, response comes back normal speed. Ask "is the first character of the password 'a'?" — delay means yes, no delay means no. Repeat that across every character and every possible value, and you can reconstruct a password one character at a time without ever seeing a single byte of actual data in the response. Slow, but completely mechanical — which is exactly why it's usually automated with a tool like sqlmap in practice.

## Real-world impact

Once you've got a working time-based oracle, what you can do with it depends heavily on the database's privileges and how the app is wired up behind the scenes, but in general it opens the door to:

- Pulling data out of the database character by character
- Confirming or ruling out specific values (usernames, table names, etc.)
- Extracting credentials given enough time and automation
- Depending on privileges, possibly writing to the database too — not just reading

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- Blind SQLi doesn't need visible output to be exploitable — timing alone is enough of a side channel.
- Always establish a baseline response time before trusting a delay as a signal.
- `SLEEP()` (or its equivalent) is the workhorse function here — get it to fire reliably and you've got a working boolean oracle.
- This technique scales into full data extraction with conditional logic, just slowly and usually via automation.
- Same fix, every time: parameterized queries. No amount of clever filtering beats not concatenating input into SQL in the first place.

## Fixing it

Parameterized queries / prepared statements are the actual fix. Don't build SQL with string concatenation:

```sql
SELECT * FROM users WHERE id = '<user_input>'
```

Use a bound parameter instead:

```sql
SELECT * FROM users WHERE id = ?
```

Worth pairing with:

- Validating user-controlled input wherever it makes sense
- Running database accounts with the least privilege they can get away with
- Not leaking detailed DB errors to clients
- Keeping an eye on unusual response-time patterns, since that's exactly what this attack abuses

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
