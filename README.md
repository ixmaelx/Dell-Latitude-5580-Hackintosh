# Dell Latitude 5580 — OpenCore EFI (macOS Ventura)

Working OpenCore EFI for the **Dell Latitude 5580** running **macOS 13.7.8 Ventura**.
Nearly everything works. Shared to save others the trial-and-error.

> ⚠️ **READ THIS FIRST — you MUST generate your own SMBIOS before booting.**
> The `SystemSerialNumber`, `MLB`, `SystemUUID` and `ROM` in this EFI are **intentionally blank/zeroed**.
> Booting with empty or shared serials will break iMessage/FaceTime and can burn a serial. See [Setup](#setup) below.

## Hardware

| | |
|---|---|
| **Model** | Dell Latitude 5580 (P60F) |
| **CPU** | Intel Core i7-7820HQ (Kaby Lake) |
| **iGPU** | Intel HD Graphics 630 (`AAPL,ig-platform-id` = `0x591B0000`) |
| **dGPU** | NVIDIA GeForce 940MX — **disabled** (`-wegnoegpu`), no macOS drivers |
| **Audio** | ALC (AppleALC, `layout-id` 33 — try 11/13/28 if yours differs) |
| **Wi-Fi/BT** | Intel Dual Band Wireless-AC 8265 |
| **Touchpad** | ALPS I²C (dual-pointing, with TrackPoint) |
| **SMBIOS** | MacBookPro14,3 |

## What works / what doesn't

| Component | Status | Notes |
|---|---|---|
| Internal display (1080p) | ✅ | Full QE/CI, Metal 3 |
| Intel HD 630 acceleration | ✅ | |
| Audio (speakers, mic, jack) | ✅ | AppleALC |
| **Wi-Fi** | ✅ | AirportItlwm **2.3.0** — ⚠️ **macOS-version-locked**, see [below](#-wi-fi-is-tied-to-your-exact-macos-version) |
| **Bluetooth** | ✅ | Intel 8265 — see [Bluetooth fix](#bluetooth-intel-8265--ventura-134) |
| Battery / sleep / lid | ✅ | SMCBatteryManager |
| Keyboard + backlight | ✅ | VoodooPS2 |
| **Touchpad gestures + TrackPoint + 3 buttons** | ✅ | see [TrackPoint fix](#trackpoint-buttons--alpshid) |
| CPU temp / fan sensors | ✅ | SMCProcessor + SMCSuperIO (read with Hot, Stats, iStat…) |
| Ethernet | ✅ | IntelMausi |
| USB | ✅ | (port map not included — see notes) |
| **HDMI / external display** | ❌ | **Hard-wired to the NVIDIA dGPU** — impossible in macOS. See below. |

### About the HDMI port (important)

The built-in **HDMI is physically wired to the NVIDIA 940MX**, which has **no macOS drivers** (last NVIDIA Web Drivers were High Sierra 10.13; OCLP only re-enables Kepler, not Maxwell). Verified with a live hot-plug test: **zero** display/ACPI events when connecting HDMI → the iGPU has no electrical path to that port. USB-C DP Alt Mode is also routed via Thunderbolt/NVIDIA and does not output video either.

**Only working second-screen option: a DisplayLink USB adapter** (software driver, GPU-independent) + DisplayLink Manager app. This is a hardware-wiring limitation, not a config issue — no framebuffer patch can fix it.

### ⚠️ Wi-Fi is tied to your exact macOS version

This EFI uses **AirportItlwm 2.3.0 built for macOS Ventura 13.7**. **AirportItlwm has a separate build for every macOS version** — the one here will **fail to load (no Wi-Fi) on a different macOS version**. If you run anything other than Ventura 13.7.x:

- Download the AirportItlwm build **matching your exact macOS version** from [OpenIntelWireless releases](https://github.com/OpenIntelWireless/itlwm/releases) and replace `Kexts/AirportItlwm.kext`.
- **Or** use **`itlwm.kext`** (already included in `Kexts/`, disabled by default): it is *not* version-locked and works on any macOS, but doesn't integrate with the native Wi-Fi menu — you control it with the [HeliPort](https://github.com/OpenIntelWireless/HeliPort) app. To switch: disable `AirportItlwm.kext` and enable `itlwm.kext` in `Kernel > Add`.

Same idea applies when you **update** macOS: update AirportItlwm to the matching build first, or you'll lose Wi-Fi after the update.

## Setup

1. **Generate your own SMBIOS** with [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) for `MacBookPro14,3`, then fill in `PlatformInfo > Generic`:
   - `SystemSerialNumber`, `MLB`, `SystemUUID`
   - `ROM` → use your machine's real Ethernet MAC (en0), or the last 6 bytes of your Wi-Fi MAC.
2. **BIOS settings** (F2 at boot):
   - Disable **Secure Boot**
   - Set **SATA Operation → AHCI**
   - Disable **CFG Lock** (or keep `AppleXcpmCfgLock` quirk enabled, which this EFI does)
   - Set **DVMT Pre-Allocated → higher if possible**; if you can set it in BIOS, remove `framebuffer-stolenmem`/`framebuffer-fbmem` from the iGPU device props
   - Enable **Virtualization (VT-x)** if you want VMs (Hypervisor.framework needs it)
3. Copy `EFI/` to your USB/ESP, boot, install/boot Ventura.

## Notable fixes (the valuable bits)

### Bluetooth (Intel 8265) — Ventura 13.4+
Two things were needed:
1. Kexts: `BlueToolFixup`, `IntelBluetoothFirmware`, `IntelBTPatcher` in `Kernel > Add`.
2. **The Ventura 13.4+ NVRAM fix** — otherwise `bluetoothd` crash-loops (EXC_GUARD, "Address: NULL") even though firmware uploads fine. Under `NVRAM > Add` **and** `NVRAM > Delete`, GUID `7C436110-AB2A-4BBB-A880-FE41995C9F82`:
   - `bluetoothInternalControllerInfo` = `<0000000000000000000000000000>` (14 bytes)
   - `bluetoothExternalDongleFailed` = `<00>` (1 byte)

   (Add writes fresh zeros; Delete wipes stale/corrupt values each boot.)

### TrackPoint buttons — AlpsHID
Plain VoodooI2C gives touchpad gestures but the **TrackPoint's 3 physical buttons don't work** (known VoodooI2C ALPS limitation). Fix = **[AlpsHID](https://github.com/blankmac/AlpsHID) v1.2** + the **patched VoodooI2CHID** that ships in the AlpsHID release zip (the stock VoodooI2CHID lacks the needed fix). Load order: `VoodooI2C` → `VoodooI2CHID` → `AlpsHID`. Result: gestures **and** TrackPoint **and** all 3 buttons work together.

## Kexts

AirportItlwm/itlwm 2.3.0 · AppleALC 1.9.7 · Lilu 1.7.2 · VirtualSMC 1.3.7 (+SMCBatteryManager, SMCLightSensor, **SMCProcessor, SMCSuperIO** — all 1.3.7) · WhateverGreen 1.7.0 · IntelMausi 1.0.8 · RestrictEvents 1.1.6 · **BlueToolFixup 2.7.2 · IntelBluetoothFirmware 2.4.0 · IntelBTPatcher 2.4.0** · **AlpsHID 1.2 · VoodooI2C 2.9.1 · VoodooI2CHID (juico-patched) · VoodooInput 1.1.6** · VoodooPS2Controller 2.3.7 · USBToolBox

> **Kext versions matter.** VirtualSMC plugins (SMCProcessor/SMCSuperIO/etc.) must match VirtualSMC's version. AirportItlwm must match your macOS version (see Wi-Fi note). When updating macOS, update Lilu + all Lilu-based kexts together.

## ACPI (SSDTs)

SSDT-BRT6, SSDT-EC-USBX, SSDT-GPI0, SSDT-GPRW, SSDT-HPET, SSDT-I2C, SSDT-PLUG, SSDT-PNLF, SSDT-SBUS-MCHC, SSDT-UPRW, SSDT-XOSI

## Credits

Based on the Dell Latitude 5480 Hackintosh work by [QuanTrieuPCYT](https://github.com/QuanTrieuPCYT/Dell-Latitude-5480_Hackintosh), plus [Acidanthera](https://github.com/acidanthera), [OpenIntelWireless](https://github.com/OpenIntelWireless), [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) and [blankmac/AlpsHID](https://github.com/blankmac/AlpsHID).

---
*Provided as-is. Always generate your own SMBIOS. Not affiliated with Apple or Dell.*
