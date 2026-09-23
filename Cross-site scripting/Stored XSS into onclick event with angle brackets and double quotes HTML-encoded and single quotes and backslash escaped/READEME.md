# Stored XSS into onclick Event — Angle Brackets & Double Quotes Encoded, Single Quotes Backslash-Escaped

**Category:** Cross-Site Scripting (XSS)
**Lab:** Stored XSS into `onclick` event with angle brackets and double quotes HTML-encoded and single quotes and backslash escaped
**Status:** Solved ✅

---

## What's going on here

This is the toughest filter yet in this series of comment-field XSS labs. Angle brackets and double quotes are HTML-encoded (so no breaking out of the tag), and this time single quotes are backslash-escaped too (so no breaking out of the JS string either, at least not the obvious way). The "Website" field ends up reflected inside an `onclick` handler on the comment author's name — so we're injecting directly into a JavaScript string literal that's embedded in an HTML attribute, and every escape route that usually works here has been closed off.

**Goal:** get a comment posted that fires `alert()` when the author's name is clicked, despite all three of those defenses being active.

## Where the trick lives

The key thing to notice: single quotes get backslash-escaped by the server, but that escaping only happens to *literal* single-quote characters in your input. It has no idea what to do with `&apos;` — the HTML entity for an apostrophe — because as far as the server's filter is concerned, that's just five harmless characters (`&`, `a`, `p`, `o`, `s`) plus a semicolon. Nothing quote-shaped for it to touch.

But the browser doesn't see it that way. When it parses the HTML and extracts the `onclick` attribute's contents to run as JavaScript, it first HTML-decodes the attribute value — entities and all — *before* handing that decoded text to the JS engine. So `&apos;` silently turns back into a real `'` character at exactly the point where it matters, completely bypassing the backslash-escaping that only ever saw the encoded form.

## Working through it

**1. Post a comment with a throwaway marker**

Submit a comment with a random alphanumeric string in the **Website** field, intercepting with Burp and sending the request to Repeater. Nothing fancy yet — just planting a comment we can inspect.

**2. Load the post and check where your input landed**

Reload the blog post (or replay the GET request in a second Repeater tab) and look at the rendered HTML around the comment author's name. You should find your string sitting inside an `onclick` attribute — confirming it's landing in a JavaScript execution context, not just plain HTML.

**3. Confirm the escaping behavior**

Worth checking directly: submit a comment with a literal single quote in the Website field, and you'll see it come back with a backslash in front of it (`\'`) — confirming the server is actively defending against a straightforward string breakout. A raw `'` alone won't get you anywhere here.

**4. Submit the real payload**

Post another comment, and set the Website field to:

```
http://foo?&apos;-alert(1)-&apos;
```

**5. Verify it fires**

Reload the post. Right-click the author's name, copy the link/URL if you want to sanity-check what actually landed in the DOM, then just click the name directly — the `onclick` handler should fire and pop `alert(1)`.

## The payload

```
http://foo?&apos;-alert(1)-&apos;
```

Here's what happens to it end to end:

1. **Server-side:** the filter scans for literal `'` characters to backslash-escape. It finds none — `&apos;` doesn't look like a quote to it — so the payload passes through completely untouched.
2. **HTML parsing:** the browser reads the `onclick` attribute's raw text and, as a normal part of parsing any HTML attribute, decodes HTML entities in it. Each `&apos;` becomes a real `'`.
3. **JavaScript evaluation:** by the time the JS engine actually runs the `onclick` code, the string literal that was supposed to safely contain our whole payload has been broken into three pieces by those now-real quotes — effectively turning the surrounding code into something like:

   ```js
   'http://foo?' - alert(1) - ''
   ```

   The first `'` closes the string early. The `-` after it isn't inside a string anymore — it's a subtraction operator, which means the JS engine has to evaluate whatever comes next as an expression. `alert(1)` is a perfectly valid function call sitting right there, so it runs — the alert box pops as a side effect of the engine working out what to subtract. The nonsensical arithmetic result (subtracting a URL string, `undefined`, and an empty string) doesn't matter at all; the alert has already fired by the time that resolves to `NaN`.

## Why this works

Two separate layers of the page — the server-side sanitizer and the browser's HTML/JS parser — disagree about what "a single quote" looks like. The server only recognizes the literal character `'`. The browser's entity-decoding step means an *encoded* representation of that character is just as good, once it gets decoded downstream. Any filter that escapes dangerous characters by literally scanning for them is vulnerable to this if there's a decoding step happening later that the filter isn't aware of — the filter and the eventual consumer of the data are looking at two different representations of the same underlying character.

This is really the same category of bug as the WAF/XML-entity bypass from an earlier lab in this series: a security check applied at one stage of processing, defeated by an encoding transformation that happens at a *later* stage the check never anticipated.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- Backslash-escaping (or any character-level filtering) only protects against the literal character it's watching for — it says nothing about encoded representations of that character that get decoded further downstream.
- HTML entity decoding happens as a normal part of attribute parsing, even for `on*` event handler attributes whose contents are about to be executed as JavaScript — this is easy to forget since those attributes "feel" like they should just be JS source.
- You don't need to fully close and reopen the exact string structure the developer intended — the JS engine will happily execute a function call sitting inside what looks like a broken arithmetic expression, as long as it's syntactically valid up to that point.
- Whenever a filter defends against one specific representation of a dangerous character, it's worth testing whether an equivalent encoded form slips through and gets decoded later by something else in the pipeline.

## Fixing it

- Apply escaping *after* all decoding has happened, or make sure the escaping logic accounts for every representation (literal and encoded) of the characters it's trying to neutralize — encode-then-filter is inherently fragile compared to context-aware output encoding done correctly the first time.
- Avoid inserting user-controlled data into inline event handler attributes (`onclick`, `onmouseover`, etc.) altogether — these are a genuinely hard context to sanitize safely, since they mix HTML attribute parsing rules with JavaScript syntax rules in a single string.
- Use `addEventListener()` in a separate `<script>` block instead of inline handlers, and keep user data out of executable contexts entirely — pass it in as data (e.g., a `data-*` attribute) that gets read and used safely by trusted JS, rather than concatenated directly into code.
- A Content Security Policy that disallows inline event handlers (`unsafe-inline`) would have prevented this specific technique from having any effect, even with the underlying bug present.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
