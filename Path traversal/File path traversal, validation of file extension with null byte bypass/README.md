# File Path Traversal — Validation of File Extension with Null Byte Bypass

**Category:** Path Traversal
**Lab:** File path traversal, validation of file extension with null byte bypass
**Status:** Solved ✅

---

## What's going on here

This lab layers a different defense on top of the same basic image-display vulnerability: the app checks that the supplied filename *ends with* the expected extension (`.png`, presumably, given the image context). That's a common and reasonable-sounding check — if you can only ever request something ending in `.png`, you shouldn't be able to walk off into `/etc/passwd`. The catch is a classic, old-school bug: how the validation layer and the underlying filesystem-reading layer disagree about where a string actually ends.

**Goal:** retrieve `/etc/passwd`, despite the app requiring the requested filename to end in the expected image extension.

## The null byte trick

This is one of the oldest tricks in the path traversal playbook, and it exploits a fundamental difference between how high-level string validation works versus how many lower-level file APIs (especially older C-based ones, which a lot of language runtimes ultimately call into) interpret strings.

A **null byte** — represented as `%00` when URL-encoded, or `\0` in raw form — is the traditional string terminator in C. Plenty of underlying file-handling code, even when called from a "safer" higher-level language, still treats a null byte as "the string effectively ends here," regardless of whatever characters follow it in the actual byte sequence.

So if the validation check operates on the *full* string (correctly seeing `.png` at the very end, after the null byte and everything past it), but the code that actually *opens the file* stops reading at the null byte, the two layers end up looking at effectively different strings — one padded out with a fake extension to satisfy validation, the other truncated right where we actually want the real path to end.

## Working through it

**1. Intercept a product image request**

Browse to a product page and catch the image request in Burp's Proxy — you'll see the `filename` parameter.

**2. Send it to Repeater**

Move it over so we can adjust the parameter freely.

**3. Build a path with a null byte before the fake extension**

Set `filename` to:

```
../../../etc/passwd%00.png
```

This is doing two things at once: the `../../../` walks back up to the filesystem root (same traversal technique as the earlier labs), landing us at `etc/passwd`. Then `%00.png` tacks a null byte followed by a plausible extension onto the end.

**4. Send it and check the response**

If the null byte trick works as expected here, the file-extension validation sees the whole string — including the trailing `.png` — and is satisfied. But when the file is actually opened on disk, the underlying read stops at the null byte, meaning the real file accessed is just `../../../etc/passwd`, with everything from `%00` onward silently discarded. The response should contain the raw contents of `/etc/passwd`.

## The payload

```
filename=../../../etc/passwd%00.png
```

## Why this works

Two different layers of the application are looking at the same string but interpreting it differently. The extension-validation logic almost certainly does something like checking `filename.endsWith('.png')` against the complete string it received — and since our string genuinely does end in `.png` as far as basic string comparison goes, that check passes cleanly. But whatever eventually opens the file on disk is very likely calling into lower-level file-handling code (directly or through several layers of abstraction) that treats a null byte as a hard string terminator — a holdover from C's string-handling conventions that still shows up in a surprising number of modern runtimes and libraries under the hood.

The result: validation happens against one interpretation of the string (the full, `.png`-suffixed version), and the actual file access happens against a different interpretation (the truncated, null-terminated version) — and the attacker gets to pick which parts land on which side of that gap. This is functionally the same category of bug as the earlier "filter before decode" lab in this series: a security check and the code it's meant to protect aren't looking at a shared, canonical representation of the data.

Note: this specific bypass is largely a *historical* technique at this point — most modern language runtimes (Java, .NET, Python, etc.) explicitly reject or safely handle null bytes in strings used for file paths precisely because this vulnerability was so widespread for so long. It's included in the Academy specifically because it's foundational to understanding the broader class of "validation layer vs. consumption layer" mismatches, even in environments where the exact null-byte trick no longer works.

## Tools used

- Burp Suite (Proxy + Repeater)
- Browser

## Takeaways

- A null byte (`%00`) can cause certain underlying file APIs to stop reading a string early, even when a higher-level validation check correctly examined the full string including everything after the null byte.
- Extension-based validation (`endsWith('.png')`) only proves something about the *string*, not about what file actually gets opened once that string is handed to lower-level, potentially null-terminated-string-aware code.
- This is largely a legacy vulnerability class today — most modern runtimes reject null bytes in path-related strings by default — but it's a foundational example of the broader "two layers disagree about the data" bug pattern that shows up repeatedly across different encodings and mechanisms.
- Combining a traversal sequence with a bypass trick (traversal to escape the directory, null byte to defeat extension checking) is a common pattern — individual defenses often only address one dimension of the attack.

## Fixing it

- Use a modern language/runtime with up-to-date libraries, since most current platforms explicitly reject null bytes in file path strings, closing off this specific bypass entirely.
- Never trust extension-based validation as a meaningful security boundary on its own — it says nothing about the rest of the path, and (as this lab shows) can itself be defeated depending on how the underlying string is later consumed.
- Canonicalize and fully resolve any user-supplied path before performing any validation, and confirm the resolved path both stays within the intended directory *and* carries the expected extension — checking these properties on the final, resolved result rather than the raw input string.
- As always, prefer indirect references (IDs mapped server-side to real paths) over accepting raw filenames or paths from user input at all.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
