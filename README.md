# tango-porting-guide

**简体中文** | [English](doc/en/README.md) | [Español](doc/es/README.md)

CPU 不支持 AArch32 的 Android 设备跑不了 32 位 APK; 本项目讲解如何在不改源码的情况下, 把 Tango 二进制翻译器移植进已编译的 system 镜像, 免 root 运行 32 位 APK, 并提供一个真机验证过的参考实现.

[![License](https://img.shields.io/badge/License-GPL--3.0--or--later-orange)](LICENSE)

## 文档

- 项目概览: [README](README.md)

- 设计思路: [DESIGN](doc/DESIGN.md)

- 项目状态: [LOG](doc/LOG.md)
- 历史记录: [HISTORY](doc/HISTORY.md)
- 变更日志: [CHANGELOG](doc/CHANGELOG.md)

- 第三方声明: [THIRD_PARTY_NOTICES](doc/THIRD_PARTY_NOTICES.md)

## 简介

| 内容 | 说明 |
| --- | --- |
| [移植教程](doc/porting-guide.md) | 前提检查, 6 个移植步骤, 分阶段验证和排错表 |
| [调研记录](doc/research.md) | 与小米 14 Ultra, 一加 12, 一加 13T 官方集成的逐项对照; 声明 32 位 ABI 后整机重启的真因 (高通 WALT 调度器的 32 位任务检查); QEMU 测试 |
| [参考实现](https://github.com/CharlesGool/tb321fu-tango-patch) | 拯救者 Y700 三代 TB321FU (骁龙 8 Gen 3, Android 15) 的完整补丁, V883 (2026-10-31) 真机运行 32 位 APK 正常 |

[Tango](https://www.amanieusystems.com/) 是 Amanieu Systems 的 ARM32→ARM64 二进制翻译器. 它的官方集成方式是给 AOSP 源码打补丁; 本项目的方法只改写 system 镜像, 适用于 GSI 和拿不到源码的 ROM.

参考实现之外的设备和 Android 版本没有验证. 参考实现的已知问题见 [tb321fu-tango-patch 的 LOG](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/LOG.md#缺陷).

## 要求

- arm64 设备, CPU 没有 AArch32; 内核 ≥ 5.6 并开启 `CONFIG_COMPAT` 和 `CONFIG_BINFMT_MISC`
- 能拿到设备内核的源码和配置, 用来编译 `tango32.ko`
- system 镜像带 32 位库 (`app_process32`, 32 位 bionic 和 framework 原生库)
- Bootloader 已解锁, 能修改并刷入 system 镜像
- Tango 集成包, 从 Amanieu Systems 获取并接受其 EULA

逐项检查方法见[移植教程的前提](doc/porting-guide.md#前提).

## 安装

本项目是教程和参考实现, 无需安装.

### 快速安装

```bash
git clone https://github.com/CharlesGool/tango-porting-guide.git
```

### 常规安装

参考实现在独立仓库 [tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch), 是 [TB321FU-GSI-Mods](https://github.com/CharlesGool/TB321FU-GSI-Mods) 的补丁. 构建参数和使用方法见该仓库.

```bash
git clone https://github.com/CharlesGool/tb321fu-tango-patch.git
```

## 指南

1. 按[前提](doc/porting-guide.md#前提)检查目标设备.
2. 按[移植教程](doc/porting-guide.md)的步骤 1 到 6 改写 system 镜像, 每一步对照[参考实现](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/README.md)的对应部分.
3. 先刷阶段 1 (不声明 32 位 ABI), 验证翻译器和内核模块, 再刷阶段 2, 见[验证](doc/porting-guide.md#验证).
4. 开不了机或整机重启时查[排错表](doc/porting-guide.md#排错).

## 升级

暂无.

## 卸载

无需卸载, 删除克隆的目录即可. 从设备上去掉 Tango 要刷回不含它的 system 镜像.

## 致谢

- [Tango](https://www.amanieusystems.com/) by Amanieu Systems: 翻译器, `tango32` 内核模块, binfmt 注册和 SELinux 策略的来源
- 小米 14 Ultra 和一加 12 的官方 Tango 集成: 32 位 zygote 和 ART GC 做法的参照

## 许可证

本项目以 GNU General Public License v3.0 或更新版本授权, SPDX 标识符 `GPL-3.0-or-later`, 全文见 [LICENSE](LICENSE). 本项目不包含 Tango, `tango32` 模块和其他厂商的 32 位库, 见 [THIRD_PARTY_NOTICES](doc/THIRD_PARTY_NOTICES.md).
