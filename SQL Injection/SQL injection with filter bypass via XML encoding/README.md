# SQL Injection with Filter Bypass via XML Encoding

**Lab:** SQL injection with filter bypass via XML encoding
**Status:** Solved ✅

---

## What's going on here

This one's less about finding the injection — that part's pretty quick — and more about getting past something actively trying to stop you: a WAF sitting in front of the app, watching for obvious SQLi signatures. The vulnerable feature is a stock checker that sends `productId` and `storeId` to the backend as XML, and the query results come straight back in the response, so once we're past the filter this is a fairly standard UNION attack.

**Goal:** get past the WAF, pull `username`/`password` pairs out of the `users` table, and log in as admin.

## Finding the injection point

**1. Watch the stock check request**

Browse the shop, pick a product, and check its stock. Catch the request in Burp — you'll see it's sending `productId` and `storeId` as XML in the body, not as normal form fields or query params.

**2. Send it to Repeater**

Standard move at this point — we're going to be iterating on the `storeId` value a lot.

**3. Confirm the value gets evaluated, not just matched**

Instead of jumping straight to SQL syntax, first check whether the app is actually evaluating your input as an expression. Change the store ID from something like `2` to `2+1`:

```xml
<storeId>2+1</storeId>
```

If the response comes back with stock numbers for a *different* store than ID 2 (say, the numbers you'd expect for store 3), that confirms the value isn't just being string-matched — it's being evaluated, which is a strong signal this feeds into something like a SQL expression server-side.

**4. Try a UNION SELECT — and get blocked**

Now go for the real thing:

```xml
<storeId>1 UNION SELECT NULL</storeId>
```

Instead of a normal response, you'll get flagged — something like an "Attack detected" message. This is the WAF doing its job: it's pattern-matching for obvious SQLi keywords like `UNION SELECT` sitting in plain text, and it caught us.

## Getting past the WAF

This is the actual point of the lab. The filter is looking for recognizable SQL syntax in the raw request — so the fix isn't a cleverer SQL payload, it's making the payload *not look like SQL* until it's already inside the XML parser, which happily decodes entities before anything else gets a chance to inspect it.

**5. Install Hackvertor**

Grab the **Hackvertor** extension from the BApp Store if you don't already have it — it's the tool PortSwigger recommends for this, and it makes entity-encoding painless instead of doing it by hand.

**6. Encode the payload as XML entities**

Highlight your injected payload in the request, right-click, and go **Extensions → Hackvertor → Encode → dec_entities** (or `hex_entities` — either works). This turns your plain-text SQL keywords into their numeric character-reference equivalents, so `UNION SELECT` might come out looking like a string of `&#85;&#78;&#73;&#79;&#78;...` instead of readable text.

**7. Resend and confirm the bypass**

Send it again. The WAF, which was only looking for literal keyword strings, sees a wall of numeric entities and has nothing to flag. No "Attack detected" this time — but critically, the XML parser on the backend still decodes those entities back into real SQL *before* it ever reaches the database, so the query itself executes exactly as intended. The filter and the parser are looking at the data at two different stages, and that gap is the whole vulnerability.

## Building the actual exploit

**8. Work out the column count**

With the WAF out of the way, go back to figuring out the query shape. Trying more than one column back (`UNION SELECT NULL,NULL`, etc.) causes the app to report `0 units` — which, in context, is really an error being swallowed and displayed as a zero rather than a genuine "no stock" result. That tells you the original query only returns a single column, so anything you inject needs to fit into that one slot.

**9. Concatenate username and password into that one column**

Since there's only room for one value, smash the two columns you actually want together with a separator:

```sql
1 UNION SELECT username || '~' || password FROM users
```

**10. Wrap it in Hackvertor's tag and send**

Rather than manually encoding every time, Hackvertor lets you wrap the payload directly in a conversion tag that gets encoded automatically when the request goes out:

```xml
<storeId><@hex_entities>1 UNION SELECT username || '~' || password FROM users</@hex_entities></storeId>
```

Send it. If it worked, the stock number that comes back in the response isn't a stock number at all — it's a list of `username~password` pairs, pulled straight out of the `users` table and rendered wherever the app normally shows the unit count.

**11. Log in**

Find the administrator's row in that output and log in with those credentials. Lab solved.

## Payload reference

| Purpose | Payload |
|---|---|
| Check if input is evaluated | `<storeId>2+1</storeId>` |
| Trigger the WAF (gets blocked) | `<storeId>1 UNION SELECT NULL</storeId>` |
| Final exploit, entity-encoded via Hackvertor | `<storeId><@hex_entities>1 UNION SELECT username \|\| '~' \|\| password FROM users</@hex_entities></storeId>` |

## Why this works

WAFs generally work by pattern-matching the raw bytes of a request against known-bad signatures — `UNION SELECT`, `OR 1=1`, and so on. That's a fundamentally different layer than the one that actually processes the data. Here, the backend parses the request as XML, and XML parsers decode character entities as a completely normal part of parsing — it's not a bug, it's just what XML does. The WAF inspects the request *before* that decoding happens, sees harmless-looking numeric entities, and lets it through. The XML parser then decodes those entities into plain SQL keywords, and the database receives the exact same malicious query it always would have — the filter just never got a chance to see it in its recognizable form.

This is really a class of bug worth remembering: whenever a WAF and the actual parser disagree about *when* to interpret encoded data, that gap is exploitable. Same root idea shows up with double-encoding, Unicode normalization tricks, and various other encoding-layer bypasses.

## Tools used

- Burp Suite (Proxy + Repeater)
- Hackvertor extension (BApp Store)
- Browser

## Takeaways

- WAFs filter on surface-level patterns, not on what the backend will eventually parse the data into — that mismatch is the bypass.
- XML entity encoding is a legitimate, boring part of XML parsing, which is exactly why it's such a good disguise for a payload.
- Confirming that input is *evaluated* (not just reflected) before reaching for SQL syntax saves time and avoids tripping filters early.
- When a query can only return a single column, string concatenation (`||`, `CONCAT()`, depending on the DB) lets you smuggle multiple values through it anyway.
- A `0 units` / seemingly-benign response can actually be a swallowed database error — worth treating oddly-specific "empty" results as a signal, not a dead end.

## Fixing it

The underlying SQL injection still needs the standard fix — parameterized queries / prepared statements, full stop. But this lab also points at a broader lesson: a WAF is not a substitute for fixing the actual vulnerability. It's a compensating control, and encoding tricks like this one demonstrate how easily pattern-based filters can be routed around. On top of proper parameterization:

- Don't rely on a WAF as your primary defense against injection — treat it as a secondary layer, not the fix itself.
- Make sure any WAF or input filter normalizes/decodes data the same way the application's own parsers will, or it's inspecting a different string than the one that actually gets processed.
- Validate and constrain input types strictly (e.g., if `storeId` should only ever be a small integer, reject anything that isn't).

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
