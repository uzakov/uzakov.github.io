---
layout: post
title: Executing arbitrary JavaScript from third-party origin when processing HTTP Basic Auth in Firefox: the story behind Bugzilla 1944926 and no CVE.
---

*All opinions in this post are my own (and my colleague's), and do not represent any employer, organization, or institution with which we are or have been affiliated.*



Whenever I'm asked for a password on the web, I try to be deliberately boring. **This is a simplified explanation for the sake of the article**, but the habit is basically:

- I prefer using bookmarked URLs (or typed URLs I've used a hundred times).
- And if something feels even slightly off, I click the lock icon and inspect the certificate / site identity details to confirm I'm really talking to the domain I think I am.
- Use VMs

One day I visited a URL and got HTTP Basic Authentication prompt. But when I clicked the lock icon to double-check the site identity, I got a surprise: (The domain below are examples for illustration; it's not the actual URL I visited.)
![HTTP Auth URLs mismatch]({{ site.url }}/public/images/2025/image8.png "Title")
The **domain shown in the identity/certificate UI did not match the domain shown in the URL bar**, while Firefox was actively asking me for credentials. See how the URL in the URL bar is referencing Twillio while cert is showing a different URL


My first thought was: What's going on?
Did I get hacked?
![Hmmmmm]({{ site.url }}/public/images/2025/image1.gif "Hmmmm")

That question turned into a rabbit hole, and eventually into an attack vector we could actually demo: by starting an async XHR on an attacker page and immediately navigating the victim to a legit site's HTTP Basic Auth prompt, Firefox can still run the attacker's XHR callback while the URL bar shows the legit site, letting us trigger a download at that moment and then call window.stop() so that no matter what the user types (or even if they cancel), the auth flow is terminated and the victim is dropped back onto our page where we can socially engineer them to open the "trusted-looking" file and go from there. 

ELI5: You send the victim to a malicious site, then redirect them to a legitimate site's HTTP Basic Auth prompt. While the URL bar shows the legitimate site, you start a file download from the malicious site, and no matter what the victim types into the auth window, you force them back to your page, where you use social engineering (and possibly another bug) to convince them to open the file and continue the attack.

Philip: [https://www.linkedin.com/in/sysmus3p](https://www.linkedin.com/in/sysmus3p) and I filed with Mozilla: [Bug 1944926](https://bugzilla.mozilla.org/show_bug.cgi?id=1944926) - “Executing arbitrary JavaScript from a third-party origin when processing HTTP Basic Auth.” And yes: **no CVE** (which we do not agree with).


## What we tried
After an initial investigation to make sure I hadn't been hacked, the URL wasn't Punycode, and no software on the machine was interfering, I realized this was a bug and that something interesting could be done, so I reached out to Philip.

The bug is about the **gap between what Firefox is showing in the UI and which origin is actually running JavaScript** while an HTTP authentication prompt is displayed.

### First look
![HTTP Auth]({{ site.url }}/public/images/2025/image10.png "HTTP Auth")
The previous page is not unloaded, but HTTP Auth URL is displayed in the address bar. To be clear: This is NOT a Same Origin Policy(SOP) bypass.

We may be able to execute arbitrary JavaScript from previous origin (attacker controlled domain) while the URL in the address bar is displaying a different URL.
![HTTP Auth]({{ site.url }}/public/images/2025/image11.png "HTTP Auth")

Few ideas we had:

### Idea 1: JavaScript Keylogger - FAILED
What if we setup JS even handler to intercept key presses?

Can we log credentials that victim puts into HTTP Basic Auth this way?
![Idea 1]({{ site.url }}/public/images/2025/image12.png "Idea 1")

Firefox paused execution of JavaScript when redirecting to HTTP Auth. Events/setInterval are paused. This limitation seems reasonable and secure.
![Cat nope]({{ site.url }}/public/images/2025/image13.png "Cat nope")
### Idea 2: 

- You start on an **attacker-controlled** domain: `attacker.tld`.
- The page navigates to `thirdparty.tld`, which is protected by HTTP Basic Auth.
- While the Basic Auth dialog for `thirdparty.tld` is on screen and the URL bar already shows `thirdparty.tld`, Firefox will still:
  - fire certain **XMLHttpRequest (XHR) event handlers** from the old page, and  
  - allow that code to call things like `window.stop()` and trigger a file download **while the URL bar still displays** `thirdparty.tld` (the download is from `attacker.tld`).
- The attacker then forces the user back to `attacker.tld` after the user enters credentials or cancels.
- The attacker asks the user to open the downloaded file.

![Idea 2]({{ site.url }}/public/images/2025/image14.png "Idea 2")

Attack demo:

![Idea 2]({{ site.url }}/public/images/2025/image2.gif "Idea 2")


From a user's perspective, this can look like:

> "I just authenticated to 'trustedwebsite.com' and now it's asking me to open a file that downloaded while I was logging in."

Under the hood, though, that download is being initiated by JavaScript from 'attacker.local', not from the trusted site. The identity panel (the "site information" popup) still exposes that fact - but the URL bar and the timing are working against the user.

Mozilla's security team eventually decided this is **a bug, but not a security bug** in their classification system - so: *no CVE*.


## The core idea of the bug

Our setup used two hostnames pointing to the same machine:

- 'attacker.local'
- 'thirdparty.local'

And an app behind to simulate:

- an attacker controlled landing page, and  
- a "trusted" site protected by HTTP Basic Auth.

At a very high level, the exploit chain looks like this:

1. Start on the attacker page 
   The user lands on 'http://attacker.local:8080' and redirected to trusted.local

2. Fire an asynchronous XHR with a delay 
   The attacker page sends an asynchronous XHR to an endpoint like '/sleep', which simply waits a bit before responding. A handler is registered for the XHR's state change.

3. Immediately navigate to a Basic Auth-protected resource
   The script then sets 'document.location' to something under 'thirdparty.local' that requires HTTP Basic Auth.  
   - Firefox starts navigating.
   - The URL bar updates to 'thirdparty.local'.
   - A modal Basic Auth prompt appears asking for credentials for 'thirdparty.local'.

4. XHR handler still runs, even though the page is "frozen"  
   At this point, the user sees the auth dialog and the new URL. The old page is visually gone and input is blocked, but under the hood:
   - The XHR completes, and  
   - Its event handler is invoked on the original document.

   This behaviour - XHR event handlers firing while a modal dialog is open - is an old known quirk in Firefox.

5. In the handler, control the timing and outcome

   In that event handler, the attacker can:

   - Use a synchronous XHR as a crude "sleep" to make sure the Basic Auth dialog is fully visible.
   - Call 'window.stop()' to cancel the authentication flow once the user submits credentials.
   - Trigger a file download from 'attacker.local' at just the right moment.

From the browser's perspective, all of this code still runs in the attacker's origin, there is no SOP bypass or cross-origin script execution. But from the user's perspective, the sequence is:

1. Land on a page.
2. See the browser ask for credentials for `trustedwebsite.com` in the URL bar.
3. A download appears at the same time as the credential prompt.
4. Enter credentials.
5. The prompt closes, and they end up back on what looks like a follow-up page that mentions a document.


## What the user actually sees (and why it's confusing)

The behaviour we reported has two UI oddities:

1. **Address bar vs identity panel mismatch**  
   - The URL bar shows the third-party domain ('thirdparty.tld' / the "trusted" site).  
   - If the user clicks the site identity button (the lock / info icon), Firefox shows details tied to the attacker origin - because that's where the running document actually lives.

2. **Download timing that looks like it came from the trusted site**  
   Because the attacker can delay the download until *after* the user has arrived at the trusted origin and seen the auth prompt, mental model becomes:  
   > "I authenticated to X, X gave me a file."  

   In reality, it's:  
   > "I visited attacker.tld, attacker.tld downloaded a file while Firefox was busy showing an auth dialog for X."


## Why we thought this might deserve a CVE

When we filed the report, we framed it as:

"Executing JavaScript from an attacker origin while Firefox is visually representing a different origin, in a security-sensitive context (HTTP auth)."

From a phishing and malware delivery perspective, this is attractive because:

- **HTTP Auth is still widely used** in internal tools and admin interfaces, often in front of pages that handle sensitive operations.  
- There is a visual mismatch between context and URL displayed, while as an attacker you can still perform certain actions

To be clear: the browser isn't mislabeling the file's origin in the download UI, and we're not escaping any sandbox. The issue is about perception and timing: during the HTTP Basic Auth prompt, the URL bar can show the **legitimate site**, while the user can still be served a file/download initiated by JavaScript from the **attacker-controlled domain**. This is not code execution on the trusted domain'it's a UI/attribution mismatch at the worst possible moment.

Still, from our point of view as people who care about phishing and UX, this felt like an interesting edge case.

PS: We reported the same issue to Tor, and Tor is also vulnerable/affected by this bug.
PPS: LLM was used mainly to help check grammar.

Further links and reading:

[1] https://bugzilla.mozilla.org/show_bug.cgi?id=1944926 has demo files you can use to demo this "attack"/bug

[2] https://www.malwarebytes.com/blog/threat-intel/2023/10/clever-malvertising-attack-uses-punycode-to-look-like-legitimate-website - how punycode gets used to trick users

---
