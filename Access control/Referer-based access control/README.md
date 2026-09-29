# Referer-Based Access Control

**Category:** Access Control
**Lab:** Referer-based access control
**Status:** Solved ✅

---

## What's going on here

This lab's admin functionality doesn't actually check who's making the request — it checks the `Referer` header, and specifically whether it looks like the request originated from a page within the admin panel itself. The reasoning presumably being: "only someone already inside the admin panel would have a page to link/redirect from, so if the Referer shows that, the request must be legitimate." That's a fundamentally weak basis for an access control decision, because the `Referer` header is entirely client-controlled and trivially spoofable.

**Goal:** log in as low-privilege user `wiener` and promote yourself to administrator by forging a `Referer` header the app mistakenly trusts.

You're given `administrator:admin` to explore the real admin panel first, and `wiener:peter` — the account we're actually escalating.

## Working through it

**1. Log in as the administrator**

Use `administrator:admin` and get familiar with the promote-user functionality in the admin panel.

**2. Promote a test user and capture the request**

Promote `carlos` to administrator, with Burp's Proxy running, and send that captured request to Repeater.

**3. Confirm the Referer check actually matters**

Open a private/incognito window and log in there as `wiener:peter` — our target low-privilege account. Try browsing directly to:

```
/admin-roles?username=carlos&action=upgrade
```

You should get rejected — unauthorized. This confirms the endpoint isn't purely checking session/role server-side in a way that would block wiener outright regardless of headers; it's specifically the *missing Referer header* (since navigating directly via URL bar doesn't send one) that's triggering the rejection here, which tells us the access control logic is Referer-dependent rather than a straightforward permission check.

**4. Swap in wiener's session on the captured request**

Back in Burp Repeater, take the request you captured in step 2 (which still has a legitimate-looking `Referer` header pointing at the admin panel, since that's exactly where it was originally sent from) and replace its session cookie with wiener's session cookie from your incognito login.

**5. Change the target username**

Update the `username` parameter from `carlos` to `wiener`, so the request is now asking to promote yourself.

**6. Send it**

Replay the request. Since it still carries the original, legitimate `Referer` header from when the admin genuinely sent it, and the app is trusting that header as its access control signal rather than actually verifying the session's role server-side, the promotion goes through — even though the session cookie now belongs to an entirely unprivileged user.

**7. Confirm the escalation**

Check wiener's account — you should now have administrator privileges, solving the lab.

## The trick in one sentence

The app checks whether a request *looks like* it came from inside the admin panel (via `Referer`) instead of checking whether the *session making the request* actually has admin privileges — and since `Referer` is just a header we can reuse or forge freely, that check protects nothing.

## Why this works

`Referer` is meant to tell a server "here's the page the user was on when they clicked/submitted whatever led to this request" — it's genuinely useful for things like analytics or basic UX logic. It was never designed to be a trustworthy security signal, because the browser sends whatever value is associated with the *previous page*, and that's something entirely within the requester's control. Modifying it, omitting it, or reusing one captured from a completely different, legitimately-authorized request costs nothing and requires no special access.

Here specifically, the flaw is almost architectural: the presence of a plausible `Referer` value is being treated as a proxy for "this request came from someone already authorized to be in the admin panel" — but that's conflating *where a request appears to originate* with *who is actually making it*. A captured admin request's `Referer` header stays exactly as valid-looking after we swap out the session cookie underneath it, because nothing about the `Referer` value itself ties it to the specific session that generated it.

## Tools used

- Burp Suite (Proxy + Repeater)
- Two browser sessions (regular + incognito, to hold both accounts simultaneously)

## Takeaways

- The `Referer` header is a client-supplied, fully spoofable value and should never be treated as an access control mechanism — it's useful for informational purposes only, never for authorization decisions.
- Capturing one legitimate, correctly-headed request and simply swapping out its session cookie is a fast way to test whether an app is conflating "this request has the right shape" with "this request came from an authorized session."
- Testing what happens when a header a request normally carries is simply *missing* (like navigating directly via URL bar, which sends no `Referer`) is a great way to confirm that header is actually load-bearing for access control, before going to the trouble of forging it.
- This bug pattern — trusting a client-controlled signal instead of verifying server-side session state — shows up under a lot of different disguises (Referer, custom headers, hidden form fields, even the request path itself); the underlying lesson generalizes well beyond this one header.

## Fixing it

- Never use the `Referer` header (or any other client-controlled value) as an access control mechanism — authorization decisions must be based on server-side session state tied to an authenticated, verified identity.
- Check the current session's actual role/permissions on every sensitive request, independent of any headers describing where the request claims to have originated.
- If Referer-based logic exists for legitimate reasons (like basic UX flows or analytics), keep it entirely separate from anything security-relevant, and never let its absence or presence gate access to privileged functionality.
- Regularly audit admin and privileged endpoints specifically for this pattern — genuine session/role verification should be happening on the server for every request, with no shortcuts based on incidental request metadata.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
