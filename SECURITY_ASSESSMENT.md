# Halliday G2 Android companion app and wearable update-path assessment

## Scope and evidence

Assessment target: Android package `com.halliday.oasis` (version `0.1.3`, build `134`). The review combined JADX recovery, native Flutter/AOT string and call-path review, on-device installation, and a Bluetooth HCI trace of a user-authorised OTA update. The assessed APK SHA-256 is `824f64ea5b5244b12ef779a62ee7ad098fe778177bb3f9437b0545f44e6f9338`.

Sensitive trace fields, device identifiers, credentials, and the firmware payload are deliberately redacted from this published document.

## Executive conclusion

The initial BLE connection and OTA process have material security weaknesses. Most urgently, the wearable exposes an OTA Wi-Fi access-point mode with HTTP and FTP services and uses weak, device-disclosed credentials. An attacker within radio range may be able to join this temporary network and interact with its update surface. Given the product's intended use around meetings containing PII and its microphone capability, this is a high-risk design until device-side authentication, encryption, and firmware-signature enforcement are independently verified.

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

The application checks for an update at the vendor gateway, starts and records OTA state through the listed APIs, then orchestrates the glasses over BLE. HCI evidence confirmed a real OTA transaction: wearable AP mode, HTTP service availability, FTP credentials supplied in cleartext BLE traffic, a target OTA archive name, and a successful transfer status. The firmware payload itself was not recovered from the Android app sandbox during this run.

## Findings

| ID | Severity | Finding | Security impact |
|---|---|---|---|
| F-01 | High | OTA AP with HTTP/FTP update surface and weak disclosed credentials | Nearby attackers can attempt unauthorised access to the wearable update network and services. |
| F-02 | High | Firmware authenticity is not independently proven by the app | If the device bootloader does not enforce a vendor signature, a compromised gateway/update path could lead to malicious firmware. |
| F-03 | Medium | GATT connection callback accepts connected state without first requiring success status | Initial connection may be reported as ready after a failed or incomplete GATT setup. |
| F-04 | Medium | Auto-connect path can return without closing the GATT client | Repeated connection attempts may leak client resources and degrade reconnect reliability. |
| F-05 | Medium | BLE advertisement identity is not authentication | A spoofing peripheral can potentially trigger pairing/connection logic unless the proprietary bond command cryptographically authenticates the glasses. |
| F-06 | Low | CCCD ordering race | The app writes the descriptor before enabling local notifications; a first indication can be lost on fast peripherals. |

## Initial-connection analysis

The Android BLE plugin's connection-state handler treats state `CONNECTED` as sufficient for the application-level connected callback, without requiring the GATT status to indicate success. The callback should fail closed unless the status is `GATT_SUCCESS`, and the app should only advertise readiness after service discovery, characteristic validation, notification setup, and an authenticated session handshake complete.

The auto-connect callback also has an early-return branch that does not close the GATT client. Make `disconnect()`/`close()` idempotent and ensure every terminal error and retry path releases the old client.

The pre-bond advertisement identifier should be considered a locator, not proof of identity. Bind the wearable using an authenticated, replay-resistant exchange after GATT encryption is established. For a product that handles meeting audio and PII, mutual authentication and downgrade protection are appropriate baseline controls.

## Firmware and microphone-risk evaluation

The observed OTA workflow creates an especially sensitive trust boundary: it can alter firmware on a microphone-equipped wearable used in PII-rich meetings. App-side transport controls and a server-provided hash are not sufficient evidence of firmware authenticity. The wearable bootloader must verify a versioned vendor signature using a public key embedded in immutable or appropriately protected device storage; it must reject unsigned, modified, replayed, or downgraded images.

The vendor gateway and update APIs should require short-lived, audience-bound access tokens; use TLS with certificate validation/pinning where appropriate; do not put device service credentials in BLE payloads; and ensure update URLs are authorized per device and expire promptly. The glasses should disable FTP in production. If local recovery is required, use an authenticated, encrypted, device-bound update protocol with least privilege and an explicit physical-presence control.

For microphone and PII assurance, perform a separate firmware and hardware review covering: microphone enablement paths, recording indicator integrity, remote-control permissions, audio buffering/storage, uplink destinations, encryption keys, consent/audit logs, and secure erase. Network egress monitoring and reproducible firmware extraction are required before claiming that meeting audio cannot be accessed or exfiltrated.

## Remediation priority

1. Remove or lock down the HTTP/FTP OTA service; rotate all deployed OTA credentials and make them per-device, ephemeral, and unavailable over BLE.
2. Enforce signed-image verification and anti-rollback in the wearable boot chain; commission an independent verification test using altered firmware.
3. Repair BLE state handling, GATT cleanup, and notification setup ordering.
4. Require authenticated pairing/session establishment before accepting device commands or OTA state.
5. Obtain the exact OTA image and bootloader details for offline firmware reverse engineering and a microphone/privacy control review.

## Validation completed

- Decompiled the Java/Kotlin portion of the APK and inspected the embedded Flutter/native layer.
- Installed the APK on an authorised USB-connected Android device.
- Captured and decoded a Bluetooth HCI trace during a successful OTA update from wearable firmware 0.16.x to 0.18.1.
- Confirmed the vendor gateway paths and the wearable's local OTA AP, HTTP, and FTP behaviour.

## Limitations

The firmware archive was not retained from the application sandbox, so this report does not assert a complete firmware decompilation or a final conclusion about microphone exfiltration. The raw evidence is retained separately and should be handled as sensitive security material.
