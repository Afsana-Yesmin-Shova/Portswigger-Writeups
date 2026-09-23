# DOM XSS in document.write Sink Using location.search Inside a Select Element

**Category:** Cross-Site Scripting (XSS)
**Lab:** DOM XSS in `document.write` sink using source `location.search` inside a select element
**Status:** Solved ✅

---

## What's going on here

This is a proper DOM-based XSS — no server round-trip involved in the vulnerable part at all. The bug lives entirely in client-side JavaScript on the product page. The stock checker feature reads a `storeId` value straight out of the URL's query string and uses `document.write()` to build an `<option>` inside a `<select>` dropdown, without any sanitization along the way.

**Goal:** break out of that `<select>` element and get an `alert()` to fire.

## The source and the sink

In DOM XSS terms, this lab is about as textbook as it gets:

- **Source:** `location.search` — the query string portion of the current URL, which is entirely attacker-controlled
- **Sink:** `document.write()` — writes raw, unescaped HTML directly into the page

The dangerous script pulls `storeId` out of the URL and hands it to `document.write()` to generate a dropdown option — something like:

```html
<select>
  <option value="1">Store 1</option>
  <script>document.write('<option value="' + storeId + '">Store ' + storeId + '</option>');</script>
</select>
```

Since `document.write` writes raw markup with zero escaping, whatever ends up in `storeId` becomes literal HTML on the page.

## Working through it

**1. Spot the vulnerable code**

Open the browser's dev tools on a product page and look at the page source or the JS driving the stock checker. You'll see it reading `storeId` from `location.search` and feeding it into `document.write()` to populate the dropdown.

**2. Confirm you control that value**

Add a `storeId` parameter to the URL yourself, with a random alphanumeric string as the value:

```
/product?productId=1&storeId=abc123xyz
```

Load it. If your random string shows up as one of the options in the dropdown, that confirms the value flows straight from the URL into the page.

**3. Inspect the dropdown to confirm the context**

Right-click the dropdown and choose Inspect. You should see your string sitting inside an `<option>` tag, inside the `<select>` element — this tells you exactly what markup you need to break out of.

**4. Build the breakout payload**

Since we're inside a `<select>`, closing that tag out cleanly is the first move: `"></select>` gets us out of both the attribute (in case there's a quote to close) and the element itself. From there, drop in an image tag with a broken `src` and an `onerror` handler — a classic way to get JavaScript to run without needing `<script>` tags, which some contexts block:

```
"></select><img src=1 onerror=alert(1)>
```

**5. Load the full exploit URL**

```
/product?productId=1&storeId="></select><img%20src=1%20onerror=alert(1)>
```

(Spaces are URL-encoded as `%20` since this is going in a query string.)

Load that URL. The malformed `src=1` on the `<img>` tag fails to load anything, which fires the `onerror` handler — and that handler is just `alert(1)`. You should see the alert box pop immediately, no click required, since `document.write` executes as part of the page loading.

## The payload

```
"></select><img src=1 onerror=alert(1)>
```

Broken down:

- `">` — closes out any open attribute/tag context we might be in
- `</select>` — closes the select element we were injected inside of, so our new tag isn't just more (inert) text inside a dropdown
- `<img src=1 onerror=alert(1)>` — a self-contained payload that doesn't need `<script>` tags at all; a deliberately broken image source guarantees the `onerror` event fires, running our JavaScript

## Why this works

`document.write()` is one of the most dangerous sinks in the DOM XSS world because it does exactly what its name says — writes whatever string it's given straight into the page's HTML, no escaping, no filtering, nothing. It doesn't matter that the value started out as `location.search`, a source the developer might not have thought of as "user input" in the traditional server-side sense — the URL is entirely under the visitor's control, and anyone can craft a link with any `storeId` they want.

Once we know our value lands inside a `<select>`, the exploit is just standard markup breakout: close whatever's currently open, then inject something that executes on its own. The `<img onerror=...>` trick is popular specifically because it doesn't rely on `<script>` tags being processed — `document.write` does execute injected `<script>` tags too in most cases, but `onerror` is a reliable fallback that works in more contexts and needs no closing tag interplay.

## Tools used

- Browser (URL bar + dev tools)
- Optionally Burp Suite, though this lab is fully explorable without intercepting any traffic — the whole vulnerability lives client-side

## Takeaways

- DOM-based XSS doesn't require anything to touch the server at all — the source, the sink, and the vulnerability can live entirely in JavaScript that already shipped to the browser.
- `location.search` is just as attacker-controlled as any server-side input; anyone can construct a URL with arbitrary query parameters.
- `document.write()` is a genuinely hazardous sink — it writes raw markup, so any unsanitized data reaching it becomes live HTML.
- Always check the actual DOM context your input lands in (via Inspect) before crafting a breakout payload — a `<select>` needs different markup to escape than, say, a plain `<div>` or a JS string.
- `<img src=x onerror=...>` is a reliable go-to when you need JS execution without depending on `<script>` tag parsing behavior.

## Fixing it

- Avoid `document.write()` entirely where possible — it's a legacy API with no built-in escaping, and safer DOM manipulation methods (`textContent`, `createElement` + `appendChild`, etc.) exist for basically every use case.
- If dynamic content must be inserted into HTML, HTML-encode it first, and be aware that encoding rules differ depending on which context (tag body, attribute, URL, JS string) the data lands in.
- Never trust `location.search`, `location.hash`, or any other client-side "source" just because it's not a traditional server parameter — from the browser's perspective, it's fully attacker-controlled.
- Apply a Content Security Policy that restricts inline event handlers and script execution, which would blunt payloads like this one even if the underlying sink issue wasn't fixed.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
