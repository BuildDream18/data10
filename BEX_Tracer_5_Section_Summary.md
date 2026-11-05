# BEX Tracer — Current Progress Summary

## 1. The BEX Challenge

The BEX challenge asks me to detect which Chrome extensions are enabled in the current browser round.

There are:

```text
89 total extensions
12 enabled extensions per round
6 rounds per submission
8 JavaScript detector files
```

My detector files run as normal JavaScript inside:

```text
baitpage.test
```

The challenge is difficult because normal webpage JavaScript cannot directly ask Chrome which extensions are enabled.

The page cannot normally use:

```javascript
chrome.management.getAll()
```

It also cannot directly read Chrome extension folders, Chrome process information, other extensions' background/service-worker state, or other extensions' private network traffic.

So I have to detect extensions indirectly using signals such as:

```text
DOM changes
CSS changes
window variables
CustomEvents
postMessage replies
Web Accessible Resources
Resource Timing
special page behavior
```

### MCC scoring

The challenge uses **Matthews Correlation Coefficient (MCC)**.

Important values:

```text
TP = correct extension detected
TN = correct extension not detected
FP = wrong extension detected
FN = enabled extension missed
```

Each round has:

```text
12 ON
77 OFF
```

With no false positives:

| Correct detections | MCC |
|---:|---:|
| 1 | 0.270 |
| 2 | 0.384 |
| 3 | 0.473 |
| 4 | 0.550 |
| 5 | 0.618 |
| 6 | 0.681 |
| 12 | 1.000 |

The most important point is:

> False positives are very expensive.

So I should not guess many extensions just to increase recall.

---

## 2. What I Have Done Until Now

I have already completed a large amount of research and testing.

### Full 89-extension analysis

I reviewed the full 89-extension set and organized them into the required 8 groups:

```text
ad_blockers
network_vpn
developer_tools
media_video
capture_recording
save_research
tabs_workflow
identity_security
```

### Built the 8 detector files

I prepared:

```text
ad_blockers.js
network_vpn.js
developer_tools.js
media_video.js
capture_recording.js
save_research.js
tabs_workflow.js
identity_security.js
```

### Researched extension source and manifests

For many extensions, I checked:

```text
manifest.json
content scripts
background/service worker behavior
host permissions
Web Accessible Resources
externally_connectable
DOM injection
CSS injection
page messaging
CustomEvents
window globals
```

### Tested DOM-based detection

I researched unique extension-created elements such as:

```text
#seoquake-seobar-panel
#jsonFormatterParsed
#easy-auto-refresh-modal
[data-test-id="pinterest-save-button"]
#DDMExtension-testelementid
```

This method is precise when the element is truly extension-specific.

### Tested CSS-based detection

For SelectorsHub, I researched special CSS outlines on elements using attributes such as:

```text
xpath="1"
xpath="2"
css="1"
css="2"
```

This is a promising extension-specific fingerprint.

### Tested page-message detection

I found strong communication-based candidates.

**Microsoft Single Sign On**
- The page can send a specific message.
- The extension creates a response node.
- This is one of the strongest candidates because it uses a real extension communication protocol.

**Scopus Document Download Manager**
- I found `DDMRequestEvent` with `TestCommunication`.
- The extension can return:
  - `data-extension-loaded="true"`
  - `data-extension-enabled="true"`

### Researched Trust Wallet detection

I investigated EIP-6963:

```text
eip6963:requestProvider
eip6963:announceProvider
```

Trust Wallet can identify itself with:

```text
com.trustwallet.app
```

### Researched Web Accessible Resources

Current research found approximately:

```text
17 WAR-enabled extensions
15 using dynamic URLs
2 potentially static
```

The two important remaining static candidates are:

```text
Adblock for Youtube
Fathom AI Note Taker
```

### Tested Resource Timing

I researched:

```javascript
performance.getEntriesByType("resource")
```

to find resources actually injected by extensions.

### Researched cookie-banner behavior

For `I Don't Care About Cookies`, I researched both extension-specific classes and whether:

```text
#accept-cookies
```

becomes hidden.

### Researched network/server capability

I also created a network capability classification:

```text
89 total extensions
67 network-capable
22 non-network in the manifest-based classification
```

Examples of domains found include:

```text
*.smartscreen.microsoft.com
mail.google.com
app.vidow.io
freedownloadmanager.org
downloadhelper.net
uvpn.me
onenote.officeapps.live.com
autorefresh.io
ecosia.org
hotcleaner.com
camelcamelcamel.com
```

### Tested and rejected bad approaches

I also tested or analyzed:

```text
generic WAR probing
<object> extension probes
timing side-channels
generic ad-block detection
localhost/internal API probing
Chrome DevTools Protocol
filesystem access
chrome.management
generic chrome.runtime.sendMessage
```

These approaches either failed or produced too many false positives.

---

## 3. Why I Got a Low Score

My best documented score so far is:

```text
MCC:       0.2329
Precision: about 1.0
Recall:    about 8%
```

This means:

> When my detector says an extension is ON, it is usually correct.

But:

> I miss many extensions that are actually ON.

So the main problem is **low recall**.

### Main reasons

1. **Many extensions do not touch the bait page**
   - background-only
   - popup-only
   - toolbar-only
   - service-worker-only

2. **Many extensions only work on special websites**
   - YouTube
   - Gmail
   - Google Meet
   - Zoom
   - Amazon
   - academic/vendor sites

   But the challenge page is `baitpage.test`.

3. **Many extensions require user action**
   - toolbar click
   - start recording
   - start download
   - open QR scanner
   - activate PiP
   - configure refresh
   - open popup
   - save/clip action

4. **Isolated-world JavaScript cannot always be seen**
   - extension globals may not be visible to the normal page.

5. **Most WAR URLs are dynamic**
   - `use_dynamic_url: true`
   - this prevents simple hardcoded probing.

6. **Generic signals cannot identify exact extensions**
   - one blocked ad can be caused by many different ad blockers.

7. **Network traffic is often private**
   - service worker
   - popup
   - background context
   - proxy
   - `webRequest`
   - `declarativeNetRequest`

8. **Some source analysis has version differences**
   - a marker in a newer version may not exist in the pinned challenge version.

9. **Some aggressive experiments created false positives**
   - timing side-channel
   - `<object>` extension probes
   - generic ad-block detection

Because MCC strongly penalizes false positives, those methods lowered the score.

---

## 4. Available Solutions

The best remaining solutions are the ones that identify one specific extension with very low false-positive risk.

### Solution 1 — Finish Adblock for Youtube static WAR detection

Need to:

```text
extract exact static WAR path
test extension ON
test extension OFF
test target OFF + other extensions ON
repeat several times
```

### Solution 2 — Finish Fathom static WAR detection

Need to:

```text
find exact static resource
verify it is not dynamic
test ON/OFF/mixed
```

### Solution 3 — Improve I Don't Care About Cookies detection

Inspect:

```text
#accept-cookies
```

and determine whether the extension hides it.

This must be validated carefully to avoid false positives.

### Solution 4 — Use Microsoft SSO page-message discovery

This is one of the strongest candidates because it uses a real extension response.

Need:

```text
ON validation
OFF validation
mixed validation
multiple fresh Chrome runs
```

### Solution 5 — Use Scopus TestCommunication

Use:

```text
DDMRequestEvent
TestCommunication
```

and only count a fresh extension-created response.

### Solution 6 — Continue SelectorsHub CSS fingerprinting

Use exact CSS differences rather than generic DOM changes.

Need:

```text
exact pinned version confirmation
ON/OFF/mixed validation
```

### Solution 7 — Continue Trust Wallet EIP-6963 detection

Use provider discovery metadata only.

Need exact pinned-version validation.

### Solution 8 — Continue Resource Timing harvesting

Use:

```javascript
performance.getEntriesByType("resource")
```

when a unique extension-loaded resource appears.

### Solution 9 — Search remaining extension source for explicit bridges

High-value patterns:

```text
window.postMessage
CustomEvent
document.dispatchEvent
addEventListener
MAIN-world injection
externally_connectable
page-created response nodes
unique CSS
unique DOM
static WAR resources
```

### Solution 10 — Build strict ON/OFF/mixed tests

Every positive rule should pass:

```text
Target ON                  → signal present
Target OFF                 → signal absent
Target OFF + others ON     → signal absent
Repeated fresh Chrome runs → same result
```

### Current expected improvement

Current measured score:

```text
0.2329
```

Current working estimate after the strongest remaining fixes:

```text
about 0.35–0.45 MCC
```

This is still an estimate and must be confirmed with the real scorer.

---

## 5. Summary

The BEX challenge asks me to identify:

```text
12 enabled extensions
out of 89
```

using normal JavaScript inside:

```text
baitpage.test
```

I have already:

```text
analyzed all 89 extensions
built all 8 detector files
researched manifests and extension source
tested DOM fingerprints
tested CSS fingerprints
found page-message handshakes
researched WAR resources
tested Resource Timing
researched Trust Wallet discovery
researched network/server capability
classified 67 network-capable extensions
tested several failed approaches
```

My current best documented score is:

```text
MCC 0.2329
```

The score is low mainly because:

```text
many extensions expose no unique signal on baitpage.test
many are site-specific
many require user actions
many work only in background/service workers
most WAR paths are dynamic
private extension network traffic is hidden from the page
generic signals create false positives
```

The strongest remaining solutions are:

```text
Adblock for Youtube static WAR
Fathom static WAR
IDCAC cookie-banner detection
Microsoft SSO discovery
Scopus TestCommunication
SelectorsHub CSS fingerprint
Trust Wallet EIP-6963
Resource Timing
more explicit page-to-extension bridges
strict ON/OFF/mixed testing
```

The main strategy is:

> **Increase recall without sacrificing precision.**

That is the safest path to improving MCC.
