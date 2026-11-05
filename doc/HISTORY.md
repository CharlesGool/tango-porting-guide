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

### 真机测试经过

| 版本 | 日期 | 现象 | 原因和处理 |
| --- | --- | --- | --- |
| V68 | 2026-10 | 卡第一屏开机循环, `reboot,boringssl-self-check-failed` | 联想 vendor 的 32 位 BoringSSL 自检程序不存在且 `reboot_on_failure`, 与 Tango 无关; 改为接管 vendor 自检 |
| V77 | 2026-10 | 卡第二屏, system_server 每 60 秒崩溃 | system 的 `ro.zygote=zygote64_32` 被 vendor 的 `zygote64` 覆盖, 32 位 zygote 没起, `abilist32` 却已声明 |
| V78 | 2026-10 | 卡第二屏, 32 位 zygote 每 5 秒重启 | init 在 `early-init` 的 `insmod` 没加载成功, 32 位库 CFI 检查 SIGILL |
| V79 | 2026-10 | 开机动画中途整机硬重启 | herald 域 (当时叫 omk) 在 `post-fs-data` 加载模块成功, 32 位 zygote 运行约 8 秒后重启; 另发现 32 位启动镜像与 CC GC 不匹配 |
| V80, V81 | 2026-10 | 不声明 32 位 ABI 时能开机 | 模块可在 `post-fs-data` 加载; 改为该时机加载 |
| V82 | 2026-10 | 带开机保险, 现象同 V79 | 保险在下一次开机回退纯 64 位 |
| V881 | 2026-10-31 | 32 位 zygote 运行后约 25 秒整机重启 | 回退时原版 `init.rc` 拉起 `zygote_secondary` 触发 `updatable_crashing`; `enable_uffd_gc` 只改 system 无效; 旧标记未清; `guard.txt` 被覆盖. V882 全部修复, 服务改名 `tango_zygote32` |
| V882 | 2026-10-31 | 带 32 位开机约 28 秒后内核 `BUG` | console-ramoops 指向联想 `sched-walt.ko` 的 32 位任务自检 |
| V883 | 2026-10-31 | 32 位 APK 正常运行 | 开机保险写 `panic_on_walt_bug=0`. 用户确认通过 |

V68 到 V883 是 TB321FU-GSI-Mods 前身工程的整体版本号, 不是本补丁的版本号.

### 组织变更

- 2026-10-31: 5 个模块从 `mods/tango/` 移到 Mods 的 `patches/tango/`.
- 2026-11-01: 5 个补丁合并为一个补丁 `required/general/tango` 的 5 个部分, 补丁版本 V1.0.
- 2026-11 初 (日期未核实): 运行时命名从 omk 改为 herald (`heraldctl`, herald 域).
- 2026-11-05: 拆成独立私有仓库 tb321fu-tango.
- 2026-11-05: 改名 tango-porting-guide 并公开, 改为通用移植教程, 补丁移到 `examples/tb321fu/` 作为参考实现.
