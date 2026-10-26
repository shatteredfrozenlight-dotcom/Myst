# Myst

Offline, account-free Android app. Built with a minimalist **Myst-inspired dark abyss aesthetic** featuring a scrollable slide-out navigation drawer, a push-to-talk centerpiece, live connection status indicator, ephemeral secure chat, VOX voice activation, hardware button mapping, and relay target configuration.

- **Radio (home):** hold to talk (or use VOX voice activation / mapped hardware button), one speaker at a time, audio heard once and never stored.
- **Chat:** ephemeral text. Deleted shortly after it is shown and at most 15 minutes after sending, not configurable. Memory only.
- Identity is a cryptographic key (no phone number, email or account). No servers, no telemetry. The manifest declares INTERNET only because Android requires it for any socket, even LAN-only UDP; `UdpTransport.isLocalNetwork` refuses every non-local address and CI checks the guard exists. Screenshots blocked (`FLAG_SECURE`, also on chat, dialogs, and navigation drawer).
- License: GPL-3.0. See PROVENANCE.md.

## Encryption (implemented and tested)
| Use | Construction |
|---|---|
| 1:1 sessions, key handover | `Noise_XX_25519_ChaChaPoly_SHA256`: mutual authentication, hidden static keys, forward secrecy. Per-message explicit nonce + 64-packet replay window (tolerates mesh reordering). |
| Channel chat | ChaCha20-Poly1305, fresh subkey per message: HKDF-SHA256(channelKey, random 16-byte salt). Payload carries id + timestamp; stale (>15 min), future, duplicate and forged messages are rejected. |
| Channel voice | ChaCha20-Poly1305, subkey per burst: HKDF-SHA256(channelKey, burstId); nonce = sequence number (0..65535, never wraps); header (burst id, seq, flags) authenticated; +16 bytes per packet. |
| Joining a channel | `myst://join/<key>?n=<name>` link or QR display; treat as a secret. |
| Primitives | Bundled Noise-Java (pure Java, runs on every Android version) and the platform HMAC. No custom primitives. |

**Verified:** 133 tests pass successfully (109 core Java crypto/mesh/voice tests and 24 app Kotlin state/UI integration tests), including the official noise-c test vector for this exact protocol (byte-exact handshake, hash and transport messages), RFC 5869 HKDF vector, tamper-at-every-byte tests, replay and reorder tests, and mutation checks.

## Layout & Features
- `core/` crypto layer (Java), vendored Noise-Java, ephemeral store, transport interface/policy, voice packet format, floor control, and mesh router.
- `app/` Compose UI with **Myst Dark Theme** (`#07080D`), scrollable slide-out navigation drawer (`ModalNavigationDrawer`), PTT centerpiece, live network connection status (`[N nearby · LAN]`), VOX voice activation, hardware button mapping, persistent settings storage (`SharedPreferences`), join & configure relay (IP:port target), invite/join dialogs, and hold-to-wipe.
- `AudioEngine` supporting high-efficiency Opus encoding (Android 10+) with seamless AAC-LC fallback.
- `tools/ci/check-privacy.sh` forbids INTERNET and other permissions, logging, file/prefs/db/network APIs.
- `docs/` audit of bitchat-android and the porting plan.

## Next steps
1. Project successfully compiled, signed release/debug configured, and verified (`./gradlew :core:test :app:testDebugUnitTest :app:assembleDebug` with all 133 tests passing).
2. Expand physical radio hardware integration beyond simulation.
3. Conduct independent security review before production deployment.

## Threat model (honest version)
Defends against: passive eavesdroppers, malicious relays (end-to-end encrypted), casual inspection after use (nothing on disk, no screenshots).
Does NOT defend against: someone photographing the screen, a modified client, a compromised phone, observation of who is nearby (radio presence), someone who obtains the channel key, or device seizure while the app is running.
Unaudited. Do not rely on it for safety until independently reviewed.

## Status and known limits (honest list)
- Verified: 133 tests pass successfully (109 core Java tests + 24 app integration tests, covering Noise XX official vectors, HKDF vectors, tamper/replay/reorder protection, mesh simulator, loopback UDP, and app state/UI integration).
- Measured on the simulator, 6x6 mesh flooding reach within 7 hops: 100% at 0% loss, 99% at 5%, 97% at 10%, 84% at 20%. Voice audio surviving 1/2/3 hops at 10% loss: 89/79/70%.
- Channel traffic has no forward secrecy, and anyone holding the key can forge messages.
- Ephemerality is best effort. Unaudited: get independent review before relying on it.

## Channels, keys and receipts
- **Saved channel.** After you join (or create) a channel, its key, owner and member list are saved, encrypted by a non-exportable Android Keystore key. Nothing reconnects by itself; messages are never saved. Panic wipe deletes the Keystore key and both files.
- **Owner model.** The channel creator's public key is in the invite. Joiners send the owner a sealed join request; the owner approves it (check the ID in person). Only the owner can remove people.
- **Removing someone.** The owner makes a new key, destroys the old one, and sends the new key to each remaining member in a one-shot Noise_X message (only that member can open it). Members accept it only from the owner and only with a higher epoch. Unacknowledged drops are re-sent every 10 s and survive restarts. The removed identity is banned from re-requesting. They keep whatever they already received; there is no forward secrecy.
- **Read = gone in 30 s.** A message is deleted 30 seconds after it is first shown (15 minutes at the latest, unread or not). Memory only.
- **Receipts.** Encrypted with the channel key: received (always), read and heard (can be switched off). Counts only; receipts do not say who. A key holder can forge receipts, like chat.
- Limits: if the owner is offline, nobody can be removed or admitted; join requests are retried for 10 minutes after joining; the read/heard switch is not saved across restarts; one channel at a time.

## Transport fallback & Toggles
`TransportSupervisor` keeps LAN on, switches on Wi-Fi Aware/Direct after ~5 s with nobody found, then Bluetooth after another ~5 s, and switches fallback radios off again once LAN has been stable for a minute and they are idle. Users can also independently toggle WLAN, Wi-Fi Direct/Aware, and Bluetooth LE via the scrollable Myst side navigation drawer.

## Relay-only mode & Relay Configuration
Switch on **Relay Mode** or configure a specific relay IP:port target in the scrollable Myst side navigation drawer to extend range (offline local relays recommended; online internet bridge relays at your own risk). The router never decrypts, so a relay needs no channel key; the app opens nothing, stores nothing, sends no receipts and cannot chat or talk until you stop relaying. `relay/` provides a headless relay for an always-on machine (see docs/RELAY.md).
