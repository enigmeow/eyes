# Android Research

**Context:** Companion to [apple.md](./apple.md). Originally researched as the comparison
case. **As of 2026-09-08 this is the primary direction** — the iOS path required a
permanently tethered Mac, which was ruled out as unacceptable for a phone my mother
carries.

**Date of research:** 2026-09-08 (updated same day: build options, then hands-on setup)
**Status:** **Dev environment verified working** — phone paired over Wi-Fi, UI tree and
screenshots confirmed, no cable and no app required. See §4. Nothing built yet.

---

## TL;DR

Everything iOS forbids, Android grants with a single user-facing toggle.

`AccessibilityService` lets an app **read the full UI tree of every app** and **inject taps
and gestures anywhere on the device**. No root, no ADB, no tethered computer, no
certificate rotation. It survives reboot. It ships on the Play Store. This is why
essentially every mobile GUI agent in the research literature is built on Android.

**Confirmed hands-on 2026-09-08:** a Galaxy S25 Ultra pairs over Wi-Fi with no cable, and
`uiautomator dump` reads the full UI tree **with zero accessibility services enabled**. The
plumbing question is closed; see §4.

**And we probably don't have to write it.** Two open-source apps already implement the
accessibility bridge — one of them (MobileRun Portal) is a drop-in APK that serves the UI
tree over HTTP with no ADB at all. See §3.

The real costs are not technical:
1. Google Play's `isAccessibilityTool` review gate (surmountable — our use case is
   genuinely the intended one; moot entirely for a sideloaded single-user build)
2. TalkBack quality varies by OEM, unlike VoiceOver
3. Asking a blind user with years of VoiceOver muscle memory to switch phones is a
   serious, possibly disqualifying, human cost

---

## 1. AccessibilityService — the capability

An Android `AccessibilityService` "requests permission to observe user actions, retrieve
window content, and perform gestures on behalf of the user."

### Reading the screen

The **accessibility tree** is a hierarchical representation of screen content. The service
works with `AccessibilityNodeInfo` objects representing buttons, lists, text fields — with
labels, bounds, and state.

Key configuration:

| Setting | Effect |
|---|---|
| `android:canRetrieveWindowContent="true"` | Required to read window content at all |
| `FLAG_RETRIEVE_INTERACTIVE_WINDOWS` | Access content of **all** interactive windows, not just the focused one. Without it, `getWindows()` returns empty, `getWindow()` returns null, and you get no `TYPE_WINDOWS_CHANGED` events. Ignored unless `canRetrieveWindowContent` is set. |
| `android:canPerformGestures="true"` | Required for `dispatchGesture()` |

Plus a live event stream (`AccessibilityEvent`) so the service is notified when the screen
changes — no polling needed.

### Acting on the screen

Three independent mechanisms:

1. **`performAction(AccessibilityNodeInfo.ACTION_CLICK)`** — semantic. Find the node, check
   `isClickable()`, activate it. Robust against layout shifts.
2. **`dispatchGesture()`** — synthetic taps, swipes, multi-touch at raw coordinates.
3. **`performGlobalAction()`** — Back, Home, Recents, notification shade, etc.

**Known gap:** on recent Android versions, the accessibility tree of **certain Jetpack
Compose dialogs is partially inaccessible** to the service. Documented workaround: fall back
from `ACTION_CLICK` on the node to `dispatchGesture()` at the button's coordinates.

> **Design implication:** build tapping as semantic-first with a coordinate fallback from
> day one. Don't discover this in a dialog that matters.

---

## 2. Why this beats the iOS architecture

The whole apple.md Path A stack — Mac mini, USB tether, port 8100, 7-day or annual cert
rotation, runner supervision — collapses into an app that runs on the phone.

| | iOS (WebDriverAgent) | Android (AccessibilityService) |
|---|---|---|
| External hardware | Mac, permanently | None |
| Credential expiry | 7 days free / 1 year paid | None |
| Survives reboot unattended | ❌ runner must be relaunched | ✅ enablement persists |
| Physical tether | USB | None |

That third row is the one that decided the project direction.

---

## 3. Do you need to build a native app?

**Something native must exist. You almost certainly don't have to write it.**

### Why native is unavoidable at the bottom

`AccessibilityService` is declared in the Android manifest and implemented as an Android
`Service` subclass in Kotlin/Java. There is no JavaScript or Dart escape hatch. In React
Native or Flutter you would write the *identical Kotlin* as a native module (TurboModule /
Platform Channel) and then pay bridge overhead on top. Cross-platform frameworks buy nothing
here.

But it is a **thin** floor. The service only needs to: read the tree, dispatch gestures, and
talk to something over a socket. All planning, LLM calls, and voice handling live outside it.

### Option A — MobileRun Portal (zero Android code, no tether) ⭐ start here

`droidrun/mobilerun-portal` — a prebuilt accessibility-bridge APK that hosts servers **on the
device**:

- **HTTP on port 8080**, **WebSocket on port 8081**, both auth-token protected
  (token generated locally, visible in-app or via ContentProvider)
- Exposes UI-tree extraction, gestures, screenshots, keyboard/clipboard
- **Reachable over the network without ADB**
- **Reverse-WebSocket mode**: the phone dials *out* to your server — no inbound ports, no
  port forwarding, works behind carrier NAT/CGNAT. Right shape for a phone she carries.
- Requires: AccessibilityService enabled + overlay permission
- Built with Kotlin/Gradle; signed APKs from GitHub Releases (`v*` tags) or CI artifacts

**⚠️ Documentation trap:** the MobileRun *framework* quickstart says ADB is mandatory, and
the `chukfinley/droidrun-mcp-server` fork is USB-tethered. **That is a property of their
Python client's default path, not of the Portal app.** Talk to the Portal's HTTP/WS
endpoints directly and ADB drops out entirely. Don't let the framework docs talk you into a
tether we already rejected.

Also note the framework's default cap of **15 steps per agent run**, and that Anthropic
support is an optional package extra (`[anthropic]`) rather than a default provider.

Verdict: **install an APK, write the agent loop in any language, talk HTTP.** Fastest route
to "Claude tapped my phone."

### Option B — OpenDroid (the whole thing, already built)

`yashab-cyber/opendroid` — Apache 2.0, ~1.4k stars, ~425 commits, single maintainer.

- Runs **entirely on-device**. No ADB, no computer.
- AccessibilityService for screen reading and control; 60+ action executors
- **Anthropic Claude** among 12 providers, with automatic failover
- **Offline wake-word detection, speech-to-text, and TTS** (optional ElevenLabs voices) —
  precisely the voice layer we'd otherwise build
- Self-planning agent loop with memory across sessions
- Dual-tier fallback: if hardware screen capture fails, drops to layout text-scraping
- Clean architecture, Dagger-Hilt DI, Jetpack Compose UI
- Build: JDK 21 + Android SDK 35, `./gradlew assembleDebug`

**⚠️ Trust this less than its README does.** The repo bills itself "production-ready." The
author's own launch post says: *"Early alpha — bugs exist, contributors welcome."* No
independent reviews found. Single maintainer.

Verdict: **read the code before it goes near her phone.** But as a reference implementation
for the voice UX and the action-executor taxonomy, it is worth a great deal.

### Option C — Termux + ADB-to-localhost (no app at all)

Android 11+ wireless debugging allows `adb pair localhost:<port>` **from Termux on the device
itself** — no root, no computer. Then:

- `uiautomator dump` → the UI tree
- `input tap <x> <y>` → injected touches

Setup: `pkg install android-tools -y`, get the 6-digit pairing code from Wireless Debugging
settings, pair and connect against `127.0.0.1`.

**✅ Verified working 2026-09-08 (§4)** — though from the Mac over Wi-Fi rather than from
Termux on-device. Reads the full tree with no accessibility service enabled, and the mDNS
device name removes the reboot/port fragility this option was marked down for.

Still not a production foundation — developer options must stay on, there's no live event
stream, and gesture injection via `input tap` is slower and blinder than
`AccessibilityService`. But as **the tool for answering the blocking question it is now the
recommended first step**, ahead of installing anything.

### Option D — Tasker + AutoInput (no-code)

AutoInput's accessibility service queries the current window's `AccessibilityNodeInfo` tree,
applies filters (text, class name, content description, clickable state) and acts on matched
nodes. Tasker triggers it. Clunky and not a foundation to build on, but a legitimate way to
sanity-check whether a specific app's tree is usable before writing anything.

### Option E — write the thin Kotlin service yourself

A few hundred lines. Worth doing **after** the prototype proves the trees are good enough,
because it buys the things that actually matter for her: wake-word behaviour, barge-in, how
actions get narrated aloud, reboot persistence, and battery discipline.

### Recommended sequence

1. **Portal APK + own agent loop** — validates the real unknown (is the accessibility tree
   good enough *on the specific apps she uses*) with no Android development.
2. **Mine OpenDroid** for the voice layer and action taxonomy.
3. **Write the thin Kotlin service** once the approach is proven.

---

## 4. Verified setup (2026-09-08)

**Hands-on on a real device. Everything in this section is confirmed working, not researched.**

### Hardware and connection

| | |
|---|---|
| Phone | Samsung Galaxy S25 Ultra (SM-S938U1), codename `pa3q` |
| OS | **Android 16, SDK 36** |
| Host | MacBook Air M4, macOS 26 (Darwin 25.6.0) |
| adb | 1.0.41 / 37.0.1, Homebrew, `/opt/homebrew/bin/adb` |
| Link | **Wi-Fi only — no cable was used at any point** |

### Pairing procedure that worked

1. Phone: Settings → About phone → Software information → tap **Build number** ×7
2. Settings → Developer options → **Wireless debugging** → ON
3. Tap the **Wireless debugging label** (not the toggle) → **Pair device with pairing code**
4. Host: `adb pair <ip>:<pair-port> <6-digit-code>`
5. Close the popup; read **IP address & Port** off the main screen — *a different port*
6. Host: `adb connect <ip>:<connect-port>`

Confirmed: **Android 11+ wireless debugging needs no initial USB connection.** The old
`adb tcpip 5555` cable requirement applies to Android 10 and earlier only.

### ⚠️ Gotchas hit for real

**A VPN on the phone breaks pairing.** The first attempt reported `10.5.0.2` — a NordVPN
tunnel address, not a LAN address. The Mac had a route to it via `utun4` (MTU 1420,
WireGuard-shaped) but the pairing port was unreachable and `adb pair` failed with
`protocol fault (couldn't read status message)`.
**Fix: turn the VPN off on the phone** so it gets a `192.168.1.x` address on the same LAN.
NordVPN running on the *Mac* was harmless — local `192.168.1.x` still routed direct via `en0`.

**Two different ports, and it is not obvious.** The pairing popup and the main Wireless
debugging screen show different ports. Popup port → `adb pair` only. Main-screen port →
`adb connect`.

**macOS has no `timeout`.** Burned a pairing code on `timeout 25 adb pair ...` →
`command not found`, and the code expired before the retry. Use `gtimeout` (coreutils) or
omit it.

**Pairing codes expire fast.** Keep the popup open; regenerate freely if a step goes wrong.

### mDNS solves the randomized-port problem ⭐

adb registers the device under a stable mDNS name:

```
adb -s adb-R5CXC3JDSYR-YBS99B._adb-tls-connect._tcp shell ...
```

**Confirmed responding.** This name survives reboots and port changes, so the randomized-port
annoyance documented all over the internet (and in this file's earlier draft) does not apply
once paired. **Use the mDNS name as the device handle, not `ip:port`.**

Minor quirk: `adb mdns services` returns an empty list even while the name resolves fine.
Cosmetic; ignore it.

### Capability test — all passed

| Capability | Command | Result |
|---|---|---|
| UI tree | `uiautomator dump` | ✅ 65 nodes, 7 clickable, 14 labelled |
| Screenshot | `exec-out screencap -p` | ✅ 177 KB PNG |
| Foreground app | `dumpsys window \| grep mCurrentFocus` | ✅ `com.android.settings` |

### 🔑 Key finding: reading the tree needs no accessibility service at all

`settings get secure enabled_accessibility_services` returned **`null`** — nothing enabled —
and `uiautomator dump` still produced a complete tree.

**This makes Option C (§3) substantially stronger than the research suggested.** Tree quality
on her real apps can be evaluated *today*: no Portal APK, no OpenDroid, no accessibility
service, no app development, no cable.

The AccessibilityService is still required for the *production* agent — live
`AccessibilityEvent` streams, gesture injection, and running on-device without a host. It is
**not** required to answer the blocking question.

---

## 5. Keeping it alive (do not defer this)

Android will kill a naive background service, and the failure mode is mystery silence — the
worst possible failure for a blind user who cannot see that nothing is happening.

**Two corrections from the verified setup (§4):** the target device runs **Android 16 /
SDK 36**, one version newer than the Android 15 limits below — re-check against 36 rather
than trusting these. And it is a **Samsung**: One UI is markedly more aggressive than stock
Android about killing background services and silently re-enabling battery optimization.
This moves the section from "important" to "will definitely bite you."

- Background services are killed after a period in background (Android 8.0 / API 26+).
- A **foreground service with a persistent notification** is required for indefinite
  operation — but note that on Android 15+, only certain FGS types (e.g. `mediaPlayback`,
  `location`) run indefinitely; others (`dataSync`, `mediaProcessing`) are time-boxed.
  **Choosing the right FGS type is an open design question.**
- From **Android 12+**, a foreground service cannot be launched from the background unless
  **battery optimization is disabled for the app**. This must be part of first-run setup.
- Reboot: the *accessibility service enablement setting* persists, which is the big win over
  iOS. Restarting our own service after reboot still needs `BOOT_COMPLETED` handling, or
  JobScheduler/AlarmManager revival.

---

## 6. The Play Store gate

A review process, not a technical wall. **Moot for a sideloaded single-user build.**

### `isAccessibilityTool="true"`

> "Only services designed to help people with disabilities access their devices or overcome
> challenges due to their disabilities are eligible to declare themselves as accessibility
> tools by setting `isAccessibilityTool=true`."

Qualifying apps are **exempt from prominent-disclosure and consent requirements**.

**We qualify.** Building a screen reader/agent for a blind user is precisely the intended use.

### What review requires

- A **short video** demonstrating the app's main purpose
- A written explanation of the target audience and the specific disabilities addressed
- Without an approved declaration you **cannot publish**, and a **live app can be removed**

### If you don't qualify

Complete an accessibility declaration in Play Console *and* implement an in-app prominent
disclosure that lives inside the app, appears in normal use without digging through menus,
describes what data is accessed and how it is used, and requires affirmative consent.

### Play Protect

Declaring `isAccessibilityTool="true"` when the app is **not** genuinely assisting users with
disabilities triggers a Play Protect warning to users. Honest declaration matters.

---

## 7. The security reality (why this cuts both ways)

Android's own security community calls `AccessibilityService` "a single toggle to total
device control" and "a11y god mode." It is a well-documented malware vector — banking trojans
use exactly these APIs to read screens and click buttons.

Directly relevant to our design:

- The capability we want **is** the capability attackers want. There is no narrower permission.
- Once granted, the agent can read every screen — banking, passwords typed into fields, 2FA
  codes.
- Prior art on the risk surface: *"From Assistants to Adversaries: Exploring the Security
  Risks of Mobile LLM Agents"* (arXiv 2505.12981).

**Implication:** the safety requirements in CLAUDE.md are *more* urgent here, not less.
Narrate before acting, hard-stop on financial and irreversible actions, and be deliberate
about what leaves the device.

Additional consideration now that third-party code is in play: **the Portal APK and OpenDroid
would both hold full screen-read authority.** Auditing what they transmit is not optional.
The Portal is documented as able to operate fully offline "unless explicitly configured" —
verify that claim rather than trusting it.

---

## 8. Android's existing accessibility for blind users

### TalkBack
Android's built-in screen reader. Explore-by-touch, gesture navigation — functionally
analogous to VoiceOver.

**Android also permits alternative screen readers** (Commentary, Jieshuo, others). iOS has
exactly one: VoiceOver. More choice; also more fragmentation.

### TalkBack + Gemini
- **Gemini Nano on-device** for image descriptions (announced Sept 2024)
- **Gemini 1.5 Flash in the cloud** for heavier work
- Natural-language **questions about an image**, with follow-ups ("what colour?", "what material?")
- Expanded to **questions about the whole screen** — e.g. while shopping, "is there a discount available?"

**Meaningfully ahead of anything Apple ships to blind users today.** Hands-on testing (Centre
For Accessibility Australia) found Gemini's image descriptions **more detailed and accurate
than VoiceOver's**.

Caveat from that same testing: Gemini's descriptions **vary between runs** for the same
image, whereas VoiceOver's are stable. For a blind user re-checking the same screen,
inconsistency is a real usability problem, not a nitpick.

### Gemini Live
Camera + screen sharing, free for all Android users — and subsequently rolled out to iOS.
**Worth testing on her existing iPhone before building anything**; it may already deliver
part of the value.

### Voice Access
Android's Voice Control equivalent — number/label overlays and spoken taps. Same
architectural role as iOS Voice Control in the apple.md Path C hybrid.

### Gemini "autonomous task engine" (March 2026)
Reported as a core Android system integration that accesses local databases, schedules
calendars, and **clicks buttons via accessibility sandboxes**. If accurate, Google is
shipping first-party what we would be building.
*(Confidence: low — single low-quality source. Verify.)*

---

## 9. Why the research literature lives on Android

Nearly every mobile GUI-agent paper found targets Android:

> "Many AI agents use Android's accessibility framework to interact with UI elements via
> high-level APIs, supporting actions like click, scroll, and text input **without requiring
> ADB, making it suitable for production deployments**."

Papers worth reading:
- **AppAgentX: Evolving GUI Agents as Proficient Smartphone Users** (arXiv 2503.02268)
- **LLM-Powered GUI Agents in Phone Automation: Surveying Progress and Prospects** (arXiv 2504.19838) — survey, good starting point
- **TaskAudit: Detecting Functiona11ity Errors in Mobile Apps via Agentic Task Execution** (arXiv 2510.12972) — directly about accessibility failures found by agents
- **From Imperative to Declarative: Towards LLM-friendly OS Interfaces** (arXiv 2510.04607)
- **From Assistants to Adversaries: Security Risks of Mobile LLM Agents** (arXiv 2505.12981)

The common vision pipeline (screenshot → OmniParser labels interactive elements with bounding
boxes → annotated image to the LLM) is **unnecessary here.** The real accessibility tree gives
exact labels and exact coordinates — strictly better input than parsed pixels. DroidRun makes
this explicit: accessibility tree, "no screenshots required," which is why it's fast.

---

## 10. Android vs iOS — direct comparison

| | Android | iOS |
|---|---|---|
| Read other apps' UI tree | ✅ `AccessibilityService` | ❌ no API |
| Inject taps / gestures | ✅ `dispatchGesture()`, `ACTION_CLICK` | ❌ no API |
| Live screen-change events | ✅ `AccessibilityEvent` | ❌ |
| Needs tethered computer | ❌ no | ✅ yes (WebDriverAgent) |
| Certificate rotation | ❌ none | ✅ 7 days free / 1 year paid |
| Survives reboot unattended | ✅ yes | ❌ runner must be relaunched |
| Ready-made open-source bridge | ✅ Portal APK, OpenDroid | ❌ |
| Shippable to an app store | ✅ with review | ❌ |
| Screen reader | TalkBack (+ alternatives) | VoiceOver (only) |
| Screen reader consistency | Varies by OEM/device | Uniform |
| First-party AI screen Q&A | ✅ Gemini in TalkBack | ⚠️ Siri onscreen awareness, just arriving |
| Malware exposure of the API | High — known trojan vector | N/A (API doesn't exist) |

---

## 11. The honest assessment

**If the goal is "my mother uses her phone by voice," Android is the correct platform.** It is
not close. The capability is first-class, permanent, untethered, and legal — and largely
pre-built.

**If the goal is "my mother uses her *iPhone* by voice," Android is irrelevant** and
apple.md Path A is the answer. That path was rejected on 2026-09-08 because it requires a
permanent Mac tether.

The remaining cost is human, not technical:

- She has years of VoiceOver muscle memory. A blind user relearning a screen reader is a
  **months-long** cost, not a weekend.
- Her existing ecosystem — contacts, messages, family, photos, her apps — is on iOS.
- TalkBack quality varies by device, so hardware choice matters more than it does on iPhone.
  (Pixel is the safe answer — Google's own.)
- Counterweight: Gemini-in-TalkBack is genuinely better *today* at describing screens than
  anything Apple offers her.

### The middle path worth considering

A **second device**. Keep her iPhone as the primary phone she knows. Add a cheap Android as
the "agent device" for specific hard tasks. Avoids the relearning cost, gets the capability,
costs less than a Mac mini.

Weak point: her accounts, data, and apps live on the iPhone. Needs thought about whether the
tasks that are hardest for her are even reachable from a second device.

---

## 12. Open questions

### Answered

- [x] **Do we need to build a fully native Android app?**
  Something native must exist — `AccessibilityService` is Kotlin/Java only, and RN/Flutter
  buy nothing. But **MobileRun Portal** is a drop-in APK serving the UI tree over HTTP with
  no ADB, and **OpenDroid** already implements the entire agent including voice. Writing our
  own thin service is a later optimization, not a prerequisite. (§3)
- [x] **Can the Android path avoid a tethered computer?** Yes — Portal's on-device HTTP/WS
  servers and reverse-WebSocket mode, or OpenDroid running fully on-device. (§3)

### Answered by the hands-on setup (§4)

- [x] **Can we develop without a data cable?** Yes, entirely. Android 11+ wireless debugging
      needs no initial USB. Paired and connected over Wi-Fi; UI tree and screenshots verified.
- [x] **Is the randomized-port problem real?** Not once paired — adb's stable mDNS device
      name survives reboots and port changes. Confirmed responding.
- [x] **Do we need an accessibility service just to read the tree?** No. `uiautomator dump`
      returned a full tree with `enabled_accessibility_services = null`. Tree-quality testing
      needs nothing installed.

### Open — blocking

- [ ] **Is the UI tree good enough on the specific apps she actually uses?** The one remaining
      unknown, and now testable immediately via `uiautomator dump` — no APK, no cable, no
      code. **Blocked only on enumerating her real daily tasks.**
- [ ] Which **foreground service type** keeps the agent alive indefinitely on **Android 16 /
      SDK 36**, without being time-boxed — and what does **Samsung One UI** do to it? (§5)
- [ ] Does the **Compose-dialog accessibility gap** affect any of her required apps?

### Open — evaluation

- [ ] Audit what the **Portal APK** actually transmits. It claims fully-offline operation
      "unless explicitly configured" — verify, don't trust. (Lower priority now that tree
      testing needs no APK.)
- [ ] Audit **OpenDroid**'s real maturity. README says production-ready; author says early
      alpha. Which is true?
- [ ] Test **Gemini Live screen sharing on her existing iPhone** against her real tasks —
      possible meaningful value today with zero engineering.
- [ ] How good is TalkBack on the specific device we'd choose?
- [ ] Two-device split: which of her hard tasks are reachable from a second phone at all?
- [ ] Is there prior art — an LLM agent built specifically **for blind users**, as opposed to
      for QA automation? (Everything found so far is automation-first.)
- [ ] Verify the March 2026 "Gemini autonomous task engine" claim against a primary Google
      source.
- [ ] Realistic timeline and rejection risk for `isAccessibilityTool` review, if this ever
      goes beyond her.

---

## Sources

### Platform APIs
- [Create an accessibility service — Android Developers](https://developer.android.com/guide/topics/ui/accessibility/service)
- [Developing an Accessibility Service for Android — Google Codelabs](https://codelabs.developers.google.com/codelabs/developing-android-a11y-service)
- [AccessibilityServiceInfo.FlagRetrieveInteractiveWindows — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/android.accessibilityservices.accessibilityserviceinfo.flagretrieveinteractivewindows?view=net-android-35.0)
- [AccessibilityService.DispatchGesture — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/android.accessibilityservices.accessibilityservice.dispatchgesture?view=net-android-35.0)
- [Restrictions on starting a foreground service from the background — Android Developers](https://developer.android.com/develop/background-work/services/fgs/restrictions-bg-start)
- [Background optimization — Android Developers](https://developer.android.com/topic/performance/background-optimization)
- [Android Foreground Services: Types, Permissions and Limitations — Softices](https://softices.com/blogs/android-foreground-services-types-permissions-use-cases-limitations)
- [Building an Android service that never stops running — Roberto Huertas](https://robertohuertas.com/2019/06/29/android_foreground_services/)

### Existing implementations
- [droidrun/mobilerun-portal — GitHub](https://github.com/droidrun/mobilerun-portal)
- [droidrun/mobilerun — GitHub](https://github.com/droidrun/mobilerun)
- [MobileRun framework quickstart](https://docs.mobilerun.ai/framework/quickstart)
- [droidrun/mobile-harness — GitHub](https://github.com/droidrun/mobile-harness)
- [chukfinley/droidrun-mcp-server — GitHub](https://github.com/chukfinley/droidrun-mcp-server)
- [yashab-cyber/opendroid — GitHub](https://github.com/yashab-cyber/opendroid)
- [OpenDroid launch post — author, "early alpha"](https://www.threads.com/@yashabcyber/post/Db8D0YoE45d/introducing-open-droid-an-open-source-autonomous-ai-agent-for-android-not-a/)
- [ghost-in-the-droid/android-agent — GitHub](https://github.com/ghost-in-the-droid/android-agent)
- [minhalvp/android-mcp-server — GitHub](https://github.com/minhalvp/android-mcp-server)
- [Termux ADB wireless debugging gist — kairusds](https://gist.github.com/kairusds/1d4e32d3cf0d6ca44dc126c1a383a48d)
- [Android Debug Bridge (adb) — Android Developers](https://developer.android.com/tools/adb)
- [ADB over Wi-Fi: fixing changing ports — tutorialpedia](https://www.tutorialpedia.org/blog/adb-over-wi-fi-android-11-on-windows-how-to-keep-a-fixed-port-or-connect-automatically/)
- [Wireless debugging keeps turning off (Android 12) — XDA](https://xdaforums.com/t/android-12-developer-options-adb-wireless-debugging-option-keeps-turning-off.4461375/)
- [Termux ADB guide — XDA Forums](https://xdaforums.com/t/termux-adb-running-adb-within-android-using-termux-wireless-adb.4724780/)
- [AutoInput — Google Play](https://play.google.com/store/apps/details?id=com.joaomgcd.autoinput)

### Cross-platform framework bridging
- [Creating a Native Module in React Native — Medium](https://medium.com/@mohamed.ma872/creating-a-native-module-in-react-native-df1a694c89a7)
- [How Flutter calls native code (Kotlin/Java/Swift) — Medium](https://medium.com/@Abhiramtsabu/bridging-the-gap-how-flutter-calls-native-code-kotlin-java-swift-e696541ab92a)

### Play Store policy
- [Use of the AccessibilityService API — Play Console Help](https://support.google.com/googleplay/android-developer/answer/10964491?hl=en)
- [Permissions and APIs that Access Sensitive Information — Play Console Help](https://support.google.com/googleplay/android-developer/answer/16558241?hl=en)
- [Developer Guidance for Google Play Protect Warnings](https://developers.google.com/android/play-protect/warning-dev-guidance)
- [How to Get Android Restricted Permissions Approved on Google Play — Newly](https://newly.app/how-to/android-restricted-permissions)

### Security
- [Android's AccessibilityService: A Single Toggle to Total Device Control — Chocapikk](https://chocapikk.com/posts/2026/android-a11y-god-mode/)
- [Android Accessibility Service: The Unexplored Goldmine — Mihir Patel](https://mhrpatel12.medium.com/android-accessibility-service-the-unexplored-goldmine-d336b0f33e30)

### Accessibility for blind users
- [TalkBack uses Gemini Nano to increase image accessibility — Android Developers Blog](https://android-developers.googleblog.com/2024/09/talkback-uses-gemini-nano-to-increase-low-vision-accessibility.html)
- [New AI and accessibility updates across Android, Chrome and more — Google Blog](https://blog.google/company-news/outreach-and-initiatives/accessibility/android-gemini-ai-gaad-2025/)
- [Android accessibility updates: dark theme, Gemini in TalkBack — Google Blog](https://blog.google/products-and-platforms/platforms/android/accessibility-update-expanded-dark-theme-gemini-talkback/)
- [What's new with TalkBack 16.0 — Android Accessibility Help](https://support.google.com/accessibility/android/answer/16294093?hl=en)
- [Hands-on: Android 15 Gemini vs. iPhone's VoiceOver — Centre For Accessibility Australia](https://www.accessibility.org.au/hands-on-android-15-ai-gemini-integration-for-alternative-text-vs-iphones-voiceover/)
- [Gemini Live free for everyone, screen sharing on iPhone — TechRadar](https://www.techradar.com/computing/artificial-intelligence/gemini-live-is-now-free-for-everyone-on-android-and-ios-and-you-can-finally-share-your-screen-and-camera-on-iphone)
- [Gemini video and screen sharing — AppleVis forum](https://www.applevis.com/forum/ios-ipados/gemini-video-screen-sharing)
- [Android Accessibility Settings — Consumer Reports](https://www.consumerreports.org/adaptive-living-aging-accessibility/android-accessibility-settings-and-features-a3469083425/)

### Research papers
- [AppAgentX: Evolving GUI Agents as Proficient Smartphone Users — arXiv 2503.02268](https://arxiv.org/pdf/2503.02268)
- [LLM-Powered GUI Agents in Phone Automation: Surveying Progress and Prospects — arXiv 2504.19838](https://arxiv.org/pdf/2504.19838)
- [TaskAudit: Detecting Functiona11ity Errors in Mobile Apps — arXiv 2510.12972](https://arxiv.org/pdf/2510.12972)
- [From Assistants to Adversaries: Security Risks of Mobile LLM Agents — arXiv 2505.12981](https://arxiv.org/pdf/2505.12981)
- [From Imperative to Declarative: LLM-friendly OS Interfaces — arXiv 2510.04607](https://arxiv.org/pdf/2510.04607)
- [Gemini Autonomous Task Engine — Vucense](https://vucense.com/ai-intelligence/agentic-ai/gemini-autonomous-task-engine-android-2026/)
