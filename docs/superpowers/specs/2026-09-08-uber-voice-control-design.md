# Design: Voice-Controlled Uber

**Date:** 2026-09-08
**Status:** Approved design, not yet implemented
**Project:** [eyes](../../../CLAUDE.md) — voice control of a phone for a blind user

---

## 1. Goal

Make Uber fully operable by voice, with no visual interaction, on Android.

The user presses a button, says where they want to go, hears the ride read back,
confirms by voice, and is then told — without asking — where the car is, when it
arrives, and what it looks like.

This is the first vertical slice of the wider project. Uber is the target because it
is a high-value, self-contained task with a clear success condition: the user gets in
the right car.

### Target user

A blind iPhone user (the author's mother), fluent in VoiceOver. **This milestone runs
on the author's own Galaxy S25 as a proving ground**, not on her device. Design
decisions are made for her; validation happens on his hardware.

### Definition of done for v1

From an unlocked, pocketed phone: press the button, speak a destination, hear the ride
read back, say yes, and be told when the car arrives and what it looks like — without
looking at or touching the screen at any point.

*Operation from the lock screen is desirable but unverified — see §10. v1 assumes the
phone is unlocked.*

---

## 2. Constraints discovered during research

These are findings, not assumptions. They shaped the design and should be re-verified
if they appear to change.

### Uber's API cannot do this

All Uber Rider API endpoints require Business Development approval; there is no
self-serve tier. More decisively, the documented endpoints are limited to
`POST /requests`, `/estimates/price`, `/estimates/time`, `/products`, `/me`, and
`/payment-methods`.

**There are no documented endpoints for ride status, ETA, driver details, or vehicle
information.** Those three things are the core of this feature. No access tier
provides them. Status must therefore come from the device.

### Uber deep links work and are the right booking mechanism

```
uber://riderequest?pickup=my_location
  &dropoff[latitude]=<lat>&dropoff[longitude]=<lng>
  &dropoff[formatted_address]=<address>
```

Also available as a universal link (`https://m.uber.com/looking?...`), which Uber
recommends over the custom scheme.

Deep links **pre-fill but do not book** — a human must confirm. This is treated as a
feature: Uber itself enforces the confirmation gate.

*Open item:* the docs list `client_id` as required. Whether one can be obtained
without BD approval is untested. See §10.

### Uber's UI is drivable, and the good screens are the ones that matter

Verified by dumping real trees on 2026-09-08:

| | Home screen | Booking flow |
|---|---|---|
| resource-ids | 0 | 39 |
| WebViews | 0 | 0 |
| Compose views | 5 | 0 |
| Labelled clickables | 25/25 (via descendants) | 16/17 |

The booking flow uses native widgets with stable semantic ids
(`ub__location_edit_search_pickup_view`, `com.ubercab:id/edit_text`,
`pill_button_guest`). The destination field arrives `focused=true`.

**Lesson worth carrying to other apps: judge an app by its functional screens, not its
home screen.** The prediction that the home screen would be the easy case was wrong.

### Notifications are readable

`NotificationListenerService` reads any app's notifications with one user permission
grant. Uber actively posts notifications (observed:
`location_foreground_service_uber_feature_channels`).

Ride-status notification content is **unverified** — it requires a ride in progress.
See §8.

### The trigger has a supported path

`accessibility_shortcut_target_service` is unbound on the target device. Android's
accessibility shortcut can target any installed accessibility service, including
third-party ones; Samsung adds a "Side and Volume up buttons" variant.

*Open item:* whether the shortcut delivers a press event or merely toggles the service
on and off. See §10.

### On-device speech is good enough

`createOnDeviceSpeechRecognizer()` is available (Android 13+; device runs Android 16 /
SDK 36). Gemini Nano via AICore is supported on the Galaxy S25. On-device word error
rate is 6–12% on clean speech.

Critically, **v1's grammar is closed**, so recognition accuracy is not the binding
constraint — see §4.

### uiautomator cannot see the keyboard

The IME is a separate window. `uiautomator dump` returns only the focused app window —
a full keyboard on screen produced zero keyboard nodes. This does not affect v1 (we
never touch the keyboard), but it is why the production path uses
`AccessibilityService` with `FLAG_RETRIEVE_INTERACTIVE_WINDOWS` rather than adb.

---

## 3. Approach

**Drive Uber structurally where possible; scrape only where nothing else exists.**

```
volume keys ─▶ AccessibilityService wakes
   │
   ├─ STT ─▶ "ride to my daughter's place"
   ├─ fuzzy match ─▶ named place ─▶ {lat, lng, address}
   ├─ deep link ─▶ Uber opens pre-filled
   ├─ read tree ─▶ price / ETA / product ─▶ speak readback ─▶ await "yes"
   ├─ performAction(ACTION_CLICK) on Confirm
   └─ monitor: NotificationListener (events) + tree (details) ─▶ narrate
```

### Approaches considered and rejected

**Pure accessibility agent** — drive every screen by tree, including typing the
destination and selecting from search results. More general, but reintroduces the
fragile typing-and-disambiguation step that deep links eliminate. Address selection by
voice from a results list is precisely where a wrong tap sends her to the wrong city.

**Notification-only** — deep link to book, notifications for status, never read the
tree. Simplest and most redesign-proof, but limited to what Uber chooses to put in a
notification. Vehicle description is a stated requirement and would likely be lost.

### Why the chosen approach

Each step uses the most structural mechanism available. Deep links cannot break from a
UI redesign. Notifications are stable text. Only the readback and vehicle-detail
extraction depend on the tree, and those degrade gracefully — a broken selector costs
detail, not the ride.

---

## 4. No LLM in v1

v1 calls no language model, local or remote.

The grammar is closed:

```
"ride to <place>"     place ∈ ~20 pre-registered named places
"yes" / "no"
"where is my ride"
"cancel"
```

With a closed vocabulary the system **matches rather than transcribes**. `"my daughters
place"`, `"my daughter's place"`, and `"my daughters plays"` all fuzzy-match one entry.
A 10% word error rate against free text becomes near-zero intent error against twenty
known phrases.

Consequences: no network dependency, no API cost, no added latency, works offline,
nothing to fail when Wi-Fi drops. Relevant benchmark framing from the research: *a 95%
accurate response in 800ms loses to a 91% accurate response in 280ms.*

**Deferred to v2, as escalation and never as a dependency:** free-form addresses
outside the named list; interpreting an unexpected Uber screen; conversational repair
("no, the other doctor"). Interfaces should leave room for these; v1 must not call out
to anything.

**Risk:** Whisper measurably outperforms Google's recognizer on non-native and atypical
speech. If the eventual user's speech differs from the recognizer's training
distribution, on-device Google may underperform. Testable at M2 with her actual voice,
before anything is built on top.

---

## 5. Components

```
┌─────────────────────────────────────────────────────┐
│  Orchestrator          state machine, owns the flow │
└───┬───────┬───────┬───────┬───────┬─────────────────┘
    │       │       │       │       │
┌───▼──┐ ┌──▼───┐ ┌─▼────┐ ┌▼─────┐ ┌▼──────────┐
│Trigger│ │Voice │ │Place │ │Uber  │ │RideMonitor│
│       │ │ IO   │ │Resolv│ │Driver│ │           │
└───────┘ └──────┘ └──────┘ └──┬───┘ └─┬────────┘
                               │       │
                          ┌────▼───┐ ┌─▼──────────────┐
                          │DeepLink│ │Notification    │
                          │+ Tree  │ │Listener + Tree │
                          └────────┘ └────────────────┘
```

| Component | Responsibility | Depends on |
|---|---|---|
| **Trigger** | `AccessibilityService`. Wakes the agent on button press, and **owns the accessibility connection** — it is the only component that holds it. All tree reads and node actions go through it. | Android accessibility framework |
| **VoiceIO** | STT in, TTS out, audio-focus arbitration. Isolated because it is most likely to be swapped, and must coexist with TalkBack. | `SpeechRecognizer`, `TextToSpeech` |
| **PlaceResolver** | Spoken destination → `{lat, lng, formatted_address}`. Named places first, `Geocoder` fallback. Not a general address search in v1. | Local place store, `Geocoder` |
| **UberDriver** | Build and fire the deep link; read facts from Uber's tree and click Confirm **via Trigger's connection**. **All selectors live in one file.** | Trigger, Android Intents |
| **RideMonitor** | Merge notification events and tree reads into one `RideState` stream. | `NotificationListenerService`, Trigger |
| **Narrator** | Decide what is spoken and when. Dedupe, rate-limit, suppress repeats. | VoiceIO |
| **Orchestrator** | The state machine. The only component that knows the whole flow. | all of the above |

`Narrator` is separate from `VoiceIO` because *what to say* and *how to say it* change
for different reasons. `UberDriver`'s selectors are isolated in one file so that an
Uber redesign breaks exactly one place, and breaks it as a failing test.

---

## 6. Data flow

### State machine

```
IDLE ──button──▶ LISTENING ──speech──▶ RESOLVING ──place──▶ LAUNCHING
                     ▲                     │                    │
                     └─────unclear─────────┘                    ▼
                                                            READBACK
                                                          ┌────┴────┐
                                              "no"/unclear│         │"yes"
                                                    ▲     │         ▼
                                                    └─────┘    CONFIRMING
                                                                    │
                                                                    ▼
  COMPLETE ◀──────────────────────────────────────────────── MONITORING
```

`CANCELLED` and `FAILED` are reachable from any state.

**Safety is structural, not procedural.** There is no edge from `RESOLVING` or
`LAUNCHING` to `CONFIRMING`. The only path to booking passes through `READBACK` and an
explicit affirmative. Anything unrecognised returns to `READBACK` and re-reads. This is
enforced by the state machine and asserted by unit tests, not by remembering to check.

### Happy path

1. **Button** → audio tone (not speech — she must know it is listening before words begin)
2. **STT** → `"get me a ride to my daughter's place"`
3. **Fuzzy match** → `{intent: REQUEST_RIDE, place: "daughter"}`
4. **PlaceResolver** → `{41.88, -87.63, "222 S Riverside Plz, Chicago, IL"}`
5. **Deep link** → Uber opens pre-filled
6. **UberDriver** waits for the confirm screen; reads price, product, ETA
7. **Readback** → *"UberX to 222 South Riverside Plaza. Twenty-four dollars. Six minutes away. Say yes to book."*
8. **"yes"** → `performAction(ACTION_CLICK)` on the confirm node
9. **MONITORING** begins

Steps 1–2 and 7–8 are the only points of interaction. Everything else is silent.

### RideState

```kotlin
data class RideState(
  val phase: Phase,          // SEARCHING, DRIVER_ASSIGNED, EN_ROUTE,
                             // ARRIVED, IN_TRIP, COMPLETE, CANCELLED
  val etaMinutes: Int?,
  val driverName: String?,
  val vehicle: Vehicle?,     // make, model, color, plate
  val etaLastChanged: Instant?
)
```

**Merge policy.** Notifications are authoritative for `phase` — they arrive with the
phone pocketed and are stable text. The tree is authoritative for `vehicle` and refines
`etaMinutes`, but requires Uber foregrounded (it will be, after the deep link).
Whichever source fires, merged state is diffed against the previous state and only the
diff reaches the `Narrator`.

### "How far away, and if it has stopped"

Two distinct things, both built:

- **Arrival** — `phase == ARRIVED`. The car is at the curb. Gets a distinct announcement.
- **Stalled en route** — `etaMinutes` has not decreased in ~3 minutes. Derived, not
  reported by Uber. This is the situation where a sighted person would glance at the map
  and see traffic.

### Narration

Event-driven, never periodic.

| Event | Spoken |
|---|---|
| Driver assigned | *"Booked. Silver Toyota Camry, plate A-B-C 1-2-3-4. Marcus. Six minutes away."* |
| ETA crosses ~2 min | *"Your ride is about two minutes away."* |
| Arrived | *"Your ride is here. Silver Toyota Camry, plate A-B-C 1-2-3-4."* |
| Stalled | *"Your driver hasn't moved for a few minutes. Still five minutes away."* |
| Cancelled | *"Your driver cancelled. Say 'find another' to rebook."* |

Three rules the `Narrator` enforces:

1. **Never speak the same fact twice** unless the underlying value changed.
2. **Plate numbers are always spelled out.** `ABC1234` heard once as a word is useless.
3. **The vehicle description repeats on arrival**, even though it was given at booking.
   That is the moment it is needed, and she should not have to hold it in memory for six
   minutes.

The user may press the button at any time and ask *"where is my ride?"* — same state,
read on demand.

---

## 7. Failure handling

**The governing rule: silence is the worst outcome.** She cannot see a spinner, a
greyed-out button, or an error toast.

| Failure | Behaviour |
|---|---|
| No speech detected | Tone, then *"I didn't hear anything."* Return to IDLE. Never hang open. |
| Place not recognised | *"I don't know where that is. You can say home, doctor, or daughter."* — enumerate the actual list |
| Uber not installed or won't open | *"I can't open Uber."* Stop. Do not retry silently. |
| Confirm button not found | **Abort. Tap nothing.** *"Something looks different in Uber. I've stopped — please check the screen or ask someone."* A missing selector must never degrade into a guessed tap. |
| Readback missing price or ETA | Read what is available and name what is missing: *"I couldn't read the price."* Never omit silently — a missing number sounds like a normal readback. |
| Ambiguous confirmation | Return to `READBACK` and re-read. There is no "assume yes". |
| Monitoring silent > 2 min | *"I've lost track of your ride."* Better to admit it than let her stand at a curb assuming she will be told. |
| Service killed by the OS | Heartbeat; on restart, *"I stopped watching your ride — checking again."* |

The two most important are **selector-not-found** and **monitoring-went-quiet**, because
both have a tempting silent failure mode, and silence is exactly what she cannot detect.

### TalkBack coexistence

Not present on the development device (TalkBack installed but disabled). It will be
enabled on hers.

- **Competing TTS.** Ours must take audio focus and duck TalkBack, never talk over it.
- **`dispatchGesture()` behaves differently under explore-by-touch.** `UberDriver` must
  use `performAction(ACTION_CLICK)` on nodes, which is unaffected. Coordinate tapping is
  a last resort. TalkBack makes the semantic-first design mandatory rather than merely
  preferred.

---

## 8. Testing

### The core problem

Everything after Confirm is observable only during a real, paid ride. There is no
sandbox and no API. The arrival flow — the part that matters most — is the last thirty
seconds of a ride you waited ten minutes for.

### The fix: capture fixtures first

Book one real ride with adb in record mode, dumping the accessibility tree and every
notification every few seconds from request to completion. That single ride yields a
replay corpus covering driver assignment, ETA ticks, arrival, trip, and completion.

`UberDriver`, `RideMonitor`, and `Narrator` then develop and test against recordings —
offline, free, at any hour. **One fare buys the whole development cycle.** This requires
no app code and can be done with the existing adb connection.

### Per component

| Component | Method | Rationale |
|---|---|---|
| `PlaceResolver` | Unit, table-driven | Feed real misrecognitions → assert correct place. This is where the closed-grammar bet is verified. |
| `Narrator` | Unit, event sequences | Given a `RideState` stream, assert exact utterances. Catches double-speaking and missed arrivals. |
| State machine | Unit | Assert illegal transitions are **rejected**, especially any path to `CONFIRMING` that skips readback. |
| `UberDriver` | Instrumented, against fixtures | Uber redesigns surface as failing tests, not as a wrong tap mid-ride. |
| `RideMonitor` | Unit, replay | Recorded notification + tree streams → assert merged `RideState`. |
| End-to-end | Manual, real rides | Only at milestone boundaries. Budget for them. |

TDD throughout: tests before implementation.

---

## 9. Milestones

| # | Milestone | Deliverable | Cost |
|---|---|---|---|
| **M0** | Capture fixtures | Replay corpus from one real ride. No app code. De-risks the largest unknown. | 1 fare |
| **M1** | Trigger | Button press produces a log line. Settles the mechanism question in §10. | — |
| **M2** | Voice loop | Say *"ride to the doctor"*, hear it read back. **Test with her voice here**, before building on top. | — |
| **M3** | Launch | Hands-free from button to Uber's pre-filled confirm screen. | — |
| **M4** | Readback and confirm | First real booking, end to end by voice. | fares |
| **M5** | Monitoring and narration | Built against M0 fixtures, validated on one real ride. | 1 fare |
| **M6** | Survival | Foreground service type, battery exemption, Samsung One UI background killing, restart after reboot. | — |

M0–M3 cost one fare total. M4 is where real bookings begin.

M6 is last because it is tuning rather than architecture — but it is what makes the
result usable rather than a demo. See [android.md §5](../../../android.md).

---

## 10. Open questions

Blocking, to be answered at the milestone indicated:

- [ ] **M1 — How does the button reach us?** Two candidates: the accessibility shortcut
      (supported and discoverable, but may only *toggle a service* rather than deliver a
      press event), or `onKeyEvent()` with `canRequestFilterKeyEvents` (a real press
      event, heavier permission, volume-key filterability unconfirmed). Verify before
      building anything on top.
- [ ] **M0 — What do Uber's ride-status notifications actually contain?** The entire
      `RideMonitor` design assumes phase transitions are legible in notification text.
      Unverifiable without a ride in progress.
- [ ] **M0 — Does the tree expose vehicle make, colour, and plate during a ride?** If
      not, the arrival announcement loses its most important content and the design needs
      revisiting.
- [ ] **M3 — Can a `client_id` be obtained without Uber BD approval?** Deep-link docs
      list it as required. If it cannot, re-evaluate whether deep links work without one,
      or fall back to driving the destination field by tree.
- [ ] **M2 — Does on-device recognition handle her voice well?** Test with her actual
      speech before committing to the on-device recognizer.
- [ ] **M1 — Does any of this work from the lock screen?** Whether the trigger fires,
      whether STT runs, and whether Uber can be driven while locked are all unknown.
      A phone that must be unlocked first is materially worse for her, but v1 assumes it.

Non-blocking:

- [ ] Which foreground-service type survives indefinitely on Android 16 / SDK 36, and
      what Samsung One UI does to it.
- [ ] Whether trees or notifications are ever sent off-device, and what redaction is
      needed if so. Ride trees contain home-adjacent addresses in plain text.

---

## 11. Out of scope for v1

Camera-based car identification · free-form addresses outside the named list ·
multi-stop rides · scheduled rides · Uber Eats · any app other than Uber · deployment to
her phone (this milestone is the author's S25)

**In-app management of named places.** The ~20 places are pre-configured by the author
(config file or a developer-facing screen). A blind user cannot add or edit places by
voice in v1. This is a real gap for eventual independent use, but it is setup rather
than operation, and it does not block the ride flow.

Camera identification is a separate subsystem — live video, plate and colour
recognition, and guiding a blind person toward a moving vehicle. It deserves its own
design.
