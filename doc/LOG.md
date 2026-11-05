---
name: project-log
description: 项目缺陷, 限制, 决策和交接
metadata:
  version: "1.0.0"
  lang: "zh-CN"
---

# 日志

## 多语言

**简体中文** | [English](en/LOG.md) | [Español](es/LOG.md)

## 文档

- 项目概览: [README](../README.md)

- 设计思路: [DESIGN](DESIGN.md)

- 项目状态: [LOG](LOG.md)
- 历史记录: [HISTORY](HISTORY.md)
- 变更日志: [CHANGELOG](CHANGELOG.md)

- 第三方声明: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## 记录

- [历史记录](HISTORY.md)
- [变更日志](CHANGELOG.md)

## 缺陷

暂无. 参考实现的已知缺陷见 [tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/LOG.md#缺陷).

## 限制

- 教程只在 TB321FU (骁龙 8 Gen 3, Android 15) 上验证过, 其他设备和版本的步骤是根据小米和一加的官方集成推断的.
- Tango 是专有软件, 本项目不能提供翻译器或可直接刷入的镜像.

## 决策

| 日期 | 决策 | 状态 | 原因 |
| --- | --- | --- | --- |
| 2026-10-31 | 不改 `ro.zygote`, 自带 32 位 zygote 服务 | 接受 | vendor 的 `ro.zygote` 会覆盖 system 的值, 且原版 rc 会带入 32 位自检和连带重启. |
| 2026-10-31 | `ro.dalvik.vm.enable_uffd_gc=false` | 接受 | Tango 不支持 userfaultfd, 32 位 ART 只能用 CC GC. |
| 2026-10-31 | 开机保险写 `panic_on_walt_bug=0` 修复整机重启 | 接受 | 改 `sched-walt.ko` 需要重打 `vendor_boot`, sysctl 已在真机验证有效. |
| 2026-10-31 | 换用一加 12 的 32 位 GPU 库 | 拒绝 | 真因是 WALT 自检, 现有库在真机上正常. |
| 2026-11-01 | 5 个独立补丁合并为一个补丁的 5 个部分 | 接受 | 各部分功能上必须一起使用. |
| 2026-11-05 | 拆成独立仓库, 作为补丁的唯一来源, Mods 保留同步副本 | 接受 | 与 herald-core 的组织方式一致, 便于单独维护和发布. |
| 2026-11-05 | 仓库改名 tango-porting-guide 并公开, 定位为通用移植教程 | 接受 | 用户要求公开如何移植 Tango, 名称不带设备名. |
| 2026-11-05 | TB321FU 补丁拆到独立公开仓库 tb321fu-tango-patch, 本仓库只保留文档 | 接受 | 补丁只有一个来源, 教程面向所有设备. |

## 交接

- 当前分支: `main`.
- 已完成: 编写通用[移植教程](porting-guide.md)和[调研记录](research.md); 补丁拆到 [tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch), 本仓库只保留文档; 文档翻译为英文和西班牙语.
- 检查: 文档相对链接和外部链接检查通过. 教程未在第二台设备上验证.
- 剩余步骤:
  - [ ] 在第二台设备或另一个 Android 版本上验证教程, 修正标为需确认的内容.
- 阻碍: 无.
- 临时项目规则: 未发现.
