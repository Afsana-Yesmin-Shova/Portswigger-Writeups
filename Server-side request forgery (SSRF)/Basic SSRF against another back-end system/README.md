# Basic SSRF Against Another Back-End System

**Category:** Server-Side Request Forgery (SSRF)
**Lab:** Basic SSRF against another back-end system
**Status:** Solved ✅

---

## What's going on here

This is the foundational SSRF lab — no filters, no redirects, no blind detection needed. The stock checker fetches data from wherever we tell it to, with zero restrictions, which means it's a fully functional server-side proxy we can point at any internal address we like. The twist here isn't really a defense to bypass at all — it's that we don't actually know the exact internal IP of the admin interface in advance. We know it's *somewhere* on the `192.168.0.X` subnet, listening on port `8080`, and the real task is scanning for it efficiently rather than exploiting some clever bypass.

**Goal:** use the unrestricted stock checker to sweep the internal `192.168.0.X` range for an admin interface on port `8080`, then use that interface to delete `carlos`.

## Working through it

**1. Trigger the stock checker and capture the request**

Visit a product page, click **Check stock**, and catch the resulting request in Burp. This time, instead of Repeater, send it straight to **Intruder** — since we're about to brute-force an entire address range, Intruder's purpose-built for exactly this.

**2. Set the base target address**

In Intruder, set the `stockApi` parameter to:

```
http://192.168.0.1:8080/admin
```

**3. Mark the last octet as the payload position**

Highlight just the final `1` in the IP address — the last octet — and click **Add §** to wrap it in Intruder's payload markers, turning it into:

```
http://192.168.0.§1§:8080/admin
```

That's the one piece of the address we're going to vary across the scan.

**4. Configure the payload set**

In the **Payloads** panel, set the payload type to **Numbers**, then enter:

- **From:** `1`
- **To:** `255`
- **Step:** `1`

This tells Intruder to generate every value from 1 through 255 and substitute each one into that final octet position in turn — effectively sweeping the entire `192.168.0.1` through `192.168.0.255` range, one request per address.

**5. Launch the attack**

Click **Start attack**. Intruder fires off 255 requests, one for each candidate IP in the range, each one asking the stock checker to fetch `/admin` from that address on port `8080`.

**6. Sort by status code**

Once the attack finishes, click the **Status** column header to sort results by HTTP status code. Most of the sweep will return something indicating no response or a connection failure (since most of those addresses don't have anything listening on port 8080 at all) — but you should see exactly one request return a clean **200** status, standing out from the rest.

**7. Inspect the successful hit**

That single `200` response is the real admin interface — meaning we've just identified the exact internal IP address hosting it, purely by sweeping the subnet rather than knowing it in advance.

**8. Send it to Repeater and finish the job**

Right-click that successful request and send it to Repeater. Change the path portion of the `stockApi` value from `/admin` to:

```
/admin/delete?username=carlos
```

Keep the correct IP address you just discovered in step 7. Send it — this deletes `carlos` through the now-located admin interface, solving the lab.

## The payload pattern

Intruder sweep:

```
stockApi=http://192.168.0.§1§:8080/admin
```

(payload type: Numbers, range 1–255, step 1)

Final exploit, once the live IP is known:

```
stockApi=http://192.168.0.<discovered-IP>:8080/admin/delete?username=carlos
```

## Why this works

This lab is really about demonstrating SSRF's most direct, unmitigated impact: when a server-side feature will fetch literally any URL it's given, with no validation whatsoever, it becomes a general-purpose proxy into whatever network that server happens to sit on. Internal services are frequently deployed with the assumption that they're "safe" simply because they're not directly reachable from the public internet — no authentication, weaker access controls, sometimes nothing standing between a request and a sensitive action at all. SSRF defeats that assumption entirely by turning the vulnerable public-facing server into an unwitting proxy, since requests made *from* that server to internal addresses are treated as trusted, internal traffic by whatever they reach.

The IP sweep itself isn't exploiting a bug in the stock checker beyond the core SSRF — it's just basic network reconnaissance, automated through Intruder because manually trying 255 individual addresses by hand would be painfully slow. Any SSRF-capable feature with no destination restrictions can be used this way to map out an otherwise-invisible internal network, entirely from outside it.

## Tools used

- Burp Suite (Proxy + Intruder + Repeater)
- Browser

## Takeaways

- An SSRF vulnerability with zero destination validation is effectively a general-purpose internal network scanner and proxy, not just a way to reach one specific known internal URL.
- Intruder is the right tool whenever an attack needs to sweep across a range of values (IP octets, ports, usernames, etc.) — manually trying each one individually doesn't scale.
- Internal services often have weak or no authentication specifically because they're assumed to be unreachable from outside — SSRF breaks that assumption by routing attacker-controlled requests through a trusted internal vantage point.
- Sorting Intruder results by status code (or response length, or any other distinguishing signal) is a fast way to spot the one meaningful result buried in a large batch of otherwise-uniform failures.
- Once you've located a live internal service via SSRF, treat it exactly as you would any other discovered endpoint — enumerate its functionality, since the admin interface itself may have further exploitable actions beyond the first one you find.

## Fixing it

- Apply a strict allowlist to any server-side URL-fetching functionality, specifying exactly which hosts/IPs are permitted and rejecting everything else by default — this single fix would have prevented the entire scan from working in the first place.
- Never assume internal network position is a sufficient security boundary on its own — internal services should still require genuine authentication and authorization, since SSRF (among other techniques) can grant an external attacker an internal vantage point.
- Segment networks so that public-facing application servers don't have broad, unrestricted access to sensitive internal services — even a successful SSRF should ideally only reach a narrow, intentional set of destinations.
- Disable or tightly scope any feature that fetches arbitrary, user-influenced URLs server-side; if the functionality is genuinely needed, restrict it as narrowly as the use case allows rather than leaving it fully open.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
