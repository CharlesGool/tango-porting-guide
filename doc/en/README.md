# tango-porting-guide

<!-- Uncomment the next line when the project has an icon -->
<!-- <img align="right" src="../resources/logo.png" width="160" alt="tango-porting-guide"> -->

[简体中文](../../README.md) | **English** | [Español](../es/README.md)

Android devices whose CPU lacks AArch32 cannot run 32-bit APKs; this project explains how to port the Tango binary translator into a compiled system image without changing source code, so 32-bit APKs run without root, and provides a reference implementation verified on a real device.

[![License](https://img.shields.io/badge/License-GPL--3.0--or--later-orange)](../../LICENSE)

## Documentation

- Project overview: [README](README.md)

- Design rationale: [DESIGN](DESIGN.md)

- Project status: [LOG](LOG.md)
- Historical records: [HISTORY](HISTORY.md)
- Version changelog: [CHANGELOG](CHANGELOG.md)

- Third-party notices: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## Introduction

| Content | Description |
| --- | --- |
| [Porting guide](porting-guide.md) | Prerequisite checks, 6 porting steps, staged verification and a troubleshooting table |
| [Research notes](research.md) | Item-by-item comparison with the official integrations on the Xiaomi 14 Ultra, OnePlus 12 and OnePlus 13T; the root cause of the whole-device reboot after declaring the 32-bit ABI (the 32-bit task check in the Qualcomm WALT scheduler); QEMU tests |
| [Reference implementation](https://github.com/CharlesGool/tb321fu-tango-patch) | Complete patch for the Lenovo Legion Y700 Gen 3 TB321FU (Snapdragon 8 Gen 3, Android 15); V883 (2026-10-31) runs 32-bit APKs correctly on a real device |

[Tango](https://www.amanieusystems.com/) is an ARM32-to-ARM64 binary translator from Amanieu Systems. Its official integration patches the AOSP source; the method in this project only rewrites the system image, and suits GSIs and ROMs whose source is unavailable.

Devices and Android versions other than the reference implementation's are unverified. For the known issues of the reference implementation, see [LOG of tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/LOG.md#缺陷).

## Requirements

- An arm64 device whose CPU has no AArch32; kernel ≥ 5.6 with `CONFIG_COMPAT` and `CONFIG_BINFMT_MISC` enabled
- Access to the device kernel's source and config, used to build `tango32.ko`
- A system image that ships 32-bit libraries (`app_process32`, 32-bit bionic and framework native libraries)
- An unlocked bootloader, so the system image can be modified and flashed
- The Tango integration package, obtained from Amanieu Systems with its EULA accepted

For item-by-item check methods, see the [prerequisites of the porting guide](porting-guide.md#prerequisites).

## Install

This project is a guide and reference implementation; no installation is needed.

### Quick Install

```bash
git clone https://github.com/CharlesGool/tango-porting-guide.git
```

### Normal Install

The reference implementation lives in the separate repository [tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch), a patch for [TB321FU-GSI-Mods](https://github.com/CharlesGool/TB321FU-GSI-Mods). For build parameters and usage, see that repository.

```bash
git clone https://github.com/CharlesGool/tb321fu-tango-patch.git
```

## Usage

1. Check the target device against the [prerequisites](porting-guide.md#prerequisites).
2. Rewrite the system image by steps 1 to 6 of the [porting guide](porting-guide.md), comparing each step with the matching part of the [reference implementation](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/README.md).
3. Flash stage 1 first (no 32-bit ABI declared) to verify the translator and the kernel module, then flash stage 2; see [verification](porting-guide.md#verification).
4. If the device fails to boot or reboots entirely, consult the [troubleshooting table](porting-guide.md#troubleshooting).

## Upgrade

None yet.

## Uninstall

No uninstall is needed; delete the cloned directory. To remove Tango from a device, flash back a system image that does not contain it.

## Acknowledgements

- [Tango](https://www.amanieusystems.com/) by Amanieu Systems: source of the translator, the `tango32` kernel module, the binfmt registration and the SELinux policy
- The official Tango integrations on the Xiaomi 14 Ultra and OnePlus 12: reference for the 32-bit zygote and ART GC approach

## License

This project is licensed under the GNU General Public License v3.0 or later, SPDX identifier `GPL-3.0-or-later`; the full text is in [LICENSE](../../LICENSE). This project does not include Tango, the `tango32` module or other vendors' 32-bit libraries; see [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md).
