# SSRF with Blacklist-Based Input Filter

**Category:** Server-Side Request Forgery (SSRF)
**Lab:** SSRF with blacklist-based input filter
**Status:** Solved ✅

---

## What's going on here

Same stock checker setup as the previous SSRF lab, but this time the defense is a more direct attempt at filtering: the app is apparently blocking requests that literally contain `127.0.0.1` (the standard loopback address), and separately filtering out the word "admin" from the path. Two blacklist-style checks, stacked on top of each other — and blacklists, as a general rule, only ever catch the specific things their author thought to list. This lab is a clean demonstration of two completely different techniques for sliding past that kind of check.

**Goal:** reach `http://localhost/admin` and delete `carlos`, bypassing both the IP-based block and the string-based "admin" block.

## Working through it

**1. Trigger the stock checker and capture the request**

Visit a product, click **Check stock**, catch the request in Burp, and send it to Repeater.

**2. Try the obvious loopback address — and get blocked**

Set the `stockApi` parameter to:

```
http://127.0.0.1/
```

Send it. You should get blocked — confirming the app is specifically watching for `127.0.0.1` as a string or recognized pattern and refusing to proceed when it sees it.

**3. Bypass the IP block with an alternate loopback representation**

Rather than fighting the filter head-on, just stop using the representation it's looking for. Try:

```
http://127.1/
```

This is a lesser-known but completely valid way of writing the same loopback address — `127.1` is shorthand that gets interpreted by the underlying networking stack as `127.0.0.1`, since IPv4 addresses can be expressed with fewer than four dot-separated octets, with the missing portions implicitly zero-filled. The filter, almost certainly doing a literal string match (or at best a narrow pattern match) against `127.0.0.1`, has no idea this alternate notation means the exact same thing — but the server's actual network layer resolves it identically. Send this, and it should go through cleanly, reaching the internal stock API as if we'd typed the full loopback address.

**4. Add the admin path and hit the second block**

Now try:

```
http://127.1/admin
```

This time you get blocked again — but for a different reason. This confirms there's a *second*, separate filter specifically watching for the literal string "admin" in the path.

**5. Bypass the string filter with double URL-encoding**

Obfuscate just the letter "a" in "admin" by double-URL-encoding it:

```
http://127.1/%2561dmin
```

Here's what's happening: `%61` is the single-URL-encoded form of the letter `a`. Encoding that *again* turns the `%` into `%25` (its own encoded form), giving us `%2561` — a doubly-wrapped representation of the same character. A filter doing a straightforward substring check against the literal word "admin" in the raw request won't recognize `%2561dmin` as containing that word at all, since at a glance it's a completely different string. But by the time the request actually reaches the backend and gets decoded — potentially through two separate decoding passes along the way, same idea as the double-encoding trick from the path traversal labs earlier in this series — `%2561` unwraps first to `%61`, and then to the literal character `a`, reconstructing the full word "admin" exactly where the application actually processes the path.

**6. Send it and confirm access**

With this payload, the request should slip past both filters and actually reach the internal admin interface, returning its content in the response.

**7. Extend the payload to delete carlos**

Having confirmed the admin interface is reachable, extend the path to the delete action:

```
http://127.1/%2561dmin/delete?username=carlos
```

Submit this as the `stockApi` value, and the stock checker follows through, deleting `carlos` and solving the lab.

## The payloads

| Purpose | Payload |
|---|---|
| Confirm `127.0.0.1` is blocked | `http://127.0.0.1/` |
| Bypass the IP blacklist | `http://127.1/` |
| Confirm "admin" string is blocked | `http://127.1/admin` |
| Bypass the string blacklist (double-encoded "a") | `http://127.1/%2561dmin` |
| Final exploit | `http://127.1/%2561dmin/delete?username=carlos` |

## Why this works

Both bypasses here share the same underlying weakness: a blacklist can only ever recognize the *exact representations* its author anticipated, and both IP addresses and URL paths have more than one valid way of being written.

For the IP filter: `127.0.0.1`, `127.1`, `0x7f.1`, `017700000001` (octal), and even a plain decimal integer representation are all technically valid ways of expressing the same loopback address, depending on how permissively the underlying networking library parses them — most do accept several of these. A filter checking for one specific textual form misses every other equally-valid form that resolves to the identical destination.

For the string filter: double URL-encoding exploits the fact that request processing often involves more than one decoding pass between where a filter inspects the data and where the application finally consumes it. A filter looking for the literal substring "admin" in a *partially*-decoded request body simply won't find it if that substring is still wrapped in an extra layer of encoding at the moment the filter runs — the decoding that would reveal "admin" hasn't happened yet from the filter's point of view, even though it will happen later, once it's too late to matter.

Both techniques boil down to the same lesson that's shown up repeatedly across this whole lab series: filtering on a specific textual pattern is fundamentally weaker than filtering on the actual, final, resolved meaning of the data.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- IP addresses have multiple valid alternate notations (shortened dotted-decimal, octal, hex, plain integer) that resolve identically at the networking layer but look completely different as text — a blacklist checking for one specific string representation misses all the others.
- Double URL-encoding remains a reliably effective filter-bypass technique any time a value passes through more than one decoding stage between the filter and the actual consuming logic — this is the same trick seen in the path traversal and WAF-bypass labs earlier in this series, just applied here to an SSRF blacklist.
- Blacklists targeting SSRF destinations are particularly weak because "the loopback address" has so many different valid spellings — this is exactly why allowlist-based approaches are so strongly preferred for this vulnerability class specifically.
- When one blacklist bypass gets you partway and you hit a second block, it's worth testing whether it's a genuinely separate check (as it was here) rather than assuming the first bypass should have handled everything.

## Fixing it

- Never rely on blacklisting specific string representations of dangerous destinations (IP addresses, hostnames, paths) — the number of valid alternate encodings and notations is large enough that a blacklist will always be incomplete.
- Use an allowlist approach instead: explicitly define which exact hosts/IPs the stock checker (or any SSRF-capable feature) is permitted to contact, and reject everything else by default, rather than trying to enumerate everything dangerous.
- Fully decode and canonicalize both the IP/hostname and the path components of a URL *before* applying any validation — checking the final, resolved form rather than the raw, possibly-still-encoded request data.
- Consider resolving the hostname to its actual IP address server-side and validating that resolved IP against an allowlist, rather than pattern-matching against the textual hostname/IP as supplied in the request — this closes off alternate-notation tricks entirely, since all the different spellings resolve to the same underlying address.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
