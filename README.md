# tb321fu-tango

**简体中文** | [English](doc/en/README.md) | [Español](doc/es/README.md)

拯救者 Y700 三代 (TB321FU) 的 CPU 没有 AArch32, 刷 GSI 后不能运行 32 位 APK; 本项目是把 Tango 二进制翻译器集成进 system 镜像的 TB321FU-GSI-Mods 补丁, 免 root 运行 32 位 APK.

[![License](https://img.shields.io/badge/License-GPL--3.0--or--later-orange)](LICENSE)

## 文档

- 项目概览: [README](README.md)

- 设计思路: [DESIGN](doc/DESIGN.md)

- 项目状态: [LOG](doc/LOG.md)
- 历史记录: [HISTORY](doc/HISTORY.md)
- 变更日志: [CHANGELOG](doc/CHANGELOG.md)

- 第三方声明: [THIRD_PARTY_NOTICES](doc/THIRD_PARTY_NOTICES.md)

## 简介

- 用 [Tango](https://www.amanieusystems.com/) 把 32 位 ARM 程序翻译成 arm64 执行, 32 位 APK 可以安装和运行, 不需要 root.
- 补丁 `tango` 由 5 个部分组成: 翻译器和 `binfmt_misc` 注册, SELinux 策略, `tango32` 内核模块, 32 位 GPU 库, 32 位 zygote 和开机保险. 每个部分的说明见 [patch/README.md](patch/README.md).
- 解决了联想 vendor 上声明 32 位 ABI 后的开机循环和整机重启问题, 原因见[调研记录](doc/research.md).
- V883 (2026-10-31) 真机测试通过. 已知问题见 [LOG](doc/LOG.md#缺陷).

本仓库是该补丁的唯一来源. [TB321FU-GSI-Mods](https://github.com/CharlesGool/TB321FU-GSI-Mods) 的 `patches/required/general/tango/` 是同步副本.

## 要求

- 拯救者 Y700 三代 (TB321FU), 原厂联想 Android 15 vendor, boot 内核为 GKI `6.1.112-android14-11-gd03cb1d7abcd-ab12592111`
- 用 [TB321FU-GSI-Mods](https://github.com/CharlesGool/TB321FU-GSI-Mods) 构建 system 镜像, 并启用 [herald-core](https://github.com/CharlesGool/herald-core) 补丁
- 使用者自备, **禁止**提交到仓库或放进分发的刷机包:
  - Tango v2.1.0 集成包 (需要 `prebuilts/tango_translator`), 从 Amanieu Systems 获取, 遵守其 EULA
  - 按上述内核编译的 `tango32.ko`, 编法见 [kernel-module](patch/parts/003.kernel-module/README.md#编译-tango32ko)
  - 小米 14 Ultra xiaomi.eu OS3.0.304.0 `vendor/lib` 中的 30 个 32 位库, 文件清单见 [manifest.sha256](patch/parts/004.vendor-lib32/manifest.sha256)
  - 原厂 vendor 镜像

## 安装

本项目不单独安装, 由 TB321FU-GSI-Mods 在构建时打进 system 镜像.

### 快速安装

TB321FU-GSI-Mods 已带同步副本, 默认构建就包含本补丁. 把上述输入放到 Mods 仓库的默认位置后构建:

```bash
make 9008 NOTE="tango"
```

默认位置是 Mods 仓库的 `private/tango/pkg/`, `private/tango/tango32.ko`, `private/xiaomi-vendor-lib32/`.

### 常规安装

用本仓库的版本替换 Mods 中的副本, 再构建:

```bash
rsync -a --delete patch/ <TB321FU-GSI-Mods>/patches/required/general/tango/
```

## 指南

### 参数

参数在 Mods 的 `devices/<设备>/conf.d/tango.conf` 中设置, 同名环境变量优先.

| 参数 | 含义 |
| --- | --- |
| `TANGO_PKG` | Tango 集成包目录 |
| `TANGO32_KO` | `tango32.ko` 路径 |
| `TANGO_KERNEL_RELEASE` | 目标内核发布串, 构建时与 .ko 的 `vermagic` 核对 |
| `XIAOMI_LIB32_DIR` | 32 位库目录, 结构同 `/vendor/lib` |
| `STOCK_VENDOR_IMG` | 原厂 vendor 镜像 (Mods 的 `device.conf` 已设置) |
| `TANGO_DEBUG` | 翻译器日志级别 `off`, `info` 或 `debug`; 不是 `off` 时写到 `/metadata/tango_log` |

### 检查和排错

```bash
adb shell heraldctl tango status        # 模块, 32 位 ABI, 是否回退
adb shell heraldctl tango logs          # 导出日志到 /data/local/tmp/tango_log
adb pull /data/local/tmp/tango_log
```

其他命令和开机保险见 [zygote32](patch/parts/005.zygote32/README.md#heraldctl-命令).

## 卸载

### 快速卸载

不刷机, 立即关闭翻译, 重启后恢复 (32 位 ABI 仍声明, 32 位 APK 无法运行):

```bash
adb shell setprop tango.enabled 0
```

### 完全卸载

在 Mods 中去掉本补丁重新构建并刷机:

```bash
make 9008 WITHOUT=tango NOTE="不带 32 位"
```

## 致谢

- [Tango](https://www.amanieusystems.com/) by Amanieu Systems: 翻译器, `tango32` 内核模块和 SELinux 策略的来源
- 小米 14 Ultra 和一加 12 的官方 Tango 集成: 32 位 zygote 和 ART GC 做法的参照

## 许可证

本项目以 GNU General Public License v3.0 或更新版本授权, SPDX 标识符 `GPL-3.0-or-later`, 全文见 [LICENSE](LICENSE). Tango, `tango32` 模块和小米的库不随本项目分发, 各自的授权见 [THIRD_PARTY_NOTICES](doc/THIRD_PARTY_NOTICES.md).
