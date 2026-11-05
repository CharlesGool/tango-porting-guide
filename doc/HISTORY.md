---
name: project-history
description: 历史记录
metadata:
  version: "1.0.0"
  lang: "zh-CN"
---

# 历史记录

## 多语言

**简体中文** | [English](en/HISTORY.md) | [Español](es/HISTORY.md)

## 文档

- 项目概览: [README](../README.md)

- 设计思路: [DESIGN](DESIGN.md)

- 项目状态: [LOG](LOG.md)
- 历史记录: [HISTORY](HISTORY.md)
- 变更日志: [CHANGELOG](CHANGELOG.md)

- 第三方声明: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## 历史记录

真机测试经过 (V68 到 V883) 见 [tb321fu-tango-patch 的 HISTORY](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/HISTORY.md#真机测试经过).

### 组织变更

- 2026-10-31: 5 个模块从 `mods/tango/` 移到 Mods 的 `patches/tango/`.
- 2026-11-01: 5 个补丁合并为一个补丁 `required/general/tango` 的 5 个部分, 补丁版本 V1.0.
- 2026-11 初 (日期未核实): 运行时命名从 omk 改为 herald (`heraldctl`, herald 域).
- 2026-11-05: 拆成独立私有仓库 tb321fu-tango.
- 2026-11-05: 改名 tango-porting-guide 并公开, 改为通用移植教程.
- 2026-11-05: 补丁拆到独立仓库 tb321fu-tango-patch, 本仓库只保留文档.
