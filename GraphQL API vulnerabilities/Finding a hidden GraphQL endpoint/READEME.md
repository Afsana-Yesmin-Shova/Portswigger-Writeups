# Finding a Hidden GraphQL Endpoint

**Category:** GraphQL API Vulnerabilities
**Lab:** Finding a hidden GraphQL endpoint
**Status:** Solved ✅

---

## What's going on here

Unlike the previous lab, there's no obvious GraphQL traffic to just observe in Burp's history here — the endpoint powering user management isn't linked from anywhere the UI visibly uses. We have to actually go find it first, the same way you'd hunt for any undocumented API endpoint during recon. And once we do find it, the site's made a genuine effort to block introspection — meaning the usual "just ask the schema to describe itself" shortcut doesn't work cleanly either. This lab is really two separate challenges stacked on top of each other: discovery, then defeating an introspection filter.

**Goal:** locate the hidden GraphQL endpoint, bypass its introspection block to map out the schema, and use what you find to delete `carlos`.

## Finding the endpoint

**1. Probe common GraphQL endpoint paths**

GraphQL APIs tend to live at a small set of conventional paths — `/graphql`, `/api`, `/graphql/v1`, and similar. In Burp Repeater, send requests to a handful of these common suffixes and see what comes back.

**2. Notice the telling error on `/api`**

A `GET` request to `/api` returns something like `"Query not present"`. That's not a generic 404 or routing error — it's the kind of error a GraphQL server specifically produces when it receives a request with no `query` parameter at all. That's a strong hint we've found a real GraphQL endpoint, just one that happens to be expecting requests over `GET` rather than the more typical `POST`.

**3. Confirm it with a minimal universal query**

Since it's responding over `GET`, the query needs to be passed as a URL parameter rather than in a request body. Try the simplest possible GraphQL query — asking for the meta-field `__typename`, which every GraphQL type exposes automatically:

```
/api?query=query{__typename}
```

**4. Confirm the response**

You should get back:

```json
{
  "data": {
    "__typename": "query"
  }
}
```

That confirms it beyond any doubt — this is a working GraphQL endpoint, accepting queries via `GET` parameters.

## Getting past the introspection block

**5. Try a standard introspection query**

Right-click the request in Repeater and choose **GraphQL → Set introspection query** — this inserts Burp's full, standard introspection query (the one that asks the schema to describe every type, field, and argument it supports), URL-encoded as the `query` parameter. Send it.

**6. Notice introspection is explicitly blocked**

Rather than a full schema dump, you'll get a response indicating introspection is disallowed. So the developers here have deliberately added some kind of filter specifically watching for introspection attempts.

**7. Insert a newline to slip past the filter**

Modify the query by adding a newline character right after the `__schema` keyword, and resend:

```
query IntrospectionQuery {
  __schema
  {
    queryType { name }
    ...
```

(The key change: instead of `__schema {` with just a space, it becomes `__schema` followed by a literal newline, then `{`.)

**8. Confirm the bypass worked**

This time, the response contains the full introspection output — every type, field, and argument in the schema. The filter was evidently built around a regex looking specifically for the exact substring `"__schema{"` (or `__schema {`, collapsed without meaningfully accounting for other whitespace) — and a newline character between the two tokens is different enough, byte-for-byte, that the pattern no longer matches, even though the query is still 100% syntactically valid GraphQL and gets parsed and executed identically by the actual GraphQL engine.

## Exploiting what we found

**9. Save the discovered schema to the site map**

Right-click the successful introspection response and choose **GraphQL → Save GraphQL queries to site map**, then browse to **Target → Site map** and use the **GraphQL** tab to look through what's available.

**10. Find the getUser query and test it**

Locate a `getUser` query, send it to Repeater, and fire it against the hidden endpoint. With no `id` specified (or a default that doesn't correspond to a real user), you'll likely get back `"getUser": null`.

**11. Find carlos's user ID**

In the GraphQL tab, adjust the `id` variable and try different values until the response returns a user record matching `carlos` — in this lab, that turns out to be ID `3`.

**12. Find the delete mutation**

Back in the site map's GraphQL schema listing, look for a `deleteOrganizationUser` mutation — it takes a user ID as its input parameter, which is exactly what we need.

**13. Send the delete mutation**

Send that mutation to Repeater and submit it with `id: 3`:

```graphql
mutation {
  deleteOrganizationUser(input: {id: 3}) {
    user {
      id
    }
  }
}
```

(As a `GET` request against the discovered endpoint, this gets URL-encoded into the `query` parameter.)

**14. Confirm the result**

If the mutation succeeds, `carlos` has been deleted — solving the lab.

## Key techniques used

| Step | Technique |
|---|---|
| Finding the endpoint | Probing common paths, recognizing a GraphQL-flavored error message |
| Confirming it | Minimal `__typename` query over `GET` |
| Bypassing the introspection filter | Inserting a newline inside `__schema {` to dodge a literal-string/regex match |
| Exploiting it | Using discovered `getUser` and `deleteOrganizationUser` operations directly |

## Why this works

**The discovery half** comes down to GraphQL APIs being common enough now that a fairly small, well-known set of conventional paths covers a lot of real-world deployments — and their error messages, even generic-sounding ones like "Query not present," are often distinctive enough to fingerprint once you know what a GraphQL server's typical error language looks like.

**The introspection bypass half** is a really clean example of the gap between syntax and surface-level string matching. GraphQL's parser doesn't care about incidental whitespace — a newline, extra spaces, or different line breaks between tokens are all semantically identical to the parser, which only cares about the token stream, not the exact bytes between tokens. A security filter built as a simple string or regex match against the raw query text, though, absolutely does care about those bytes. Whoever implemented this introspection block clearly intended to catch any introspection attempt, but implemented the check against one specific textual representation of `__schema {` rather than against the actual semantic structure of the query — leaving every other valid way of writing the identical query completely unguarded.

## Tools used

- Burp Suite (Proxy + Repeater, with GraphQL-aware tooling)
- Browser

## Takeaways

- Unlinked, undocumented API endpoints are still discoverable through basic recon — trying common naming conventions and paying attention to distinctive error messages goes a long way.
- GraphQL servers can and do accept queries over `GET`, with the query passed as a URL parameter — worth testing both methods when probing an unfamiliar endpoint.
- Introspection defenses built as simple string/regex matching against raw query text are inherently fragile, because GraphQL's actual parser is whitespace-insensitive in ways a naive text filter usually isn't accounting for.
- Once introspection succeeds, the full schema (every query, mutation, type, and field) becomes available for exploration — turning a "find the vulnerability" problem into a much more straightforward "pick the right operation" problem.
- Security filters implemented as blacklists against specific text patterns are a recurring weak point across this whole lab series — SQLi WAFs, path traversal filters, and now GraphQL introspection blocks all share the same fundamental flaw.

## Fixing it

- Don't rely on hidden/unlinked endpoints as a security measure — obscurity isn't a substitute for actual access control, and endpoints are discoverable through straightforward enumeration.
- If introspection needs to be blocked, implement the check against the query's actual parsed structure (abstract syntax tree), not against raw text patterns — a proper GraphQL-aware filter should recognize a `__schema` query regardless of incidental whitespace or formatting differences.
- Better yet, disable introspection entirely at the server configuration level in production, rather than trying to selectively filter introspection-shaped queries while leaving the endpoint otherwise open.
- Apply real authentication and field-level authorization to sensitive queries and mutations (like `getUser` and `deleteOrganizationUser`) regardless of whether introspection is blocked — a determined attacker can often still guess or enumerate meaningful operation names even without a full schema dump.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
