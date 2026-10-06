# SameSite Lax Bypass via Method Override

**Category:** Cross-Site Request Forgery (CSRF) — Bypassing SameSite Restrictions
**Lab:** SameSite Lax bypass via method override
**Status:** Solved ✅

---

## What's going on here

Unlike the "no defenses" lab, this site actually does benefit from a real, modern CSRF defense — just not one it explicitly configured. The session cookie carries no CSRF token in the form, but modern browsers apply a default `SameSite=Lax` restriction to any cookie that doesn't explicitly set a `SameSite` value at all. That default restriction genuinely blocks the most obvious CSRF technique: a forged cross-site `POST` won't carry the session cookie along with it under `Lax` rules. The bug that makes this lab solvable isn't in the cookie policy at all — it's a method-override feature on the server itself, which turns out to be exactly the loophole `SameSite=Lax` leaves open.

**Goal:** change a logged-in victim's email address via CSRF, despite the session cookie's default `Lax` restriction blocking a straightforward cross-site `POST`. Chrome is specifically recommended for testing, since the victim's simulated browser uses it and `SameSite` default behavior can differ between browsers.

## Understanding Lax, and the gap it leaves open

`SameSite=Lax` — the default modern browsers apply when a cookie doesn't specify `SameSite` explicitly — doesn't block *all* cross-site requests from carrying cookies. It specifically still allows the cookie through for **top-level navigation `GET` requests** — the ordinary case of a user clicking a link or being redirected to a new page, which browsers treat as a safe enough category of cross-site interaction to still include cookies for. What `Lax` *does* block is cookies on cross-site `POST` requests (among other non-`GET`, non-top-level cases), which is exactly the kind of request a standard CSRF form submission normally relies on.

So: a forged cross-site `POST` to `/my-account/change-email` won't carry the session cookie, and the request will fail as unauthenticated. But a forged cross-site `GET`, triggered via top-level navigation (like redirecting the browser's own address bar, not an embedded image or iframe), absolutely will carry it. The only problem: the actual endpoint requires `POST`. Unless, that is, the server itself provides a way to make a `GET` request behave like a `POST`.

## Working through it

**1. Study the change-email request**

Log in with `wiener:peter` in Burp's browser, submit the "Update email" form, and find the request in **Proxy → HTTP history**. You'll see it's a `POST /my-account/change-email`, carrying no CSRF token at all.

**2. Check how the session cookie's SameSite attribute is set**

Look at the response to the earlier `POST /login` request — specifically its `Set-Cookie` header. You'll notice it doesn't explicitly specify any `SameSite` value for the session cookie. Since the website hasn't opted into a stricter policy, the browser falls back to its default — `Lax` — automatically.

**3. Recognize what that default actually permits**

Given `Lax`'s behavior, the session cookie *will* still be sent on a cross-site `GET` request, as long as it results from a top-level navigation (redirecting the whole page, not loading something in the background via `fetch` or an `<img>` tag). That's our opening — if only the endpoint accepted `GET`.

**4. Confirm the endpoint rejects GET as-is**

Send the original `POST` request to Repeater, right-click, and **Change request method** to convert it to `GET`. Send it — you should get a response indicating the endpoint only accepts `POST`.

**5. Try a method override parameter**

Rather than giving up, try appending a `_method` parameter to the query string, set to `POST`:

```
GET /my-account/change-email?email=foo%40web-security-academy.net&_method=POST HTTP/1.1
```

Plenty of backend frameworks support exactly this kind of override as a convenience feature — letting clients that can't easily send real `PUT`/`DELETE`/etc. requests (older HTML forms, for instance, which only support `GET` and `POST` natively) signal their *intended* method via a parameter instead, while the actual HTTP verb stays `GET` or `POST`.

**6. Send it and confirm it's accepted**

If the app supports this override, the request should now go through successfully, despite physically being a `GET` request on the wire.

**7. Verify in the browser**

Check your account page — your email should now show the new value, confirming the override genuinely changed it.

## Building the actual exploit

**8. Go to the exploit server**

In the **Body** section, we need something that triggers a genuine top-level navigation to our crafted URL — not an embedded resource load, since `Lax` specifically only exempts top-level navigations. A simple JavaScript redirect does exactly that:

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=pwned@web-security-academy.net&_method=POST";
</script>
```

Setting `document.location` causes the entire browser tab to navigate to that URL — a genuine top-level navigation, which is precisely the category `Lax` still permits cookies on.

**9. Store and test on yourself**

Store the exploit, click **View exploit**, and confirm your own email address changes as a result.

**10. Swap in a different email for the real delivery**

Update the email value so it's not the one you just used for testing, avoiding any "address already taken" conflict.

**11. Deliver it**

Store again, then click **Deliver to victim**. When the simulated victim's browser (Chrome, per the lab's note) loads the page, the script fires a top-level navigation to our crafted URL — the session cookie goes along for the ride under `Lax` rules, the `_method=POST` override convinces the server to treat this `GET` as the real email-change action, and the victim's email changes, solving the lab.

## The payload

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=pwned@web-security-academy.net&_method=POST";
</script>
```

## Why this works

Two separate, individually-reasonable pieces combine into a working bypass here. `SameSite=Lax` is a genuinely effective modern CSRF mitigation — but it was deliberately designed with an exception for top-level navigation `GET` requests, specifically to avoid breaking ordinary, legitimate cross-site linking behavior (clicking a link from an external site into a logged-in session shouldn't silently log you out or misbehave). That's a sensible tradeoff on its own.

The method-override feature is also individually reasonable — plenty of legitimate use cases exist for letting a client signal an intended HTTP method that the literal request verb doesn't support natively. The problem only emerges once both are present on the same endpoint: the override effectively turns a `GET` request — the one category `Lax` still permits across sites — into something that performs exactly the same sensitive action a `POST` was supposed to be required for. `SameSite=Lax` was never actually bypassed in a technical sense; the attack simply routes around the specific category of request it restricts, using a feature the developer added for an unrelated reason.

## Tools used

- Burp Suite (Proxy + Repeater)
- The lab's exploit server
- Browser (Chrome specifically, matching the victim's simulated browser)

## Takeaways

- `SameSite=Lax` (the modern browser default) still permits cookies on cross-site `GET` requests that result from top-level navigation — it's not a blanket block on all cross-site cookie-bearing requests, and that exception is exactly what needs testing whenever `Lax` is the only defense in place.
- Method-override parameters (`_method`, `X-HTTP-Method-Override`, and similar conventions) are convenient for legitimate client compatibility reasons, but they quietly reopen any protection that assumed a sensitive action could only be reached via a specific HTTP verb.
- `document.location = "..."` in a `<script>` tag is the key building block for forcing a genuine top-level navigation from a CSRF delivery page — distinct from loading a URL via `fetch`, an `<img>`, or an iframe, none of which count as top-level navigation for `Lax` purposes.
- Whenever an app relies solely on `SameSite=Lax` (rather than `Strict`) and also happens to support method overriding on any sensitive endpoint, that combination is worth testing specifically — it's a known, named bypass pattern, not just a one-off academy trick.

## Fixing it

- Set `SameSite=Strict` on session cookies for any application where cross-site top-level navigation access genuinely isn't needed — `Strict` blocks cookies on cross-site requests entirely, including top-level navigations, closing this gap completely.
- Avoid supporting method-override parameters on sensitive, state-changing endpoints — if method overriding is needed for legitimate compatibility reasons elsewhere, exclude security-critical actions like account changes from that mechanism entirely.
- Implement genuine CSRF tokens as the primary defense regardless of `SameSite` cookie settings — `SameSite` is a strong complementary browser-level mitigation, but relying on it alone (especially at the `Lax` level) leaves exactly this kind of gap.
- Audit any endpoint reachable via multiple methods (real or overridden) for consistent authorization and CSRF protection across all of them — a defense applied to only one method variant of an endpoint protects nothing if another variant performs the identical action.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
