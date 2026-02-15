# Intel Wi-Fi 6/6E (AX210) on macOS Sonoma / Sequoia / Tahoe (Hackintosh)

<p align="center">
  <img src="img/Wi-Fi.png" alt="Wi-Fi" />
</p>

## Context (why this repo)
Starting with **macOS Sonoma**, Apple removed legacy Broadcom Wi-Fi drivers used by many pre-2017 Macs and common Hackintosh cards (e.g., Fenvi T-919). A common workaround is **OCLP Root Patches**, but that typically requires relaxing macOS security (e.g., SecureBootModel disabled and SIP partially disabled).

This repo documents an alternative approach: using an **Intel AX210 (Wi-Fi 6E)** with **OpenIntelWireless** drivers, allowing you to keep **normal macOS security posture** (no OCLP root patches, no SecureBootModel/SIP downgrades).

> ✅ Goal: Wi-Fi + Bluetooth working on modern macOS with OpenCore, without root patches.

---

## Table of contents
- [Hardware](#hardware)
- [Before you start (if migrating from Broadcom/OCLP)](#before-you-start-if-migrating-from-broadcomoclp)
- [Wi-Fi setup](#wi-fi-setup)
  - [Option A: itlwm + HeliPort](#option-a-itlwm--heliport)
  - [Option B: AirportItlwm (native menu)](#option-b-airportitlwm-native-menu)
- [Bluetooth setup (Monterey+)](#bluetooth-setup-monterey)
- [Sleep: instant wake fix (Bluetooth)](#sleep-instant-wake-fix-bluetooth)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)
- [Credits / References](#credits--references)

---

## Hardware
You can buy the AX210 in two common formats:

1) **Prebuilt PCIe card** (AX210S PCIe, e.g. Ziyituod and similar)
2) **M.2/NGFF AX210 module + PCIe x1 adapter** (A+E Key)

<table>
<tr>
<td><img src="img/Card and adapter.png" alt="Card and adapter"></td>
<td><img src="img/AX210 card.jpg" alt="AX210 card"></td>
</tr>
</table>

---

## Before you start (if migrating from Broadcom/OCLP)
If you previously used **Broadcom/Fenvi + OCLP Root Patches**, revert those changes first.

### OpenCore config.plist cleanup
- Disable Broadcom/OCLP Wi-Fi kexts (examples):
  - `IOSkywalk.kext`
  - `IO80211FamilyLegacy.kext`
  - `AirPortBrcmNIC.kext`
- Remove/disable any **IOSkywalk blocking** entries you added for Broadcom.
- Restore SIP:
  - `csr-active-config` → `00000000`
- Restore Apple Secure Boot model:
  - `SecureBootModel` → anything **other than** `Disabled`

### OCLP
Open **OpenCore Legacy Patcher (OCLP)** → **Post-Install Root Patch** → **Revert Root Patches**.

---

## Wi-Fi setup
OpenIntelWireless provides two mutually exclusive approaches:

> **DO NOT use `itlwm.kext` and `AirportItlwm.kext` at the same time. Choose ONE.**

Kext downloads:
- itlwm releases (Wi-Fi): https://github.com/OpenIntelWireless/itlwm/releases (latest stable listed as **v2.3.0**)  
- HeliPort releases (app): https://github.com/OpenIntelWireless/HeliPort/releases (latest listed as **v1.3.1**)  

(References: itlwm v2.3.0 :contentReference[oaicite:4]{index=4}, HeliPort v1.3.1 :contentReference[oaicite:5]{index=5})

### Option A: itlwm + HeliPort
**Best compatibility**, works by exposing Wi-Fi as an Ethernet-like interface and you manage networks with **HeliPort**.

- Install:
  - `itlwm.kext` (OpenCore → `EFI/OC/Kexts`)
  - HeliPort app (installed in macOS)
- Pros:
  - Often the most reliable across macOS changes
- Cons:
  - No native Wi-Fi menu integration (HeliPort handles networks)

### Option B: AirportItlwm (native menu)
Uses Apple `IO80211Family` path so it behaves like “normal” Wi-Fi in macOS menus.

- Install:
  - `AirportItlwm.kext` (OpenCore → `EFI/OC/Kexts`)
- Pros:
  - Native Wi-Fi UI (no HeliPort)
- Cons / limitations:
  - **No AirDrop**
  - Continuity is limited (Handoff/Universal Clipboard may be partial)
  - Known limitations can vary by macOS build

### Version / macOS notes
- **macOS Sonoma 14.4** changed parts of the Wi-Fi stack; driver selection may differ by sub-version.
- **macOS Sequoia** and later may require updated builds; if AirportItlwm is not available/working, fall back to **itlwm + HeliPort**.

---

## Bluetooth setup (Monterey+)
For **macOS Monterey and newer**, OpenIntelWireless Bluetooth typically needs 3 kexts:

- `IntelBTPatcher.kext` (requires Lilu)
- `IntelBluetoothFirmware.kext`
- `BlueToolFixup.kext` (from Acidanthera’s BrcmPatchRAM package)

Downloads:
- IntelBluetoothFirmware releases (latest listed as **v2.4.0**): https://github.com/OpenIntelWireless/IntelBluetoothFirmware/releases :contentReference[oaicite:6]{index=6}

> Ensure Lilu is up to date if you use IntelBTPatcher.

---

## Sleep: instant wake fix (Bluetooth)
Some systems wake immediately after sleep when using Intel Bluetooth kexts.

A known workaround:
- Add `SSDT-GPRW.aml` to `EFI/OC/ACPI`
- Enable ACPI patch: “Change GPRW to XPRW, needs SSDT-GPRW.aml”

**Trade-off:** system may only wake via **power button** (no keyboard/mouse wake).

### SSDT-GPRW (example)
```c++
DefinitionBlock ("", "SSDT", 2, "DRTNIA", "GPRW", 0x00000000)
{
    External (XPRW, MethodObj)    // 2 Arguments
    Method (GPRW, 2, NotSerialized)
    {
        If (_OSI ("Darwin"))
        {
            If ((0x6D == Arg0))
            {
                Return (Package (0x02) { 0x6D, Zero })
            }
            If ((0x0D == Arg0))
            {
                Return (Package (0x02) { 0x0D, Zero })
            }
        }
        Return (XPRW (Arg0, Arg1))
    }
}
