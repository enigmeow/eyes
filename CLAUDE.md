# eyes

Giving Claude the ability to read and operate a phone screen, so a blind user can
drive their apps entirely by voice.

**Who this is for:** my mother. She is blind, uses an iPhone, and is a fluent
VoiceOver user. She is the actual user, not a hypothetical one — design decisions
should be checked against what she can verify by ear.

**Status:** research phase, no code yet. **Platform decision made 2026-09-08: Android.**
**Dev environment verified 2026-09-08** — phone paired over Wi-Fi, UI tree readable. See
android.md §4.

---

## Research

Read these before proposing an approach. They are the accumulated findings, with
sources and confidence levels marked.

- **[apple.md](./apple.md)** — iOS. Deprioritized, kept for durable constraints and as a
  fallback. Why an App Store app can't do this, what Apple already ships that we shouldn't
  rebuild, and the four candidate paths (WebDriverAgent, iPhone Mirroring, ReplayKit,
  App Intents/Shortcuts).
- **[android.md](./android.md)** — Android. **Primary direction.** `AccessibilityService`
  grants exactly what iOS forbids, untethered and persistent. Includes the build-options
  analysis (§3) and the keep-it-alive constraints (§4).

---

## The state of play, in short

iOS has **no API** to read another app's UI tree or inject a tap — an absent API, not a
policy hurdle. The only iOS path with real capability (WebDriverAgent) needs a permanently
tethered Mac plus certificate rotation. **Rejected 2026-09-08: unacceptable for a phone she
carries.**

Android's `AccessibilityService` grants full cross-app screen reading and gesture injection
with one user toggle. No tether, no expiry, and enablement survives reboot.

**We likely don't need to write an Android app either.** `AccessibilityService` must be
native Kotlin (RN/Flutter buy nothing), but two open-source implementations already exist:

- **MobileRun Portal** (`droidrun/mobilerun-portal`) — drop-in APK serving the UI tree over
  HTTP :8080 / WS :8081 **on-device, no ADB**, plus a reverse-WebSocket mode that dials out
  (no inbound ports, works behind CGNAT).
- **OpenDroid** (`yashab-cyber/opendroid`) — Apache 2.0, fully on-device, Claude support,
  wake word + STT + TTS already built. README says "production-ready"; the author's own
  launch post says "early alpha." **Audit before trusting it.**

## Device — connecting to the test phone

Samsung Galaxy S25 Ultra (SM-S938U1), **Android 16 / SDK 36**. Paired over Wi-Fi, **no cable
required or used**. Use the stable mDNS name as the handle — it survives reboots and adb's
randomized ports:

```
adb -s adb-R5CXC3JDSYR-YBS99B._adb-tls-connect._tcp shell ...
```

If it is not connected: on the phone, Settings → Developer options → Wireless debugging → ON,
then `adb connect <ip>:<port>` using the port from that screen. Re-pair only if that fails.

**The phone must not be on a VPN** — it then reports a tunnel address (e.g. `10.5.0.2`) that
is unreachable for pairing. A VPN on the *Mac* is fine.

Reading the screen needs **nothing installed** — no APK, no accessibility service:

```
adb -s <device> shell uiautomator dump /sdcard/ui.xml && adb -s <device> pull /sdcard/ui.xml
adb -s <device> exec-out screencap -p > shot.png
```

## ⚠️ The blocking question

**Is the UI tree good enough on the specific apps she actually uses?**

Everything else is settled: the platform, the plumbing, and the fact that no app needs writing
to find out. This is the one real unknown, and it **cannot be answered without her actual task
list.**

**Sequence:**
1. **Enumerate her real daily tasks.** Blocks everything. Nothing can be evaluated without it.
2. **Test Gemini Live screen sharing on her existing iPhone** against those tasks. Zero
   engineering, on the phone she already knows. May cover part of the need outright.
3. **Set up Voice Control + VoiceOver** properly with custom commands — establishes the true
   baseline and shows which tasks genuinely need an agent.
4. **`uiautomator dump` those apps on the test phone** and judge tree quality honestly.
5. Only then: Portal APK, or the thin Kotlin service.

Do not skip 1-3 to get to the fun part. They are cheap and they set the requirements.

## Non-negotiable design constraints

These shape any code written here and should not be deferred to "later":

- **Narrate before acting.** She cannot visually verify what the agent did. Every
  action is announced before it is committed.
- **Hard-stop on anything financial or irreversible.** No exceptions, no
  configuration flag to disable it.
- **Authentication stays in her hands.** Face ID, passwords, and payment
  confirmations are never automated. WebDriverAgent enforces this anyway — treat it
  as a feature, not a limitation.
- **The capability we want is the capability attackers want.** Full screen read plus
  tap injection is the same surface banking trojans use — on Android this is a
  well-documented malware vector, not a hypothetical. Be deliberate about what leaves
  the device, and audit any third-party APK that holds this authority.
- **Silence is the worst failure mode.** She cannot see that nothing is happening. Android
  will kill a naive background service; foreground-service type and battery-optimization
  exemption are correctness requirements, not polish. See android.md §5.

---

## Working notes

- Research docs carry explicit confidence markers. Anything sourced from a single
  low-quality page is labelled as such — verify before relying on it.
- Both docs end with an open-questions checklist. Update them as questions get
  answered rather than starting new documents.
