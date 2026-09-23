# Reflected XSS in a JavaScript URL with Some Characters Blocked

**Category:** Cross-Site Scripting (XSS)
**Lab:** Reflected XSS in a JavaScript URL with some characters blocked
**Status:** Solved ✅

---

## What's going on here

This is easily the most advanced lab in the series so far. On the surface it looks trivial — your input gets reflected straight into a JavaScript context, no HTML encoding to fight through. But the app is filtering out a chunk of the characters you'd normally reach for, most notably **spaces**, which quietly rules out almost every standard payload you'd try first. `alert(1337)` on its own would be easy — the actual challenge is getting there without ever typing a literal space, and without relying on the obvious syntax.

**Goal:** get `alert()` to fire with `1337` somewhere in the message it displays — using a payload that survives the character filter.

## Where the input lands

The `postId` parameter gets reflected into a `javascript:` URL — almost certainly something powering the "Back to blog" link at the bottom of the post page, roughly shaped like:

```js
javascript:loadPost({x: 'POSTID'})
```

That's a single-argument function call, with our input sitting inside a string that's itself inside an object literal. Normally you'd just close the string, close the object, and tack on your own statement. The wrinkle is doing all of that — plus the actual exploit logic — with no spaces available.

## Building the payload, piece by piece

The full payload (URL-decoded) is:

```
'},x=x=>{throw/**/onerror=alert,1337},toString=x,window+'',{x:'
```

Dropped into the original code, it turns:

```js
loadPost({x: 'POSTID'})
```

into:

```js
loadPost({x: ''},x=x=>{throw/**/onerror=alert,1337},toString=x,window+'',{x:''})
```

Since `loadPost(...)` is a function call, JavaScript is totally fine with extra comma-separated arguments even if the function only uses the first one — the rest just get evaluated and quietly discarded, which is exactly the loophole this exploit lives in. Each "extra argument" here is really just an excuse to smuggle in a full JS expression that runs as a side effect of being evaluated. Let's go through them one at a time:

**Argument 1 — `{x: ''}`**
Just the original object, now with an empty string instead of the postId. Harmless, keeps `loadPost` happy.

**Argument 2 — `x=x=>{throw/**/onerror=alert,1337}`**
This is where the real work happens. It defines a global variable `x` as an arrow function. Since the function needs to `throw` — and `throw` is a *statement*, not an expression, so it can't be squeezed into a one-line arrow body without braces — the arrow function uses a full `{ }` block instead of the usual implicit-return shorthand.

Inside that block:
```js
throw (onerror=alert, 1337)
```
This is a comma expression: `onerror=alert` runs first, setting the global `onerror` handler to the `alert` function itself — meaning any uncaught exception from here on gets handed straight to `alert()`. The comma operator then discards that and evaluates to `1337`, which is what actually gets thrown. Since spaces aren't allowed between `throw` and what follows it, a blank comment (`/**/`) is used in place of the space — comments are whitespace as far as the JS parser cares, so `throw/**/onerror` parses identically to `throw onerror` with an actual space.

**Argument 3 — `toString=x`**
Sets the global `toString` variable equal to that arrow function we just defined. This matters because of what comes next.

**Argument 4 — `window+''`**
This is the trigger. Adding an empty string to the `window` object forces JavaScript to coerce `window` into a string — and part of that coercion process is calling `window.toString()`. Since we just overwrote the global `toString` (which, unqualified in this scope, resolves to `window.toString`) with our arrow function, this line is really invoking our function. That's what actually fires the `throw` — and since we never wrote a literal `(` to call our own function directly, we got JavaScript's own type-coercion machinery to call it for us.

**Argument 5 — `{x: ''}`**
Just padding, there to keep the argument count from breaking anything downstream. Not load-bearing.

## Putting it all together

Once `window+''` runs, here's the actual sequence of events:

1. `window.toString` gets invoked (because of the string coercion).
2. That's our arrow function, which throws the value `1337` — but right before throwing, it sets `window.onerror = alert`.
3. Since the throw is uncaught, the browser's global error handler kicks in — which is now `alert`.
4. The browser calls `onerror` with details about the error, including a message that contains our thrown value, `1337`.
5. `alert()` displays that message — satisfying the lab's requirement that `1337` shows up somewhere in the alert.

## The payload

```
'},x=x=>{throw/**/onerror=alert,1337},toString=x,window+'',{x:'
```

URL-encoded, as it needs to be sent in the query string:

```
%27%7D%2Cx%3Dx%3D%3E%7Bthrow%2F**%2Fonerror%3Dalert%2C1337%7D%2CtoString%3Dx%2Cwindow%2B%27%27%2C%7Bx%3A%27
```

(Or the lab's own shorthand, `%27` for `'` and `%3D` etc. — same idea.)

Load the resulting URL and click "Back to blog" at the bottom of the page to trigger it — the exploit lives in that link's `javascript:` handler, so it only fires once that link is actually clicked.

## Why this works

Two separate constraints made the "obvious" payloads impossible: no spaces, and a context where you can't just write a normal `alert(1337)` call cleanly. The exploit routes around both by leaning entirely on features of JavaScript that don't require the syntax being filtered:

- **No spaces?** Blank comments (`/**/`) are functionally identical to whitespace to the parser, so `throw/**/x` parses exactly like `throw x`.
- **Need to call a function without writing a clean function call?** Overwrite `toString` and force a type coercion — JS will call your function for you as a side effect of trying to convert an object to a string.
- **Need to pass a value to `alert` without calling it directly?** Hijack the browser's global `onerror` handler, which gets automatically invoked (with arguments!) whenever an uncaught exception occurs.

None of this is really "new" JavaScript — it's all standard language behavior (comma operator, comment-as-whitespace, type coercion, global error handling) repurposed to route around a character filter that was only checking for the *shapes* it expected malicious code to take.

## Tools used

- Burp Suite (Proxy + Repeater, to explore the reflection and test the filter)
- Browser

## Takeaways

- Character/keyword blacklists are almost always incomplete — JavaScript has enough redundant ways to achieve the same effect (comments as whitespace, coercion-triggered function calls, exception handlers) that a filter targeting "the obvious payload" rarely covers every path to code execution.
- The comma operator is a genuinely useful tool for smuggling multiple side-effecting expressions into a single statement, especially inside constrained contexts like extra function-call arguments.
- Overwriting `toString` (or `valueOf`) and then forcing a coercion is a classic technique for invoking a function without writing literal call syntax — worth remembering any time parentheses or explicit calls are restricted.
- `window.onerror` will happily receive whatever you `throw`, and pointing it at `alert` turns any uncaught exception into a fully controlled `alert()` call.
- Advanced filter bypasses like this usually come from combining several small, individually-boring JS quirks rather than one clever trick.

## Fixing it

- Never reflect user input directly into a JavaScript execution context, filtered or not — character blacklists are a losing game against a language with this much syntactic flexibility.
- If dynamic values must reach client-side JS, pass them as properly-escaped JSON data consumed by static, trusted code — not as raw text spliced into a script.
- Apply a strict Content Security Policy that blocks `javascript:` URIs and inline script execution; this exploit specifically depends on a `javascript:` link being clickable at all.
- Treat any character-blacklist-based filter as a stopgap, not a fix — the underlying injection point is the real vulnerability, and it needs proper output encoding or elimination, not smarter blacklisting.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
