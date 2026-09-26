# Clobbering DOM Attributes to Bypass HTML Filters

**Category:** DOM-Based Vulnerabilities — DOM Clobbering
**Lab:** Clobbering DOM attributes to bypass HTML filters
**Status:** Solved ✅

---

## What's going on here

This lab targets a different piece of the DOM clobbering puzzle than the last one — instead of clobbering a variable to smuggle in a malicious *value*, here we clobber a property that the sanitizer itself relies on internally, which breaks its filtering logic entirely and lets us inject whatever HTML attributes we want. The sanitizer in play is **HTMLJanitor**, and the bug lives in exactly how it decides which attributes are allowed to survive.

**Goal:** get `print()` to fire in a victim's browser by clobbering HTMLJanitor's own attribute-filtering mechanism, then use the exploit server to make the payload trigger automatically once delivered. Chrome only for this one — the intended solution doesn't work in Firefox.

## How the filter actually breaks

HTMLJanitor filters attributes on an element by iterating over that element's `attributes` property — a live, array-like collection the DOM exposes automatically for every element, listing all of its current attributes. The library's filtering logic checks `attributes.length` and loops through it to decide what survives.

Here's the catch: `attributes` isn't a hardcoded, protected property — it's just another DOM-exposed name, and DOM clobbering can override it exactly the same way it overrides any other implicit global or element property. If we can get an element with `id="attributes"` sitting *inside* the element being filtered, that inner element clobbers the outer element's own `attributes` property, replacing the real (correctly-populated) attribute list with a reference to our injected element instead. Since our injected element obviously doesn't have a numeric `.length` the way a real attribute list does, `attributes.length` becomes `undefined` — and the filtering loop, which presumably expects to iterate zero-or-more real attributes, effectively does nothing at all. The element sails through completely unfiltered, attributes and all.

## Working through it

**1. Post the clobbering comment**

Go to a blog post and submit a comment containing:

```html
<form id=x tabindex=0 onfocus=print()><input id=attributes>
```

Breaking this down:

- `<form id=x tabindex=0 onfocus=print()>` — this is the actual element we want HTMLJanitor to leave untouched. It carries a dangerous `onfocus` handler that calls `print()`, and `tabindex=0` makes it focusable via script (forms aren't focusable by default, but adding a `tabindex` makes any element eligible to receive focus).
- `<input id=attributes>` — nested inside the form, this is the clobbering element. Since its `id` is `attributes`, it clobbers the form's own `attributes` property once both elements are in the DOM together, breaking HTMLJanitor's ability to correctly enumerate (and therefore strip) the form's actual attributes — including the `onfocus` handler that would otherwise get stripped as dangerous.

**2. Build the auto-trigger page on the exploit server**

Having the payload sit in a comment isn't enough on its own — an `onfocus` handler only fires when the element actually receives focus, and nobody's going to manually click into a hidden form field. So we need a way to programmatically focus that exact element in the victim's browser. Go to the exploit server and use:

```html
<iframe src=https://YOUR-LAB-ID.web-security-academy.net/post?postId=3 onload="setTimeout(()=>this.src=this.src+'#x',500)">
```

Swap in your actual lab ID, and make sure `postId` matches whichever blog post you left the comment on in step 1.

**3. Understand what the iframe trick does**

When this iframe loads the blog post page, its `onload` handler waits 500 milliseconds — giving the page time to fully render, including the comment section where our clobbered form lives — and then appends `#x` to the iframe's own URL. That fragment identifier tells the browser "scroll to and focus the element whose `id` is `x`," which is exactly the `id` we gave our malicious `<form>`. The browser automatically shifts focus onto it as a result of navigating to that fragment — no click required, no user interaction at all.

**4. Store and deliver**

Store the exploit and click **Deliver to victim**. When their browser loads the iframe, waits out the delay, and jumps to the `#x` fragment, focus lands on our form — firing its `onfocus` handler and calling `print()`. Lab solved.

## The payloads

Comment posted on the blog post:

```html
<form id=x tabindex=0 onfocus=print()><input id=attributes>
```

Exploit server body:

```html
<iframe src=https://YOUR-LAB-ID.web-security-academy.net/post?postId=3 onload="setTimeout(()=>this.src=this.src+'#x',500)">
```

## Why this works

This is a nice example of DOM clobbering attacking the *sanitizer itself* rather than attacking application logic downstream of sanitization. HTMLJanitor's filtering approach assumes it can trust the standard, browser-provided `attributes` collection on any element it's inspecting — a reasonable assumption in isolation, since that collection is normally a read-only, browser-maintained live list. What it doesn't account for is that a *child* element with a matching `id` can shadow that property on its *parent*, because DOM clobbering isn't scoped to `window` alone — it applies to any element whose properties can be overridden by descendant elements carrying matching `id`/`name` attributes, and `attributes` on an HTML element works exactly the same way `defaultAvatar` did on `window` in the earlier lab.

Once the sanitizer's own internal bookkeeping is corrupted, its "safe" element passes through with genuinely dangerous attributes intact — the filter isn't tricked into permitting something on an allowlist, it's disabled from actually processing the element at all.

## Tools used

- Browser (Chrome — required, this lab's approach doesn't work in Firefox)
- Blog comment form
- The lab's exploit server

## Takeaways

- DOM clobbering doesn't only target application-level global variables — it can target properties a *sanitizer library itself* relies on internally, breaking the filter's own logic rather than just smuggling a malicious value past it.
- Any code that trusts `element.attributes` (or similar DOM-exposed collections) without validating that it actually got a real, expected collection back is vulnerable to this same class of bug.
- URL fragments (`#x`) triggering automatic focus is a legitimate, standard browser behavior — reused here as a zero-click way to fire an `onfocus` handler without any real user interaction.
- Multi-stage delivery (comment now, exploit-server page later) is a recurring pattern in DOM clobbering labs — the poisoning step and the triggering step are often genuinely separate actions.
- Sanitization libraries are regular JavaScript, subject to the same DOM-clobbering assumptions as any other client-side code — using a well-known library doesn't automatically make an app immune to this bug class.

## Fixing it

- Sanitizer authors should never assume DOM-exposed properties like `attributes`, `children`, or similar live collections are trustworthy without verifying their actual type — checking `attributes instanceof NamedNodeMap` (or equivalent) before relying on `.length` and iteration would have caught this.
- Application developers should keep sanitization libraries up to date, since DOM-clobbering weaknesses like this one get identified and patched over time as the technique becomes better understood.
- Avoid attaching security-critical filtering logic to property names that legitimate untrusted content (like `id="attributes"`) can realistically collide with — a defense-in-depth approach doesn't rely on any single unguarded property staying intact.
- As with the earlier DOM clobbering lab, a strict Content Security Policy limiting inline event handlers would reduce the practical impact even if the underlying clobbering bug isn't fixed.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
