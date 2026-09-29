# File Path Traversal — Validation of Start of Path

**Category:** Path Traversal
**Lab:** File path traversal, validation of start of path
**Status:** Solved ✅

---

## What's going on here

Unlike the previous lab in this series, this one doesn't send a bare filename that the server prepends a directory to — instead, the app transmits the **entire file path** as a request parameter, and validates it by checking that the path *starts with* the expected images folder. On paper that sounds reasonable: if you can't make the path begin anywhere except the intended directory, you should be stuck inside it. In practice, "starts with the right folder" and "stays inside the right folder" are two completely different guarantees, and the gap between them is the whole vulnerability.

**Goal:** retrieve `/etc/passwd` despite the app enforcing that the supplied path must begin with the expected images directory.

## Why "starts with the right folder" isn't enough

A check like `path.startsWith('/var/www/images/')` only inspects the beginning of the string — it says nothing about what happens *after* that prefix. Since traversal sequences (`../`) are just ordinary characters as far as basic string validation is concerned, nothing stops us from starting the path exactly where the check expects, and then immediately walking right back out of that directory using enough `../` sequences to climb up to the filesystem root, before descending into wherever we actually want to go.

The validation is checking the *literal text* the path begins with, not the *actual location on disk* the path resolves to once the operating system processes all those `../` segments.

## Working through it

**1. Intercept a product image request**

Browse to a product page, and in Burp's Proxy, catch the request that fetches the image — you'll see the `filename` parameter carrying the full path this time, something like `/var/www/images/58.jpg`.

**2. Send it to Repeater**

Move it over so we can iterate on the path value.

**3. Build a path that starts correctly but ends up elsewhere**

Set `filename` to:

```
/var/www/images/../../../etc/passwd
```

This genuinely does start with `/var/www/images/`, satisfying whatever `startsWith`-style check the app is running. But once the filesystem actually resolves this path — stepping back up three directory levels via `../../../` and then down into `etc/passwd` — it ends up pointing well outside the images folder entirely, ultimately reaching `/etc/passwd`.

**4. Send it and check the response**

If the validation really is only checking the start of the string, the request goes through, and the response contains the raw contents of `/etc/passwd` instead of an image.

## The payload

```
filename=/var/www/images/../../../etc/passwd
```

## Why this works

Prefix-based validation (`startsWith`) is a check on the *string*, not on the *resolved path*. Filesystems process `../` sequences as "go up one directory," entirely independent of whatever the path started with — so a string that begins exactly where a validator expects can still resolve to a completely different location once all the traversal segments are worked through. This is really the same underlying mistake as blacklisting specific substrings: the check is looking at surface-level text patterns instead of the actual, final effect of that text once it's interpreted by the system that matters.

The number of `../` segments needed depends on how deep the starting directory is — three levels up was enough here to escape `/var/www/images/` and reach the filesystem root, from which `etc/passwd` is directly reachable. In a real assessment you'd typically try a handful of different `../` counts if the exact directory depth isn't obvious up front.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- Validating that a path *starts with* an expected directory says nothing about where that path actually *resolves to* once traversal sequences in the rest of the string are processed.
- `../` sequences work regardless of where they appear in a path, as long as they come after a starting point the filesystem can actually walk backward from — a prefix check has no way to "see" past its own matched prefix.
- This is a good reminder that any string-matching-based security check (blacklists, prefix checks, suffix checks) is fundamentally different from validating the *semantic meaning* of the data after it's fully interpreted.
- If the exact number of `../` segments needed isn't obvious, it's usually fine to just try a range of values (2, 3, 4 levels up) until one lands correctly.

## Fixing it

- Never validate a path by checking its literal starting string — instead, resolve the path to its canonical, absolute form first (resolving all `../` and symlinks), and *then* check that the resolved result genuinely lives inside the intended directory.
- Avoid accepting full file paths from user input at all where possible — use an indexed reference (a database ID, a fixed allowlist of valid filenames) that maps server-side to a real, trusted file path, removing the traversal attack surface entirely.
- If a path must be accepted directly, strip and reject any `..` segments after canonicalizing the path, not before — canonicalization first, validation second, exactly the reverse of what this lab's app did.
- Apply least-privilege file permissions to the process serving these files, so even a successful traversal has limited blast radius.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
