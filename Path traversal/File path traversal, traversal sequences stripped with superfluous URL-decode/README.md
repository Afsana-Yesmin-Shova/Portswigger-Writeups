# File Path Traversal — Traversal Sequences Stripped with a Superfluous URL-Decode

**Category:** Path Traversal
**Lab:** File path traversal, traversal sequences stripped with superfluous URL-decode
**Status:** Solved ✅

---

## What's going on here

The product image display feature takes a `filename` parameter, and the app does actually try to defend against path traversal — it strips out sequences like `../` before using the value. On its own, that would be a reasonable (if slightly naive) defense. The bug is in the *order* of operations: after stripping those sequences, the app then performs a URL-decode on what's left, before finally using the value to fetch a file. That extra decode step happens too late, and it's exactly what breaks the defense.

**Goal:** retrieve `/etc/passwd` by working around the traversal-sequence filter.

## Why the fix doesn't actually work

The intended flow is: strip dangerous traversal sequences first, *then* decode whatever's left, so that anything genuinely dangerous should already be gone before decoding happens. But if we URL-encode our traversal sequence *before* sending it, the string the filter actually inspects doesn't look like `../` at all — it looks like harmless-seeming encoded text. The filter finds nothing to strip, lets it through untouched, and only *then* does the app decode it — turning our disguised string back into a real `../` sequence at the point where it's far too late to matter.

Even better for us: if we **double-encode** the sequence, we survive an environment where the URL itself might already get decoded once automatically (by the browser, a proxy, or the web server) before the app's own filtering code ever sees it. A single URL-encoded `../` might already look like a real `../` by the time the filter runs. Double-encoding it means that even after one layer of decoding happens along the way, what the filter actually inspects is still a single-encoded, harmless-looking string — and the app's own explicit decode step is what peels off that final layer.

## Working through it

**1. Intercept a request for a product image**

Browse to any product page, and in Burp's Proxy, catch the request that loads the product image — it'll include a `filename` parameter pointing at the image file being requested.

**2. Send it to Repeater**

Standard move — we're going to be modifying this parameter and resending it a few times.

**3. Set the filename to a double-URL-encoded traversal payload**

```
..%252f..%252f..%252fetc/passwd
```

Breaking that down: `%25` is the URL-encoded form of the `%` character itself. So `%252f` decodes once to `%2f`, and only decodes a *second* time to an actual `/`. In other words, this string is `../../../etc/passwd`, but with every `/` in the traversal portion encoded twice over.

**4. Send it**

If everything's set up as expected, the traversal-sequence filter inspects this value, sees nothing resembling `../` (because at that point it's still doubly-encoded), and lets it through untouched. The app's own subsequent decode step then unwraps one layer of encoding — turning `%252f` into `%2f` — which, depending on how the request was transmitted and processed, may itself already look sufficiently different from a raw `../` to pass any remaining checks, or gets decoded a further time by whatever component reads the final filename value, ultimately resolving to real `../` sequences by the time the file is actually fetched.

**5. Check the response**

If the payload worked, the response body isn't an image at all — it's the raw contents of `/etc/passwd`, confirming successful traversal outside the intended image directory.

## The payload

```
filename=..%252f..%252f..%252fetc/passwd
```

## Why this works

This is really a timing/ordering bug dressed up as an encoding trick. Any filter that operates on a string *before* decoding is vulnerable to exactly this class of bypass: encode the dangerous part so it doesn't match what the filter is looking for, and let a decoding step that happens *after* the filter reconstruct the dangerous value. Double-encoding specifically defends against the possibility that some earlier stage in the request pipeline (a proxy, the web server itself, or the framework's own request parsing) already performs a first round of automatic decoding before your own application code even gets a look — a single layer of encoding might already be gone by the time your filter runs, but a second layer survives that automatic pass and only gets stripped by the application's own deliberate, later decode call.

The broader lesson: sanitize-then-decode is backwards. Any value should be fully decoded into its final, canonical form *first*, and only then checked for dangerous content — otherwise there's always some encoded representation of the dangerous input that slips past a filter looking at the wrong representation of the data.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- Filtering dangerous input before decoding it is a common but fundamentally broken pattern — the filter and the eventual consumer of the data need to be looking at the same, final representation of that data.
- Double URL-encoding is a standard technique for surviving an environment where one layer of automatic decoding might already happen somewhere in the request pipeline before your payload even reaches the vulnerable filter.
- `%25` (the encoded form of `%` itself) is the key building block for constructing a double-encoded payload — it's what lets `%2f` (an encoded `/`) itself be encoded a second time.
- Path traversal filters that only look for literal `../` are trivially bypassed by anything that changes the string's representation without changing its eventual meaning — encoding, alternate separators, and similar tricks are always worth trying.

## Fixing it

- Always fully decode user input into its canonical form *before* applying any security-relevant filtering — never filter first and decode afterward.
- Avoid building file paths from user input at all where possible; instead, use an indirect reference (like a database ID or a fixed allowlist of filenames) that maps to a real file path server-side, so no traversal sequence — encoded or not — can ever reach the filesystem layer.
- If direct filenames must be accepted, validate the fully-decoded result strictly (e.g., reject anything containing `/`, `\`, or `..` after decoding, and confirm the resolved absolute path still lives inside the intended directory) rather than trying to strip specific "bad" substrings.
- Apply the principle of least privilege to the process serving these files, limiting what it can access on disk even if a traversal bypass is later discovered.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
