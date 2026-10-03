# Bypassing GraphQL Brute Force Protections

**Category:** GraphQL API Vulnerabilities
**Lab:** Bypassing GraphQL brute force protections
**Status:** Solved ✅

---

## What's going on here

The login system here runs through a GraphQL API, and it has a genuinely working rate limiter — send too many login attempts from the same origin in a short window, and you'll start getting rate-limit errors back. On the surface that should kill any straightforward brute-force attempt. The flaw is architectural: the rate limiter is apparently counting **HTTP requests**, not individual login attempts — and GraphQL has a built-in feature, aliases, that lets a single HTTP request carry dozens (or hundreds) of separate operations at once. If the limiter only sees "one request," it never notices that request secretly contained a hundred password guesses.

**Goal:** brute-force `carlos`'s password using a candidate list, all crammed into as few HTTP requests as possible to stay under the rate limit.

## What GraphQL aliases actually are

Normally, if you wanted to call the same GraphQL operation multiple times with different arguments, you'd have to send multiple separate requests, since a single query/mutation document can only reference a given field once under its default name. Aliases solve a completely legitimate problem: they let you call the *same* operation multiple times within one request, each instance tagged with its own custom name, so the response can distinguish between them. Something like:

```graphql
mutation {
  attempt1: login(input: {username: "carlos", password: "123456"}) {
    success
  }
  attempt2: login(input: {username: "carlos", password: "password"}) {
    success
  }
}
```

Both of these are genuinely independent calls to the `login` mutation, with different arguments, bundled into a single request — and the response will come back with both `attempt1` and `attempt2` as separate keys, each with its own `success` field. That's the feature we're about to repurpose.

## Working through it

**1. Trigger a login attempt and find the request**

In Burp's browser, go to **My account** and try logging in with bad credentials. Check **Proxy → HTTP history** — you'll see the login goes through as a GraphQL mutation. Send it to Repeater.

**2. Confirm the rate limiter kicks in**

Resend the login mutation a handful of times with different wrong passwords, one request at a time. After a short while, you should start getting rate-limit errors back — confirming the protection is real and genuinely blocks naive, one-attempt-per-request brute-forcing.

**3. Build a single mega-request using aliases**

Instead of sending one password guess per HTTP request, build a single `mutation { }` block containing dozens of aliased `login` calls — one per password in a candidate list, each targeting `carlos`, each with a different password, and each requesting the `success` field so we can tell which one worked.

Since typing this out by hand for a long password list would be painfral, a quick script makes more sense. Something like this, run in the browser console (right-click the page → **Inspect** → **Console** tab), builds the whole alias list and copies it straight to your clipboard:

```javascript
copy(`123456,password,12345678,qwerty,...(full password list)...`
  .split(',')
  .map((password, index) => `
bruteforce${index}:login(input:{password: "${password}", username: "carlos"}) {
    token
    success
}`).join('\n'));
console.log("The query has been copied to your clipboard.");
```

(Use the full PortSwigger authentication lab password list as your candidate set — it's the standard wordlist used across their authentication labs.)

**4. Paste the generated aliases into your mutation**

In Repeater's GraphQL tab (or the Pretty tab for the raw request), wrap the clipboard contents in a single `mutation { ... }` block, so the final request looks like:

```graphql
mutation {
  bruteforce0: login(input: {password: "123456", username: "carlos"}) {
    token
    success
  }
  bruteforce1: login(input: {password: "password", username: "carlos"}) {
    token
    success
  }
  ...
  bruteforce99: login(input: {password: "12345678", username: "carlos"}) {
    token
    success
  }
}
```

**5. Clean up the request before sending**

If you built this by editing the request you originally sent to Repeater (rather than from scratch), make sure to delete any leftover GraphQL `variables` dictionary and `operationName` field, since those won't line up with our hand-built batch of aliases anymore.

**6. Send it**

Fire off this single, large request. Since it's just *one* HTTP request as far as the rate limiter is concerned — regardless of how many `login` calls it actually contains internally — it should sail through without triggering any rate-limit error at all.

**7. Find the successful attempt**

The response comes back with one result block per alias, each carrying its own `success` field. Use the response search bar (in Repeater) to search for `true` — this immediately jumps you to whichever aliased attempt actually succeeded.

**8. Read off the winning password**

Trace that successful alias back to the corresponding password in your request — that's `carlos`'s real password.

**9. Log in**

Use the discovered credentials to log in through the normal login form, solving the lab.

## The core trick

```graphql
mutation {
  bruteforce0: login(input: {password: "GUESS_1", username: "carlos"}) { token success }
  bruteforce1: login(input: {password: "GUESS_2", username: "carlos"}) { token success }
  ...
}
```

One HTTP request, dozens (or hundreds) of independent login attempts, each distinguishable in the response via its alias.

## Why this works

Rate limiters typically operate at the HTTP request layer — counting how many requests arrive from a given IP, session, or other identifier within some time window. That's a completely sensible approach for REST-style APIs, where "one request" and "one logical operation" are basically the same thing. GraphQL breaks that assumption: a single HTTP request's body can legally contain any number of independent operations, each with its own arguments and its own result in the response, because that's precisely what the specification's alias feature is designed to support.

So the rate limiter here isn't technically failing to count requests correctly — it's counting the right thing (HTTP requests), it's just that "HTTP requests" and "login attempts" have quietly stopped being equivalent once GraphQL and its aliasing feature are involved. The defense was built with a REST-API mental model in mind, and GraphQL's query language gives us a completely legitimate, specification-compliant way to make that mental model wrong.

## Tools used

- Burp Suite (Proxy + Repeater, with GraphQL tooling)
- Browser DevTools console (to generate the large aliased query)

## Takeaways

- GraphQL aliases let a single request carry many independent operations — a feature built for legitimate batching use cases, but just as usable for cramming an entire brute-force attempt into one HTTP call.
- Rate limiting implemented purely at the HTTP-request level doesn't translate cleanly to GraphQL, where "one request" can mean "a hundred logical operations" without anything unusual appearing at the transport layer.
- When a payload needs to be large and repetitive (like a hundred near-identical aliased mutations), scripting the construction — even something quick in the browser console — beats hand-typing it and avoids transcription errors.
- Searching response content directly (for `"success":true` or similar) is much faster than manually scanning a huge batched response for the one result that matters.
- This technique generalizes beyond login brute-forcing — any rate-limited GraphQL mutation or query is potentially vulnerable to the same aliasing trick, wherever the limiter is counting requests rather than operations.

## Fixing it

- Implement rate limiting that counts individual GraphQL *operations* within a request, not just the number of HTTP requests received — a single request containing 100 aliased mutations should count as 100 attempts against the limiter, not one.
- Consider limiting or disabling the use of aliases for sensitive operations like login, or capping the maximum number of aliased operations permitted in a single query/mutation document.
- Apply query complexity/cost analysis (a common GraphQL security practice) that accounts for the total number of resolver calls a request will trigger, rejecting requests that exceed a reasonable threshold regardless of how they're structured.
- Layer additional brute-force protections beyond simple rate limiting — account lockouts after repeated failures, CAPTCHA challenges, and anomaly detection all provide defense in depth that doesn't depend solely on counting HTTP requests correctly.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
