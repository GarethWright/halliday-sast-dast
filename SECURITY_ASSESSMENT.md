# Halliday G2 security assessment — final conclusions

## Scope

This document consolidates the final conclusions from an authorised static and dynamic assessment of the Halliday G2 Android companion app and wearable firmware.

| Item | Assessed value |
|---|---|
| Android package | `com.halliday.oasis` |
| App version | `0.1.3` (build 134) |
| APK SHA-256 | `824f64ea5b5244b12ef779a62ee7ad098fe778177bb3f9437b0545f44e6f9338` |
| Wearable firmware | Upgrade from 0.16.x to 0.18.1 observed; 0.18.1 package acquired and analysed |
| Methods | APK/JADX and Dart-AOT analysis, authenticated API tracing, HCI analysis, live BLE tests, OTA extraction, package/script review, and partial Ghidra analysis of the AP and Bluetooth firmware images |

Raw captures, account tokens, device serial numbers, and other PII-bearing evidence are retained outside this repository. This private report includes service credentials because they are necessary to describe the verified historical exposure accurately.

## Executive conclusions

1. **Initial ownership is not authenticated.** BLE pairing uses LE Secure Connections Just Works. A factory-reset or otherwise unowned device accepted an invented account identity and immediately granted a privileged session. An already-owned device rejects a different identity, but the first claim is not validated against the Halliday service or any cryptographic proof.
2. **Brief physical access enables takeover.** The on-device Settings → Unpair action has no owner PIN or equivalent confirmation. After local unpairing, the next nearby party can exploit the unauthenticated first-claim process. A blind remote `CmdUnbond` attempt was correctly rejected, but this does not mitigate the physical path.
3. **The firmware update trust chain is critically weak.** The production OTA archive is JAR-signed with the publicly available AOSP test key. Its `ota.sh` script writes images directly to the AP, Bluetooth, audio/sensor, and secondary AP partitions without verifying a vendor signature. Static firmware evidence showed CRC32/magic-value integrity checks, not a demonstrated vendor-authenticity check. Whether an immutable bootloader independently rejects a modified image remains to be tested on a lab unit.
4. **Historical OTA credentials were fixed defaults and reused.** Firmware 0.16.x disclosed the same low-entropy Wi-Fi and FTP credentials on repeated OTA use. Firmware 0.18.1 changes the OTA credentials every time OTA mode is entered. This is a real improvement and eliminates the historical persistent/default-password condition.
5. **Credential rotation does not fix OTA authorization.** The current per-session credentials are returned over the proprietary BLE channel after the weak first claim. A party that claims an unowned device can request the current credentials, start its OTA access point, and reach its local update services.
6. **Microphone control and partial audio capture were confirmed.** The glasses carry microphone audio over the same proprietary GATT command/notification channel used by other privileged functions. The Start Meeting request (`biz=7`, `cmd=2`, encoded command byte `0xE2`) uses a fixed, non-secret control payload. The command was accepted in live testing, entered the real meeting state, and partial microphone/audio capture was confirmed. A sustained end-to-end adversarial recording was not completed.
7. **Meeting audio is processed in Halliday's cloud, not locally.** The app forwards Opus audio received from the glasses intact as binary WebSocket frames to Halliday's servers. Transcription, translation, and AI processing occur server-side. The app contains no local transcription or pre-upload PII-scrubbing stage, so sensitive content has already left the phone before it can be identified or redacted.

## Findings

| ID | Severity | Conclusion |
|---|---|---|
| F-01 | Critical | An unowned device accepts an arbitrary first identity claim without account or cryptographic proof. |
| F-02 | Critical | Production firmware uses the public AOSP test key and the update script performs no vendor-signature check before flashing component images. |
| F-03 | High | Physical unpair is not owner-protected and exposes the device to the unauthenticated first-claim takeover. |
| F-04 | High | OTA HTTP/FTP services are reachable using credentials disclosed through the weak BLE trust boundary. |
| F-05 | Critical | A weakly bonded client can issue the microphone/meeting-enable command; live testing confirmed command acceptance, real meeting-mode activation, and partial audio capture. |
| F-06 | High | Intact Opus meeting audio is forwarded to Halliday's cloud for transcription and AI processing with no local transcription or PII scrubbing. |
| F-07 | Medium | Android reports GATT connected when `newState == CONNECTED` without also requiring `status == GATT_SUCCESS`. |
| F-08 | Medium | An auto-connect disconnect path returns without closing the GATT object, allowing stale clients and reconnect exhaustion. |
| F-09 | Low | CCCD setup ordering can lose the first notification or indication on a fast peripheral. |

## OTA credentials: historical and current behaviour

| Firmware behaviour | Wi-Fi AP password | FTP account | Persistence |
|---|---|---|---|
| Historical, observed on 0.16.x | `besfd123` | `ota` / `abc123` | Fixed defaults reused across OTA-mode entries |
| Current, confirmed on 0.18.1 | Generated value; one observed example was `m6mKT=yt` | Generated per OTA-mode entry | Changes every time OTA mode is entered |

The historical Wi-Fi password was only eight characters and the FTP password was six characters, with obvious human-readable patterns. Under the unrealistic assumption of uniform selection from 36 lowercase alphanumeric symbols, their maximum search spaces were approximately 41.4 and 31.0 bits. Their actual entropy was much lower because they were fixed product defaults. The FTP transport also provides no confidentiality.

Firmware 0.18.1 materially improves this point by generating fresh values for each OTA-mode entry. The residual problem is authorization rather than password entropy: the active values are supplied to a BLE client after the device accepts its session. On an unowned device, that session can be obtained using an arbitrary fabricated identity.

## Initial connection and ownership protocol

The reconstructed connection flow is:

```text
Advertisement match
  → LE GATT connection
  → service and characteristic discovery
  → CCCD subscription
  → proprietary BF-framed command channel
  → CmdBond identity claim
  → privileged BizRunning session
```

The custom GATT UUIDs recovered from the application and validated against live traffic are:

- Service: `04000400-0000-1000-8000-009078563412`
- Notification characteristic: `05000500-0000-1000-8000-009178563412`
- Write characteristic: `06000600-0000-1000-8000-009278563412`

The application protocol uses `BF` magic bytes, protocol/module/command fields, variable-length payload framing, and an additive checksum. The checksum detects accidental damage; it is not a message-authentication code.

Pairing is LE Secure Connections Just Works because the glasses advertise `NoInputNoOutput`. This protects a connection from passive interception but does not authenticate the person or account initiating it. Live testing established the ownership rule:

- A claimed device rejects a different identity with NCK 24.
- Privileged commands without a valid session are rejected with NCK 25.
- A fully reset/unowned device accepts an arbitrary syntactically valid identity and enters the privileged state.
- Remote unbond without a valid session is rejected.
- Local unpair from the glasses UI requires no owner secret and recreates the vulnerable unowned state.

## Firmware download and wearable services

The app uses the vendor gateway at `https://halliday-gateway.halliday-tech.com`. Recovered OTA routes are:

- `/hallidayiot/app/firmware/upgrade/check`
- `/hallidayiot/app/ota/upgrade/start`
- `/hallidayiot/app/ota/upgrade/update`
- `/hallidayiot/app/device/update/info`

The authenticated server workflow returns update metadata and the OTA archive. The phone then directs the glasses over BLE to expose a private OTA network. The observed device-side address was `192.168.4.1`; an HTTP service reported ready and FTP was used for archive transfer. BLE status messages reported the target archive and transfer result.

The acquired 0.18.1 package contained images for multiple processors and an `ota.sh` script that writes them directly to partition devices. The package's JAR certificate is the public AOSP test certificate. No vendor-signature validation occurs in the script before the writes. CRC/read-back checks establish transfer integrity, not publisher authenticity.

## Microphone and PII conclusion

The product is designed to process meeting audio and therefore operates in a high-impact PII context. Static analysis confirms microphone/audio management, meeting transcription, cloud WebSocket handling, and a phone-to-glasses meeting-enable command. Firmware analysis also found VAD/wake processing and microphone controls on the Bluetooth/multimedia processor. Live testing confirmed that the enable command is accepted after the weak bond, activates the real meeting workflow, and permits partial audio capture.

The normal application pipeline is:

```text
Glasses microphone
  → Opus frames over proprietary BLE notifications
  → Halliday Android app
  → intact binary Opus frames over an authenticated WebSocket
  → Halliday cloud transcription / translation / AI processing
  → text and result events returned to the app and glasses
```

`HeyHallidayService.handleDeviceAudio()` forwards each audio chunk received from the glasses directly to `HallidayWsClient.sendAudio()`. The app connects to `wss://halliday-gateway.halliday-tech.com/hallidayai/ws/live/agent` and sends the audio as binary WebSocket frames. The same cloud mechanism supports the `hey_halliday`, `translation`, and `meeting` services. Their start messages contain audio/session metadata—including Opus parameters, sample rate, channel count, language, and session identifiers—not locally produced transcript text.

There is no local speech-to-text path for the meeting feature and no text-only client protocol that would allow the app to submit an on-device transcript instead of audio. The phone does not decode the meeting into text, detect PII, redact names or identifiers, or otherwise minimise content before transmission. TLS protects the network connection in transit, but it does not change the data-minimisation result: Halliday's server receives the intelligible audio stream and must process it to produce the transcript.

This architecture prevents meaningful pre-transmission PII scrubbing. Meeting participants may disclose names, contact details, health data, financial information, credentials, commercial secrets, or special-category personal data before either the user or software can identify it. Once spoken, that content is included in the outgoing Opus stream. Server-side redaction can improve stored transcripts, but it occurs after disclosure of the original audio and therefore cannot prevent the cloud processor from receiving the PII.

The confirmed evidence does not establish continuous recording or malicious exfiltration by the manufacturer. It does establish that use of the advertised meeting, translation, and assistant features intentionally transfers intact audio off the phone for server-side processing, and that partial audio capture is achievable through the weak device-control boundary.

## Required remediation

1. Replace the AOSP test key with a protected Halliday production signing key and enforce signature verification plus anti-rollback in immutable boot code for every flashed partition.
2. Require a Halliday-service challenge and account-bound cryptographic proof for the first device claim, with explicit physical confirmation on the glasses.
3. Protect local unpair with an owner-authorized action and clearly notify the existing owner of ownership changes.
4. Retain the 0.18.1 per-session credential generation, but replace FTP with an encrypted, device-bound transfer protocol. Release credentials only after authenticated device and account proof, and expire them when OTA mode exits.
5. Separate link-connected, services-ready, notifications-ready, and authenticated-session-ready states in the Android client. Require `GATT_SUCCESS`, close every terminal GATT client, and enable local notifications before writing the CCCD.
6. Require authenticated and authorized control of microphone/meeting commands; provide a hardware-backed recording indicator that firmware cannot suppress.
7. Perform speech recognition on the glasses or phone wherever hardware permits. For privacy-sensitive deployments, make local-only transcription the default and allow cloud processing only through an explicit, policy-controlled opt-in.
8. Where cloud processing is necessary, perform local PII detection, redaction, segmentation, and user-approved exclusion before any audio leaves the phone. Do not describe server-side transcript redaction as pre-transmission PII protection because the server has already received the original audio.
9. Use short-lived, purpose-scoped WebSocket credentials bound to the user, device, and meeting session rather than the app's general REST bearer token. Document processor location, subprocessors, retention, training use, deletion, lawful basis, and cross-border transfers.
10. Complete a lab anti-rollback/signature-rejection test and a sustained end-to-end adversarial microphone test before approving the glasses for meetings containing PII.

## Evidence boundaries

The following conclusions are directly confirmed: BLE Just Works pairing, arbitrary first claim on an unowned unit, protection against blind remote unbond, unprotected local unpair, historical fixed OTA defaults, current per-entry credential regeneration, local HTTP/FTP OTA operation, production package acquisition, AOSP test signing, absence of signature verification in `ota.sh`, acceptance of the meeting-enable command, partial audio capture, and direct forwarding of intact Opus audio to the Halliday cloud without local transcription or PII scrubbing.

The remaining high-value tests are whether immutable boot code rejects an altered but structurally valid image and whether an independently bonded client can sustain a complete audio stream without additional authorization. Cloud-side storage, secondary use, regional processing, retention, deletion, and subprocessors cannot be established from the client binaries and require contractual and server-side evidence.
