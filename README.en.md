# Huawei MateBook 14 (2019) — macOS Tahoe 26 Hackintosh EFI

[English](README.en.md) | [简体中文](README.md)

[![macOS](https://img.shields.io/badge/macOS-26.7.3-black?logo=apple)](https://www.apple.com/macos/)
[![OpenCore](https://img.shields.io/badge/OpenCore-1.0.9-blue)](https://github.com/acidanthera/OpenCorePkg)
[![Status](https://img.shields.io/badge/status-working-brightgreen)]()

OpenCore configuration and a complete troubleshooting log for running **macOS Tahoe 26.7.3** on a Huawei MateBook 14 (2019).

> ⚠️ **No Apple copyrighted files are included. SMBIOS values are blanked — generate your own before use.**

---

## Hardware

| Component | Model | Status |
|---|---|---|
| Laptop | Huawei MateBook 14 2019 (KLV-W19 / KLV-W29) | — |
| CPU | Intel Core i5-8265U (Whiskey Lake-U, 4C/8T) | ✅ all 8 threads |
| iGPU | Intel UHD Graphics 620 (real ID `0x3EA0`, spoofed `0x3EA5`) | ✅ 1536 MB, HW decode works |
| dGPU | NVIDIA MX250 | ❌ unsupported, disabled via SSDT `_OFF` |
| RAM | 8 GB LPDDR3 2133 MHz (soldered) | ✅ |
| Storage | WDC PC SN720 SDAPNTW-512G NVMe | ✅ native, no NVMeFix required |
| Display | 14" 2160×1440 (3:2) | ✅ internal panel + brightness |
| Wi-Fi | Intel CNVi (PCI `8086:9df0`, 9560 class) | ✅ `AirportItlwm_Tahoe`, native UI |
| Bluetooth | same CNVi module | ✅ |
| Audio | Realtek ALC256 | ⚠️ see "Open items" |
| Touchpad | Synaptics I2C HID | ✅ `VoodooI2C` + `VoodooI2CHID` |

## Status

| Feature | State | Notes |
|---|---|---|
| Internal display / brightness | ✅ | `framebuffer-con0` + PNLF |
| Metal acceleration | ✅ | UHD 620, 1536 MB |
| Wi-Fi | ✅ | `AirportItlwm_Tahoe` 2.4.0, no HeliPort needed |
| Bluetooth | ✅ | `IntelBluetoothFirmware` 2.5.2 + `BTPatcher` + `BlueToolFixup` 2.7.3 |
| Keyboard / touchpad | ✅ | `VoodooPS2` + `VoodooI2C` |
| Battery percentage | ✅ | `SMCBatteryManager` |
| Sleep / wake | ⏳ | not soak-tested |
| Audio (ALC256) | ⏳ | `AppleALC 1.9.9` + `SimpleHDA`, layout 76 — see [docs/AUDIO-FIX.md](docs/AUDIO-FIX.md) |
| HDMI output | ⏳ | connector patches carried over from the working 14.5 setup |
| SD reader / fingerprint | ❌ | unsupported |
| MX250 | ❌ | unsupported (disabled by SSDT to save power) |

---

## Installation (100% from Windows, no Mac required)

This is the most valuable part of the project: **installing from scratch without access to a Mac**.

### 1. Get a recovery image

```cmd
python macrecovery.py -b Mac-27AD2F918AE68F61 -m 0000000000000000 -os latest download
```

You get `com.apple.recovery.boot\BaseSystem.dmg` (~916 MB) + `BaseSystem.chunklist`.

### 2. ⚠️ Do NOT boot `com.apple.recovery.boot` directly

> **The single most important lesson of this project.**

When booting a `(dmg)` recovery entry, OpenCore **reads the whole 916 MB image into RAM**:

```
OCRAM: Extent allocation of 960530321 bytes (A) gave 100000000
```

On an 8 GB machine this reliably resets the box. Symptom: *black screen for ~30 s after selecting the recovery entry, then it reboots into Windows.*

Restore the image to a **real HFS+ partition** instead:

```bash
dmg2img -i BaseSystem.dmg -o BaseSystem.img     # ~2.49 GB, a complete GPT disk image
```

On Windows use [dmg2img Win32](http://vu1tur.eu.org/tools/), then write the `.img` to a USB stick in **DD mode** with Rufus or balenaEtcher.

You end up with a **2.4 GB Apple HFS partition** + unallocated space.

### 3. Create the EFI partition in the remaining space

With DiskGenius:

1. Right-click the unallocated space → **Create New Partition** → **FAT32** → Save Changes
2. Right-click the new partition → **Modify Partition Parameters** → type: **EFI System Partition**
3. Copy this repo's `EFI` folder to the root of that partition

> Never touch the 2.4 GB HFS partition afterwards (no format, no repartition).

### 4. BIOS

- Secure Boot → **Disabled**
- Fast Boot → **Disabled**
- Boot mode: UEFI only

### 5. Install

1. Press F12, boot the USB → OpenCore menu → pick the entry **without the `(dmg)` suffix**
2. macOS Recovery → Disk Utility → **⌘2 (Show All Devices)**
3. Select the target partition (your Linux partition shows as `Unknown` format) → **Erase** → APFS
4. Quit Disk Utility → Reinstall macOS → requires internet (~15 GB download)

> "About 9 hours remaining" at the start is a wildly pessimistic estimate; it drops within ~10 minutes.

### 6. Reboots during installation

**You must select the right entry in OpenCore manually after every reboot:**

```
macOS Installer        ← pick this while it exists
Tahoe                  ← pick the target volume once the installer is gone
```

---

## EFI composition

### OpenCore

**1.0.9** (release) with the **Builtin text picker**.

> ⚠️ Do not use `PickerMode=External` with an incomplete theme: you get
> `OCUI: Failed to load image ...Background.icns - Not Found`, OpenCanopy refuses to render, **the menu never appears at all**, and the firmware falls through to Windows.

### ACPI

| File | Purpose |
|---|---|
| `SSDT-MB14.aml` | PNLF brightness, dGPU `_OFF`, USBX, fake EC, `_Q0A/_Q0B/_Q10` hotkeys |

Three `ACPI → Patch` renames: `_Q0A→XQ0A`, `_Q0B→XQ0B`, `_Q10→XQ10`.

### Kexts

| Kext | Version |
|---|---|
| Lilu | 1.7.3 |
| VirtualSMC | 1.3.9 |
| WhateverGreen | 1.7.2 |
| RestrictEvents | 1.1.7 |
| NVMeFix | 1.1.4 |
| SMCBatteryManager | 1.3.9 |
| VoodooPS2Controller | 2.3.8 |
| VoodooI2C / VoodooI2CHID | 2.9.1 / 1.0 |
| AirportItlwm_Tahoe | 2.4.0 |
| IntelBluetoothFirmware / BTPatcher | 2.5.2 |
| BlueToolFixup | 2.7.3 |
| AppleALC | 1.9.9 |
| HibernationFixup | 1.5.5 |

### Key settings

```
boot-args:
  -v keepsyms=1 -wegnoegpu -igfxblr -igfxblt -igfxonln=1
  revpatch=sbvmm -no_compat_check
```

| Setting | Value | Why |
|---|---|---|
| `Kernel → Quirks → DisableIoMapper` | **true** | Huawei's Insyde BIOS does not expose a VT-d switch, so VT-d is always on. With `false`, macOS hangs during PCI enumeration and the firmware resets the machine — looks exactly like "black screen, reboots after a minute". **The sneakiest trap in this project.** |
| `Kernel → Quirks → CustomSMBIOSGuid` | false | mutually exclusive with `UpdateSMBIOSMode=Create` |
| `Booter → Quirks → DevirtualiseMmio` | false | not needed by this firmware; kept at the value proven on macOS 14.5 |
| `Misc → Security → SecureBootModel` | Disabled | — |
| `NVRAM → revpatch` | `sbvmm` | MacBookPro15,2 is not on Tahoe's support list; RestrictEvents spoofs the board-id |
| GPU `AAPL,ig-platform-id` | `0x3EA504` | proven on this panel under 14.5 |
| GPU `device-id` | `0x3EA5` (spoofed from `0x3EA0`) | — |
| GPU `force-online` | `01000000` | Tahoe may fail eDP detection and output to a dead connector |
| `UEFI → Drivers` | HfsPlus / OpenRuntime / ResetNvramEntry | no OpenCanopy |

---

## Pitfalls cheat sheet

Full write-up with real log excerpts: [docs/PITFALLS.md](docs/PITFALLS.md)

| # | Symptom | Real cause | Fix |
|---|---|---|---|
| 1 | Recovery → black screen ~1 min → reboots into Windows | `DisableIoMapper=false`, Huawei BIOS can't disable VT-d | set `true` |
| 2 | Same, still black after the fix | GPU missing `force-online` | add `01000000` |
| 3 | Copying the T490 config breaks boot | T490-only leftovers: `ReservedMemory(0x8B000)`, `BlessOverride(fedora)`, `HibernateMode=NVRAM`, `IOPCIFamily` patches | delete all of them |
| 4 | **Menu never appears at all** | theme pack missing `Background.icns`; OpenCanopy fails to render | `PickerMode=Builtin` |
| 5 | With DEBUG logging, reboots at ~60 s mid-log | each log line costs ~180 ms (screen + FAT write); 401 lines ≈ 60 s → firmware timeout | `Target=65` (file only) or `0` |
| 6 | No `opencore-*.txt` produced | release builds don't do file logging; `Target` must include `0x40` | DEBUG build + `Target=67` |
| 7 | Black ~30 s, log ends at `OCRAM: Extent allocation of 960530321 bytes` | OpenCore loads the whole DMG into RAM | use `dmg2img` + real partition |
| 8 | Stuck at `Waiting on ... boot-uuid-media`, last line `MODE_SENSE_06 failed` | that USB stick can't publish a block device under macOS (firmware can read it, macOS can't) | different USB stick / SD card + reader |
| 9 | Touchpad and keyboard dead | `config-minimal.plist` has only 5 kexts — no input drivers | expected; use the full config |
| 10 | Battery icon on screen | macOS "low battery, plug in" | plug in the charger |
| 11 | *An Internet connection is required to install macOS* | BaseSystem downloads ~15 GB | connect Wi-Fi (`AirportItlwm_Tahoe`) |
| 12 | Linux partition invisible in Disk Utility | ext4 isn't mountable, only mountable volumes are listed | **⌘2**, or set Format → APFS in the Partition dialog |
| 13 | *This volume can't be resized* | clicked "Partition" instead of "Erase" | change Format → APFS → Apply |

---

## Repository layout

```
.
├── EFI/
│   ├── BOOT/BOOTx64.efi
│   └── OC/
│       ├── ACPI/SSDT-MB14.aml
│       ├── Drivers/{HfsPlus,OpenRuntime,ResetNvramEntry}.efi
│       ├── Kexts/                16 kexts (14 enabled by config.plist)
│       ├── Resources/            Fonts only (Builtin picker needs no images)
│       ├── config.plist          default: input + Wi-Fi + battery (used for the install)
│       ├── config-full.plist     adds Bluetooth / audio / hibernation
│       ├── config-minimal.plist  5 kexts, for bisecting
│       └── config-verbose.plist  troubleshooting: keeps -v
├── docs/PITFALLS.md   full troubleshooting log
├── docs/FINALIZE.md   remove -v + move EFI to the internal ESP
├── docs/AUDIO-FIX.md  audio: AppleALC + SimpleHDA + layout-id troubleshooting
├── LICENSE
└── README.md / README.en.md
```

## Before you use this

### 1. Generate your own SMBIOS

SMBIOS values are blanked. Use [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) or OCAT for **MacBookPro15,2**:

- `SystemSerialNumber`, `MLB`, `SystemUUID`, `ROM` (6-byte MAC, e.g. `112233445566`)

Put them in `PlatformInfo → Generic` and set `UpdateNVRAM` back to `true`.

> MacBookPro15,2 is not officially supported by Tahoe; booting relies on `revpatch=sbvmm` + `-no_compat_check`. Don't switch to MacBookPro16,x — it drags in T2-related behaviour and changes USB port mapping.

### 2. Check `DisableIoMapper`

Follow Dortania's guide for your own firmware. **On this MateBook it must be `true`.**

## Open items

- [ ] Audio (ALC256 / layout 76): AppleALC + SimpleHDA added, pending verification
- [ ] Sleep, wake, lid-close not soak-tested
- [ ] HDMI output not verified
- [ ] No `SSDT-PLUG` — CPU power management left at defaults
- [ ] No `USBMap` — USB 3.0 throughput untested
- [ ] iServices (iMessage / FaceTime) requires your own SMBIOS

## Credits

- [Acidanthera](https://github.com/acidanthera) — OpenCore / Lilu / VirtualSMC / WhateverGreen / RestrictEvents
- [5T33Z0 / Thinkpad-T490-Hackintosh-OpenCore](https://github.com/5T33Z0/Thinkpad-T490-Hackintosh-OpenCore) — same i5-8265U; source of the kext versions and overall structure
- [laobamac](https://github.com/laobamac) — Tahoe-compatible `AirportItlwm`
- [VoodooI2C](https://github.com/VoodooI2C) — Synaptics I2C touchpad

## License

Configuration and documentation are MIT. All bundled binaries belong to their respective projects and keep their original licenses.
