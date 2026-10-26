# Myst
Myst PTT
# Myst for Android

Private, offline push-to-talk and chat. No accounts, no servers, no telemetry. Your identity is a cryptographic key that lives only on your phone.

> **Status: early.** The protocol and logic are tested in simulation, and the Android state code interoperates with the Linux app over real UDP sockets in tests. The Android UI has not yet been compiled or run on a real device, and nothing has been independently security-audited. Treat it as a working prototype.

## What it does

- **Push-to-talk is the main screen.** Hold the button to talk. Voice is heard once and never stored.
- **Chat** opens from the bottom of the screen. It lives in memory only. A message is deleted 30 seconds after it is read, and at 15 minutes at the latest.
- **Delivery confirmations:** sent, received, and read (chat) or heard (voice). Sending read receipts can be turned off in PEOPLE.
- **Channels** are shared by invite. The channel key is saved on the phone after you join, but nothing connects automatically.
- **Removing someone** changes the channel key and sends the new key only to the people who remain.
- **Relay-only mode** turns the phone into a forwarder for other people's encrypted traffic (it cannot read it).
- **Panic wipe** erases your identity, channel key and messages.
- Works with the Myst Linux app on the same network.

## How people connect

1. The channel owner opens **INVITE**. It shows a QR code and a `myst://join/...` link.
2. The other person scans the QR with their phone's camera app (opens Myst and asks to confirm), or pastes the link into **JOIN**.
3. The owner sees the request in **PEOPLE** and presses **ALLOW** after checking the person's ID matches what they show.

Treat the invite as a password: it contains the channel key. Share it in person or over a channel you trust.

Transport today is **the same Wi-Fi network** (UDP port 47770, local addresses only). Bluetooth and Wi-Fi Aware/Direct fallbacks are designed and tested as logic but their radio drivers are not built yet.

## Security summary

| Area | What it does |
|---|---|
| Identity | X25519 key pair created on first launch, stored in `identity.bin`, wrapped with an Android Keystore AES-GCM key |
| Channel key | Saved in `channel.bin`, also Keystore-wrapped |
| Messages and voice | ChaCha20-Poly1305 with per-message subkeys derived from the channel key (HKDF) |
| Peer links | Noise_XX |
| Key changes | Sent to each remaining member sealed with Noise_X; only the owner can issue them; removed identities are banned |
| Screen | `FLAG_SECURE` on the window and dialogs: no screenshots, recordings or recents preview |
| Storage | Backups off; no analytics, crash reporting or logging |
| Network | Code refuses any non-local address (the INTERNET permission is required by Android for any socket) |

**Not protected:** local-network observers can see that Myst is in use, packet sizes, timing and which devices talk (not the content). Anyone with the invite link can read the channel. A rooted or malware-infected phone can read the screen and memory. A stolen channel key decrypts traffic recorded under it.

## Permissions

| Permission | Why |
|---|---|
| `RECORD_AUDIO` | Push-to-talk |
| `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE`, `CHANGE_WIFI_STATE`, `CHANGE_NETWORK_STATE` | Local-network sockets and Wi-Fi state |
| `CHANGE_WIFI_MULTICAST_STATE`, `WAKE_LOCK` | Keep Wi-Fi broadcasts arriving, including with the screen off |
| `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_CONNECTED_DEVICE`, `POST_NOTIFICATIONS` | Stay reachable while running; show the "Myst is running" notification |
| `BLUETOOTH_*`, `NEARBY_WIFI_DEVICES`, `ACCESS_FINE_LOCATION` | Reserved for the Bluetooth and Wi-Fi Aware/Direct transports (not active yet) |

There is no camera permission: QR codes are scanned with the phone's own camera app.

## Building

Requirements: Android Studio (recent), JDK 17, Android SDK 35.

1. Open the project root folder (the one containing `settings.gradle.kts`) in Android Studio and let Gradle sync.
2. Build: **Build > Build APK(s)**, or run `./gradlew :app:assembleDebug` if a Gradle wrapper is present.
3. Install the APK from `app/build/outputs/apk/`.

Modules: `app` (this app, Kotlin and Jetpack Compose), `core` (pure Java: crypto, mesh routing, transports), `relay` (headless relay), `desktop` (Linux app).

Because the Android UI has never been built against a real SDK, the first build may show small compile errors. Fix or report them.

## Using it

- **Both devices on the same Wi-Fi network, VPN off.** A VPN can send discovery packets out the wrong interface.
- Guest networks and routers with "client isolation" will block devices from seeing each other.
- The header shows `N nearby`. If it stays at 0, check the network and any firewall on the other device (UDP 47770).
- Voice between the phone and the Linux app needs `ffmpeg` installed on the Linux machine.

## Tests

The Android state logic is tested without a device: 24 tests in `MystStateTest` and 4 phone-versus-Linux interoperability tests in `InteropTest`. The privacy rules are enforced by `tools/ci/check-privacy.sh`.

## Not done yet

- Bluetooth and Wi-Fi Aware/Direct transports
- Opus voice
- One-to-one Noise chats in the UI
- Testing on real hardware (microphone, speaker, Wi-Fi discovery, phone AAC decoding of the Linux app's audio)
- Independent security review

See `PROVENANCE.md` for where the code and libraries come from and `LICENSE.md` for the license.
