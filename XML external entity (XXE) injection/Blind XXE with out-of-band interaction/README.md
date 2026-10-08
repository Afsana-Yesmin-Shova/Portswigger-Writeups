# Blind XXE with Out-of-Band Interaction

**Category:** XML External Entity (XXE) Injection — Blind
**Lab:** Blind XXE with out-of-band interaction
**Status:** Solved ✅

---

## What's going on here

Same stock checker, same XML parsing bug as the earlier two XXE labs — but this time, nothing from the resolved entity ever makes it back into the response. No reflected error message, no visible content difference, nothing at all distinguishing a successful injection from a failed one. Exactly like the blind SSRF lab earlier in this series, we need an entirely separate channel to prove the vulnerability exists, since the application's own response gives us nothing to go on.

**Goal:** confirm blind XXE by getting the server's XML parser to make an outbound DNS lookup and HTTP request to Burp Collaborator. Note: this lab specifically requires **Burp Suite Professional** (Collaborator isn't available in Community), and the lab's firewall only permits interaction with Collaborator's default public server — no self-hosted Collaborator instances.

## Working through it

**1. Trigger the stock checker and capture the request**

Visit a product page, click **Check stock**, and intercept the `POST` request in Burp. Send it to Repeater.

**2. Insert a DOCTYPE pointing at a Collaborator placeholder**

Between the XML declaration and the `<stockCheck>` element, insert:

```xml
<!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN"> ]>
```

**3. Insert a live Collaborator payload**

Right-click directly where `BURP-COLLABORATOR-SUBDOMAIN` sits in that DOCTYPE and choose **Insert Collaborator payload**. Burp swaps that placeholder for a unique, actively-monitored subdomain.

**4. Reference the entity in productId**

```xml
<productId>&xxe;</productId>
```

**5. Send it**

Fire off the request. As expected for a blind vulnerability, the response looks completely ordinary — no file content, no error leaking anything, nothing to indicate whether the entity actually resolved or not.

**6. Poll Collaborator for interactions**

Switch to the **Collaborator** tab and click **Poll now**. If nothing's there yet, wait a few seconds and poll again — the server-side XML parsing and subsequent HTTP fetch may take a moment to complete.

**7. Confirm the interaction**

You should see both a DNS interaction and an HTTP interaction logged, both initiated by the application as a direct result of resolving our entity. That's unambiguous proof: the XML parser really did attempt to resolve the `SYSTEM` identifier by reaching out to our Collaborator subdomain — confirming the vulnerability exists even though nothing in the actual HTTP response ever hinted at it. Lab solved.

## The payload

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "http://YOUR-COLLABORATOR-SUBDOMAIN.oastify.com"> ]>
<stockCheck>
    <productId>&xxe;</productId>
    <storeId>1</storeId>
</stockCheck>
```

## Why this works

This is really the exact same underlying bug as the file-read and SSRF XXE labs — external entities are being resolved by a parser that shouldn't be resolving them at all. The only thing different here is the application's behavior *after* resolution: instead of reflecting the resolved value anywhere observable, it's discarded or used internally in a way we never see. That distinction matters enormously for *detection*, but not at all for the underlying *risk* — a blind XXE still means the server will happily make outbound requests (or attempt local file reads) based on attacker-controlled XML, it's just that confirming it requires an independent, attacker-controlled listening point rather than reading the application's own response.

This is the same detection philosophy used for blind SQLi (time-based or boolean-oracle signals) and blind SSRF (also via Collaborator) elsewhere in this series — whenever an application's visible behavior gives you nothing to work with, the question becomes "is there any other way to observe an effect of this injection," and out-of-band network interactions are one of the most broadly applicable answers to that question across many different vulnerability classes.

## Tools used

- Burp Suite **Professional** (Collaborator required)
- Burp Repeater

## Takeaways

- A vulnerability being "blind" (no visible feedback in the response) doesn't make it any less real or less exploitable — it just changes how you go about proving and weaponizing it.
- Out-of-band detection via Burp Collaborator is a technique that generalizes across vulnerability classes — the same "insert Collaborator payload, poll for interactions" workflow confirms blind SQLi, blind SSRF, and blind XXE equally well, because the underlying question ("did the server make an outbound request based on my input") is the same each time.
- Both DNS and HTTP interactions showing up is meaningful — it confirms not just that a hostname lookup happened, but that a full outbound connection was actually attempted, which matters for assessing what else might be reachable (internal services, metadata endpoints, etc.) from the same injection point.
- Once blind XXE is confirmed this way, the same vulnerability can often be escalated further — out-of-band data exfiltration techniques (covered elsewhere in the Academy) let you actually extract file contents through a blind XXE, not just prove the injection point exists.

## Fixing it

- Disable external entity resolution in the XML parser configuration — this is the root fix, and it closes off every variant of XXE covered in this series simultaneously (visible file read, SSRF, and this blind detection scenario), regardless of whether the application happens to reflect results back to the user.
- Don't assume a lack of visible output means a feature is safe to leave unpatched — blind vulnerabilities carry the same underlying risk as their visible counterparts, just with a higher bar to detect and demonstrate.
- Monitor outbound network connections from application servers, since a blind XXE (or blind SSRF) attempt still generates real outbound traffic that network-level monitoring could potentially catch even without any visible symptom in the application itself.
- As with the other XXE labs, consider using a less expressive input format than full XML wherever the use case allows, removing the external entity attack surface entirely rather than relying solely on parser configuration to stay correctly locked down.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
