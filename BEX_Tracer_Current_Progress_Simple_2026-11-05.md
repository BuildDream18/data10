# BEX Tracer — Simple Progress Report

**Project:** Detect which Chrome extensions are enabled  
**Total extensions:** 89  
**Enabled each round:** 12  
**Rounds:** 6  
**Current best score:** MCC `0.2329`  
**Current behavior:** High precision, low recall

---

# 1. What Is the Main Problem?

I need to detect which **12 extensions are enabled out of 89**.

My code runs only as normal JavaScript inside:

```text
baitpage.test
```

The main difficulty is that a normal webpage cannot directly ask Chrome:

```text
Which extensions are enabled?
```

The page cannot normally use:

```text
chrome.management.getAll()
```

It also cannot directly read:

```text
/run/exc/...
Chrome extension folders
Chrome process command line
other extensions' service workers
other extensions' private network traffic
```

So I must detect extensions indirectly.

I look for things like:

```text
DOM elements
CSS changes
window globals
postMessage replies
CustomEvents
Web Accessible Resources
Resource Timing entries
link rewriting
unique page behavior
```

---

# 2. How the Score Works

The challenge uses **MCC — Matthews Correlation Coefficient**.

Formula:

```text
MCC =
(TP × TN − FP × FN)
/
sqrt((TP+FP)(TP+FN)(TN+FP)(TN+FN))
```

Where:

```text
TP = correct true
TN = correct false
FP = false positive
FN = missed enabled extension
```

Each round:

```text
12 extensions = ON
77 extensions = OFF
```

Examples with **0 false positives**:

| Correctly detected | MCC |
|---:|---:|
| 1 | 0.270 |
| 2 | 0.384 |
| 3 | 0.473 |
| 4 | 0.550 |
| 5 | 0.618 |
| 6 | 0.681 |
| 12 | 1.000 |

The important point is:

> False positives hurt the score a lot.

So it is better to detect fewer extensions correctly than to guess many extensions incorrectly.

---

# 3. My Current Result

My best documented result is:

```text
MCC:       0.2329
Precision: about 1.0
Recall:    about 8%
```

This means:

```text
When I return true → I am usually correct.
But I miss many extensions that are actually enabled.
```

So my main problem is:

```text
LOW RECALL
```

not:

```text
TOO MANY FALSE POSITIVES
```

---

# 4. What Solutions Have I Tried Until Now?

Yes. I have already tried many different solutions.

---

## 4.1 DOM Detection

I searched for unique HTML elements added by extensions.

Examples:

```text
#seoquake-seobar-panel
#jsonFormatterParsed
#easy-auto-refresh-modal
[data-test-id="pinterest-save-button"]
#DDMExtension-testelementid
```

### Result

This is one of the best methods.

Why?

Because a unique extension-created DOM element can identify one specific extension.

### Problem

Many extensions do not add anything to `baitpage.test`.

Some only add UI when:

```text
user clicks extension
user starts download
user starts recording
user opens QR scanner
page is YouTube
page is Gmail
page is Zoom
```

So DOM detection has:

```text
high precision
low recall
```

---

# 5. CSS Detection

I tested CSS differences created by extensions.

A good example is **SelectorsHub**.

The extension can add special outlines to elements with attributes like:

```html
<span xpath="1"></span>
<span xpath="2"></span>
<span css="1"></span>
<span css="2"></span>
```

The detector checks exact:

```text
outline style
outline width
outline color
```

### Result

This is promising because it is extension-specific.

### Problem

The version I inspected was not always exactly the same as the pinned challenge version.

So it still needs:

```text
ON test
OFF test
mixed-extension test
exact-version test
```

---

# 6. Page-to-Extension Message Detection

This is one of the strongest methods I found.

---

## 6.1 Microsoft Single Sign On

The page can send a specific message.

The extension replies by creating a response element.

The detector checks:

```text
new response element
runtime-ID-like value
correct extension version
```

### Why this is good

This is much better than guessing.

It uses a real extension communication method.

### Remaining problem

It still needs full live validation:

```text
ON
OFF
OFF + other extensions ON
many repeated rounds
```

---

## 6.2 Scopus Document Download Manager

I found a communication event:

```text
DDMRequestEvent
```

with:

```text
TestCommunication
```

The extension creates a response like:

```text
#DDMExtension-testelementid
data-extension-loaded="true"
data-extension-enabled="true"
```

### Result

This is another strong candidate.

### Remaining problem

It needs full real Chrome validation.

The detector must also make sure that an old/stale element is not counted.

---

# 7. Trust Wallet Detection

I researched EIP-6963 provider discovery.

Events:

```text
eip6963:requestProvider
eip6963:announceProvider
```

Trust Wallet can identify itself using:

```text
com.trustwallet.app
```

### Result

Promising.

### Problem

The public version I inspected was newer than the pinned challenge version.

Also wallet injection may depend on:

```text
settings
startup timing
page context
wallet state
version
```

So this is not fully validated yet.

---

# 8. Web Accessible Resource Detection

I tried detecting extension resources with URLs like:

```text
chrome-extension://<extension-id>/<resource>
```

This seemed like a very good idea at first.

---

## What I Found

About:

```text
17 extensions have WAR resources
15 use dynamic URLs
2 may have static URLs
```

The two important remaining static candidates are:

```text
Adblock for Youtube
Fathom AI Note Taker
```

---

## Why Most WAR Detection Failed

Manifest V3 can use:

```text
use_dynamic_url: true
```

That means the resource URL changes.

So I cannot safely hardcode:

```text
chrome-extension://ID/path/file.js
```

for those extensions.

### Result

Generic WAR probing failed.

### Remaining solution

Finish exact static WAR paths for:

```text
Adblock for Youtube
Fathom AI Note Taker
```

Then test them carefully.

---

# 9. Resource Timing Detection

I tried:

```javascript
performance.getEntriesByType("resource")
```

The idea is to see if an extension injected a resource into the page.

### Result

This can work.

### Problem

It only works when the extension actually injects a resource into `baitpage.test`.

Many extensions do not.

So it is useful, but it cannot detect all 89.

---

# 10. Cookie Banner Detection

I researched:

```text
I Don't Care About Cookies
```

Old checks looked for:

```text
.idcac
#idcac
.idc0_350
```

### Problem

The extension may hide the cookie banner without adding those exact classes.

### Better solution I am testing

Check:

```text
#accept-cookies
```

and see whether it becomes hidden through computed CSS.

### Remaining problem

This can only be used if tests prove:

```text
extension ON  → hidden
extension OFF → not hidden
other extensions ON → not hidden
```

Without those controls, it may cause false positives.

---

# 11. Network / Server URL Research

I also researched which extensions can communicate with remote servers.

Current classification:

```text
89 total extensions

67 = network-capable
22 = no extension-level network capability
```

Examples of domains found:

```text
*.smartscreen.microsoft.com
mail.google.com
app.vidow.io
freedownloadmanager.org
downloadhelper.net
app.downloadhelper.net
v10.downloadhelper.net
uvpn.me
onenote.officeapps.live.com
autorefresh.io
ecosia.org
hotcleaner.com
camelcamelcamel.com
```

### Important result

This research is useful for understanding extension behavior.

But:

> Network capability does NOT mean I can detect the extension from the bait page.

---

## Why Network Detection Is Still a Problem

An extension may make requests from:

```text
service worker
background page
popup
proxy layer
declarativeNetRequest
webRequest
extension iframe
```

Normal page JavaScript cannot directly inspect those private requests.

So even if I know:

```text
Extension X connects to server Y
```

the page may still have no way to see that connection.

---

# 12. Timing Side-Channel

I tested timing differences.

The idea:

```text
installed extension request
vs
nonexistent extension request
```

may have different timing.

### Result

It failed badly.

Score dropped to about:

```text
MCC ~0.03
```

### Why?

It appeared to detect:

```text
installed
```

rather than:

```text
enabled in this round
```

That caused many false positives.

### Conclusion

Do not use timing detection.

---

# 13. `<object>` Resource Probe

I also tried loading extension URLs using `<object>`.

### Result

It created many false positives.

### Why?

Chrome can trigger:

```text
onload
```

even when the resource is blocked or unusable.

So:

```text
onload
```

does not prove the extension is enabled.

### Conclusion

Do not use this method.

---

# 14. Generic Ad-Block Detection

I tested the idea of detecting blocked ads.

### Problem

Many extensions can block the same ad.

For example:

```text
uBlock Origin Lite
Adblock Plus
AdGuard
AdBlocker Ultimate
Adblock for Youtube
Pie Adblock
```

If:

```text
adsbygoogle.js is blocked
```

I still do not know which one blocked it.

If I mark all six as true:

```text
1 may be correct
5 may be false positives
```

That is very bad for MCC.

### Conclusion

Generic ad blocking cannot identify the exact extension.

---

# 15. `chrome.management.getAll()`

This would be an excellent solution.

For example:

```javascript
chrome.management.getAll()
```

could list extensions.

### Why it fails

My detector is a normal webpage.

This API is available to privileged extension code, not normal page JavaScript.

### Conclusion

Impossible from the current page context.

---

# 16. Direct Filesystem Access

The enabled extension folders may exist in a runtime location such as:

```text
/run/exc/...
```

Chrome itself knows which extensions were loaded.

### Why page JS cannot use this

The bait page cannot normally read:

```text
file:///run/exc/...
```

Chrome blocks local filesystem access from normal HTTPS pages.

### Conclusion

The information exists outside the page, but ordinary page JavaScript cannot read it.

---

# 17. Localhost / Internal Server Access

I tested ideas like:

```text
127.0.0.1
172.x.x.x
host.docker.internal
```

### Problems

Possible barriers include:

```text
Docker/container separation
CORS
Private Network Access
no readable internal endpoint
different network namespace
```

### Result

No reliable extension-state API was found.

### Conclusion

This is not a useful current solution.

---

# 18. Chrome DevTools Protocol

I investigated using Chrome debugging.

### Problem

Chrome uses a random debugging port.

The real port is stored outside the page.

The page does not know it.

There are also security restrictions around connecting from the HTTPS page.

### Conclusion

Not a practical page-JS solution.

---

# 19. `chrome.runtime.sendMessage()`

A webpage cannot simply message any Chrome extension.

The target extension must allow that website with:

```text
externally_connectable
```

### Problem

The extensions do not generally allow:

```text
baitpage.test
```

### Conclusion

Not a universal solution.

---

# 20. Site-Specific Extensions

Many extensions only work on special websites.

Examples:

```text
YouTube
Gmail
Google Meet
Zoom
Microsoft Teams
Amazon
academic websites
vendor websites
```

But the challenge page is:

```text
baitpage.test
```

### Why this is a big problem

Creating a fake YouTube-looking div does not make the page:

```text
youtube.com
```

Chrome content-script match rules use the real URL/origin.

### Conclusion

Many site-specific extensions may be invisible on the bait page.

---

# 21. User-Action-Only Extensions

Some extensions do nothing until the user:

```text
clicks the toolbar
starts recording
starts a download
opens PiP
starts QR scan
opens a popup
configures refresh
saves/clips something
```

Examples include some:

```text
download tools
recording tools
QR tools
tab tools
browser utilities
```

### Problem

The challenge may never activate those workflows.

### Conclusion

Installed/enabled does not always create a detectable page signal.

---

# 22. Background-Only Extensions

Some extensions mostly work in:

```text
service worker
background script
browser API
popup
context menu
proxy
```

They may never modify the bait page.

### Conclusion

These are very difficult or impossible to detect with normal page JS.

---

# 23. Isolated World Problem

Chrome content scripts often run in an isolated JavaScript world.

For example, an extension may set:

```javascript
window.someExtensionVariable = true;
```

But that `window` may be the extension's isolated world.

The page sees a different JavaScript environment.

### Conclusion

A variable existing in the extension content script does not automatically mean page JS can read it.

---

# 24. Solutions That Look Most Promising

These are the strongest remaining directions.

---

## Solution 1 — Microsoft SSO Discovery

Status:

```text
Promising
```

Reason:

```text
explicit page ↔ extension communication
specific response structure
specific version
```

Need:

```text
ON/OFF/mixed validation
```

---

## Solution 2 — Scopus TestCommunication

Status:

```text
Promising
```

Reason:

```text
real extension event
extension-created response
specific DOM attributes
```

Need:

```text
fresh-response checking
ON/OFF/mixed validation
```

---

## Solution 3 — Static WAR for Adblock for Youtube

Status:

```text
Not finished
```

Need:

```text
extract exact WAR file
test real Chrome
check false positives
```

---

## Solution 4 — Static WAR for Fathom

Status:

```text
Not finished
```

Need:

```text
extract exact WAR file
test ON/OFF/mixed
```

---

## Solution 5 — IDCAC Cookie Banner Differential

Status:

```text
Needs validation
```

Idea:

```text
check whether #accept-cookies is hidden
```

Need:

```text
prove only IDCAC causes the exact change
```

---

## Solution 6 — SelectorsHub CSS Fingerprint

Status:

```text
Promising
```

Need:

```text
exact pinned version
ON/OFF/mixed tests
```

---

## Solution 7 — Trust Wallet EIP-6963

Status:

```text
Promising but version-sensitive
```

Need:

```text
exact-pin confirmation
fresh profile validation
```

---

## Solution 8 — Resource Timing

Status:

```text
Useful helper
```

It can identify extension-loaded resources when they appear.

But it cannot detect extensions that inject nothing.

---

# 25. Remaining Main Problems

The remaining problems are:

---

## Problem 1 — Low Recall

Current detector is precise but finds too few enabled extensions.

This is the biggest problem.

---

## Problem 2 — Many Extensions Leave No Page Signal

They may be:

```text
background-only
popup-only
toolbar-only
site-specific
user-action-dependent
```

---

## Problem 3 — Version Differences

Some research uses a newer public extension version than the pinned challenge version.

A good marker in the new version may not exist in the challenge version.

---

## Problem 4 — Dynamic WAR URLs

Most WAR-enabled extensions use dynamic URLs.

That prevents simple static probing.

---

## Problem 5 — Private Network Traffic

The extension may connect to a server, but normal page JS cannot inspect the private request.

---

## Problem 6 — Similar Extensions Produce the Same Effect

Examples:

```text
ad blockers
VPNs
download extensions
recording extensions
```

A generic effect does not identify a unique product.

---

## Problem 7 — Need More Real Validation

Some current candidates are source-grounded but not yet fully tested in the real scorer.

---

# 26. What I Still Need to Test

For every detector that may return `true`:

### Test 1

```text
Target ON
```

Expected:

```text
true
```

### Test 2

```text
Target OFF
```

Expected:

```text
false
```

### Test 3

```text
Target OFF
Other extensions ON
```

Expected:

```text
false
```

### Test 4

Repeat with many fresh Chrome sessions.

Only after these tests should a detector be considered safe.

---

# 27. What I Should Not Spend More Time On

These methods have already shown poor value:

```text
generic timing
generic ad blocking
<object> extension URL probes
guessing dynamic WAR URLs
trying to read isolated-world globals
generic vendor-domain detection
marking many extensions true from one shared signal
```

These approaches tend to cause:

```text
false positives
```

and therefore lower MCC.

---

# 28. Current Estimated Ceiling

Current measured score:

```text
MCC 0.2329
```

Current working estimate after the strongest remaining fixes:

```text
MCC about 0.35–0.45
```

This is only an estimate.

It is not proven yet.

The real score must be measured after:

```text
Adblock WAR fix
Fathom WAR fix
IDCAC improvement
exact-pin validation
ON/OFF/mixed testing
```

---

# 29. Current Progress Summary

| Work | Status |
|---|---|
| Understand 89-extension challenge | Done |
| Build 8 detector files | Done |
| Research MCC | Done |
| Source/manifests analysis | Mostly done |
| DOM fingerprint research | Done for many candidates |
| CSS fingerprint research | Done for some candidates |
| SSO handshake research | Done |
| Scopus handshake research | Done |
| Trust Wallet research | Done, needs exact-pin validation |
| WAR research | Mostly done |
| Resource Timing research | Done |
| Network/server capability research | Done |
| 67 network / 22 non-network classification | Done |
| Timing test | Failed / rejected |
| Generic ad-block test | Rejected |
| `<object>` probe | Failed / rejected |
| Localhost/API approach | Failed / rejected |
| CDP approach | Not usable from page JS |
| Filesystem approach | Not usable from page JS |
| Adblock static WAR | Remaining |
| Fathom static WAR | Remaining |
| IDCAC better detector | Remaining |
| Exact version verification | Remaining |
| ON/OFF testing | Remaining |
| Mixed-extension testing | Remaining |
| Improve recall | Main remaining problem |

---

# 30. Simple Final Conclusion

Until now, I have found several good solutions, but none can detect all 89 extensions.

The working solutions mainly use:

```text
DOM changes
CSS changes
page-message handshakes
CustomEvents
static Web Accessible Resources
Resource Timing
extension-specific page behavior
```

The reason the score is still low is not mainly because the code is broken.

The biggest reason is:

> Many extensions do not expose anything unique to normal JavaScript on `baitpage.test`.

My current strategy is therefore:

```text
1. Keep precision high.
2. Do not guess.
3. Finish the strongest remaining WAR rules.
4. Improve the IDCAC rule.
5. Validate every candidate with ON/OFF/mixed tests.
6. Verify exact pinned extension versions.
7. Continue searching for explicit page-to-extension communication.
```

The goal is to increase recall **without creating false positives**.

That is the most important path to improving MCC.
