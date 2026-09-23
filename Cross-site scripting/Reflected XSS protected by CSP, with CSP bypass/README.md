# Reflected XSS Protected by CSP — with a CSP Bypass

**Category:** Cross-Site Scripting (XSS)
**Lab:** Reflected XSS protected by CSP, with CSP bypass
**Status:** Solved ✅

---

## What's going on here

This lab has a real, working XSS vulnerability — but the site also ships a Content Security Policy, and CSP is specifically designed to neuter exactly this kind of bug. Injected `<script>` tags or inline event handlers should just get silently blocked by the browser, no matter how clean the injection point is. The twist: the CSP itself has a hole in it, because one of its own directives is being built out of a value we control.

**Goal:** find the flaw in the CSP configuration and use it to get `alert()` to actually execute, despite the policy nominally being there to stop it. Note: PortSwigger's intended solution for this one only works in Chrome.

## First, confirm the XSS is real but blocked

**1. Try the obvious payload**

Drop a standard image-based XSS payload into the search box:

```
<img src=1 onerror=alert(1)>
```

**2. Check what happens**

Submit it and look at the response — you'll see your payload reflected back into the page HTML exactly as sent. But no alert fires. Open the browser console and you'll see a CSP violation logged, blocking the inline `onerror` handler from executing. So the injection point is genuinely there; it's just that the browser is refusing to run whatever ends up in it.

## Finding the actual hole

**3. Look at the CSP header itself**

Switch to Burp and inspect the response headers. You'll see a `Content-Security-Policy` header, and within it, a `report-uri` directive — the mechanism sites use to have the browser report policy violations back to a logging endpoint. Look closely at the value of that directive, and you'll notice it includes a `token` parameter, something like:

```
report-uri /csp?token=abc123
```

**4. Realize the token is reflected from a request parameter**

Here's the actual bug: that `token` value isn't a fixed server-side secret — it's being pulled straight from a `token` parameter in the request and echoed back into the CSP header itself. Since we control the request, we control part of the CSP header's content. And a CSP header is just a semicolon-separated list of directives — so if we can inject a semicolon and more text into that token value, we can inject an entirely new directive into the policy the browser is about to enforce.

## Building the bypass

**5. Craft a value that appends a new directive**

The plan: use the `token` parameter to close out the existing `report-uri` directive early and add our own `script-src-elem` directive, loosened to allow inline scripts:

```
;script-src-elem 'unsafe-inline'
```

`script-src-elem` specifically governs `<script>` *elements* (as opposed to inline event handler attributes, which fall under a different, more specific policy category in modern CSP). Since directives listed later in a CSP header can override earlier ones for the same category, injecting a fresh `script-src-elem 'unsafe-inline'` effectively relaxes the restriction on `<script>` tags, even if the original policy locked that down.

**6. Combine it with a plain script-tag payload**

Since we're now allowed to use inline `<script>` elements (rather than needing an event-handler-based trick like before), the payload itself can be refreshingly simple:

```
<script>alert(1)</script>
```

**7. Assemble the full URL**

Put both pieces into the request — the XSS payload in `search`, and the CSP-loosening injection in `token`:

```
https://YOUR-LAB-ID.web-security-academy.net/?search=<script>alert(1)</script>&token=;script-src-elem%20'unsafe-inline'
```

(URL-encoded, that's `%3Cscript%3Ealert%281%29%3C%2Fscript%3E` for the search payload, and `%20` for the space before `'unsafe-inline'`.)

**8. Load it and confirm**

Visit that URL in Chrome. The server reflects our `token` value into the `Content-Security-Policy` header, appending our `script-src-elem 'unsafe-inline'` directive to the policy the browser receives. That directive permits inline `<script>` elements, so our injected `<script>alert(1)</script>` is no longer blocked — the alert fires.

## The payload

Two pieces working together:

```
search=<script>alert(1)</script>
token=;script-src-elem 'unsafe-inline'
```

## Why this works

CSP is only as trustworthy as the header that delivers it. Here, the developer built part of that header dynamically from a request parameter — almost certainly to support tracking *which* request a CSP violation report corresponds to (a legitimate, common pattern) — but never accounted for the fact that CSP header syntax uses semicolons as directive separators, and that an attacker-controlled value landing unescaped in that header is just as dangerous as an attacker-controlled value landing unescaped in HTML.

This turns the CSP from a fixed, trustworthy allowlist into something the attacker can partially rewrite on the fly. Directive ordering matters here too: later directives of the same type generally take precedence, so appending a permissive `script-src-elem` after the original restrictive one effectively wins.

The broader lesson: a CSP is a genuinely strong XSS defense — but only if every part of it is static or comes from a source the attacker can't influence. The moment any piece of the header is built from user input, the policy itself becomes an injection target.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser (Chrome specifically, for this lab's intended solution)

## Takeaways

- CSP is a real, effective mitigation against XSS — this lab isn't a knock against CSP in general, it's a demonstration of what happens when the CSP header itself is partially attacker-controlled.
- Always inspect the actual CSP header value when testing a site that has one — don't just assume it's static. Look for any piece of it that might be templated from request data (tokens, nonces, callback URLs, report endpoints).
- CSP directives are semicolon-delimited, so injecting a semicolon into any attacker-influenced portion of the header is the key that unlocks adding arbitrary new directives.
- `script-src-elem` is a more specific, newer directive than the general `script-src` — worth knowing it exists and that it can be used (or abused) to target script elements specifically, separately from other script-related restrictions.
- A vulnerability and its mitigation living side by side doesn't mean you're safe — always test whether the mitigation itself has been implemented soundly, not just whether it's present.

## Fixing it

- Never construct any part of a CSP header from user-controllable input. If a token or identifier needs to appear in `report-uri` (or any directive), validate it strictly against an expected format (e.g., a fixed-length alphanumeric string) and reject anything containing characters like `;` that have syntactic meaning in the header.
- Prefer a static, hardcoded CSP wherever possible — the simpler and more fixed the policy, the smaller the attack surface for this class of bug.
- If per-request tracking is genuinely needed for violation reporting, put that identifier in the report endpoint's own logic (e.g., correlate by session or a server-side-generated value stored out of band) rather than embedding it directly in the policy the browser receives.
- Regularly test your own CSP configuration for reflected/injectable values, the same way you'd test any other security-relevant header.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
