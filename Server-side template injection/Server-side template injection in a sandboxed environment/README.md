# Server-Side Template Injection in a Sandboxed Environment

**Category:** Server-Side Template Injection (SSTI)
**Lab:** Server-side template injection in a sandboxed environment
**Status:** Solved ✅

---

## What's going on here

This lab uses Freemarker, and — unlike some SSTI labs where the template engine just has no protection at all — this one actually has a sandbox in place, meant to stop template authors from reaching dangerous Java APIs. The problem is that the sandbox is *poorly implemented*: it evidently blocks a known list of dangerous classes and methods, but Java's reflection API gives you plenty of legitimate, boring-looking method chains that eventually lead to the same place, and the sandbox never anticipated all of them.

**Goal:** break out of the sandbox and read `/home/carlos/my_password.txt`, then submit its contents to solve the lab.

You're given working credentials (`content-manager:C0nt3ntM4n4g3r`) with access to edit product description templates — that's our injection point and our testing ground, all in one.

## Working through it

**1. Log in and find the injection point**

Log in with the provided credentials and open one of the product description templates for editing. You'll notice a `product` object is available to reference directly in the template — this is a developer-supplied object, exactly the kind of thing SSTI exploitation guides point you toward exploring first, since it's a guaranteed, real Java object we can call methods on.

**2. Confirm basic code execution and explore what's callable**

Every Java object inherits a baseline set of methods from the `Object` class — `getClass()`, `toString()`, `hashCode()`, and so on — regardless of what type it actually is. Confirm you can invoke one of these:

```
${product.getClass()}
```

If that renders successfully (returning something like the class name of `product`), you've confirmed template injection works and that method calls on real Java objects are permitted through whatever sandboxing exists.

**3. Chase a known escalation path through the Java reflection API**

`getClass()` returns a `Class` object — and `Class` objects expose a whole further tree of reflection-related methods that, combined, can get you surprisingly far. This is where reading the actual Freemarker/Java documentation pays off (rather than guessing): a well-known escalation chain from `getClass()` leads through `getProtectionDomain()` → `getCodeSource()` → `getLocation()`, which returns the URL that the class in question was originally loaded from. From there, a `URI`/`URL` object gives you `resolve()`, which can be pointed at an arbitrary path, and eventually `openStream()` to actually read a file.

**4. Build the full exploitation chain**

Put together, the payload looks like this:

```
${product.getClass().getProtectionDomain().getCodeSource().getLocation().toURI().resolve('/home/carlos/my_password.txt').toURL().openStream().readAllBytes()?join(" ")}
```

Walking through each link in that chain:

- `product.getClass()` — gets the `Class` object representing `product`'s actual Java type.
- `.getProtectionDomain()` — every loaded class has an associated `ProtectionDomain`, which describes where and under what security context it was loaded.
- `.getCodeSource()` — from that domain, get the `CodeSource`, which tracks where the class's bytecode actually came from (a JAR file, a directory, etc.).
- `.getLocation()` — pulls out the actual `URL` of that source location.
- `.toURI()` — converts it to a `URI` object, which is what supports the next step.
- `.resolve('/home/carlos/my_password.txt')` — resolves a new path against that base URI. Since the path we're supplying is absolute (starts with `/`), this effectively replaces the entire path portion of the URI, giving us something equivalent to `file:///home/carlos/my_password.txt` — completely unrelated to wherever the class actually lives. This is the actual "escape" step: legitimate reflection is being repurposed to construct an arbitrary file URI.
- `.toURL()` — converts that resolved URI back into a `URL` object, now pointing at our target file.
- `.openStream()` — opens an actual `InputStream` for reading that file's contents.
- `.readAllBytes()` — reads the entire file into a byte array.
- `?join(" ")` — a Freemarker built-in function that joins array elements together using a separator string. Applied to a byte array, it renders each byte as its decimal value, space-separated.

**5. Enter the payload and save**

Paste that into the template and save it. Instead of a normal rendered page, the output will be a long string of space-separated decimal numbers — each one the ASCII/byte value of one character of the password file's contents.

**6. Convert the bytes back to text**

Take that list of decimal numbers and convert each one back into its corresponding ASCII character (any online converter, a quick script, or even Burp's own decoder tools handle this easily) to reconstruct the actual password string.

**7. Submit the solution**

Click **Submit solution** on the lab page and paste in the reconstructed password text to solve the lab.

## The payload

```
${product.getClass().getProtectionDomain().getCodeSource().getLocation().toURI().resolve('/home/carlos/my_password.txt').toURL().openStream().readAllBytes()?join(" ")}
```

## Why this works

A template engine sandbox typically works by blacklisting or restricting access to specific dangerous classes and methods — things like `Runtime.exec()` or direct filesystem APIs that are obviously catastrophic if reachable. The problem with this approach is that Java's standard library is enormous, and reflection in particular offers a huge number of *indirect* paths to sensitive functionality that don't go through any of the obviously-dangerous entry points a sandbox author might think to block.

`getProtectionDomain()` and `getCodeSource()` aren't security-sensitive-sounding method names — they exist for entirely legitimate purposes (introspecting where code was loaded from, useful for security policy enforcement, ironically enough). But chained together with `resolve()` and `openStream()`, they form a perfectly functional arbitrary-file-read primitive that a naive blacklist would never anticipate. This is the core challenge of sandboxing any sufficiently powerful language or runtime: you're trying to enumerate every dangerous *destination*, when what you actually need to control is every possible *path* to get there — and in a reflection-capable language, those paths are effectively unbounded.

## Tools used

- Browser (template editor, directly in the app)
- A decimal-to-ASCII converter (or a quick script) to decode the extracted bytes

## Takeaways

- Sandboxes built around blacklisting specific classes or methods are inherently fragile in reflection-capable languages — there's almost always another chain of "safe-looking" method calls that reaches the same dangerous capability.
- `getClass()` is a universal starting point available on any object in a Java-based template engine, and from there, standard reflection APIs (`getProtectionDomain`, `getCodeSource`, `getLocation`) can lead to genuinely powerful capabilities like arbitrary file reads.
- Reading the actual documentation for the objects and classes you have access to — rather than guessing — is often the fastest way to find one of these chains; SSTI exploitation is as much a research exercise as it is a payload-crafting one.
- Freemarker's built-ins (like `?join`) can be repurposed for exfiltrating binary data (like raw file bytes) in a text-renderable form, which is exactly what's needed here since template output has to come back as text.
- Whitelisting is a fundamentally stronger sandboxing approach than blacklisting for exactly this reason — a whitelist only exposes what's explicitly deemed safe, while a blacklist has to correctly anticipate literally everything dangerous.

## Fixing it

- Avoid exposing full Java objects (or objects from any powerful, reflection-capable language runtime) directly to template contexts wherever possible — expose only the specific, minimal data or methods templates actually need.
- If a sandbox is necessary, prefer an allowlist model — explicitly permitting only known-safe methods and classes — over a blacklist trying to enumerate every dangerous one, since the latter approach will always be playing catch-up against creative reflection chains.
- Keep template engines and their sandboxing implementations up to date; known sandbox escape techniques for popular engines like Freemarker are actively tracked and periodically patched.
- As a broader principle, never rely on a template engine's sandbox as the sole layer of defense against template injection — preventing the injection itself (proper input handling, not concatenating untrusted input into template source) remains the real fix.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
