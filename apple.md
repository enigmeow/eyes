# Apple / iOS Research

**Question:** Can Claude read my mother's iPhone screen and tap on it, so she can operate her apps entirely by voice?

**Date of research:** 2026-09-08
**Status:** ⚠️ **DEPRIORITIZED 2026-09-08.** Every viable iOS path requires a permanently
tethered Mac, which was ruled out as unacceptable for a phone she carries. Primary direction
is now [android.md](./android.md). This document is kept because the constraints are durable
and the decision may be revisited — Path C (ReplayKit + Voice Control) remains the only
untethered iOS option and is the fallback if the Android platform switch proves too costly
for her.

---

## TL;DR

Not as an App Store app — iOS has no equivalent of Android's `AccessibilityService`, so no third-party app can read another app's UI tree or inject a tap.

But there is a real yes: **WebDriverAgent**, an Apple-signed XCTest runner, gives full UI-tree reads and real taps on a stock non-jailbroken iPhone today. It costs a tethered Mac and a developer certificate. That is the path to prototype.

---

## 1. The hard wall

iOS deliberately provides no API for:

- Reading the accessibility tree of another app
- Injecting touches, taps, or gestures anywhere outside your own app

VoiceOver reads the accessibility tree from *inside the OS*, in-process. That access was never exposed to third parties, and there is no entitlement to request. This is not a policy hurdle that can be appealed; it is an absent API.

On macOS the Accessibility API does allow cross-app inspection (with the Privacy & Security > Accessibility grant), but App Sandbox blocks it, and App Store Connect refuses non-sandboxed apps that use it — so even the Mac version is store-hostile. iOS has no analogue at all.

**Consequence:** the intuitive design — "an app on her phone that watches her screen and drives her other apps" — cannot be built and shipped. Every viable path below routes around this.

---

## 2. What Apple already ships (baseline — confirm before building anything)

A homegrown agent will be worse than these at the basics. Establish what she actually uses first.

### VoiceOver
The screen reader. Gesture-driven: swipe to move focus, double-tap to activate, rotor for navigation modes. Reads the accessibility tree directly.

### VoiceOver Recognition / Screen Recognition
On-device ML that scans the rendered UI, detects buttons and controls that developers never labeled, and feeds them to VoiceOver. **Runs entirely on-device, no cloud.** This is the closest thing Apple ships to "AI reads the screen," and it is the reason many unlabeled apps are usable at all.

Sub-features: Image Descriptions, Screen Recognition, Text Recognition. Configurable via the rotor and Quick Settings.

### Voice Control
This is the sleeper. Voice Control provides:

- `"Show names"` / `"Show numbers"` / `"Show text numbers"` — overlays a label or number on every interactive element on screen
- `"Tap <name>"` / `"Tap <number>"` — performs the tap
- Custom command recording — bind a short phrase to a longer sequence
- Grid mode for arbitrary coordinates

**This is already voice-driven full-device control.** It exists, it ships, it needs no development.

**The catch:** Voice Control was designed for motor impairment, not blindness. Per blind users on AppleVis, VoiceOver and Voice Control "were never necessarily meant to be used together" and running both is awkward and conflict-prone. VoiceOver's own spoken commands are long-winded, though Voice Control's command recording can shorten them.

> **This friction may be the actual problem worth solving, and it is far smaller than building a GUI agent.** Worth an evening of setup before writing any code.

### iOS 26 accessibility additions
- **Accessibility Reader** — full-screen distraction-free reading view with custom fonts, contrast themes, narration. Settings > Accessibility > Read and Speak.
- **Braille Access** — recreates a braille notetaker across iOS/iPadOS/macOS/visionOS. Launch apps, open BRF/TXT books, Nemeth math, system notifications from a connected display.
- **Siri start/stop sounds** — Settings > Accessibility > VoiceOver > Audio. Small but relevant: a blind user otherwise cannot tell if Siri is still listening.
- Accessibility Nutrition Labels on App Store listings.

### Third-party AI tools that already exist
- **Be My Eyes / Be My AI** — GPT-4-Vision describes what the camera sees. 10M+ volunteers, 1M+ blind users. Also runs "Service AI" contact centers (Microsoft Disability Answer Desk), reportedly resolving ~90% of requests without a human.
- **Seeing AI**, **Envision**, **Aira**, **PiccyBot** — same category.

**Critical distinction:** all of these describe the *world through the camera*. None of them touch the phone's own UI. That is the gap.

---

## 3. The viable paths

### Path A — WebDriverAgent (the one that works) ⭐

`WebDriverAgent` (Appium, open source) is an XCTest-based server you build and sign yourself, install on the phone, and drive over HTTP.

**Capabilities:**
- Read the live UI element tree — real labels, real coordinates, semantic element lookup
- Tap, swipe, scroll, type
- Launch and quit apps, open URLs, web contexts
- Screenshots

It links `XCTest.framework` and calls Apple's own APIs. **No jailbreak.** Listens on port 8100 on the device.

**Requirements:**
- Mac with Xcode
- Apple signing identity
- Device connected by cable
- `libimobiledevice` (Homebrew) and `iproxy` for USB port-forwarding to reach port 8100

**Constraints:**

| Constraint | Detail |
|---|---|
| Certificate expiry | Free Apple ID → **re-sign every 7 days**. Paid account ($99/yr) → **once a year**. |
| Persistence | The runner must stay alive. Closing it halts control. Dies on reboot. |
| Connection | USB required initially; port 8100 forwarded over the same USB link. Wi-Fi possible but flakier. |
| Security prompts | Face ID, Apple ID password, payment confirmations stay manual **by design**. Automation runs through your developer cert, not a credential bypass. |
| Not a UI | "Not a polished 'see my phone on my Mac' window" — it is a scripting interface. |

**⚠️ THE OPEN QUESTION — TEST THIS FIRST**

Sources conflict on whether the XCTest runner can drive **arbitrary third-party apps** or only apps you signed yourself.

- Appium's documentation and general practice describe system-wide automation, and people do drive App Store apps with it.
- One source states: *"the sandbox stops you from poking at arbitrary third-party apps the way you can on Android"* and that full automation is guaranteed only for "an app you build and sign."

That statement may be about *gray-box* inspection (reaching into an app's process) rather than *black-box* UI driving via the accessibility layer — but it is unresolved from documentation alone.

**Everything downstream depends on this.** Test: install WDA, launch her banking app, request the UI tree. ~10 minutes. The answer decides the entire architecture.

**If it passes, the architecture is:**
```
Claude  ←HTTP→  WDA (port 8100)  ←USB→  iPhone
                     ↑
              always-on Mac mini at her house
              paid dev account (1-year cert)
```

**Related tooling seen in research:**
- `appium/WebDriverAgent` (GitHub) — canonical
- SideTap (sidetap.io) — commercial "drive your iPhone from Windows"
- `Joe15935/ios-agent-bridge` — an MCP server wrapping iOS agent control
- `chuk-mcp-ios` — MCP server for simulators **and real devices**, via Facebook IDB, over HTTP/SSE/stdio
- `joshuayoes/ios-simulator-mcp`, `whitesmith/ios-simulator-mcp`, `atom2ueki/mcp-server-ios-simulator` — **simulator only**, not useful here, but useful for building/testing the agent logic

### Path B — iPhone Mirroring + Claude computer-use ❌ DEAD

Mirror the iPhone to a Mac (macOS 15 Sequoia+ / iOS 18+), let Claude's existing computer-use tooling drive the mirrored window. Elegant. No certificates.

**Killed by one Apple design decision:**

> iPhone Mirroring only works while the iPhone is **nearby and locked**. The moment she picks up and unlocks her phone, mirroring turns off. It resumes only when the phone is locked again. It also auto-pauses after inactivity.

She would have to leave the phone face-down on a table and use a Mac. That is not "her phone, operated by voice."

**Second problem:** the mirrored content is a video stream. When an iOS device is locked for iPhone Mirroring, **the app UI contents are not available to Xcode** — and by extension not to the macOS Accessibility API. Claude would be reading pixels, not labels. Strictly worse than Path A.

Requirements for completeness: same Apple Account on both, Wi-Fi + Bluetooth on, within ~30 ft / 10 m.

### Path C — ReplayKit Broadcast Extension (read-only, App Store legal) ⭐ interesting hybrid

A **Broadcast Upload Extension** with an `RPBroadcastSampleHandler` receives **system-wide screen frames** — every app, home screen, Safari, Settings. Triggered by the user via `RPSystemBroadcastPickerView` / Control Center. This is the mechanism behind "share my entire screen" in Zoom, Teams, Meet, Discord.

**This is fully legitimate and shippable on the App Store.** It reads the whole screen.

**It cannot tap.**

**Constraints:**
- Extension is capped at **50 MB memory** and is killed if exceeded. This is tight for video frame handling — expect real engineering here.
- Screen recording stops automatically when the screen locks or a call comes in.
- Separate extension target/process from the host app.

**The hybrid worth serious thought:**

> Claude reads the screen via ReplayKit → speaks the next action out loud → she says the Voice Control phrase → **Apple performs the tap.**

This splits the problem along the line where each side is strong. Reading and understanding unfamiliar screens is the hard part for her; tapping is already solved by Apple. It ships on the App Store, needs no certificates, no Mac, no tether. The agent becomes a *guide* rather than a *driver*.

Downside: she is in the loop for every action, so it is slower, and it inherits the VoiceOver/Voice Control coexistence friction from §2.

### Path D — Apple's sanctioned rails (App Intents / Siri / Shortcuts)

**Siri AI** — the long-delayed personal-context features. Timeline:
- Announced WWDC 2024; delayed March 2025; targeted spring 2026
- Reportedly shipped in **iOS 27 developer builds on 2026-06-08** as "Siri AI": personal context, world knowledge, **onscreen awareness**
- *(Confidence: medium — sourced from a Medium article and search summaries, not an Apple press release. Verify.)*

Onscreen awareness means Siri can read the current screen and act contextually ("add this address to my contacts").

**App Intents** — a large expansion letting developers expose in-app actions to Siri. This is Apple's official answer to agentic control.

**Shortcuts `Use Model` action (iOS 26)** — calls Apple Intelligence on-device, Private Cloud Compute, or ChatGPT from inside an automation, feeding results into subsequent steps. Shortcuts run from Siri, Control Center, Action Button. iOS 26 added notification-triggered automations with keyword filtering, plus screenshot and keyboard-connection triggers.

**Verdict:** most robust layer, zero certificates, zero tethering — but the ceiling is whatever each app's developer chose to expose. Great for "text my son." Useless for an insurance portal that ships no App Intents.

**Recommendation: build her top ~10 daily tasks here regardless of what else happens.** It is the layer least likely to break.

**Apple's research direction:** Ferret-UI Lite (Feb 2026 paper) — an on-device GUI-grounding model, 53.3% on ScreenSpot-Pro, >15% better than UI-TARS-1.5 (7B). Apple is clearly building toward Siri seeing and controlling apps natively. Not shipping to third parties.

### Path E — EU DMA interoperability (long shot, probably not applicable)

Under the DMA, Apple must open iOS features to third parties. iOS 26.3 delivered proximity pairing for third-party accessories and notification forwarding. By 2026-06-01 Apple was due to ship more (notifications, proximity pairing, audio switching).

**But:** as of 2026-03-22, **not one of 56 formal interoperability requests had resulted in a solution.** Apple continues to litigate its obligations; Meta has challenged Apple's plan.

Accessibility-tree access is not among the mandated features, and this is EU-only. Not a path — noted for completeness.

---

## 4. Comparison table

| Path | Reads screen | Taps | App Store | Needs Mac tether | Works with phone in hand |
|---|---|---|---|---|---|
| A. WebDriverAgent | ✅ real UI tree | ✅ | ❌ | ✅ yes | ✅ |
| B. iPhone Mirroring | ⚠️ pixels only | ✅ | n/a | ✅ yes | ❌ **fatal** |
| C. ReplayKit + Voice Control | ✅ pixels, system-wide | via user's voice | ✅ | ❌ no | ✅ |
| D. App Intents / Shortcuts | ⚠️ only exposed data | ⚠️ only exposed actions | ✅ | ❌ no | ✅ |

---

## 5. Recommended plan

1. **Test the WDA third-party-app question.** Everything hinges on it; ~10 minutes.
2. **In parallel, set up Voice Control + VoiceOver properly** with custom commands. This establishes the real baseline and reveals which tasks actually need an agent.
3. **If WDA passes:** Claude ↔ WDA over HTTP, always-on Mac mini at her house, paid dev account for the 1-year cert. Plan for runner supervision/auto-restart.
4. **If WDA fails:** ReplayKit reads + Voice Control taps (Path C), plus App Intents for the top ten tasks (Path D).
5. **Either way:** build Path D's high-frequency tasks. Most robust layer.

---

## 6. Safety requirement (design in, do not bolt on)

An agent with tap authority on her phone can move money, and **she cannot visually verify what it did.**

Whatever is built must:
- Narrate its intended action before committing
- Hard-stop at anything financial or irreversible
- Keep Face ID / password / payment confirmation in her hands (WDA enforces this anyway, which is a feature not a limitation)

---

## 7. Open questions

### Resolved by decision, not by evidence

The WebDriverAgent question below was never tested. It became moot when the Mac tether was
rejected on 2026-09-08 — even a "yes" would not have produced an acceptable system. Recorded
here because it is the first thing to answer if this path is ever revisited.

- [ ] ~~**Can WDA drive third-party App Store apps, or only self-signed apps?**~~ (untested; moot)
- [ ] ~~Does WDA survive reboot, or must someone re-launch it?~~ (moot)
- [ ] ~~Is WDA-over-Wi-Fi stable enough to avoid a permanent USB tether?~~ (moot — Wi-Fi still
      requires the Mac, which was the actual objection)
- [ ] ~~Does WDA's accessibility tree read correctly while VoiceOver is active?~~ (moot)

### Still live — these matter regardless of platform

- [ ] **Test Gemini Live screen sharing on her existing iPhone** against her real tasks. It
      is available on iOS now. Possible meaningful value today with zero engineering, on the
      phone she already knows. Cheapest experiment available.
- [ ] **Set up Voice Control + VoiceOver properly** with custom commands. `"Show names"` →
      `"Tap Send"` is already voice-driven device control. This establishes the true baseline
      and reveals which tasks genuinely need an agent — which is required input for the
      Android work too.
- [ ] **Enumerate her actual daily tasks.** Nothing here can be evaluated without them, and
      the Android plan now blocks on the same list.

### Live only if iOS is revisited (Path C fallback)

- [ ] Is the ReplayKit 50 MB extension ceiling workable for streaming frames to a vision model?
- [ ] Does Voice Control's `"Show names"` overlay appear in a ReplayKit capture? **If yes,
      Path C gets much stronger** — Claude could read the labels and speak the exact phrase
      back to her, making the read/act split clean.
- [ ] Confirm iOS 27 / Siri AI onscreen-awareness status against an Apple primary source
      (current sourcing is a Medium article; confidence medium).

## Sources

- [Control your iPhone from your Mac — Apple Support](https://support.apple.com/en-us/120421)
- [Control your iPhone from your Mac (guide) — Apple Support](https://support.apple.com/guide/iphone/control-your-iphone-from-your-mac-iph505911a40/ios)
- [Manage iPhone Mirroring — Apple Personal Safety Guide](https://support.apple.com/guide/personal-safety/manage-iphone-mirroring-on-your-iphone-or-mac-ips70daa1bcf/web)
- [UI automation with iPhone Mirroring — Apple Developer Forums](https://developer.apple.com/forums/thread/765955)
- [appium/WebDriverAgent — GitHub](https://github.com/appium/WebDriverAgent)
- [Control & Automate Your iPhone With WebDriverAgent — Jean Galea](https://jeangalea.com/control-automate-iphone-webdriveragent/)
- [WebDriverAgent — The Heart of iOS E2E Testing — Thuyen's Corner](https://trinhngocthuyen.com/posts/tech/mobile-e2e-wda/)
- [There's No ADB for iPhone — Joche Ojeda](https://www.jocheojeda.com/2026/07/04/no-adb-for-iphone-automating-react-native-ui-tests-on-real-ios-devices/)
- [SideTap](https://sidetap.io/)
- [Use VoiceOver Recognition on your iPhone — Apple Support](https://support.apple.com/en-us/111799)
- [Use Voice Control commands to interact with iPhone — Apple Support](https://support.apple.com/guide/iphone/use-voice-control-iph2c21a3c88/ios)
- [Use Voice Control on your iPhone — Apple Support](https://support.apple.com/en-us/111778)
- [Using VoiceOver and Voice Control together — AppleVis](https://www.applevis.com/forum/ios-ipados/using-voiceover-voice-control-together-ios)
- [Voice Control evaluation criteria — App Store Connect Help](https://developer.apple.com/help/app-store-connect/manage-app-accessibility/voice-control-evaluation-criteria/)
- [Apple's VoiceOver Screen Recognition — AFB AccessWorld](https://afb.org/aw/spring2024/apple-screen-recognition-machine-learning-accessibility)
- [iOS and iPadOS 26 Accessibility Updates — AFB](https://afb.org/blog/entry/ios-26-accessibility-features)
- [What's New in iOS 26 Accessibility for Blind and DeafBlind Users — AppleVis](https://www.applevis.com/blog/whats-new-ios-26-accessibility-blind-deafblind-users)
- [iOS 26 Brings More Braille Access — Helen Keller Services](https://www.helenkeller.org/ios-26-braille-access/)
- [Live Screen Broadcast with ReplayKit — WWDC18](https://developer.apple.com/videos/play/wwdc2018/601/)
- [iOS Screen Sharing: ReplayKit + Broadcast Extension — Forasoft](https://www.forasoft.com/blog/article/how-to-implement-screen-sharing-in-ios-1193)
- [LiveKit iOS screen sharing docs](https://github.com/livekit/client-sdk-swift/blob/main/Docs/ios-screen-sharing.md)
- [Broadcast extension memory limit — Apple Developer Forums](https://developer.apple.com/forums/thread/651367)
- [What's new in Shortcuts for iOS 26 — Apple Support](https://support.apple.com/en-us/125148)
- [Here's What You Can Do With the iOS 26 Shortcuts App — MacRumors](https://www.macrumors.com/2025/06/11/ios-26-shortcuts-app/)
- [App Intents and the next Siri — Mac O'Clock](https://medium.com/macoclock/app-intents-is-the-part-of-ios-27-that-explains-why-siri-might-work-this-time-3e69ffa52fb5)
- [Apple Plans to Release Delayed Siri Features in Spring 2026 — MacRumors](https://www.macrumors.com/2025/06/12/apple-intelligence-siri-spring-2026/)
- [Apple's Ferret-UI Lite — AppleInsider](https://appleinsider.com/articles/26/02/21/apples-latest-ferret-ai-model-is-a-step-towards-siri-seeing-and-controlling-iphone-apps)
- [Changes for apps in the EU — Apple Developer](https://developer.apple.com/support/dma-and-apps-in-the-eu/)
- [Apple keeps challenging its interoperability obligations under the DMA — FSFE](https://fsfe.org/news/2026/news-20260420-01.en.html)
- [EU credits DMA as Apple opens iOS 26.3 to third-party accessories — Digital Watch](https://dig.watch/updates/eu-credits-dma-as-apple-opens-ios-26-3-to-third-party-accessories)
- [Be My Eyes](https://www.bemyeyes.com/)
- [Accessibility Permission In Sandbox — Apple Developer Forums](https://developer.apple.com/forums/thread/789896)
