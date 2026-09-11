# Cross-Platform Feasibility & Feature Matrix — verified Sept 2026
 
> Research-backed feasibility study. Every row cites the platform API and the
> verification source. **This document gates implementation**: nothing gets
> built on an assumption; if a row says unsupported/unavailable, the UI will
> say so — no fake buttons, no bypasses.
>
> Verdict legend: ✅ supported · 🔶 permission-gated (official, user-granted) ·
> 🔶-app needs companion app · ⚠️ limited · ❌ unavailable to third-party software.
>
> Distribution-policy note: Google Play / App Store policies are distinct from
> technical feasibility. This project ships as source; where a capability is
> only blocked by *store policy* (e.g., SMS without default-handler), the row
> says "policy-blocked on Play" and the capability remains available in
> sideloaded/self-built builds with explicit user disclosure.
 
## 1. Verified findings that shaped the architecture
 
### Apple (developer.apple.com, verified via doc JSON API)
 
1. **ScreenCaptureKit** captures "displays, apps, and windows that your app
   can capture" (`SCShareableContent` = `SCDisplay`/`SCRunningApplication`/
   `SCWindow`). **No iPhone/iPad device source exists** — a Mac app cannot
   capture a connected iPhone's screen. iPhone capture is Apple-only (iPhone
   Mirroring, macOS 15+, Apple Silicon — Apple-only feature, no third-party API).
2. **ReplayKit** on iOS: `RPBroadcastSampleHandler` ("An object that processes
   buffer objects as received from ReplayKit") inside a Broadcast Upload
   Extension, launched via `RPSystemBroadcastPickerView` ("A view displaying a
   broadcast button that, when tapped, shows a broadcast picker") or
   `RPBroadcastActivityViewController`. This is the **sanctioned** full-device
   capture path to a **custom destination** — how Twitch/YouTube mobile
   streaming apps work. Requires a companion iOS app.
3. **iPhone Mirroring**: Apple-only, macOS 15+/Apple Silicon Mac + iOS 18+
   iPhone; no third-party API. We surface it as the "use Apple's feature"
   recommendation rather than reimplementing it.
4. **Mac built-in Screen Sharing** (macOS Sonoma+ "High Performance" mode is
   Apple-proprietary): the VNC-compatible mode ("VNC viewers may control
   screen with password" in System Settings → Screen Sharing) is accessible
   to standard third-party RFB clients. → Our Mac↔Mac control path.
5. **WebCodecs** (W3C + caniuse): `VideoDecoder` H.264 with `format: 'annexb'`
   accepts raw Annex-B access units without a description; Safari/WKWebView
   full support since Safari 26 (partial from 16.4). macOS 26 WKWebView on
   this machine qualifies for hardware-decode, `latencyMode: 'realtime'`,
   `hardwareAcceleration: 'prefer-hardware'`.
6. **Tauri v2 IPC Channel** (`tauri::ipc::Channel`): streams binary chunks
   (`Channel<&[u8]>`) Rust→webview — the recommended mechanism for streamed
   data; no documented size/rate cap. → Our in-app H.264 delivery path,
   avoiding a loopback WS server entirely.
 
### Android (developer.android.com — full report in research log)
 
- **MediaProjection** → `MediaCodec` H.264: supported, consent-gated. API 29+
  foreground service; API 34+ `FOREGROUND_SERVICE_MEDIA_PROJECTION` + type
  declaration; **Android 14 one-consent-per-capture** (each
  `createVirtualDisplay` = new consent; reuse throws). `PARAMETER_KEY_REQUEST_SYNC_FRAME`
  + `PARAMETER_KEY_VIDEO_BITRATE` confirmed on `MediaCodec` for IDR-on-demand
  and runtime bitrate adaptation.
- **AccessibilityService `dispatchGesture`**: sanctioned input injection (API
  24+); user must enable in Settings → Accessibility (no runtime prompt).
- **NotificationListenerService**: read + dismiss + RemoteInput **replies**;
  same enablement also unlocks `MediaSessionManager.getActiveSessions`
  (media control of other apps) — one enablement, two features.
- **SMS/calls**: technically possible; **Play policy requires default-handler**
  for SMS/call-log groups → ship disabled-by-default, direct-APK disclosure.
  `ACTION_DIAL` (no permission) always fine; `ACTION_CALL` needs `CALL_PHONE`.
- **Clipboard**: Android 10+ **foreground-only reads** ("Unless your app is
  the default IME or the app that currently has focus… cannot access clipboard
  data") — we sync clipboard only while the companion is foregrounded, and say so.
- **mDNS**: `NsdManager` today; **Android 16 Local Network Protections**
  (rollout 25Q2–26Q2) will gate it behind `ACCESS_LOCAL_NETWORK` or the
  `FLAG_SHOW_PICKER` per-service consent — must be designed for now.
- **Battery, photo picker (`PickVisualMedia` — permissionless), contacts
  (`READ_CONTACTS`)**: all clear.
- **Background app-launch**: restricted since Android 10 — deep-link opens
  require foreground app state or notification tap; we can't silently launch
  arbitrary apps on the phone from the Mac (and won't pretend to).
 
### Windows (learn.microsoft.com — full report in research log)
 
- Notification listener = packaged (MSIX) apps only (`userNotificationListener`
  capability); Win32 clipboard + `AddClipboardFormatListener` fine;
  `Windows.Graphics.Capture` picker-gated fine; `SendInput` fine (UIPI-bounded);
  DNS-SD native via `DnsServiceRegister`/`DnsServiceBrowse` (windns.h) — no
  Bonjour SDK needed. Phone Link's app-streaming/cross-device graph features
  are Microsoft/OEM-privileged (documented as such in the matrix).
- **Mac-side build is this branch's target.** Windows is architecture-prepared
  (Rust core is `#[cfg]`-platformed), not built here — stated honestly.
 
## 2. Feature matrix (the required table)
 
Legend: **Dir** — A→D = Android→Desktop (desktop views/controls Android),
D→A reverse, D↔D = desktop↔desktop (Mac↔Mac). **Status** uses the five-state
capability model of PROTOCOL §11 (`supported | needs-permission | needs-
companion-app | unsupported-platform | unavailable`).
 
### Android ↔ Mac (Phone Link parity — companion app required on Android)
 
| Platform | Feature | Dir | API / Technology | Permission | Status | Limitation |
|---|---|---|---|---|---|---|
| Android↔Mac | Discovery/pairing | both | `NsdManager` + mDNS `_crosslink._tcp` + pinned mTLS | Local network (Android 16+) | 🔶-app | multicast blocked on some APs → QR/manual-IP fallback |
| Android↔Mac | Persistent sessions | both | mTLS WebSocket + heartbeat/backoff | — | 🔶-app | — |
| Android↔Mac | Notifications mirror | A→D | `NotificationListenerService` | Notif access (user-enabled) | 🔶-app + 🔶 | restricted-settings gate for sideloaded APKs |
| Android↔Mac | Notification replies | D→A | `RemoteInput.addResultsToIntent` + action `PendingIntent` | via listener | 🔶-app + 🔶 | apps that post no RemoteInput actions can't be replied to |
| Android↔Mac | Media control + now-playing | both | `MediaSessionManager.getActiveSessions(listener)` / `MediaController` | via listener | 🔶-app + 🔶 | one enablement unlocks notifications+media |
| Android↔Mac | File transfer | both | chunked mTLS WS + SHA-256 | — | 🔶-app | — |
| Android↔Mac | Clipboard | both | `ClipboardManager` | focus-gated (Android 10+) | 🔶-app | **foreground-only on Android side**; Mac side full |
| Android↔Mac | Photos | A→D | Photo Picker `PickVisualMedia` | permissionless | 🔶-app | user-selected items only |
| Android↔Mac | Battery/device status | A→D | `BatteryManager` | none | 🔶-app | — |
| Android↔Mac | Screen viewing | A→D | `MediaProjection` → `MediaCodec` H.264 | per-capture consent | 🔶-app + 🔶 | Android 14: consent per capture; FGS type API 34+ |
| Android↔Mac | Screen control (touch) | D→A | `AccessibilityService.dispatchGesture` | Accessibility (user-enabled) | 🔶-app + 🔶 | sanctioned injection path; needs explicit enablement |
| Android↔Mac | Screen control (text) | D→A | `ACTION_SET_TEXT` on focused nodes / gestures | via accessibility | 🔶-app + 🔶 | IME-less text entry; app-dependent reliability |
| Android↔Mac | SMS/MMS | both | `SmsManager`, `RECEIVE_SMS` receiver | SMS perms + Play default-handler policy | ⚠️ policy-blocked on Play | works sideloaded; off by default, explicit opt-in |
| Android↔Mac | Calls (dial) | D→A | `ACTION_DIAL` (no perm) / `ACTION_CALL` (`CALL_PHONE`) | CALL_PHONE for direct dial | 🔶-app | dialer-UI path is permissionless |
| Android↔Mac | Call state | A→D | `TelephonyCallback` | `READ_PHONE_STATE` | ⚠️ Play-restricted | not in default-handler group exception; optional |
| Android↔Mac | Contacts | A→D | `ContactsContract` | `READ_CONTACTS` | 🔶-app | — |
| Android↔Mac | App launch/deep-link | D→A | `startActivity` from foregrounded app | BAL restrictions | ⚠️ | works only when companion is foregrounded; otherwise notification tap |
| Android↔Mac | Multi-device | both | per-peer sessions (up to 8) | — | 🔶-app | — |
| Android↔Mac | Adaptive quality | both | `PARAMETER_KEY_VIDEO_BITRATE`, fps ladder | — | 🔶-app | — |
 
### iPhone/iPad ↔ Mac (Apple investigation — companion iOS app needed for anything beyond Apple-only features)
 
| Platform | Feature | Dir | API / Technology | Permission | Status | Limitation |
|---|---|---|---|---|---|---|
| iPhone/iPad↔Mac | Screen viewing (phone→Mac) | iPhone→Mac | ReplayKit Broadcast Upload Extension (`RPBroadcastSampleHandler`) + our mTLS WS | ReplayKit consent per broadcast | 🔶-app + 🔶 | the ONLY sanctioned third-party full-device capture; user starts it via `RPSystemBroadcastPickerView` button in companion iOS app |
| iPhone/iPad↔Mac | Screen viewing (Mac→phone) | Mac→iPhone | VideoToolbox H.264 encode → companion iOS app `VTDecompressionSession` decode | — | 🔶-app | requires iOS companion; hardware decode fine |
| iPhone/iPad↔Mac | Screen control | either | — | — | ❌ | **no public API to inject input into iOS**; iPhone Mirroring (Apple-only) is the only path; we surface "use Apple's iPhone Mirroring" guidance |
| iPhone/iPad↔Mac | Photos | iPhone→Mac | `PHPickerViewController` (user-selected) | none | 🔶-app | — |
| iPhone/iPad↔Mac | Clipboard | both | `UIPasteboard` (foreground reads only) | iOS 16+ paste prompt | 🔶-app | foreground-only; paste prompt on read; no background paste |
| iPhone/iPad↔Mac | Notifications of other apps | iPhone→Mac | — (no API) | — | ❌ | `UNUserNotificationCenter` is own-app-only |
| iPhone/iPad↔Mac | SMS/iMessage | either | — (no API) | — | ❌ | Apple-only (iMessage cloud/Continuity); no third-party path |
| iPhone/iPad↔Mac | File transfer | both | our chunked mTLS WS; iOS side `FileManager`/share sheet | — | 🔶-app | app-sandbox storage |
| iPhone/iPad↔Mac | Calls/call state | either | — (CallKit own-calls only) | — | ❌ | — |
| iPhone/iPad↔Mac | Contacts | iPhone→Mac | `CNContactPickerViewController` | user-selected | 🔶-app | — |
| iPhone/iPad↔Mac | Battery status | iPhone→Mac | `UIDevice.batteryLevel` (needs `batteryMonitoringEnabled`) | none | 🔶-app | coarse/limited accuracy |
| iPhone/iPad↔Mac | AirPlay mirroring | iPhone→Mac | macOS built-in AirPlay Receiver (OS feature) | — | ✅ (OS) | this is an OS feature of macOS, not our app; we detect + deep-link to it |
| iPhone/iPad↔Mac | iPhone Mirroring | iPhone→Mac | Apple feature (macOS 15+, Apple Silicon) | — | ❌ third-party | Apple-only; we surface guidance to use it |
| iPhone/iPad↔Mac | Discovery/pairing/sessions/transfer/media-remote | both | same as Android rows (iOS companion implements protocol v2) | Local network (iOS 14+) | 🔶-app | — |
 
### Mac ↔ Mac (both run our desktop app; or via built-in Screen Sharing)
 
| Platform | Feature | Dir | API / Technology | Permission | Status | Limitation |
|---|---|---|---|---|---|---|
| Mac↔Mac | Screen viewing | M→M | ScreenCaptureKit (Swift agent) → VideoToolbox H.264 → WebCodecs | TCC Screen Recording | 🔶 | capture consent; no iPhone-source (verified) |
| Mac↔Mac | Screen control | M→M | VNC client → macOS Screen Sharing "VNC viewers may control screen with password" mode | VNC password (user-set) | 🔶 | High-Performance mode is Apple-proprietary; third-party = VNC-compatible mode only |
| Mac↔Mac | Screen control (our app both ends) | M→M | Screen viewing + `CGEventPost` input injection | Accessibility (AX) | 🔶 | sanctioned system-wide injection; per-app AX permission |
| Mac↔Mac | File transfer, clipboard, media, notifications, battery, discovery | both | same core protocol | — | ✅ | notifications = in-app mirror only (system notif display optional) |
| Mac↔Mac | Screen streaming quality | both | adaptive bitrate/fps + keyframe recovery | — | 🔶 | measured in Phase 5 |
 
### Windows desktop (architecture-prepared; NOT built on this branch — stated honestly)
 
| Platform | Feature | Dir | API / Technology | Permission | Status | Limitation |
|---|---|---|---|---|---|---|
| Windows | All core (discovery, transfer, clipboard, media, screen via `Windows.Graphics.Capture`, input via `SendInput`) | both | Tauri/Rust core + windns `DnsServiceRegister`/`Browse`, Win32 clipboard, `SendInput` | clipboard none; capture picker; AX equivalent | planned | requires Windows VM/build env; `#[cfg]` seams already placed |
| Windows | Notification mirroring | W↔A | `UserNotificationListener` | MSIX package + capability + user consent | planned | unpackaged Win32 needs MSIX helper |
| Windows | Phone-Link-exclusive features (app streaming, cross-device graph, Instant Hotspot) | — | Microsoft/OEM-privileged | — | ❌ third-party | documented as Microsoft-internal; not replicable |
 
## 3. Consequences for this branch (scope decisions)
 
1. **iOS companion**: feasible for viewing (ReplayKit send + VideoToolbox
   receive), photos, clipboard (foreground), transfer. Control is impossible
   → UI says so. **Decision: document + scaffold protocol-parity in this
   branch; full iOS companion is a follow-up** (Xcode build required — CLT
   alone can't sign/install iOS targets). Stated honestly in docs.
2. **Mac↔Mac control**: primary = our app on both Macs (Screen viewing +
   CGEventPost injection, both permission-gated). Secondary = VNC client into
   the OS Screen Sharing server (works without our app on the target, VNC
   password mode). Both implemented.
3. **Android control**: AccessibilityService + dispatchGesture (sanctioned);
   reported `needs-permission` until enabled; never claimed working before tested.
4. **Streaming subsystem** (not a demo): WebCodecs `annexb` + realtime +
   prefer-hardware decode; Tauri IPC Channel binary streaming; keyframe
   recovery; adaptive bitrate/fps ladder with hysteresis; bounded queues and
   backpressure (frame drop before memory growth); measured latency/FPS/CPU.
5. **SMS/calls on Android**: implemented behind explicit opt-in with policy
   disclosure (works in sideloaded builds; Play would require default-handler).
6. **Windows**: `#[cfg]` seams + this matrix; no Windows binary from this branch.
 
