---
name: project-design
description: 项目架构与设计约束
metadata:
  version: "1.0.0"
  lang: "zh-CN"
---

# tango-porting-guide — 设计

## 多语言

**简体中文** | [English](en/DESIGN.md) | [Español](es/DESIGN.md)

## 文档

- 项目概览: [README](../README.md)

- 设计思路: [DESIGN](DESIGN.md)

- 项目状态: [LOG](LOG.md)
- 历史记录: [HISTORY](HISTORY.md)
- 变更日志: [CHANGELOG](CHANGELOG.md)

- 第三方声明: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## 设计目标

- [x] 讲清在不改源码的情况下, 把 Tango 移植进已编译的 Android system 镜像的通用步骤.
- [x] 每一步都对应一个真机验证过的参考实现 ([tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch), TB321FU, Android 15).
- [x] 给出与厂商官方集成的对照和整机重启真因, 作为步骤的依据.
- [ ] 在第二台设备或另一个 Android 版本上验证教程.
- 非目标: 提供可直接刷入的镜像或任何设备的补丁代码.

## 架构

本项目只有文档:

| 文件 | 内容 |
| --- | --- |
| [porting-guide.md](porting-guide.md) | 通用移植教程: 前提, 6 个步骤, 验证和排错; 每一步链接到参考实现的对应部分 |
| [research.md](research.md) | 调研记录: 最终做法, 与小米和一加的对照, 真因, QEMU 测试 |

| 顶层目录 | 职责 |
| --- | --- |
| `doc/` | 项目文档 |

参考实现在独立仓库 [tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch). 本项目没有代码, 界面文本和构建输出.

## 设计约束

- 仓库**禁止**包含 Tango, `tango32.ko` 和其他厂商的专有文件.
- 教程中未在真机验证的内容**必须**标为需确认.
- 教程的每一步**必须**能在参考实现中找到对应文件; 参考实现的路径变化时**必须**同步更新链接.
