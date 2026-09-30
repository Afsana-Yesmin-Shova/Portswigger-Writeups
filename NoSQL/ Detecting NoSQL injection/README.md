# Detecting NoSQL Injection

**Category:** NoSQL Injection
**Lab:** Detecting NoSQL injection
**Status:** Solved ✅

---

## What's going on here

This lab is the NoSQL equivalent of the very first SQLi lab in this series — a product category filter that's vulnerable, and the goal is to surface hidden/unreleased products by bending the underlying query's logic. The database here is MongoDB, though, which means the injection mechanics look pretty different from classic SQL even though the underlying idea — user input reshaping a query's logic — is exactly the same.

**Goal:** get the category filter to display products that haven't been released yet, by injecting into MongoDB's query syntax.

## A quick note on how MongoDB injection differs from SQL

MongoDB queries aren't built out of `SELECT ... WHERE ...` strings the way SQL is — but plenty of real-world MongoDB usage still ends up constructing queries by concatenating strings that eventually get evaluated as JavaScript expressions server-side (this is especially true of older `$where`-clause patterns, which this lab is built around). So even though we're not dealing with `'` breaking out of a SQL string, we're dealing with something structurally similar: user input landing inside a JavaScript expression that the database will evaluate, and a single quote is still often the character that breaks things open.

## Working through it

**1. Trigger the category filter and capture the request**

In Burp's browser, click into any product category. Head to **Proxy → HTTP history**, find that category filter request, right-click it, and send it to Repeater.

**2. Confirm input reaches something that parses it as code**

In Repeater, change the category parameter to a single `'` character and send it. If that causes a JavaScript syntax error in the response, that's a strong signal our input is landing directly inside a JS expression the server evaluates — not just being safely treated as a literal string value.

**3. Confirm it really is JavaScript being built via concatenation**

Try:

```
Gifts'+'
```

(URL-encode this before sending — highlight it in Repeater and hit `Ctrl-U`.)

If this *doesn't* cause a syntax error, that's meaningful: `'+'` is valid JavaScript string concatenation syntax. The fact that it parses cleanly suggests our input is being spliced into an existing JS string via concatenation — exactly the pattern that makes this exploitable, since we can use real JS operators to reshape the logic around our input.

**4. Test boolean conditions to confirm we can influence the query's logic**

Try injecting an always-false condition:

```
Gifts' && 0 && 'x
```

(URL-encoded.) You should get zero products back — consistent with the underlying condition evaluating to false and the query returning nothing.

Then try an always-true condition:

```
Gifts' && 1 && 'x
```

This time you should see the normal Gifts category results — confirming we have real, working control over a boolean condition embedded in the query logic.

**5. Build a payload that makes the whole condition always true**

```
Gifts'||1||'
```

Instead of `&&`, this uses `||` (logical OR) — meaning regardless of whatever the original category-matching condition evaluates to, `||1||` guarantees the overall expression is true. If the app was previously restricting displayed products based on both category *and* a released/unreleased flag using similar boolean logic, forcing the whole condition to always evaluate true should knock out that restriction entirely, the same way `'--` did to a SQL `AND released = 1` clause back in the SQLi labs.

**6. View and verify the response**

Right-click the response in Repeater and choose **Show response in browser**, then copy the resulting URL and load it directly in Burp's browser.

**7. Confirm unreleased products are showing**

If the injection worked, the page now displays products that were previously hidden — unreleased items sitting right alongside the normal category listing, exactly like the WHERE-clause SQLi lab from earlier in this series, just reached through MongoDB's query syntax instead of SQL's.

## The payloads

| Purpose | Payload |
|---|---|
| Trigger a JS syntax error to confirm injection | `'` |
| Confirm string concatenation is happening | `Gifts'+'` |
| Test a false condition | `Gifts' && 0 && 'x` |
| Test a true condition | `Gifts' && 1 && 'x` |
| Force the whole condition to always be true | `Gifts'||1||'` |

(All of these need URL-encoding before sending — `Ctrl-U` on a highlighted selection in Burp Repeater does this quickly.)

## Why this works

This particular flavor of NoSQL injection happens when application code builds a MongoDB query by string-concatenating user input into something that ultimately gets evaluated as a JavaScript expression (classically via `$where`, which lets a MongoDB query embed literal JS logic). Once that's happening, MongoDB injection starts looking almost identical to classic SQL injection in spirit — user input can break out of the intended string context and inject real operators (`&&`, `||`) that reshape the boolean logic of the whole query, exactly the same way `'--` or `' OR 1=1` does in SQL.

The methodology mirrors SQLi detection closely too: throw a single quote first to look for a syntax error (confirming injection), confirm you can still produce *valid* syntax with a quote-and-concatenation payload (confirming it's not just erroring out uselessly), then test true/false conditions to confirm you have logical control, and finally weaponize that control to strip out whatever restriction was hiding data from you.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- NoSQL databases aren't immune to injection just because they're not "SQL" — any time user input is concatenated into a query (especially one that gets evaluated as code, like MongoDB's `$where`), the same fundamental injection risks apply.
- The detection methodology is genuinely transferable from SQLi: single quote to trigger an error, a quote-plus-operator payload to confirm valid injected syntax, then boolean true/false tests to confirm logical control before building the real exploit.
- `||` (OR) is the NoSQL/JS equivalent of SQL's `OR 1=1` — a fast way to force an entire condition to evaluate true regardless of the original logic.
- Reading response behavior carefully (a JS syntax error vs. a clean response vs. a changed result set) is how you build confidence in an injection point before committing to a specific exploit payload.

## Fixing it

- Never construct MongoDB queries (or any NoSQL query) by concatenating user input into a string that gets evaluated as code — this is exactly the same underlying mistake as string-concatenated SQL.
- Avoid `$where` and similar JavaScript-evaluating query operators entirely where possible; MongoDB's standard query operators (`$eq`, `$in`, `$gt`, etc.) can express most application logic without ever needing to evaluate arbitrary JS.
- Use parameterized queries / the driver's built-in query-builder methods, which handle values as data rather than as code to be concatenated and evaluated.
- Apply strict input validation and type checking on any value that will be used in a database query, regardless of whether the backend is SQL or NoSQL.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
