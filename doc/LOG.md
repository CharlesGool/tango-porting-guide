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

- [ ] 默认 `ENFORCE=0` 时, `tango-guard.sh` 开机会删除 `fallback` 标记, 所以 `heraldctl tango off` 写的回退标记不生效.
- [ ] Tango v2.1.0 在 QEMU 上 32 线程程序走致命路径时偶发挂死, 或把 SIGSEGV 误报为 SIGABRT (v2.0.7 也有); 真机上未观察到.
- [ ] `panic_on_walt_bug=0` 同时关闭了联想 WALT 的其他 BUG 检查.
- [ ] 32 位 Adreno `V@0762.4` 与设备 64 位 `V@0762.28` 和内核 KGSL 的兼容性没有单独验证.

## 限制

- 不能按需启动 32 位 zygote, 也不能预翻译: 都需要修改 framework 或 installd 源码, 二进制 GSI 改不了.
- 不能分发完整刷机包: Tango EULA 禁止复制和再分发, 小米的库是专有文件.
- `tango32.ko` 只能用于发布串完全相同的内核, 换原厂 boot 后**必须**重编.

## 决策

| 日期 | 决策 | 状态 | 原因 |
| --- | --- | --- | --- |
| 2026-10-31 | 不改 `ro.zygote`, 自带 32 位 zygote 服务 | 接受 | vendor 的 `ro.zygote` 会覆盖 system 的值, 且原版 rc 会带入 32 位自检和连带重启. |
| 2026-10-31 | `ro.dalvik.vm.enable_uffd_gc=false` | 接受 | Tango 不支持 userfaultfd, 32 位 ART 只能用 CC GC. |
| 2026-10-31 | 开机保险写 `panic_on_walt_bug=0` 修复整机重启 | 接受 | 改 `sched-walt.ko` 需要重打 `vendor_boot`, sysctl 已在真机验证有效. |
| 2026-10-31 | 换用一加 12 的 32 位 GPU 库 | 拒绝 | 真因是 WALT 自检, 现有库在真机上正常. |
| 2026-11-01 | 5 个独立补丁合并为一个补丁的 5 个部分 | 接受 | 各部分功能上必须一起使用. |
| 2026-11-05 | 拆成独立仓库, 作为补丁的唯一来源, Mods 保留同步副本 | 接受 | 与 herald-core 的组织方式一致, 便于单独维护和发布. |
| 2026-11-05 | 仓库改名 tango-porting-guide 并公开, 定位为通用移植教程, TB321FU 补丁作为参考实现放在 `examples/tb321fu/` | 接受 | 用户要求公开如何移植 Tango, 名称不带设备名. |

## 交接

- 当前分支: `main`.
- 已完成: 从 TB321FU-GSI-Mods `main` 提取 Tango 补丁 (代码未改) 到 `examples/tb321fu/`; 按项目模板建立文档; 补丁说明按现行代码订正; 调研记录整理为 [research.md](research.md); 编写通用[移植教程](porting-guide.md); 仓库改名为 tango-porting-guide 并公开.
- 检查: shell 脚本 `bash -n` 和 Python `py_compile` 通过; 文档相对链接和外部链接检查通过. 本仓库内没有可运行的构建或测试, 参考实现未在本仓库重新构建, 教程未在第二台设备上验证.
- 剩余步骤:
  - [ ] 在 Mods 中改为从本仓库同步 `examples/tb321fu/` (参照 herald-core 的 `make sync-core`), 并让 Mods 文档指向本仓库.
  - [ ] 修复 `heraldctl tango off` 在 `ENFORCE=0` 时不生效的缺陷.
  - [ ] 向 Amanieu Systems 确认 Tango 集成文件 (binfmt 注册串, sepolicy) 的授权, 见 [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md#合规审查).
- 阻碍: 无.
- 临时项目规则: 未发现.
