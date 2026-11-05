---
name: project-design-en
description: Project architecture and design constraints
metadata:
  version: "1.0.0"
  lang: "en"
---

# tango-porting-guide — Design

## Multi-language

[简体中文](../DESIGN.md) | **English** | [Español](../es/DESIGN.md)

## Documentation

- Project overview: [README](README.md)

- Design rationale: [DESIGN](DESIGN.md)

- Project status: [LOG](LOG.md)
- Historical records: [HISTORY](HISTORY.md)
- Version changelog: [CHANGELOG](CHANGELOG.md)

- Third-party notices: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## Design Goals

- [x] Explain the general steps for porting Tango into a compiled Android system image without changing source code.
- [x] Map every step to a reference implementation verified on a real device ([tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch), TB321FU, Android 15).
- [x] Provide a comparison with the vendors' official integrations and the root cause of the whole-device reboot, as the rationale for the steps.
- [ ] Verify the guide on a second device or another Android version.
- Non-goals: providing flashable images or patch code for any device.

## Architecture

This project contains documentation only:

| File | Content |
| --- | --- |
| [porting-guide.md](porting-guide.md) | General porting guide: prerequisites, 6 steps, verification and troubleshooting; each step links to the matching part of the reference implementation |
| [research.md](research.md) | Research notes: final approach, comparison with Xiaomi and OnePlus, root cause, QEMU tests |

| Top-level directory | Responsibility |
| --- | --- |
| `doc/` | Project documentation |

The reference implementation is in the separate repository [tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch). This project has no code, UI text or build output.

## Design Constraints

- The repository **MUST NOT** contain Tango, `tango32.ko` or other vendors' proprietary files.
- Guide content not verified on a real device **MUST** be marked as to be confirmed.
- Every step of the guide **MUST** map to a file in the reference implementation; when the reference implementation's paths change, the links **MUST** be updated accordingly.
