---
name: project-history-en
description: Historical Records
metadata:
  version: "1.0.0"
  lang: "en"
---

# Historical Records

## Multi-language

[简体中文](../HISTORY.md) | **English** | [Español](../es/HISTORY.md)

## Documentation

- Project overview: [README](README.md)

- Design rationale: [DESIGN](DESIGN.md)

- Project status: [LOG](LOG.md)
- Historical records: [HISTORY](HISTORY.md)
- Version changelog: [CHANGELOG](CHANGELOG.md)

- Third-party notices: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## Historical Records

The real-device test history (V68 to V883) is in [HISTORY of tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/HISTORY.md#真机测试经过).

### Organizational Changes

- 2026-10-31: Moved the 5 modules from `mods/tango/` to `patches/tango/` in Mods.
- 2026-11-01: Merged the 5 patches into the 5 parts of a single patch, `required/general/tango`, patch version V1.0.
- Early 2026-11 (date not verified): Renamed the runtime from omk to herald (`heraldctl`, herald domain).
- 2026-11-05: Split into a separate private repository, tb321fu-tango.
- 2026-11-05: Renamed to tango-porting-guide and made public, turned into a general porting guide.
- 2026-11-05: Moved the patch to the separate repository tb321fu-tango-patch; this repository keeps only documentation.
