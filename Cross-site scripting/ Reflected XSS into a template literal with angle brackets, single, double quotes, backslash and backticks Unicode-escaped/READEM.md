# Reflected XSS into a JavaScript Template Literal (Angle Brackets, Quotes, Backslash & Backticks All Escaped)

**Category:** Cross-Site Scripting (XSS)
**Lab:** Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped
**Status:** Solved ✅

---

## What's going on here

On paper this looks like the most locked-down context in the whole series — angle brackets, single quotes, double quotes, backslashes, *and* backticks are all being escaped. That's every character you'd normally reach for to break out of a JS string or HTML context. And yet this one turns out to be the easiest exploit in the series, because none of that filtering actually matters.

The reason: your input isn't landing inside a normal string. It's landing inside a **JavaScript template literal** — the backtick-delimited string type (`` `...` ``) that ES6 introduced — and template literals have a feature plain strings don't: `${...}` interpolation, which evaluates whatever's inside it as live JavaScript and inserts the result. We don't need to escape anything. We just need to use the feature that's already there.

**Goal:** get `alert()` to execute from inside the search box, using nothing but the template literal's own interpolation syntax.

## Where the injection lands

Somewhere in the page's JS, there's something like:

```js
var searchTerm = `You searched for: ${userInput}`;
```

If `userInput` already sits directly inside the backticks of an existing template literal, then anything shaped like `${...}` that we put into it doesn't need to escape out of a string boundary at all — it's already *inside* live template syntax. The interpolation placeholder is a first-class part of the string, evaluated by the JS engine as an expression, not just text.

## Working through it

**1. Submit a test string and see where it lands**

Type a random alphanumeric string into the search box, submit it, and catch the request in Burp — send it to Repeater.

**2. Check the response for template literal context**

Look at the JS in the response around where your search term got reflected. You should see it sitting between a pair of backticks — confirming you're inside a template literal, not a regular single- or double-quoted string.

**3. Replace your input with interpolation syntax**

Instead of trying to escape out of anything, just use the feature that's already there:

```
${alert(1)}
```

**4. Send and verify**

Send the modified request. If you want to double-check it in the browser directly, right-click the response, copy the URL, and paste it into your address bar — loading that page should pop the alert immediately, since the search term (and the code embedded in it) gets evaluated as soon as the page runs.

## The payload

```
${alert(1)}
```

That's genuinely the whole thing. `${ }` is the template literal interpolation syntax — whatever expression sits inside it gets evaluated by the JS engine and the result gets stitched into the string. `alert(1)` is a completely valid expression, so it runs, pops the alert, and its return value (`undefined`) gets silently interpolated into the surrounding text — which doesn't matter, since the alert already fired by then.

## Why this works

Every defense the app has in place is aimed at a different problem: preventing you from breaking *out* of the existing string or tag. Escaping `` ` `` stops you from prematurely ending the template literal. Escaping `'` and `"` stops you from escaping a regular string. Escaping `<` and `>` stops you from injecting raw HTML. All of that is solid, standard defense — against the standard attack.

But template literal interpolation isn't a way of breaking out of the string at all. `${...}` is a legitimate part of template literal syntax that's evaluated *while staying fully inside* the backticks. None of the escaped characters are needed to trigger it. The developer filtered every character that would let you escape the string, but never considered that the string type itself has a built-in code-execution feature that doesn't require escaping anything.

This is a good reminder that XSS filtering has to account for the full syntax of whatever context the data lands in — not just the characters that delimit that context, but any language features active *within* it.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- Template literals (`` `...` ``) are a fundamentally different beast from regular JS strings — they support live expression evaluation via `${}`, which regular `'...'` or `"..."` strings simply don't have.
- Escaping every character that would let you *break out* of a context doesn't help if the context has its own built-in way to *execute code from inside*.
- Always check exactly what kind of string/context your input lands in — a template literal calls for a completely different exploitation approach than a plain string, even though both look like "text between quote-ish characters."
- This is one of the simplest payloads in the whole series precisely because the developer over-focused on escape characters and missed the actual attack surface.

## Fixing it

- Never insert unsanitized user input directly into a template literal that will be evaluated as JavaScript — `${}` interpolation is exactly as dangerous as `eval()` if the string containing it comes from an untrusted source.
- If user input must appear inside dynamically-generated JavaScript, insert it as a properly JSON-encoded string literal (with a JS-aware encoder, not a naive character blacklist), and make sure whatever encodes it is aware of template-literal-specific syntax like `${}` and nested backticks.
- Better yet, avoid building JavaScript by string concatenation/interpolation at all — pass data to the client as JSON and let trusted, static JS code consume it as data, never as code.
- A Content Security Policy that disallows inline scripts would have blocked this particular payload from having any effect, even with the underlying injection point still present.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
