# CSRF Where Token Validation Depends on Token Being Present

**Category:** Cross-Site Request Forgery (CSRF) — Bypassing Token Validation
**Lab:** CSRF where token validation depends on token being present
**Status:** Solved ✅

---

## What's going on here

This is probably the simplest possible flaw a CSRF token implementation can have, and it's a genuinely common one in the real world. The "Update email" form carries a `csrf` token, and the app really does validate it — tamper with the value, and the request gets rejected. But the validation logic apparently only checks "does the submitted token match the expected one?" It never asks the more basic question first: "was a token submitted at all?" If the parameter is simply missing from the request entirely, there's nothing to compare, and the check seems to just... not fire.

**Goal:** change a logged-in victim's email address via CSRF, by submitting a forged request that omits the `csrf` parameter entirely rather than trying to guess or forge its value.

You've got `wiener:peter` to test with, and the same reminder as the other email-change labs: use a different email for your real delivered exploit than whatever you test with on your own account.

## Working through it

**1. Submit the legitimate form and capture the request**

Log in with `wiener:peter` in Burp's browser, use the "Update email" form, and find the resulting request in **Proxy → HTTP history**. It should carry both `email` and `csrf` parameters.

**2. Confirm tampering with the token gets rejected**

Send it to Repeater, modify the `csrf` value to something else, and resend. You should get the request rejected — confirming validation is genuinely happening when a token is present.

**3. Try removing the token entirely**

Instead of changing the value, delete the `csrf` parameter from the request body altogether, and resend.

**4. Observe that it's accepted**

If the validation logic really is only checking "does the token match" and skipping that check whenever there's no token to compare against, the request goes through successfully even with no `csrf` parameter at all — a pretty clear sign the check is conditional on the parameter's presence, rather than genuinely required.

## Building the exploit

**5. Build the CSRF form, omitting the csrf field entirely**

Since the fix is simply "don't send the field," the exploit is a standard self-submitting form that just never includes a `csrf` input:

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="anything@web-security-academy.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

Grab your lab's actual URL via right-click → **Copy URL** on the captured request.

**6. Store and test on yourself**

Go to the exploit server, paste this into the **Body** section, click **Store**, then **View exploit** to confirm it successfully changes your own email.

**7. Swap in a fresh email for delivery**

Update the `email` value so it's different from whatever you just tested with.

**8. Store again and deliver**

Click **Store**, then **Deliver to victim**. The victim's browser auto-submits the form — no `csrf` parameter included anywhere — and since the server-side check apparently has nothing to compare against and simply lets the request through, their email changes without their knowledge, solving the lab.

## The payload

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="anything@web-security-academy.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

Identical in shape to the "no defenses" lab's exploit — the only meaningful difference is that this time there genuinely is a CSRF token the legitimate app uses, we're just never sending it at all.

## Why this works

This is a classic case of validation logic being written as "if a token is present, check that it's correct" rather than "a correct token must be present." Those two statements sound almost identical, but they have completely different security implications: the first one has an implicit escape hatch built right into its own structure — simply not including the field at all sidesteps the entire check, since there's nothing to validate against. The second statement, properly implemented, would reject a request with a missing token exactly as readily as it rejects one with a wrong token.

This is an easy mistake to make in code, especially if token validation is implemented as something like "if request has a csrf parameter, verify it matches the session's expected value" without an accompanying "and reject the request outright if that parameter is absent in the first place." It's also one of the most commonly seen real-world CSRF implementation flaws, precisely because it's such a subtle, easy omission.

## Tools used

- Burp Suite (Proxy + Repeater)
- The lab's exploit server
- Browser

## Takeaways

- A CSRF token check needs to explicitly require the token's presence, not just verify its correctness when present — these are two different checks, and skipping the first one while implementing only the second is a surprisingly common and serious mistake.
- Testing both "tamper with the token's value" and "remove the token parameter entirely" should be a standard part of CSRF testing methodology — they probe genuinely different failure modes in the validation logic.
- This vulnerability pattern requires zero special payload construction — the entire bypass is simply *not including* a field a naive attacker might assume needs to be forged or guessed.
- This is one of several common, named CSRF token implementation flaws (alongside method-dependent validation, tokens not tied to sessions, and others covered elsewhere in this lab series) — worth knowing the whole family of these mistakes when assessing a real CSRF token implementation.

## Fixing it

- Explicitly reject any request to a CSRF-protected endpoint that's missing the token parameter entirely — "no token" should fail exactly like "wrong token," never slip through as an implicit pass.
- Structure validation logic as "require a token AND verify it matches," not "if a token exists, verify it matches" — the former has no accidental escape hatch, the latter does.
- Use a dedicated, well-tested CSRF protection library or framework feature rather than hand-rolling token validation logic, since these implementation details are easy to get subtly wrong and frameworks generally handle them correctly by default.
- Pair token validation with `SameSite` cookie restrictions as defense in depth, so that even a flawed token check isn't the only thing standing between the app and a successful CSRF attack.

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
