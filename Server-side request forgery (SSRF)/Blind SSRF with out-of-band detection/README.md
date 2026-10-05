# Blind SSRF with Out-of-Band Detection

**Category:** Server-Side Request Forgery (SSRF)
**Lab:** Blind SSRF with out-of-band detection
**Status:** Solved ✅

---

## What's going on here

Every SSRF lab so far in this series has been visible — we send a crafted request, and the response hands back something useful, whether that's stock data, admin panel content, or a deleted user. This one's different: the vulnerable feature is some analytics software that fetches whatever URL is sitting in the `Referer` header, server-side, whenever a product page loads — but it never shows us anything about what that fetch returned. No content, no error, nothing reflected back. We have to confirm the vulnerability exists purely by observing that the request happened at all, from somewhere we're independently watching.

**Goal:** get the server to make an outbound HTTP (and/or DNS) request to Burp Collaborator, proving blind SSRF exists, even with zero visible feedback from the app itself.

## Why "blind" changes the approach

With a visible SSRF, you point the vulnerable feature at something you control or something interesting, and read the result directly out of the response. Blind SSRF removes that entirely — the server might very well be making the request we want, but if nothing about that request's outcome ever surfaces anywhere we can see it, we need an independent way to detect that it happened at all. That's exactly what Burp Collaborator is for: it gives you a unique, disposable domain that silently logs any DNS lookup or HTTP request that hits it, completely independent of whatever the vulnerable application's own response says.

## Working through it

**1. Capture a normal product page request**

Visit any product page with Burp's Proxy running, and send the resulting request to Repeater.

**2. Find the Referer header**

In the request, locate the `Referer` header — this is what the backend analytics software is reading and fetching from, per the lab description.

**3. Swap it for a live Collaborator payload**

Highlight the `Referer` header's value, right-click, and choose **Insert Collaborator payload**. Burp replaces the existing value with a unique, freshly-generated Collaborator domain that it's actively monitoring.

**4. Send the request**

Fire it off. As expected, the response you get back looks completely ordinary — the page loads like it always does. That's expected and fine; we're not looking for anything in this response at all.

**5. Poll Collaborator for interactions**

Switch to the **Collaborator** tab in Burp and click **Poll now**. Since the server-side fetch happens as a separate, asynchronous action (not something the browser waits on before returning the page), the callback might not be instant — if nothing shows up right away, wait a few seconds and poll again.

**6. Confirm the interaction**

You should see one or more interactions logged — likely both a DNS lookup and a full HTTP request, since resolving the Collaborator domain and then actually connecting to it both show up as separate, independently detectable events. That confirms it: the application really did take our injected `Referer` value and make an outbound server-side request to it, with zero indication of that fact anywhere in the actual page response. Lab solved.

## The payload

No custom payload to build by hand here — the whole technique is "Insert Collaborator payload" into the `Referer` header, letting Burp generate and track a unique domain for you:

```
Referer: http://YOUR-UNIQUE-SUBDOMAIN.oastify.com/
```

## Why this works

The vulnerability itself is nothing exotic — it's the same root cause as every other SSRF in this series: a server-side feature makes an outbound request to a URL it takes from user-controllable input, with no validation on where that URL actually points. What makes this lab worth doing separately is the detection method, not the bug. Plenty of real-world SSRF vulnerabilities never return anything useful in the response — maybe the fetched content gets processed and discarded, maybe it's used for some internal side effect, maybe errors are swallowed silently. None of that means the vulnerability isn't real or isn't dangerous; it just means you can't *confirm* it by reading the HTTP response the normal way.

Out-of-band detection sidesteps that entirely by moving the proof to a completely separate channel. If a DNS lookup or HTTP request shows up on infrastructure we control, at the exact domain we just injected, that's unambiguous evidence the server-side fetch happened — regardless of whether the application's visible behavior gave us any hint of it.

## Tools used

- Burp Suite **Professional** (Collaborator requires Pro — Community edition can't generate or poll Collaborator payloads)
- Burp Repeater

## Takeaways

- Blind vulnerabilities aren't unconfirmable — they just require shifting from "read the response" to "watch an independent, attacker-controlled channel" for proof.
- Burp Collaborator is the standard tool for this across multiple vulnerability classes — the same core technique (insert a Collaborator payload, poll for interactions) applies to blind SSRF just as it did to out-of-band SQL injection extraction earlier in this series.
- A completely normal-looking HTTP response tells you nothing definitive about what the server did or didn't do behind the scenes — asynchronous or fire-and-forget server-side actions can be happening regardless of what comes back to the browser.
- Checking for both DNS and HTTP interactions matters — a strict network configuration might allow DNS resolution but block the actual outbound HTTP connection (or vice versa), so seeing either one is still meaningful confirmation, and seeing both tells you more about exactly how far the SSRF reaches.
- Many real-world SSRF findings are blind by default — this detection technique isn't just an academic exercise, it's often the only practical way to confirm SSRF exists in production systems where responses never leak any details.

## Fixing it

- Apply the same defenses as any other SSRF vulnerability: validate and restrict outbound destinations via an allowlist, disable unnecessary URL-fetching functionality, and don't trust client-controlled headers (like `Referer`) as safe input for server-side requests.
- Treat the `Referer` header specifically with suspicion for any server-side processing — it's entirely client-supplied and trivially spoofable, the same lesson as the earlier Referer-based access control lab in this series, just applied here to a different consequence.
- Monitor and log outbound requests made by backend services, so that even blind SSRF attempts (ones that never surface in the application's visible behavior) leave a detectable trail internally.
- Segment network access so that even if a blind SSRF is exploited, the range of internal or external destinations the server can actually reach is tightly restricted — limiting blast radius regardless of whether the vulnerability is ever directly observed by the app's own response.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
