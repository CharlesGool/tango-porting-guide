---
name: project-log-en
description: Project bugs, limitations, decisions, and handoff
metadata:
  version: "1.0.0"
  lang: "en"
---

# Log

## Multi-language

[简体中文](../LOG.md) | **English** | [Español](../es/LOG.md)

## Documentation

- Project overview: [README](README.md)

- Design rationale: [DESIGN](DESIGN.md)

- Project status: [LOG](LOG.md)
- Historical records: [HISTORY](HISTORY.md)
- Version changelog: [CHANGELOG](CHANGELOG.md)

- Third-party notices: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## Records

- [History](HISTORY.md)
- [Changelog](CHANGELOG.md)

## Bugs

None yet. For known bugs of the reference implementation, see [tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/LOG.md#缺陷).

## Limitations

- The guide has only been verified on TB321FU (Snapdragon 8 Gen 3, Android 15); steps for other devices and versions are inferred from the Xiaomi and OnePlus official integrations.
- Tango is proprietary software; this project cannot provide the translator or flashable images.

## Decisions

| Date | Decision | Status | Reason |
| --- | --- | --- | --- |
| 2026-10-31 | Do not change `ro.zygote`; ship our own 32-bit zygote service | Accepted | The vendor's `ro.zygote` overrides the system value, and the stock rc would bring in the 32-bit self-test and the restart chained to it. |
| 2026-10-31 | `ro.dalvik.vm.enable_uffd_gc=false` | Accepted | Tango does not support userfaultfd, so 32-bit ART can only use the CC GC. |
| 2026-10-31 | Write `panic_on_walt_bug=0` in the boot guard to fix the whole-device reboot | Accepted | Changing `sched-walt.ko` requires repacking `vendor_boot`; the sysctl is verified effective on a real device. |
| 2026-10-31 | Switch to OnePlus 12's 32-bit GPU libraries | Rejected | The root cause is the WALT self-check, and the current libraries work on a real device. |
| 2026-11-01 | Merge 5 separate patches into 5 parts of one patch | Accepted | The parts must be used together functionally. |
| 2026-11-05 | Split into a separate repository as the sole source of the patch; Mods keeps a synced copy | Accepted | Consistent with how herald-core is organized; easier to maintain and release on its own. |
| 2026-11-05 | Rename the repository to tango-porting-guide and make it public, positioned as a general porting guide | Accepted | The user asked to publish how to port Tango; the name carries no device name. |
| 2026-11-05 | Move the TB321FU patch to the separate public repository tb321fu-tango-patch; this repository keeps only documentation | Accepted | The patch has a single source, and the guide targets all devices. |

## Handoff

The current handoff is maintained in the [Simplified Chinese LOG.md](../LOG.md#交接).
