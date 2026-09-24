# Multistep Clickjacking

**Category:** Clickjacking
**Lab:** Multistep clickjacking
**Status:** Solved ✅

---

## What's going on here

This lab is a step up from basic clickjacking, and it's a good demonstration of why a confirmation dialog isn't actually a defense against this attack class — it's just a second thing to clickjack. The "Delete account" flow here is genuinely well protected against CSRF (there's a real token in play), and there's a "Are you sure?" confirmation step layered on top specifically to guard against accidental or forced clicks. Neither of those stops us, because clickjacking doesn't need to forge a request at all — it just needs the *real, logged-in user* to click the real buttons themselves, in the right sequence, without realizing it.

**Goal:** build a two-stage clickjacking page that tricks the victim into clicking "Delete account" and then confirming "Yes" on the resulting dialog — using two separate decoy elements layered over one invisible iframe of the real account page.

## Why a single decoy isn't enough here

The earlier clickjacking lab only needed one aligned click, because one click was the whole attack. Here, deleting the account is a two-step flow: click "Delete account," then click "Yes" on a confirmation prompt that appears afterward. Since both steps happen inside the same iframe (the confirmation renders on the same page, just after the first click), we need **two** decoy elements, each aligned to wherever the relevant button sits at that particular moment — one aligned to "Delete account" as it initially appears, and a second aligned to wherever "Yes" appears once the confirmation shows up.

## Working through it

**1. Log in and get familiar with the target page**

Log in with the provided credentials (`wiener:peter`) and go to the account page, so you know roughly where "Delete account" sits and what the confirmation dialog looks like once triggered — this makes eyeballing your pixel offsets much easier later.

**2. Start from the exploit server template**

Go to your lab's exploit server and paste in:

```html
<style>
	iframe {
		position:relative;
		width:$width_value;
		height:$height_value;
		opacity:$opacity;
		z-index:2;
	}
   .firstClick, .secondClick {
		position:absolute;
		top:$top_value1;
		left:$side_value1;
		z-index:1;
	}
   .secondClick {
		top:$top_value2;
		left:$side_value2;
	}
</style>
<div class="firstClick">Test me first</div>
<div class="secondClick">Test me next</div>
<iframe src="YOUR-LAB-ID.web-security-academy.net/my-account"></iframe>
```

Same core idea as basic clickjacking — a low-`z-index` decoy underneath, a semi-transparent iframe of the real site on top — just with two separate decoy `<div>`s instead of one, each independently positioned.

**3. Point it at the account page**

Swap `YOUR-LAB-ID` in for your lab instance, so the iframe loads the real, logged-in `/my-account` page.

**4. Set a starting size**

Use rough values to start — 500px width, 700px height work well as a baseline, adjustable later if needed.

**5. Position the first decoy over "Delete account"**

Set `$top_value1` and `$side_value1` so `.firstClick` lines up with wherever the "Delete account" button sits in the iframe — something around 330px from the top and 50px from the left is a reasonable starting point, though exact values depend on your layout.

**6. Position the second decoy over the "Yes" confirmation button**

Set `$top_value2` and `$side_value2` for `.secondClick` so it lines up with the "Yes" button that appears *after* the first click — roughly 285px from the top and 225px from the left as a starting point. Since this button only appears once the confirmation dialog shows, you're essentially pre-positioning a decoy over where you expect it to land once the page state changes.

**7. Keep the iframe semi-visible while you calibrate**

Set `$opacity` to `0.1` for now, so you can actually see the real page underneath while lining things up. `0.0001` is the value you'll switch to for the final, delivered version — visually imperceptible but still fully clickable.

**8. Store and preview**

Click **Store**, then **View exploit**.

**9. Check alignment on the first click**

Hover over "Test me first." The cursor should change to a hand if it's correctly sitting on top of the real "Delete account" button. If not, tweak `.firstClick`'s `top`/`left` values until it does.

**10. Click through to the confirmation step, then check the second alignment**

Click **Test me first** — this should trigger the real delete flow's confirmation dialog, rendering inside the iframe. Now hover over "Test me next" and confirm the cursor changes to a hand there too, meaning it's correctly aligned with the real "Yes" button that just appeared. Adjust `.secondClick`'s position if it's off.

**11. Finalize and deliver**

Once both decoys are confirmed to align correctly, rename "Test me first" → "Click me first" and "Test me next" → "Click me next," drop the opacity down to `0.0001`, and click **Store** again. Then click **Deliver exploit to victim**.

The victim sees two innocuous-looking prompts, clicks them in the order presented — and each click actually lands on a real button in the invisible iframe underneath: first "Delete account," then "Yes" on the resulting confirmation. Neither the CSRF token nor the confirmation dialog does anything to stop this, because every request involved is completely genuine, submitted by the real logged-in user, clicking real buttons — they just don't know that's what they're doing.

## Why this works

CSRF tokens defend against *forged* requests — ones an attacker tries to construct and submit without the legitimate user's direct interaction. That defense is intact here; we're never forging anything. Clickjacking sidesteps it entirely by getting the real user to perform the real action themselves, with full session and token validity, just without their conscious awareness of what they're clicking.

The confirmation dialog is meant to catch *accidental* clicks or lightweight CSRF attempts — the assumption being that even if an attacker tricks someone into one click, a second explicit confirmation adds a layer of intentionality that's hard to forge. But that assumption only holds if the confirmation step is something the attacker can't also predict and align a decoy against. Since the confirmation dialog renders in a predictable location, at a predictable point in the flow, it's just as clickjackable as the original button — it just needs its own decoy, positioned and timed correctly. A defense against one click doesn't generalize to a defense against a sequence of clicks if every click in that sequence is independently clickjackable.

## Tools used

- Browser (Chrome, required for the victim simulation)
- The lab's exploit server

## Takeaways

- CSRF tokens and confirmation dialogs solve different problems than clickjacking does — clickjacking doesn't forge anything, it just misdirects genuine user interaction, so defenses built around detecting forgery don't apply.
- A multi-step confirmation flow is not automatically clickjacking-resistant — if every step happens inside the same frame, every step can get its own aligned decoy.
- Building multistep clickjacking exploits is really just basic clickjacking, repeated per step, with careful attention to how the page's DOM state changes between clicks (since the second decoy has to be positioned for a button that doesn't even exist until after the first click fires).
- The only real fix for clickjacking is preventing framing altogether — no amount of in-page confirmation logic addresses the root cause, since the attacker never needs to interact with the confirmation logic directly, only align a decoy over it.

## Fixing it

- Set `X-Frame-Options: DENY` (or `SAMEORIGIN` where legitimate framing is required) on the account pages — this is the actual fix, and it would have prevented both the single-click and multistep versions of this attack equally.
- Use a Content Security Policy with `frame-ancestors 'none'` as the modern equivalent, ideally alongside `X-Frame-Options` for broader browser compatibility.
- Don't treat confirmation dialogs as a clickjacking mitigation — they're a reasonable UX safeguard against accidental actions, but they add essentially no protection against a determined clickjacking attack, since the whole point of clickjacking is aligning decoys over exactly this kind of secondary interaction.
- For especially sensitive actions (account deletion being a good example), consider requiring something clickjacking can't easily replicate — re-entering a password, a CAPTCHA, or another explicit proof of intent that isn't just "click here again."

---

*Solved as part of the PortSwigger Web Security Academy — for learning/authorized testing only. Don't run this against anything you don't have permission to test.*
