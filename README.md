# I Built a Tiny Passkey Authenticator, Then Had to Explain Why You Shouldn't Trust It

**BSides Perth 2026** · Friday, 13 November 2026 · George Coldham  
**Session:** 30 minutes, including questions · **Track:** Identity / Hardware Security

[**Download the presentation (PowerPoint)**](./BSides-Perth-Pico-FIDO-Blueprint-Updated.pptx)

> A working authenticator is not automatically a trustworthy authenticator.

## About the talk

Passkeys eliminate shared passwords and provide strong resistance to phishing. But **why should we trust the device holding the private key?**

For this talk, I built a FIDO2 authenticator from a low-cost RP2350 development board, open-source Pico FIDO firmware and an oversized red pushbutton. It registered a passkey and signed in successfully. Then I investigated what the device actually does to protect its secrets, which security controls were enabled, and whether I could simply copy it.

The session covers:

1. **Why passkeys resist phishing.** Registration creates credential-specific cryptographic material; the service stores a public key. During sign-in, the authenticator signs a fresh challenge. WebAuthn's origin and relying-party checks prevent a phishing site from simply using a credential registered for the genuine site.
2. **Building and demonstrating a FIDO2 authenticator.** A modified Pico FIDO 7.6 build, a Waveshare RP2350 board and a physical user-presence button. The PIN provides **user verification (UV)**; the button provides **user presence (UP)**.
3. **Where the secrets live.** Pico FIDO's source describes an encrypted/wrapped device master key and credential information stored in external flash. Its protection design can use RP2350 one-time-programmable (OTP) secrets and device identity, but also contains alternative code paths. Source-code intent is not proof of how a particular board was provisioned.
4. **What the hardware reported.** We inspected the original device with `picotool` and found Secure Boot **off**, debug **enabled**, and secure debug **enabled**.
5. **What happened when we tried to copy it.** A standard BOOTSEL full-flash read failed with a permission error. We reproduced a working authenticator on a second board, **not** the original device's credentials.
6. **What would make it more trustworthy.** Properly configured boot integrity, debug restrictions, OTP provisioning, protection of boot/readout paths, and independent assurance—along with the remaining architectural differences from a YubiKey.

## The build

| Component | What we used |
| --- | --- |
| Microcontroller board | **Waveshare RP2350-Plus-16MB** (RP2350 revision A4 on the tested original) |
| Firmware | **Pico FIDO 7.6**, with local Waveshare-specific changes |
| User-presence control | Large red **normally-open, momentary** pushbutton |
| Button wiring | Board **BOOT/P4 test point** to **GND**; **P4 is not the GP4 GPIO pin** |
| Test site | [WebAuthn.io](https://webauthn.io/) with disposable test credentials |
| Inspection tool | Raspberry Pi [`picotool`](https://github.com/raspberrypi/picotool) |

The local firmware changes included selection of the Waveshare board, its LED on GPIO25, and safer BOOTSEL sampling across the RP2350's cores. The external button reproduces the onboard BOOTSEL press. The proposed decorative LED ring **was not fitted** to the final device.

## What we actually found

| Observation | Result | Important limitation |
| --- | --- | --- |
| Original authenticator | Registered and authenticated successfully with its external button | A working login does not prove physical security |
| User presence on spare board | After the PIN step, authentication **timed out** when the button was not pressed | Demonstrated for the tested ceremony on the spare, not every possible FIDO command |
| `picotool info -a` on original | RP2350 A4; Pico FIDO 7.6; 16 MB external flash | Identifies reported configuration and image metadata |
| Secure Boot | `secure boot: 0` — **off** | A verified image hash is **not** enforced signed boot |
| Debug configuration | `debug enable: 1`; `secure debug enable: 1` | No SWD debugger was attached or used to extract keys |
| Full-flash read | Stopped at about **6%** with **`permission failure`** | A partial read is not a complete backup or a credential extraction |
| Spare board | Same firmware installed; registered and used its **own** credential | **No cross-device credential clone was demonstrated** |

### Why the flash read failed

The original device's partition table described the following **non-secure BOOTSEL** permissions within the reported 0–4 MB region:

| Region | Flash offsets | BOOTSEL access |
| --- | --- | --- |
| Firmware | `0x000000–0x100000` | Read / write |
| Binding (8 KB) | `0x100000–0x102000` | **Write only** |
| Data | `0x102000–0x400000` | Read only |

The standard all-flash save stopped near the unreadable Binding region. That is **consistent with** the permission failure; we did not conclusively isolate the tool's internal failure path. The board has **16 MB total flash**, while the table above only describes the reported 0–4 MB partitioned region.

**The conclusion is deliberately narrow:** the standard BOOTSEL full-read path was blocked. We did **not** prove the device cannot be cloned, nor test direct flash-chip access, SWD extraction, replacement firmware or fault injection. We also did not verify the original unit's OTP provisioning and page locks.

## Further reading and project resources

### Passkeys, WebAuthn and FIDO2

- [WebAuthn.io](https://webauthn.io/) — hands-on WebAuthn registration and sign-in demonstration.
- [Passkeys.dev](https://passkeys.dev/) — approachable developer documentation and passkey resources.
- [W3C Web Authentication (WebAuthn) Level 3](https://www.w3.org/TR/webauthn-3/) — formal browser API, credential lifecycle, origin and relying-party rules.

### Pico FIDO, hardware and tools

- [Pico FIDO — upstream repository](https://github.com/polhenarejos/pico-fido) — the open-source authenticator project by Pol Henarejos and contributors.
- [Pico FIDO v7.6 release](https://github.com/polhenarejos/pico-fido/releases/tag/v7.6) — the version used for this experiment; newer upstream releases may behave differently.
- [Waveshare RP2350-Plus wiki](https://www.waveshare.com/wiki/RP2350-Plus) — board information and supporting resources.
- [Waveshare RP2350-Plus schematic (PDF)](https://files.waveshare.com/wiki/RP2350-Plus/RP2350_Plus.pdf) — including the BOOT/P4 button circuit.
- [Raspberry Pi picotool](https://github.com/raspberrypi/picotool) — RP2350 firmware and BOOTSEL inspection utility.
- [Raspberry Pi RP2350 datasheet (PDF)](https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf) — chip architecture, boot process, OTP and debug controls.

### Hardware security and comparison

- [Understanding RP2350's security features (Raspberry Pi PDF)](https://pip-assets.raspberrypi.com/categories/1260-security/documents/RP-009377-WP-1-Understanding%20RP2350_s%20security%20features.pdf) — Secure Boot, encrypted boot and device lockdown.
- [YubiKey hardware guide](https://developers.yubico.com/Developer_Program/Guides/YubiKey_Hardware.html) — secure-element architecture and physical user-presence design.
- [YubiKey 5 firmware overview](https://docs.yubico.com/hardware/yubikey/yk-tech-manual/yk5-firmware-overview.html) — firmware characteristics and model-specific capabilities.

## Scope and caveats

This repository contains **the conference presentation and this README only**. It does **not** include the local modified firmware worktree, build artifacts, credential material, OTP secrets, or partial flash dumps. The upstream Pico FIDO links are references, **not** a byte-for-byte reproduction of the local Waveshare firmware.

This was a practical learning exercise, not an independent certification or a comprehensive penetration test. In particular, **FIDO2 compatibility, phishing resistance, and physical resistance to credential extraction are different security claims**.

---

**The lesson:** It authenticated. We measured some security controls. One straightforward flash-copy attempt was blocked. None of those facts, on its own, establishes that a homemade authenticator deserves the same trust as a purpose-built commercial security key.
