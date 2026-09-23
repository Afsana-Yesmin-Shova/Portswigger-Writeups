# DOM XSS in an AngularJS Expression (Angle Brackets & Double Quotes HTML-Encoded)

**Category:** Cross-Site Scripting (XSS)
**Lab:** DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded
**Status:** Solved ✅

---

## What's going on here

This lab is interesting because the app's actually doing something right — angle brackets and double quotes are getting HTML-encoded, which normally shuts down most XSS attempts cold. No `<script>` tags, no attribute breakouts, none of the usual tricks work here.

But the search results get rendered inside an element with an `ng-app` attribute — which means this part of the page is running under **AngularJS**, and Angular has its own template syntax that has nothing to do with HTML tags at all. That's the gap: the filter is guarding the HTML layer, but Angular adds an entirely separate execution layer on top of it that doesn't care about angle brackets.

**Goal:** get an AngularJS expression to execute and pop an `alert()`.

## A quick primer on what's exploitable here

AngularJS scans any DOM node marked with `ng-app` and treats double-curly-brace syntax — `{{ ... }}` — as a live expression to evaluate, not literal text. That's a completely legitimate framework feature, normally used for simple things like `{{ user.name }}` to display a value. The problem is that Angular expressions aren't sandboxed the way you'd hope — older AngularJS versions in particular allow expressions to reach JavaScript's `constructor` chain, which is a well-known way to escape the "just template data" mental model and get to arbitrary code execution.

Since curly braces aren't angle brackets or quotes, none of the app's existing encoding does anything to stop this.

## Working through it

**1. Find where your input lands**

Type a random alphanumeric string into the search box and submit it. Then view the page source and find where that string got reflected.

**2. Confirm it's inside an ng-app context**

Look at the surrounding markup — you should see your search term sitting inside (or near) an element carrying the `ng-app` attribute. That's your signal this is Angular territory, and curly-brace expressions will get evaluated here rather than just displayed as text.

**3. Swap the search term for an AngularJS sandbox-escape expression**

Instead of a plain string, search for:

```
{{$on.constructor('alert(1)')()}}
```

**4. Submit and watch it fire**

Click search. If everything lines up, Angular evaluates that expression as part of rendering the results — and since the expression walks through `constructor` to build and immediately invoke a new function containing `alert(1)`, the alert box pops right there.

## The payload

```
{{$on.constructor('alert(1)')()}}
```

Here's what each piece is doing:

- `{{ ... }}` — the AngularJS interpolation syntax that tells Angular "evaluate this as an expression," which is what gets us execution without needing a single angle bracket or quote in the HTML sense
- `$on` — a reference to an existing function accessible from the Angular scope
- `.constructor` — every JavaScript function has a `constructor` property pointing back to the `Function` constructor itself — this is the actual escape hatch, since `Function` lets you build a brand-new function out of a string of code
- `('alert(1)')` — calling that `Function` constructor with a string turns the string into the body of a new function; effectively this constructs `function() { alert(1) }`
- The trailing `()` — immediately invokes the function we just built, which is what actually runs `alert(1)`

## Why this works

The app's defenses — encoding `<`, `>`, and `"` — are aimed squarely at preventing you from injecting raw HTML or breaking out of an HTML attribute. That's a perfectly reasonable defense against classic XSS. But it does nothing against AngularJS's own template engine, which operates as a second, independent layer of interpretation sitting on top of the rendered HTML. Curly braces aren't HTML syntax, so there's nothing for the encoding to catch.

This is really a case of "client-side template injection" — a cousin of server-side template injection, but happening inside a JavaScript framework's expression evaluator instead of a backend templating engine. Whenever a page uses a framework that treats certain unescaped syntax as executable — Angular's `{{ }}`, and other frameworks have their own equivalents — any user input that ends up inside a live template context bypasses whatever HTML-level encoding is in place, because the two layers just aren't looking at the same thing.

## Tools used

- Browser (view source + search box)

## Takeaways

- HTML encoding only protects the HTML layer — it says nothing about template engines or other interpreters that might process the same string afterward.
- Any element scoped under `ng-app` (in older AngularJS apps) treats `{{ }}` as live code, not text, which turns "input reflected into the page" into "input reflected into an execution context."
- The `constructor` property chain is a classic JavaScript sandbox-escape technique — worth recognizing any time you're looking at a constrained expression evaluator, whether that's Angular, a templating engine, or anything else trying to limit what user-supplied expressions can do.
- Client-side template injection is its own bug class, distinct from classic reflected/stored XSS, and needs to be tested for separately — a payload that would be totally inert as raw HTML can still be dangerous inside a framework's template syntax.

## Fixing it

- Upgrade off legacy AngularJS if at all possible — newer Angular versions (2+) use a fundamentally different template system that isn't vulnerable to this class of sandbox escape, and even later AngularJS 1.x releases tightened the expression sandbox considerably.
- Never mix untrusted user input directly into a region of the page governed by `ng-app`/Angular templates without understanding that HTML encoding alone won't protect it.
- Apply a strict Content Security Policy — it won't stop the sandbox escape itself, but it can prevent the resulting `alert()`/arbitrary code from doing anything more damaging in a real attack (like exfiltrating cookies).
- Treat any client-side templating or expression language the same way you'd treat a server-side one: assume user input reaching it needs escaping specific to *that* context, not just the surrounding HTML.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
