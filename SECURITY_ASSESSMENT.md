# Halliday G2 Android companion app and wearable update-path assessment

## Scope and evidence

Assessment target: Android package `com.halliday.oasis` (version `0.1.3`, build `134`). The review combined JADX recovery, native Flutter/AOT string and call-path review, on-device installation, and a Bluetooth HCI trace of a user-authorised OTA update. The assessed APK SHA-256 is `824f64ea5b5244b12ef779a62ee7ad098fe778177bb3f9437b0545f44e6f9338`.

Sensitive trace fields, device identifiers, credentials, and the firmware payload are deliberately redacted from this published document.

## Executive conclusion

The initial BLE connection and OTA process have material security weaknesses. The critical risk is a practical trust-chain failure: a production OTA package was retrieved through the real update flow, is signed with the publicly available AOSP test certificate, and its on-device flashing script did not perform cryptographic image verification before writing component images. The wearable also exposes an OTA Wi-Fi access-point mode with HTTP and FTP services and discloses weak credentials over BLE. Together, these conditions create a plausible proximity path to arbitrary wearable firmware replacement.

An unbonded wearable was also observed accepting a fabricated initial identity claim without proof of account ownership. Because link-layer pairing is Just Works, an attacker near a factory-reset, returned, resold, or physically unpaired device can claim it and access privileged OTA functions. For a microphone-equipped product used in PII-rich meetings, this is a critical security and privacy issue.

## Reconstructed protocol

```text
Phone app
  | BLE advertisements / GATT connection
  | proprietary command + notification traffic
  v
Wearable
  | enters temporary OTA access-point mode
  | exposes HTTP service and FTP transfer service on private LAN
  v
Wearable OTA services (192.168.4.1 observed)
  ^
  | phone sends network and OTA commands; transfer status returns via BLE
  |
Vendor gateway: https://halliday-gateway.halliday-tech.com
  /hallidayiot/app/firmware/upgrade/check
  /hallidayiot/app/ota/upgrade/start
  /hallidayiot/app/ota/upgrade/update
  /hallidayiot/app/device/update/info
```

The application checks for an update at the vendor gateway, starts and records OTA state through the listed APIs, then orchestrates the glasses over BLE. HCI evidence confirmed a real OTA transaction: wearable AP mode, HTTP service availability, FTP credentials supplied in cleartext BLE traffic, a target OTA archive name, and a successful transfer status. The production firmware was subsequently acquired through the authenticated vendor update flow and analysed offline.

## Consolidated findings

| ID | Severity | Finding | Security impact |
|---|---|---|---|
| F-01 | Critical | OTA firmware trust chain is broken | A retrieved production package uses the public AOSP test certificate; the observed update script did not cryptographically authenticate component images before flashing. |
| F-02 | Critical | Initial device claim is unauthenticated | A newly unbonded device accepted a fabricated identity claim and then accepted privileged OTA commands. |
| F-03 | High | OTA AP with HTTP/FTP update surface and low-entropy initial passwords | Nearby attackers can recover the credentials from BLE traffic or cheaply guess them, then access the wearable update network and services. |
| F-04 | High | Microphone/audio path is exposed after weak bond | Audio uses the same proprietary BLE command/notification channel; a recovered Start Meeting command requires no demonstrated session secret. End-to-end capture remains to be repeated under controlled lab conditions. |
| F-05 | Medium | GATT connected callback does not require successful status | Initial connection may be reported as ready after a failed or incomplete GATT setup. |
| F-06 | Medium | Auto-connect path can return without closing the GATT client | Repeated connection attempts may leak client resources and degrade reconnect reliability. |
| F-07 | Low | CCCD ordering race | The app writes the descriptor before enabling local notifications; a first indication can be lost on fast peripherals. |

### F-03 password strength and disclosure detail

The live OTA trace exposed two initial service passwords: an eight-character Wi-Fi AP password and a six-character FTP password. Both used simple lowercase-letter/digit constructions; the literal values are withheld from this repository. Even if each character had been independently and uniformly selected from all 36 lowercase alphanumeric symbols—which the observed human-readable patterns do not support—the nominal search-space ceilings would be only about 41.4 bits and 31.0 bits respectively. Their effective entropy is materially lower because they follow predictable textual patterns and appear to be product defaults rather than device-generated random secrets.

The FTP password is below a reasonable modern minimum and is feasible to guess online if rate limiting is absent. The eight-character AP password merely meets the minimum WPA passphrase length and is unsuitable for protecting a security-sensitive update service. More importantly, brute force is unnecessary in the observed workflow: both credentials are transmitted to the phone in readable BLE application payloads after the weak initial bond. Anyone able to complete that bond can obtain the passwords directly. If the same values are reused across devices or update sessions, compromise of one trace becomes a fleet-level or persistent credential exposure.

These are not independent authentication factors: the party that reaches the weak BLE trust boundary is given the credentials for the next Wi-Fi/FTP boundary. Password rotation alone is therefore insufficient. The local update service needs device-bound, high-entropy, single-use credentials delivered only after authenticated account and device proof, with a short expiry, connection throttling, and no plaintext FTP.

## Initial-connection analysis

The Android BLE plugin's connection-state handler treats state `CONNECTED` as sufficient for the application-level connected callback, without requiring the GATT status to indicate success. The callback should fail closed unless the status is `GATT_SUCCESS`, and the app should only advertise readiness after service discovery, characteristic validation, notification setup, and an authenticated session handshake complete.

The auto-connect callback also has an early-return branch that does not close the GATT client. Make `disconnect()`/`close()` idempotent and ensure every terminal error and retry path releases the old client.

The pre-bond advertisement identifier is only a locator, not proof of identity. Link-layer pairing was observed to use LE Secure Connections Just Works because the wearable has no input/output capability; this prevents passive eavesdropping but does not authenticate a connecting party. Bind the wearable using an authenticated, replay-resistant exchange after GATT encryption is established, and require a physical confirmation plus account-bound proof for first claim and unpairing.

## Firmware and microphone-risk evaluation

The observed OTA workflow creates an especially sensitive trust boundary: it can alter firmware on a microphone-equipped wearable used in PII-rich meetings. The retrieved production OTA package was signed with the public AOSP test key, and review of the update script found no cryptographic verification before writing target partitions. Server-provided hashes and TLS do not substitute for a device-side signature check. The wearable bootloader must verify a versioned vendor signature using a public key embedded in immutable or appropriately protected device storage; it must reject unsigned, modified, replayed, and downgraded images.

The vendor gateway and update APIs should require short-lived, audience-bound access tokens; use TLS with certificate validation/pinning where appropriate; do not put device service credentials in BLE payloads; and ensure update URLs are authorized per device and expire promptly. The glasses should disable FTP in production. If local recovery is required, use an authenticated, encrypted, device-bound update protocol with least privilege and an explicit physical-presence control.

For microphone and PII assurance, perform a separate firmware and hardware review covering: microphone enablement paths, recording indicator integrity, remote-control permissions, audio buffering/storage, uplink destinations, encryption keys, consent/audit logs, and secure erase. Network egress monitoring and reproducible firmware extraction are required before claiming that meeting audio cannot be accessed or exfiltrated.

## Remediation priority

1. Immediately suspend OTA distribution to affected devices; replace the AOSP test key with a protected vendor signing key and enforce signature verification plus anti-rollback in the boot chain.
2. Remove or lock down the HTTP/FTP OTA service; rotate all deployed OTA credentials and make them per-device, ephemeral, and unavailable over BLE.
3. Require account-bound, challenge-response first claim with physical confirmation, and protect local unpair with an owner-authorized action.
4. Repair BLE state handling, GATT cleanup, and notification setup ordering.
5. Reproduce controlled end-to-end microphone capture testing and audit audio indicators, storage, transmission, consent, and deletion controls.

## Validation completed

- Decompiled the Java/Kotlin portion of the APK and inspected the embedded Flutter/native layer.
- Installed the APK on an authorised USB-connected Android device.
- Captured and decoded a Bluetooth HCI trace during a successful OTA update from wearable firmware 0.16.x to 0.18.1.
- Confirmed the vendor gateway paths and the wearable's local OTA AP, HTTP, and FTP behaviour.
- Retrieved and unpacked the production OTA package; verified the public test signing certificate and reviewed the flashing script.
- Tested initial claiming on a factory-reset/unbonded unit and observed acceptance of a fabricated identity.

## Limitations

The firmware archive has been analysed, but a complete disassembly of every processor image and a final microphone-exfiltration conclusion are not asserted. The raw evidence is retained separately and should be handled as sensitive security material.
