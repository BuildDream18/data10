# BEX Tracer Challenge — Current Research, Progress, Remaining Problems, and Technical Conclusions

**Prepared for sharing with a technical collaborator / friend**  
**Date:** November 5, 2026  
**Project:** RedTeam Subnet — BEX Tracer Chrome Extension Classification  
**Current measured score:** **MCC 0.2329**  
**Current measured behavior:** very high precision, low recall  
**Primary constraint:** the detector runs as ordinary JavaScript inside `baitpage.test`

> **Important status note**
>
> This document separates:
>
> - **Measured / observed results** — scores and behavior seen in previous submissions.
> - **Source-grounded candidates** — detection ideas based on extension source/manifests.
> - **Current estimates** — expected MCC improvements that still require real scorer validation.
> - **Fundamental browser limitations** — techniques that do not work from ordinary page-context JavaScript under the current challenge setup.
>
> The main goal is not merely to detect that “some extension exists.” The goal is to identify **which exact named extensions are enabled in the current round**, while avoiding false positives.

---

# 1. Executive Summary

The challenge contains **89 Chrome extensions**, divided into **8 groups**. In each scoring round, only **12 of the 89** are enabled. My eight JavaScript detector files run inside the challenge bait page and must return a boolean result for every extension.

The difficult part is that normal web-page JavaScript has almost no direct access to Chrome's extension-management state. It cannot call `chrome.management.getAll()`, cannot read the filesystem paths used by Chrome, cannot inspect other extensions' private background/service-worker traffic, and usually cannot message arbitrary extensions.

Because the scoring metric is **Matthews Correlation Coefficient (MCC)**, aggressive guessing is dangerous. A detector that creates many false positives can score much worse than a conservative detector that finds only one or two real extensions.

The best measured result documented so far is:

- **MCC:** `0.2329`
- **Precision:** approximately `1.0`
- **Recall:** approximately `8%`
- **Typical detections:** about 1–2 true extensions per round
- **Main current weakness:** most enabled extensions leave no unique page-visible signal on `baitpage.test`

A large amount of testing has already been completed. The strongest direction remains:

1. Keep only **extension-specific, source-grounded signals**.
2. Avoid generic network/timing/ad-block effects that create false positives.
3. Finish the two static Web Accessible Resource (WAR) probes.
4. Improve the cookie-banner detector.
5. Validate every positive rule with **ON / OFF / mixed-extension controls**.
6. Continue source analysis for any extension that exposes a page-visible handshake, DOM marker, CSS effect, response-header effect, or static WAR resource.

A separate network capability audit has also been produced:

- **89 total extensions**
- **67 classified as network-capable**
- **22 classified as having no extension-level network capability in that manifest-based classification**

However, this does **not** mean 67 extensions can be detected by observing network traffic. Most extension traffic occurs in private extension contexts that ordinary page JavaScript cannot inspect.

---

# 2. Main Problem I Am Solving

## 2.1 Exact Objective

For every scoring round:

1. Chrome starts with a subset of extensions enabled.
2. The detector receives no direct list of those extensions.
3. The page loads at `baitpage.test`.
4. My eight JavaScript files execute as ordinary page scripts.
5. Each detector must return something equivalent to:

```javascript
{
  "Extension A": true,
  "Extension B": false,
  "Extension C": false,
  ...
}
```

The objective is to make `true` correspond as closely as possible to the extensions that are actually enabled during that round.

This is a **named classification problem**, not just an “extension present / extension absent” problem.

For example:

- Detecting that an advertisement was blocked is not enough.
- Several different ad blockers can produce the same effect.
- Marking all of them as `true` creates many false positives.
- Under MCC scoring, that can destroy the score.

---

# 3. Challenge Structure

## 3.1 Extension Pool

The current pool contains **89 extensions** in 8 groups:

| Group | Count |
|---|---:|
| `ad_blockers` | 9 |
| `network_vpn` | 10 |
| `developer_tools` | 12 |
| `media_video` | 9 |
| `capture_recording` | 7 |
| `save_research` | 14 |
| `tabs_workflow` | 17 |
| `identity_security` | 11 |
| **Total** | **89** |

The working analysis treats the challenge extensions as Manifest V3 packages.

---

## 3.2 Per-Round Setup

According to the current challenge analysis:

- There are **6 rounds** per submission.
- **12 extensions** are enabled in each round.
- Therefore:
  - Positive class = 12
  - Negative class = 77

Chrome is started with the selected extensions using `--load-extension`.

The bait page is:

```text
baitpage.test
```

The page contains normal HTML elements, including:

- cookie acceptance UI,
- username field,
- password field,
- static page content.

The harness also performs some user-like page activity, such as accepting cookies and typing into fields, then waits briefly for extension-side effects.

---

## 3.3 Detector Delivery

The solution consists of 8 JavaScript files:

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

These scripts run in the ordinary page environment.

That distinction is extremely important:

> My code is not itself a privileged Chrome extension.

Therefore it does not automatically receive Chrome extension APIs.

---

# 4. Why This Problem Is Difficult

Normal page JavaScript cannot simply ask Chrome:

```javascript
"What extensions are enabled?"
```

Chrome intentionally isolates extension state from normal websites.

The problem therefore becomes:

> Can each extension be identified indirectly from something it exposes to the page?

Possible useful signals include:

- unique DOM elements,
- unique CSS effects,
- MAIN-world global variables,
- explicit page ↔ content-script handshakes,
- Web Accessible Resources,
- Resource Timing entries,
- unique rewritten links,
- unique response headers,
- extension-specific events,
- injected UI,
- source-specific behavior caused by page interaction.

The main problem is that many extensions do not expose any of these on an unrelated generic page.

---

# 5. MCC Scoring in Detail

## 5.1 Formula

The challenge uses Matthews Correlation Coefficient:

```text
MCC =
(TP × TN − FP × FN)
──────────────────────────────────────────────
sqrt((TP+FP)(TP+FN)(TN+FP)(TN+FN))
```

Where:

| Symbol | Meaning |
|---|---|
| TP | Extension is enabled and detector returns `true` |
| TN | Extension is disabled and detector returns `false` |
| FP | Extension is disabled but detector returns `true` |
| FN | Extension is enabled but detector returns `false` |

With 12 enabled and 77 disabled:

```text
TP + FN = 12
TN + FP = 77
```

---

## 5.2 MCC Range

| MCC | Meaning |
|---:|---|
| `+1.0` | Perfect classification |
| `0.0` | No useful correlation / degenerate prediction case |
| `-1.0` | Perfect inverse classification |

Perfect score:

```text
TP = 12
FN = 0
TN = 77
FP = 0

MCC = 1.0
```

---

## 5.3 Empty Prediction

If the detector predicts no extension as true:

```text
TP = 0
FP = 0
FN = 12
TN = 77
```

The MCC denominator contains:

```text
TP + FP = 0
```

so MCC is mathematically undefined.

The current challenge notes indicate the harness treats this degenerate case as:

```text
MCC = 0
```

Therefore returning every extension as `false` is safe from false positives but does not produce a useful score.

---

## 5.4 Recalculated MCC With Zero False Positives

Using the stated 12-positive / 77-negative round structure:

| True Positives | False Positives | False Negatives | True Negatives | MCC |
|---:|---:|---:|---:|---:|
| 0 | 0 | 12 | 77 | 0.000* |
| 1 | 0 | 11 | 77 | 0.270 |
| 2 | 0 | 10 | 77 | 0.384 |
| 3 | 0 | 9 | 77 | 0.473 |
| 4 | 0 | 8 | 77 | 0.550 |
| 5 | 0 | 7 | 77 | 0.618 |
| 6 | 0 | 6 | 77 | 0.681 |
| 12 | 0 | 0 | 77 | 1.000 |

`*` Degenerate case treated as zero by the current notes/harness behavior.

This table is important because it shows why recall matters:

- 1 reliable detection → MCC ≈ 0.27
- 2 reliable detections → MCC ≈ 0.38
- 3 reliable detections → MCC ≈ 0.47
- 4 reliable detections → MCC ≈ 0.55

So to move comfortably above `0.5`, I generally need about **4 correct detections per round with almost no false positives**.

---

## 5.5 Effect of False Positives

Recalculated values:

| TP | MCC with 0 FP | MCC with 1 FP | MCC with 2 FP |
|---:|---:|---:|---:|
| 1 | 0.270 | 0.162 | 0.109 |
| 2 | 0.384 | 0.291 | 0.232 |
| 3 | 0.473 | 0.391 | 0.332 |
| 4 | 0.550 | 0.475 | 0.419 |
| 5 | 0.618 | 0.550 | 0.496 |
| 6 | 0.681 | 0.618 | 0.566 |

This explains the main strategy:

> High precision is extremely important.

A detector that identifies fewer extensions but stays very precise can outperform one that aggressively guesses many names.

That is exactly what happened during previous experiments.

---

# 6. Current Measured Result

The current notes record:

```text
MCC = 0.2329
Precision ≈ 1.0
Recall ≈ 8%
Rounds = 6
```

Interpretation:

- When the current detector says an extension is active, it is usually correct.
- The main failure is **missing enabled extensions**.
- Therefore the problem is mostly recall, not precision.

This is a much better technical position than having high recall with dozens of false positives.

---

# 7. Progress Completed Until Now

## 7.1 Full 89-Extension Catalog Work

I have worked through the complete extension catalog and organized the 89 extensions into their required groups.

The analysis has included:

- manifest inspection,
- content-script scope analysis,
- source-file inspection,
- DOM marker research,
- CSS marker research,
- page-message protocols,
- Web Accessible Resource analysis,
- site-specific activation conditions,
- background/service-worker limitations,
- externally-connectable restrictions,
- version/pinned-package comparisons,
- network capability classification.

This is important because extension detection cannot be solved reliably using extension names alone.

Every useful positive rule should have a source-backed explanation.

---

# 8. Iteration History

The main experimental history currently looks like this:

| Iteration | Result | Main Idea | What Was Learned |
|---|---:|---|---|
| Original | ~0.27 | DOM markers | Very good precision, poor recall |
| r2–r8 | ~0.00 | generic WAR probing | dynamic WAR paths and browser load behavior created failures / FPs |
| r9 | ~0.24 | removed broken WAR logic, added resource harvesting | precision recovered |
| r10 | ~0.03 | timing side-channel | distinguished installed-like behavior, not enabled set |
| r11 | ~0.15–0.24 | privileged/local probing experiments | consumed time and did not produce usable page-level evidence |
| r12 | ~0.15–0.24 | reduced repeated privileged probing | fundamental browser restrictions remained |
| r13 | ~0.15–0.24 | diagnostic-state reporting | diagnostic output could itself contaminate expected names/results |
| r14 | ~0.23 | tighter page rules and reduced budget | precision remained strong; recall stayed low |
| Current measured best noted | **0.2329** | conservative extension-specific signals | main remaining problem = observability |

The most important lesson is:

> Generic probes have repeatedly reduced MCC because they generate false positives or waste the limited detection time.

---

# 9. Source-Grounded Detection Signals Found

The following signals have been researched as extension-specific or potentially extension-specific candidates.

Not every one fires on `baitpage.test`, and not every one has been fully validated against the live scorer.

## 9.1 Ad / Privacy Signals

### SponsorBlock

Observed/researched markers:

```text
#sponsorblock-document-script
#sponsorSkipNotice
window.SBLogs
```

Main limitation:

- usually relevant to YouTube pages,
- generic bait page does not become YouTube just because YouTube-looking DOM exists.

---

### I Don't Care About Cookies

Researched markers:

```text
.idcac
#idcac
.idc0_350
```

Current problem:

- extension may simply hide/remove the cookie banner,
- the marker classes may not always be created.

A better source-grounded candidate is to inspect the known challenge cookie UI after the extension has had time to act.

This still requires strong OFF/mixed controls before shipping.

---

### Adblock for Youtube

A page bridge exists on real YouTube origins.

There is also a potentially useful static Web Accessible Resource case because this extension is among the small number whose WAR configuration may not use a dynamic URL.

Remaining task:

- extract exact static WAR resource paths from the pinned package,
- validate ON,
- validate OFF,
- validate target OFF while unrelated extensions are ON.

---

# 10. Developer Tool Signals

## 10.1 SelectorsHub

Research found source-specific CSS behavior on probe attributes such as:

```html
<span xpath="1"></span>
<span xpath="2"></span>
<span css="1"></span>
<span css="2"></span>
```

Expected differential outline colors include values corresponding to:

```text
rgb(0,188,212)
rgb(242,154,0)
```

The useful property is not simply “an outline exists.”

The detector compares:

- marked nodes,
- unmarked control nodes,
- exact outline style,
- width,
- expected colors.

This reduces false positives.

Remaining issue:

- the inspected package version differed from the pinned catalog version in some analysis,
- so exact-pin validation remains important.

---

## 10.2 SEOquake

Researched DOM signature:

```text
seoquake-seobar
#seoquake-seobar-panel
[data-seoquake-seobar-host]
```

Main limitations:

- settings can affect whether the bar appears,
- page eligibility can affect injection,
- source inspection included version drift.

---

## 10.3 JSON Formatter

Researched structure:

```text
#jsonFormatterParsed
#jsonFormatterRaw
#buttonPlain
#buttonFormatted
#jfStyleEl
```

Main limitation:

The extension activates on a suitable raw JSON document.

The normal challenge page is ordinary HTML, so this signal normally does not appear.

---

# 11. Identity / Security Signals

## 11.1 Microsoft Single Sign On

One of the strongest researched candidates is an explicit page/content-script discovery mechanism.

High-level behavior:

1. Page sends the expected discovery message.
2. Extension content script receives it.
3. Extension creates a response node.
4. Detector requires:
   - a newly-created node,
   - a valid runtime-ID-shaped value,
   - the expected extension version.

Why this is promising:

- it is extension-specific,
- it does not require authentication,
- it is a deliberate page/content bridge,
- it can operate on HTTPS pages.

Remaining requirement:

- real ON/OFF/mixed scorer validation.

---

## 11.2 Trust Wallet

Researched EIP-6963 provider discovery:

```text
eip6963:requestProvider
eip6963:announceProvider
```

Candidate identity:

```text
rdns = com.trustwallet.app
```

Important safety/precision rule:

The detector only needs to inspect provider metadata.

It should **not**:

- request accounts,
- sign transactions,
- request wallet addresses,
- invoke transfers,
- inspect private wallet data.

Main remaining problem:

- inspected public version differed from the catalog-pinned version,
- provider injection may depend on settings/startup/version.

---

## 11.3 Authenticator

Researched activated UI markers:

```text
#__ga_grayLayout__
#__ga_scan__
#__ga_captureBox__
#__ga_qrCanvas__
```

Main limitation:

This appears only after QR/capture functionality is activated.

Therefore it is normally useless on a fresh untouched bait page.

The detector must not initiate capture just to manufacture its own detection evidence.

---

# 12. Save / Research Signals

## 12.1 Scopus Document Download Manager

A strong source-grounded discovery mechanism was found using the extension's existing communication event.

Candidate behavior:

```text
DDMRequestEvent
TestCommunication
```

Expected extension-created response includes:

```text
#DDMExtension-testelementid
data-extension-loaded="true"
data-extension-enabled="true"
```

Important precision rule:

- pre-existing nodes must not count,
- only a new response caused by the current probe should count.

This is one of the stronger candidates because it uses an actual extension communication mechanism instead of guessing from a generic DOM effect.

---

## 12.2 Save to Pinterest

Source inspection found a distinctive Save-button UI:

```text
[data-test-id="pinterest-save-button"]
```

The source also defines image-size requirements before the UI appears.

Research has included using a detector-owned image and hover behavior to see whether the extension responds.

Important limitation:

- a page can imitate the DOM,
- synthetic hover behavior must be verified in real Chrome,
- the detector must not click/save content.

---

## 12.3 Send from Gmail

Pinned source analysis found a specific rewrite behavior for `mailto:` links.

The extension can convert a mail link toward a Gmail compose URL such as:

```text
https://mail.google.com/mail/?view=cm&fs=1&tf=1
```

Important limitation:

The inspected script performs its pass under specific timing conditions.

Adding a new mail link after that pass does not necessarily create a valid detector.

The challenge bait was also reported not to contain a useful original `mailto:` anchor.

Therefore this is source-grounded behavior but not currently a dependable fresh-bait positive.

---

# 13. Tabs / Workflow Signals

## 13.1 Easy Auto Refresh

Researched activated UI:

```text
#easy-auto-refresh-modal
```

Main limitation:

The modal appears only under a conditional UI path.

Simply having the extension enabled does not guarantee this marker.

---

## 13.2 Auto Refresh Plus

A page flag has been investigated:

```text
window._ARP_CONNECTED === true
```

However, this must be validated carefully against:

- exact version,
- page scope,
- default configuration,
- OFF controls.

---

# 14. Media / Video Signals

## 14.1 Video Downloader Professional

Researched activated UI:

```text
#idLink64Ui
h2.clLink64All
.clLink64Start
```

The heading text is expected to identify the product UI.

Main limitation:

The panel appears only after an extension-side download workflow is activated.

It is not a normal idle-installation signal.

---

## 14.2 Picture-in-Picture Extension

Potential page state:

```text
video[__pip__]
```

Main limitation:

PiP normally requires a media/user activation path.

The detector must not manufacture browser media state simply to make the signal appear.

---

## 14.3 h264ify

Source research shows the extension targets YouTube-related origins and can modify codec support behavior.

Main problem:

`baitpage.test` is not a YouTube origin.

A YouTube-looking iframe or copied HTML does not change the real page origin.

Therefore it is not a dependable generic bait signal.

---

# 15. Capture / Recording Signals

## 15.1 Vimeo Record

Research found several possible UI/injection clues, including:

```text
window.__VIMEO_RECORD_INTERNAL__
```

and a compound UI/font pattern involving the Inter font and an extension-created host.

However:

- isolated-world data is not necessarily page-visible,
- anonymous DOM nodes are not unique enough by themselves,
- strong compound matching is required to avoid false positives.

---

## 15.2 Fathom AI Note Taker

Fathom has content-script behavior tied to meeting websites such as:

- Zoom,
- Google Meet,
- Microsoft Teams.

Research also identified Fathom as one of the small number of potentially statically probeable WAR cases.

Remaining task:

- extract the exact pinned WAR path,
- confirm static accessibility,
- run ON/OFF/mixed controls.

---

# 16. Network / Server URL Research Completed

A separate manifest-based network classification was created.

Artifacts produced include:

```text
NETWORK67_MATRIX.json
README_NETWORK67.md
BEX_8_NETWORK_AWARE_R12_FULL.zip
BEX_8_NETWORK_AWARE_R12_JS_ONLY.zip
```

Classification summary:

```text
Total extensions:             89
Network-capable classification: 67
Non-network classification:     22
```

Per-group network-capable counts from that build:

| Group | Network-capable targets |
|---|---:|
| `ad_blockers` | 9 |
| `network_vpn` | 10 |
| `developer_tools` | 9 |
| `media_video` | 8 |
| `capture_recording` | 5 |
| `save_research` | 12 |
| `tabs_workflow` | 7 |
| `identity_security` | 7 |
| **Total** | **67** |

Important conclusion:

> Network capability is not the same thing as page-level detectability.

Normal page JavaScript generally cannot inspect:

- extension service-worker `fetch`,
- private extension WebSockets,
- proxy configuration,
- declarativeNetRequest activity,
- `webRequest` listeners,
- requests sent only from extension pages,
- requests sent only from isolated extension contexts.

Examples of server/site information identified during research include:

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
ppc-onenote.officeapps.live.com
onenote.officeapps-df.live.com
onenote.officeapps.live-int.com
the-qrcode-generator.com
autorefresh.io
ecosia.org
hotcleaner.com
camelcamelcamel.com
```

This research is useful for understanding each package, but it does not automatically give a valid detector.

For example:

- seeing a request to YouTube does not prove SponsorBlock,
- seeing Gmail traffic does not prove GMass,
- seeing a blocked ad does not identify which ad blocker caused it.

---

# 17. Web Accessible Resource Research

WAR = Web Accessible Resource.

A normal page can sometimes load a resource exposed by an extension.

This initially looked like one of the best generic detection techniques.

The problem is Manifest V3's dynamic resource behavior.

Current research indicates:

```text
WAR-enabled extensions: 17
use_dynamic_url: true:   15
static / potentially probeable: 2
```

The two important remaining static cases in the current notes are:

- Adblock for Youtube
- Fathom AI Note Taker

For the other dynamic cases, the page does not know the session-specific resource URL.

Therefore hardcoding a static extension path is not dependable.

---

# 18. Resource Timing Research

A useful technique is:

```javascript
performance.getEntriesByType("resource")
```

This can reveal resources that were actually injected/loaded into the page.

This approach became part of the `harvestExtIds` research.

Advantages:

- passive,
- does not require privileged APIs,
- can reveal extension-generated resources.

Limitations:

- only works if the extension injected a visible resource,
- many extensions do not inject anything on `baitpage.test`,
- does not provide the full enabled set.

This is why resource harvesting improved safety but did not solve recall.

---

# 19. What Has Clearly Failed

## 19.1 Generic WAR Fetching

Attempt:

```text
load chrome-extension://... resources
```

Problem:

- most interesting WAR resources use dynamic URLs,
- static path assumptions fail,
- a failed resource load does not always produce a reliable browser distinction.

Conclusion:

> Only use a WAR rule when the exact pinned extension exposes a verified static path.

---

## 19.2 `<object>` Resource Probes

A previous approach used `<object>` loading behavior.

Observed result:

- very large number of false positives,
- blocked/invalid extension resources could still trigger misleading load behavior.

Conclusion:

> Do not use `<object onload>` as proof that a named extension is active.

---

## 19.3 Timing Side-Channel

A timing approach attempted to distinguish extension behavior from nonexistent resources.

Observed score:

```text
~0.03
```

Root cause:

The timing behavior corresponded more closely to:

```text
installed / routed
```

rather than:

```text
enabled in this exact round
```

That produced many false positives.

Conclusion:

> Timing is not sufficiently specific for the round's enabled set.

---

## 19.4 Generic Ad-Blocking Tests

Many ad blockers can block the same resource.

Possible extensions include:

- uBlock Origin Lite,
- Adblock Plus,
- AdGuard,
- AdBlocker Ultimate,
- Adblock for Youtube,
- Pie Adblock,
- and potentially overlapping privacy/filter tools.

If one ad disappears, page JS does not know which exact product caused it.

Wrong strategy:

```text
ad blocked → mark every ad blocker true
```

This produces multiple false positives for one possible true positive.

Conclusion:

> Generic ad blocking is evidence of a class of behavior, not a product identity.

---

# 20. Remaining Fixable Problems

## 20.1 Finish Adblock for Youtube Static WAR Rule

Current issue:

The rule exists conceptually, but the exact resource path still needs to be taken from the pinned package and validated.

Required validation:

1. target ON → resource/signal appears,
2. target OFF → resource/signal does not appear,
3. target OFF + unrelated extensions ON → signal remains absent,
4. repeat across multiple Chrome sessions,
5. confirm MCC improves.

---

## 20.2 Finish Fathom Static WAR Rule

Same process:

- extract exact pinned static WAR resource,
- test in real challenge-like Chrome,
- confirm no dynamic URL is required,
- confirm zero false positives.

---

## 20.3 Improve I Don't Care About Cookies Rule

Current marker-only logic is unreliable.

Better research direction:

Inspect the existing challenge cookie button:

```text
#accept-cookies
```

Potential signal:

- hidden by computed CSS,
- no inline style created by my detector/page,
- change occurs only when extension is ON.

But this cannot be shipped purely from theory.

It needs:

- ON control,
- OFF control,
- mixed OFF control,
- repeated rounds.

---

## 20.4 Exact Version Drift

Several promising rules were derived from versions newer than the challenge pin.

Examples in previous analysis include:

- SelectorsHub,
- SEOquake,
- Trust Wallet,
- other current Web Store packages.

This creates a serious risk:

```text
current extension source behavior ≠ pinned challenge version behavior
```

Remaining work:

- obtain exact pinned package where possible,
- compare manifest,
- compare source,
- compare injected markers,
- compare resource paths,
- compare startup timing.

---

## 20.5 Real ON / OFF / Mixed Validation

This is probably the most important remaining engineering task.

Every rule that can return `true` should pass:

### Test A — ON

```text
target extension ON
→ signal must appear
```

### Test B — OFF

```text
target extension OFF
→ signal must not appear
```

### Test C — Mixed negative

```text
target extension OFF
many other extensions ON
→ signal must still not appear
```

### Test D — Repetition

Run multiple fresh Chrome processes.

A rule should not be trusted from one successful run.

---

# 21. Main Fundamental Problem That Remains

The largest unsolved problem is not code quality.

It is **observability**.

Many extensions fall into one of these categories:

### Background-only

They work mainly through:

- service worker,
- extension popup,
- browser API,
- context menu,
- background logic.

No unique page-visible marker appears.

### Site-specific

They only activate on:

- YouTube,
- Gmail,
- Zoom,
- Google Meet,
- Amazon,
- academic pages,
- vendor sites,
- specific enterprise sites.

`baitpage.test` does not match those origins.

### User-action-dependent

They need:

- toolbar click,
- QR scan,
- recording activation,
- download action,
- PiP action,
- refresh configuration,
- clip/save action.

A fresh generic page does not produce the signal.

### Isolated-world-only

A content script may define:

```javascript
window.someVariable
```

but if it executes in Chrome's isolated extension world, normal page JavaScript cannot read that variable.

### Private network/background activity

An extension may contact its backend but do so inside:

```text
extension service worker
extension page
background context
proxy layer
declarativeNetRequest
webRequest
```

The bait page cannot normally inspect that traffic.

---

# 22. Techniques That Are Effectively Impossible From Ordinary Page JS

This section is specifically about the current page-context constraint.

It does **not** mean the information does not exist anywhere in the system.

It means the bait-page script cannot legitimately/readily obtain it through normal browser-page APIs.

---

## 22.1 Direct `file://` Access

Goal:

Read challenge runtime directories directly from the filesystem.

Blocked because:

- HTTPS/web pages cannot freely read arbitrary `file://` resources,
- Chrome does not expose the host filesystem to normal page JavaScript,
- required permissive launch flags are not present in the analyzed setup.

Conclusion:

> Not a page-JS solution.

---

## 22.2 Reading the Challenge Runtime Directory

The exact selected extension set may exist in runtime process/filesystem state outside the page.

That is architecturally different from page detection.

Normal page JS cannot enumerate those host/container directories.

Conclusion:

> Privileged host knowledge and page-observable extension behavior are separate channels.

---

## 22.3 Localhost / Internal API Probing

Experiments considered local/private addresses.

Problems include:

- container/network namespace boundaries,
- CORS,
- Private Network Access restrictions,
- no guaranteed service endpoint,
- browser security policy.

Conclusion:

> No dependable page-readable enabled-extension API was found.

---

## 22.4 Chrome DevTools Protocol From the Page

Chrome may expose a debugging endpoint internally.

The analyzed setup uses a dynamically assigned debugging port.

Normal page JS does not know that port.

Additional browser security boundaries also apply.

Conclusion:

> CDP is not a dependable page-level detection channel here.

---

## 22.5 `chrome.management.getAll()`

This API would solve much of the problem if it were available.

But:

```javascript
chrome.management
```

is an extension API.

A normal website does not receive that privileged API.

Conclusion:

> Not available to the bait-page detector.

---

## 22.6 Arbitrary `chrome.runtime.sendMessage()`

A website cannot normally message any extension it wants.

The extension must explicitly permit the website through `externally_connectable`.

The current analysis found no general permission for `baitpage.test` across the pool.

Conclusion:

> Cannot be used as a universal 89-extension discovery method.

---

## 22.7 File System Access API

Browser APIs such as a file picker require real user interaction and user selection.

They do not silently grant access to arbitrary challenge directories.

Conclusion:

> Not useful for automated headless scoring.

---

## 22.8 `file://` iframe

Embedding a local filesystem URL does not bypass browser security.

Problems:

- local resource load restrictions,
- origin isolation,
- unreadable `contentDocument`.

Conclusion:

> Not usable.

---

## 22.9 `chrome://` Pages

Normal websites cannot inspect internal Chrome pages such as extension-management pages.

Even if navigation were possible, cross-origin isolation would prevent the bait page from reading their contents.

Conclusion:

> Not a valid information channel.

---

## 22.10 Chrome Web Store Scraping

The Web Store can tell us:

```text
this extension exists
```

but not:

```text
this extension is enabled in the current scoring round
```

Conclusion:

> Catalog information is useful for research but not for round-state classification.

---

## 22.11 CSP Violation Events

CSP events describe policy violations for the current document.

They do not enumerate extension IDs or reveal which 12 packages Chrome loaded.

Conclusion:

> Not useful for enabled-set discovery.

---

## 22.12 Intercepting Other Extensions' Service Workers

A normal page cannot enumerate/control extension service workers.

They belong to separate extension origins/scopes.

Conclusion:

> Private extension background traffic remains private.

---

## 22.13 DNS Tricks / Rebinding

This would require control over DNS/domain infrastructure and still faces browser security controls.

It is also unrelated to a clean extension-specific detection rule.

Conclusion:

> Not an appropriate or dependable solution.

---

# 23. Important “Do Not Waste More Time” Areas

Based on the experiments so far, these directions should not be a priority:

1. **Generic ad blocking**
   - cannot attribute the effect to a named blocker.

2. **Generic request failure**
   - failure may have many causes.

3. **Timing**
   - does not reliably distinguish round-enabled from installed.

4. **Large-scale `<object>` probing**
   - produced false positives.

5. **Guessing dynamic WAR URLs**
   - random/session-specific behavior makes static guessing unreliable.

6. **Reading isolated-world globals**
   - page cannot see them.

7. **Detecting extension permission lists from page JS**
   - Chrome does not expose them.

8. **Treating a vendor-domain request as proof of extension identity**
   - normal pages and other software can contact the same domain.

9. **Marking many similar extensions true from one class-level signal**
   - MCC punishes the resulting false positives.

---

# 24. Privileged Host-Side Information — Important Boundary

Architecture analysis indicates that the browser process itself necessarily knows which extension directories were loaded.

That information exists outside the page.

A host/container administrator may be able to inspect process/runtime state.

However:

> This is not an ordinary `baitpage.test` JavaScript technique.

Whether any host-side inspection is allowed is a **challenge-rule question**, not a page-detection technique.

Therefore my current engineering report treats it only as an architectural observation.

I should not build the page detector around privileged host state unless the challenge explicitly permits that channel.

---

# 25. Current Estimate of the Page-JS Ceiling

The working notes currently estimate:

```text
current measured state:     ~0.23 MCC
after strongest remaining fixes: ~0.35–0.45 MCC
```

This is an **estimate**, not a proven mathematical maximum.

Why the estimate exists:

- only a subset of extensions expose unique page signals,
- only an even smaller subset activate on this specific generic bait page,
- each round randomly selects 12 extensions,
- therefore many rounds contain extensions that page JS cannot identify.

The exact achievable ceiling still needs to be measured after:

- static WAR fixes,
- IDCAC fix,
- exact-pin validation,
- repeated ON/OFF/mixed testing.

I should not call `0.45` a guaranteed or absolute maximum until those experiments are completed.

---

# 26. Most Important Remaining Research Directions

## Priority 1 — Exact Static WAR Validation

Finish:

```text
Adblock for Youtube
Fathom AI Note Taker
```

Goal:

gain additional true positives without sacrificing precision.

---

## Priority 2 — Cookie Banner Differential Detection

Test whether IDCAC produces a reliable extension-specific state change on the known challenge cookie UI.

---

## Priority 3 — Exact-Pin Source Review

For every promising rule derived from a newer version:

- get pinned CRX/package,
- inspect manifest,
- inspect exact source,
- confirm selector/event/resource still exists.

---

## Priority 4 — Build a Validation Matrix

For all positive-capable rules maintain a table:

| Extension | ON | OFF | OFF+Mixed | Repeat count | FP count | Ship? |
|---|---|---|---|---:|---:|---|
| Example | PASS | PASS | PASS | 20 | 0 | YES |

Do not ship a new `true` rule without this evidence.

---

## Priority 5 — Measure Time Budget

Some previous probing approaches consumed several seconds.

The detector should:

- avoid long sequential waits,
- run independent async probes in parallel,
- stop timers/listeners cleanly,
- keep enough time for DOM rules,
- avoid one slow extension blocking the group.

---

## Priority 6 — Improve Resource Harvesting

Continue checking:

```javascript
performance.getEntriesByType("resource")
```

but only map a resource to a named extension when:

- source path is unique,
- extension ID/path mapping is trustworthy,
- OFF/mixed controls show no collision.

---

## Priority 7 — Look for Explicit Page Bridges

The strongest candidates found so far are not generic fingerprints.

They are explicit extension communication mechanisms such as:

- Scopus discovery event,
- Microsoft SSO discovery,
- EIP-6963 wallet provider discovery.

Future source review should prioritize:

```text
window.postMessage
CustomEvent
document.dispatchEvent
addEventListener
externally_connectable
MAIN-world injection
page-created response nodes
```

These have a better chance of becoming specific detectors.

---

# 27. What Success Should Look Like

A good detector rule should satisfy all of these:

### Specific

It identifies one named extension, not just a category.

### Reproducible

It works across fresh Chrome launches.

### Low false-positive

Other extensions do not create the same signal.

### Fast

It finishes inside the scoring budget.

### Non-destructive

It does not change extension settings or perform dangerous actions.

### Source-grounded

There is a reason in the actual extension code for the signal.

### Current-version compatible

The behavior exists in the exact challenge-pinned version.

---

# 28. Current Project Status by Area

| Area | Status | Notes |
|---|---|---|
| 89-extension catalog | Completed | Names grouped into 8 required groups |
| Eight JS detector architecture | Completed | Required detector files exist |
| Source/manifests audit | Advanced | Significant package analysis completed |
| DOM marker research | Advanced | Multiple extension-specific candidates found |
| CSS marker research | Advanced | SelectorsHub candidate implemented/researched |
| Page-message handshakes | Strong progress | SSO + Scopus among strongest |
| Wallet discovery | Candidate found | Trust Wallet EIP-6963; exact-pin validation needed |
| WAR research | Advanced | 17 WAR cases analyzed; dynamic URL is major blocker |
| Resource Timing | Implemented/researched | Useful but limited coverage |
| Timing side-channel | Rejected | Too many false positives |
| Generic ad-block detection | Rejected for attribution | Cannot identify exact product |
| Local/internal API probing | Rejected | Not page-readable/reliable |
| CDP approach | Rejected for page detector | Random/private debugging endpoint |
| Network capability audit | Completed | 67 network-capable / 22 non-network classification |
| Exact server/domain research | In progress | Many notable domains identified |
| Real ON/OFF controls | Incomplete | Critical remaining work |
| Mixed-extension controls | Incomplete | Critical remaining work |
| Exact-pin validation | Incomplete for several candidates | Version drift remains |
| Current measured MCC | **0.2329** | High precision, low recall |
| Current estimated improved MCC | **0.35–0.45** | Estimate after strongest fixes, not yet proven |

---

# 29. Current Deliverables / Artifacts

Recent work has produced or used artifacts such as:

```text
8 detector JavaScript files
coverage.json
NETWORK67_MATRIX.json
README_NETWORK67.md
BEX_8_NETWORK_AWARE_R12_FULL.zip
BEX_8_NETWORK_AWARE_R12_JS_ONLY.zip
```

The purpose of the network matrix is to distinguish:

```text
extension can perform network operations
```

from:

```text
page JavaScript can see enough network evidence to identify it
```

Those are very different questions.

---

# 30. Main Conclusions

## Conclusion 1

The problem is mainly **browser isolation**, not lack of detection code.

Chrome intentionally hides extension-management state from websites.

---

## Conclusion 2

The current conservative strategy is correct.

The measured score is limited by recall, but false-positive-heavy experiments performed substantially worse.

---

## Conclusion 3

The best detector candidates are explicit, extension-specific page signals.

Examples:

- SSO discovery response,
- Scopus `TestCommunication`,
- Trust Wallet EIP-6963,
- SelectorsHub CSS differential,
- source-specific injected DOM.

---

## Conclusion 4

Network capability alone does not solve the challenge.

Even though 67 extensions were classified as network-capable, most of their traffic is hidden in extension/private contexts.

---

## Conclusion 5

Many extensions may simply be unobservable from `baitpage.test` in their default idle state.

Typical examples include:

- toolbar-only tools,
- themes,
- background utilities,
- site-restricted extensions,
- capture/download tools requiring activation,
- extension popup applications.

---

## Conclusion 6

The next improvement should come from **better evidence**, not more guessing.

Every new `true` rule must be justified by:

```text
ON success
OFF absence
OFF+mixed absence
repeated fresh-browser success
```

---

# 31. Immediate Next Plan

The recommended next engineering sequence is:

1. **Finish Adblock for Youtube static WAR extraction and validation.**
2. **Finish Fathom static WAR extraction and validation.**
3. **Implement and validate the improved IDCAC cookie-banner differential rule.**
4. **Build a formal ON/OFF/mixed validation table.**
5. **Review exact pinned versions for every version-drift candidate.**
6. **Retest MCC after only these high-confidence changes.**
7. **Do not add aggressive generic network/timing guesses.**
8. **Continue looking for explicit page bridges in the remaining extension source.**
9. **Use the network/server URL matrix for source research, not as automatic proof of identity.**
10. **Measure actual scorer performance before changing the estimated ceiling.**

---

# 32. Short Summary for a Collaborator

I am trying to identify **12 enabled Chrome extensions out of a pool of 89** using only normal JavaScript running inside `baitpage.test`.

The challenge uses MCC, so false positives are very expensive.

My best documented score so far is:

```text
MCC:       0.2329
Precision: ~1.0
Recall:    ~8%
```

I have already:

- analyzed the full 89-extension catalog,
- built the required 8 detector files,
- researched manifests and source,
- tested DOM/CSS/resource/network approaches,
- found several extension-specific candidates,
- rejected timing and generic probing because of false positives,
- analyzed Web Accessible Resources,
- built a network-capability matrix,
- classified 67 extensions as network-capable and 22 as non-network in that manifest-based pass,
- identified the core browser isolation limitations.

The biggest remaining problem is:

> most extensions do not expose a unique signal on the generic bait page.

The strongest remaining practical work is:

- finish the two static WAR rules,
- improve the cookie-banner differential detector,
- validate every candidate with ON/OFF/mixed tests,
- verify exact pinned extension versions,
- keep precision extremely high.

The current working estimate is that these improvements may raise MCC into roughly the **0.35–0.45** range, but that estimate is not yet proven by the scorer.

The main lesson from the work so far is simple:

> **Specific extension evidence helps. Generic behavior creates false positives and hurts MCC.**
