# Accidental Exposure of Private GraphQL Fields

**Category:** GraphQL API Vulnerabilities
**Lab:** Accidental exposure of private GraphQL fields
**Status:** Solved ✅

---

## What's going on here

GraphQL APIs expose a single endpoint that accepts flexible, client-specified queries — you ask for exactly the fields you want, and the server returns exactly that shape of data back. That flexibility is the whole appeal of GraphQL, but it's also exactly where this lab's bug lives: a query meant for one legitimate purpose (checking login credentials) happens to expose a field — the user's `password` — that was never meant to be readable by a normal client at all. The access control failure isn't in *who* can call the query; it's in *what the query is willing to return* once you know to ask for it.

**Goal:** discover a `getUser` query that leaks both username and password, use it against the administrator's account specifically, log in as them, and delete `carlos` from the admin panel.

## Working through it

**1. Explore the login flow**

In Burp's browser, go to **My account** and attempt a login (any credentials, even wrong ones — we just need to see the request shape).

**2. Find the GraphQL request**

In **Proxy → HTTP history**, you'll see the login attempt wasn't a normal form POST — it's a GraphQL **mutation**, carrying the username and password as structured query variables rather than plain form fields. Send that request to Repeater.

**3. Run an introspection query**

GraphQL APIs often expose a built-in introspection capability — essentially the API documenting its own schema, listing every available query, mutation, and field, if introspection hasn't been explicitly disabled. In Repeater, right-click inside the request body and choose **GraphQL → Set introspection query**, which swaps in Burp's pre-built introspection query for you. Send it.

**4. Save the discovered schema to the site map**

Right-click the response and choose **GraphQL → Save GraphQL queries to site map**. Burp parses the introspection results and populates **Target → Site map** with every query and mutation the schema exposes — effectively handing you a menu of everything this API can theoretically do.

**5. Spot the interesting query**

Browsing through the site map, you should find a `getUser` query that returns both a user's `username` and, more importantly, their `password` — a field that obviously shouldn't be exposed through any normal, authenticated-user-facing API call. You'll also notice this query takes a numeric `id` parameter and fetches whichever user that ID corresponds to — a classic insecure direct object reference shape, just sitting inside GraphQL's query syntax instead of a REST URL path.

**6. Send the query to Repeater and test it**

Right-click `getUser` in the site map and send it to Repeater. Hit send with its default `id` value (likely `0`) — you'll probably get an empty or null result back, since `0` doesn't correspond to a real user.

**7. Enumerate IDs until you find the administrator**

Switch to the GraphQL tab in Repeater, and try different values for the `id` variable — `1`, `2`, and so on. At some point, one of those IDs should return a result containing the word "administrator" as the username, along with their actual password in plaintext, right there in the response. In this lab, that happens to be `id: 1`.

**8. Log in as the administrator**

Take the extracted credentials and log in through the normal login form.

**9. Delete carlos**

Head to the **Admin** panel and delete the `carlos` account — solving the lab.

## The queries involved

The introspection query (Burp inserts this automatically via **GraphQL → Set introspection query**) is what reveals the schema in the first place. The actual exploit query is something along these lines:

```graphql
query {
  getUser(id: 1) {
    username
    password
  }
}
```

Just swap the `id` value as needed to enumerate different accounts.

## Why this works

This is really an access control failure disguised as an API design oversight. The `getUser` query's *existence* isn't necessarily the bug — plenty of apps need a way to look up user records internally. The bug is that this query returns a `password` field at all, to any caller who knows to ask for it, with no check on whether the requester is authorized to see that specific user's credentials (or any credentials at all). GraphQL's flexible, client-driven field selection means a schema design mistake like "this type happens to include a sensitive field" becomes directly, immediately exploitable the moment someone discovers the schema — which introspection makes trivially easy if it's left enabled.

This is also a solid demonstration of why introspection is worth disabling (or at least restricting) in production GraphQL deployments: it essentially hands an attacker the full map of everything the API is capable of returning, removing almost all of the guesswork that would otherwise be needed to even know a field like `password` exists to ask for.

## Tools used

- Burp Suite (Proxy + Repeater, with GraphQL-aware tooling)
- Browser

## Takeaways

- GraphQL's flexibility — letting clients request exactly the fields they want — means any sensitive field accidentally exposed on a type is reachable the moment a caller knows (or discovers) its name, regardless of whatever UI restrictions exist elsewhere in the app.
- Introspection queries are an enormously powerful recon tool against GraphQL APIs — if enabled, they hand you the entire schema, including fields and queries that were never meant to be used by typical clients.
- Burp's built-in GraphQL support (`Set introspection query`, `Save GraphQL queries to site map`) makes schema discovery and querying significantly faster than building these requests by hand.
- A query that works by numeric ID lookup (`getUser(id: N)`) is a GraphQL-flavored IDOR — the same enumeration techniques used against REST APIs apply directly here.
- Sensitive fields (passwords, tokens, internal flags) should never be returned by a general-purpose query reachable by normal authenticated users, regardless of whether the UI happens to never display them — the API itself is the actual boundary that matters.

## Fixing it

- Never include sensitive fields like plaintext passwords, password hashes, or tokens in any GraphQL type that's reachable by a general user-facing query — if credential verification needs to happen server-side, do it entirely within trusted backend logic, never by returning the credential to the client for comparison.
- Apply field-level and query-level access control within the GraphQL resolver layer itself, verifying the requesting user is authorized to view the specific data being asked for — not just that they're authenticated at all.
- Disable introspection in production environments, or restrict it to internal/development use only, since it provides a complete, authoritative map of the API's capabilities to anyone who queries it.
- Audit the full GraphQL schema specifically for fields that shouldn't be broadly queryable, the same way you'd audit a REST API's endpoints and response bodies for over-exposed data.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
