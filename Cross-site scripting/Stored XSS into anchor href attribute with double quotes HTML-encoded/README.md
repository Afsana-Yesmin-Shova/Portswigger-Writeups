# Stored XSS into Anchor href Attribute with Double Quotes HTML-Encoded

**Category:** Cross-Site Scripting (XSS)
**Lab:** Stored XSS into anchor `href` attribute with double quotes HTML-encoded
**Status:** Solved ✅

---

## What's going on here

This is a nice example of a filter that's technically doing its job — and it's still not enough. The site's comment feature includes a "Website" field, and whatever you type there ends up getting rendered as the `href` of a link (the commenter's name becomes clickable, pointing wherever you told it to). The app does HTML-encode double quotes, so you can't just break out of the attribute the obvious way by injecting `">`. But that's not actually the only way in.

**Goal:** get a comment posted that pops an `alert()` when someone clicks the author's name.

## Where the injection lands

The important thing to notice here is *where* your input ends up — not in the visible page text, but as the value of an `href` attribute:

```html
<a href="YOUR_INPUT">author name</a>
```

Since double quotes are encoded, we can't inject `" onmouseover="alert(1)` or similar — the closing quote we'd need just becomes `&quot;` and sits there harmlessly as text inside the attribute value. But `href` doesn't only take normal URLs. It'll just as happily accept a `javascript:` URI, and there's no quote required to make that work — we're not breaking out of the attribute at all, we're just using it exactly as intended, with a value that happens to run code.

## Working through it

**1. Post a comment with a throwaway marker**

Fill out the comment form, and in the **Website** field, put something random and easy to spot later — an alphanumeric string you'd recognize instantly, like `xsstestabc123`. Submit it with Burp intercepting, and send that request to Repeater. You don't need to do anything with this one yet — it's just there so the comment exists.

**2. Load the post and check how your input got rendered**

Reload the blog post page in the browser (or replay the GET request in a second Repeater tab) and look at the HTML that comes back. Find your comment and check the anchor tag around the author's name — you should see your random string sitting inside `href="..."`, confirming exactly where the injection point is and that it survived encoding intact.

**3. Confirm quotes get encoded**

If you're curious, you can verify the filter's behavior directly: submit a comment with a `"` in the Website field and check the response. You'll see it come back as `&quot;` — confirming double quotes are neutralized, which is exactly why the standard "close the attribute and add an event handler" trick won't work here.

**4. Submit the real payload**

Post another comment, and this time set the Website field to:

```
javascript:alert(1)
```

No quotes needed — we're not trying to escape the attribute, just filling it with a URI scheme the browser will happily execute when clicked.

**5. Verify it actually fires**

Reload the post, right-click the malicious link (your name, now pointing at the `javascript:` URI), and copy the URL. Paste it into the browser's address bar to double check it's really `javascript:alert(1)` and not something mangled by encoding along the way. Then just click the link on the page itself — clicking the author name should pop an `alert()` box, confirming the payload executes.

## Payload used

```
javascript:alert(1)
```

That's the entire exploit — no encoding tricks, no attribute breakout, just a URI scheme the app never anticipated someone would use maliciously.

## Why this works

HTML-encoding double quotes is a real, meaningful defense — it closes off the most common way of escaping an attribute context. But it's solving the wrong part of the problem. The vulnerability here isn't really about breaking *out* of the `href` attribute; it's about what's allowed to go *into* it. Browsers support several URI schemes beyond `http://` and `https://`, and `javascript:` is one of them — when a link using that scheme is clicked, the browser runs whatever follows the colon as JavaScript instead of navigating anywhere.

Because the app only sanitizes special characters, and `javascript:alert(1)` doesn't need any special characters at all to work, the filter has nothing to catch. The lesson: escaping dangerous characters protects the *syntax* of the surrounding markup, but it does nothing to restrict the *semantics* of a value the app is about to treat as an active URL.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- Encoding quotes stops attribute-breakout XSS, but it does nothing against payloads that don't need quotes in the first place.
- `href` (and similarly `src`, `action`, `formaction`, and other URL-bearing attributes) is dangerous even when perfectly quote-safe, because `javascript:` URIs are a valid value for it.
- Always check exactly *where* your input lands in the DOM before assuming a filter has closed off every route — the context matters as much as the encoding.
- Stored XSS is especially nasty here because the payload sits waiting in the database; anyone who clicks that comment author's name triggers it, not just the person who posted it.

## Fixing it

- Validate the *scheme* of any user-supplied URL, not just its characters — allow only `http://` and `https://` (and relative paths, if applicable), and reject `javascript:`, `data:`, `vbscript:`, and other executable schemes outright.
- Don't rely on character-level encoding alone to secure a context that also has scheme-level risk — encoding and allowlisting solve different problems and you often need both.
- Consider a Content Security Policy that disallows inline script execution, which would blunt the impact of `javascript:` URIs even if one slipped through.
- Treat any field that ends up as an attribute value — not just visible page content — as a potential injection point worth testing deliberately.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
