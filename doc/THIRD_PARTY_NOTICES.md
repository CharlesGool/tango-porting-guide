---
name: project-third-party-notices
description: 第三方署名与合规声明
metadata:
  version: "1.0.0"
  lang: "zh-CN"
---

# 第三方声明

## 多语言

**简体中文** | [English](en/THIRD_PARTY_NOTICES.md) | [Español](es/THIRD_PARTY_NOTICES.md)

## 文档

- 项目概览: [README](../README.md)

- 设计思路: [DESIGN](DESIGN.md)

- 项目状态: [LOG](LOG.md)
- 历史记录: [HISTORY](HISTORY.md)
- 变更日志: [CHANGELOG](CHANGELOG.md)

- 第三方声明: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## 第三方声明

### 组件清单

| 组件名称 | 版本或哈希 | 上游地址 | 许可证类型 | 使用方式 | 版权归属 | 发布合规义务 | 核验日期 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Tango Binary Translator (`tango_translator`) | v2.1.0 | https://www.amanieusystems.com/ | 专有 (Tango EULA) | 构建输入 `TANGO_PKG`, 不在仓库中 | Amanieu Systems Ltd | **禁止**复制, 再分发和用于商业目的; 使用者自行取得并接受 EULA | 2026-11-05 |
| Tango 集成文件 (`binfmt_misc` 注册串, `tango.te`, `te_macros`) | v2.1.0 集成包 | https://www.amanieusystems.com/ | 未单独声明, 不确定是否受 EULA 约束 | 改写后进入 `patch/parts/001.core/files/tango-binfmt.rc` 和 `patch/parts/002.sepolicy/tango.cil` | Amanieu Systems Ltd | 不确定, 见合规审查 | 2026-11-05 |
| `tango32` 内核模块 | v2.1.0 集成包 | https://www.amanieusystems.com/ | GPL-2.0 (源文件 SPDX 标记) | 源码和编好的 `tango32.ko` 都是构建输入, 不在仓库中 | Amanieu Systems Ltd | 分发含 `tango32.ko` 的镜像时需提供对应源码 | 2026-11-05 |
| 小米 14 Ultra 32 位 GPU 和 gralloc 库 | xiaomi.eu OS3.0.304.0, 30 个文件, 哈希见 `patch/parts/004.vendor-lib32/manifest.sha256` | 小米官方固件 | 专有 | 构建输入 `XIAOMI_LIB32_DIR`, 仓库只含文件名和 SHA-256 | 小米, 高通 | **禁止**进仓库和分发的刷机包 | 2026-11-05 |
| AOSP `init.zygote64_32.rc` 的 `zygote_secondary` 服务定义 | Android 15 | https://android.googlesource.com/platform/system/core/ | Apache-2.0 | 修改后进入 `patch/parts/005.zygote32/files/tango-zygote32.rc` | The Android Open Source Project | 保留许可证文本, 标明修改 | 2026-11-05 |

### 许可证文本

- Tango EULA: 随 Tango 集成包提供 (`EULA.txt`), 本仓库不包含.
- GPL-2.0: 随 Tango 集成包的 `kernel-module/` 源码提供, 本仓库不分发该组件.
- Apache-2.0: [third_party/aosp/LICENSE](../third_party/aosp/LICENSE).

### 源代码提供

本仓库不分发 `tango32.ko`. 使用者分发含 `tango32.ko` 的镜像时, 需要按 GPL-2.0 提供 Tango 集成包中的 `kernel-module/` 源码和 [编译方法](../patch/parts/003.kernel-module/README.md#编译-tango32ko).

### 合规审查

- `tango32` (GPL-2.0) 是独立的可加载内核模块, 不与本项目的 GPL-3.0 代码链接, 本仓库也不包含其源码或二进制.
- `tango-binfmt.rc` 的注册串和 `tango.cil` 改写自 Tango 集成包. 集成包只附带面向翻译器的 EULA, 没有为这些集成文件单独声明许可证; 它们是否属于 EULA 中的 "Software" 不确定. 公开发布前建议向 Amanieu Systems (info@amanieusystems.com) 确认或寻求专业意见.
- Tango EULA 的授权范围写的是 Raspberry Pi 3 和使用者控制的 ARMv8 设备上的个人使用, 本项目只供使用者在自己的设备上使用.
