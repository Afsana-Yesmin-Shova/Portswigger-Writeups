# Multi-Step Process with No Access Control on One Step

**Category:** Access Control
**Lab:** Multi-step process with no access control on one step
**Status:** Solved ✅

---

## What's going on here

The admin panel here has a "promote user to administrator" flow that's built as a **multi-step process** — presumably a first request that shows a confirmation prompt ("are you sure you want to promote this user?"), and a second request that actually performs the promotion once confirmed. That's a completely reasonable UX pattern. The bug is that access control was only enforced on the *first* step. Whoever built this apparently checked "is this user an admin?" when the confirmation page is requested, but never repeated that check on the step that actually executes the privilege change — treating the second request as implicitly trusted just because it's part of a flow that started with a valid check.

**Goal:** log in as the low-privilege user `wiener` and promote yourself to administrator, exploiting the fact that only one of the two steps is actually protected.

You're given two sets of credentials: `administrator:admin` to explore the legitimate admin panel and see how the promotion flow is supposed to work, and `wiener:peter`, the account we actually need to escalate.

## Working through it

**1. Log in as the administrator**

Use `administrator:admin` to log in and get a first-hand look at the promotion functionality as it's meant to be used.

**2. Walk through promoting a user, and capture the final confirmation request**

Go to the admin panel, and promote the `carlos` account to administrator, with Burp's Proxy running. You're specifically after the *last* request in that flow — the one that actually fires once you click through the confirmation, which is presumably the step performing the real privilege change. Send that request to Repeater.

**3. Log in as the low-privilege user, in a separate session**

Open a private/incognito browser window (keeping your existing admin session alive in the other window), and log in there with `wiener:peter`.

**4. Swap in wiener's session and target username**

Back in Burp, grab the session cookie from this new, non-admin login, and paste it into the captured confirmation request in Repeater — replacing the admin session cookie that was there. Then change whatever parameter identifies the target username (likely something like `username=carlos`) to `username=wiener`, so the request is now asking to promote *yourself*, authenticated as your own low-privilege session.

**5. Send it**

Replay the modified request. If access control really was only checked on the earlier step of the flow (the one that renders the confirmation prompt) and not on this final, actually-privileged action, the server processes it anyway — promoting `wiener` to administrator despite the request coming from a completely unprivileged session.

**6. Confirm the privilege escalation**

Refresh or revisit the account/admin panel as `wiener` — you should now have full administrator access, solving the lab.

## Why this works

Multi-step processes are a common and reasonable pattern for anything sensitive or destructive — an initial "are you sure?" step followed by a final confirming action reduces the risk of accidental clicks causing serious changes. But each step in that flow is still just an independent HTTP request as far as the server is concerned, and each one needs its own access control check. What seems to have happened here is that the developer checked the user's role when rendering the *first* step (the confirmation page itself — makes sense, since you'd only want an admin to even see that prompt), and then assumed that by the time the *second* request arrives, the user must already be legitimately authorized, simply because it's "part of the same flow."

But HTTP requests don't inherently carry any memory of what came before them — nothing forces someone to go through step one before firing off step two directly. An attacker with any valid session (even a totally unprivileged one) can just skip straight to replaying the final, unguarded request, and if that request alone is sufficient to perform the sensitive action, the earlier "protected" step was never actually protecting anything at all.

## Tools used

- Burp Suite (Proxy + Repeater)
- Two browser sessions (a regular window plus an incognito/private one, to hold two separate logins simultaneously)

## Takeaways

- Every step of a multi-step process needs independent access control validation — checking permissions once at the start of a flow and trusting every subsequent request in that flow is a common and dangerous mistake.
- The request that actually performs a sensitive action (not just the one that displays a confirmation prompt) is the one that matters most for access control — and it's often the one developers forget to protect, since it feels like it's "downstream" of an already-checked step.
- Capturing and replaying the *final* step of a flow with a different, lower-privileged session is a fast, reliable way to test for exactly this class of bug.
- Keeping two separate browser sessions open (a normal window plus incognito) is a simple and effective way to hold two different user identities simultaneously while testing access control issues like this one.

## Fixing it

- Enforce access control checks independently on every request that performs a sensitive or state-changing action, not just on the step that initiates or confirms the flow.
- Never assume a request is legitimate just because it's structured as "part of" a multi-step process — validate the requester's authorization on each individual request, every time.
- Where possible, tie multi-step flows together with a server-side, single-use token or state check (confirming step one was actually completed by this same authorized session before step two is allowed to run) — though this is a secondary safeguard, not a substitute for checking authorization on every step.
- Regularly audit multi-step or wizard-style admin functionality specifically for this pattern — it's a genuinely common real-world finding, not just an academy exercise.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
