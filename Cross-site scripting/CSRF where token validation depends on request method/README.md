# CSRF Where Token Validation Depends on Request Method

**Category:** Cross-Site Request Forgery (CSRF)
**Lab:** CSRF where token validation depends on request method
**Status:** Solved ✅

---

## What's going on here

This lab's "Update email" feature does actually have CSRF protection — there's a `csrf` token in the form, and the app does check it. The bug isn't that the token is missing or weak; it's that the check only applies to *some* requests, and it turns out the developer forgot to enforce it consistently across HTTP methods. If the normal flow uses `POST`, but the endpoint also happens to accept `GET`, and only the `POST` path validates the token, then switching methods quietly walks straight past the entire defense.

**Goal:** build a page that forges an email-change request against a logged-in victim's account, without ever needing a valid CSRF token.

You're given working credentials for testing (`wiener:peter`) — use those to explore the app yourself before building the final exploit, and keep in mind you can't register an email address someone else already has, so use a throwaway address for the real payload, not one you've already tested with.

## Working through it

**1. Capture the legitimate request**

Log in through Burp's browser, go to the "Update email" form, and submit a change. Find that request in your Proxy history — it should be a `POST` to something like `/my-account/change-email`, with `email` and `csrf` as body parameters.

**2. Confirm the token is actually being checked**

Send the request to Repeater, then just tamper with the `csrf` parameter's value — change a character or two — and resend. You should get the request rejected. Good: that confirms token validation is genuinely happening for this request, at least in its normal form.

**3. Try switching the method**

Right-click the request in Repeater and use **Change request method** to convert it from `POST` to `GET`. Burp handles moving the body parameters into the query string automatically. Send it — and this time, even with a garbage or missing `csrf` value, the request goes through and the email actually changes.

That's the whole bug laid bare: whatever validation logic checks the `csrf` parameter only runs along the `POST` code path. The server apparently still accepts `GET` requests to the same endpoint and processes them identically in every other respect — just without bothering to check the token.

## Building the CSRF exploit

**4. Generate (or write) the exploit HTML**

If you've got Burp Suite Professional, right-click the `GET` request in Repeater and go **Engagement tools → Generate CSRF PoC**, tick the auto-submit option, and click Regenerate — Burp builds the whole page for you.

Otherwise (Community Edition), just build it by hand. Since this is now a `GET` request, no form submission trickery with hidden POST fields is even needed — a request that just *loads* is enough, but the straightforward version PortSwigger uses is a self-submitting form:

```html
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="anything@web-security-academy.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

Note there's no `method` attribute on the `<form>` tag — that defaults to `GET`, which is exactly what we need to land on the unprotected code path. Grab the actual request URL for your lab instance by right-clicking the captured request and choosing **Copy URL**, and use that as the form's `action`.

**5. Host it on the exploit server**

Paste that HTML into the **Body** section of your exploit server and click **Store**.

**6. Test it on yourself first**

Click **View exploit** and confirm the request actually fires and the response looks right, using your own test account — this is exactly why the lab warns you to use a *different* email address for the real delivery, since you can't reuse one that's already registered.

**7. Swap in a fresh email address**

Update the `value` in the hidden `email` field to something you haven't already used, so it doesn't collide with whatever you tested with in step 6.

**8. Store and deliver**

Save the exploit again, then click **Deliver to victim**. The simulated victim's browser loads your page, the form auto-submits a `GET` request to the change-email endpoint using their existing session cookie, and since that code path never checks the CSRF token, their email address gets silently changed. Lab solved.

## Why this works

CSRF tokens work by requiring proof that the request came from a page the legitimate site actually served — something a third-party attacker page can't produce or predict. That protection is only as strong as its *enforcement*, though. Here, the validation logic was apparently written with just the normal `POST`-based form submission in mind, and never accounted for the same endpoint accepting other methods too. A `GET` request to the identical URL performs the identical action, but skips the code path where the token gets checked entirely.

This is a really common real-world mistake: security checks bolted onto one specific code path (often the "intended" one a developer was actively thinking about) while a framework or router happily accepts the same logical request through other methods that never route through that check. The vulnerability isn't a weak token — it's an incomplete enforcement surface.

## Tools used

- Burp Suite (Proxy + Repeater, Professional for the CSRF PoC generator)
- Exploit server (provided by the lab)
- Browser

## Takeaways

- A CSRF token being present and checked *somewhere* doesn't mean it's checked *everywhere* that endpoint can be reached from — always test alternate HTTP methods against the same URL.
- "Change request method" in Burp Repeater is a fast, easy way to probe exactly this class of bug — it's worth trying on every state-changing endpoint you test.
- State-changing actions (like changing an email address) should really never be reachable via `GET` in the first place — `GET` requests are supposed to be safe/idempotent by convention, and browsers, proxies, and caches all assume as much.
- Self-submitting auto-forms are the standard CSRF delivery mechanism — the victim doesn't need to click anything, just load the page.
- Test any CSRF fix (or claim of one) against every method the endpoint actually accepts, not just the one the legitimate UI happens to use.

## Fixing it

- Validate CSRF tokens on **every** method the endpoint accepts, not just the one used by the app's own front-end — if `GET` is permitted for any reason, it needs the same protection as `POST`.
- Better yet, restrict state-changing endpoints to only accept the HTTP methods that are semantically appropriate — `POST`, `PUT`, `PATCH`, or `DELETE` as relevant — and reject `GET` outright for anything that mutates data.
- Enforce CSRF checks at a shared layer (middleware, a framework-level guard) that automatically applies to a route regardless of method, rather than embedding the check inside method-specific handler logic where it's easy to forget on one branch.
- As defense in depth, pair CSRF tokens with `SameSite` cookie attributes (`Lax` or `Strict`), which limit exactly this kind of cross-site request from carrying session cookies in the first place.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
