---
name: porting-guide
description: "把 Tango 二进制翻译器移植到没有 AArch32 的 Android 设备上, 运行 32 位 APK 的步骤"
metadata:
  version: "1.0.0"
  updated: "2026-11-05"
  lang: "zh-CN"
---

# Tango 移植教程

本文说明如何在 CPU 不支持 AArch32 的 arm64 Android 设备上, 用 Tango 二进制翻译器让 32 位 APK 免 root 安装和运行. Tango 集成包的官方做法是给 AOSP 源码打补丁 (参考设备 Pixel 6); 本文针对拿不到或不想重编源码的情况, 直接改写已编译的 system 镜像, 不改 framework 和 vendor 分区. 每一步都对应 [TB321FU 参考实现](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/README.md)中的文件, 该实现在拯救者 Y700 三代 (骁龙 8 Gen 3, Android 15) 上真机验证通过. 其他设备和 Android 版本未验证, 文中标为 "需确认" 的地方要在目标设备上实测.

## 原理

Tango 是进程级翻译, 不是 native bridge:

1. 内核的 `binfmt_misc` 把每个 32 位 ARM ELF 交给静态 arm64 程序 `tango_translator` 执行.
2. 翻译器在用户态把 ARM32 指令翻译成 ARM64 执行, JIT 需要 `execmem`.
3. 需要内核按 32 位语义处理的系统调用 (compat ioctl 等), 由内核模块 `tango32` 通过 `/dev/tango32` 提供给翻译器.
4. 32 位 app 由一个 32 位 zygote fork 出来; 32 位 zygote 本身也是被翻译的进程.

所以移植要做的是: 装翻译器和内核模块, 补 SELinux 策略, 补 32 位用户态库, 声明 32 位 ABI 并启动 32 位 zygote, 再处理声明 32 位 ABI 后引发的开机问题.

## 前提

| 条件 | 检查方法 | 不满足时 |
| --- | --- | --- |
| CPU 没有 AArch32 | 有 AArch32 的设备不需要 Tango | 不需要移植 |
| 内核 ≥ 5.6, 开了 `CONFIG_COMPAT` 和 `CONFIG_BINFMT_MISC` | `adb shell uname -r`; `adb shell zcat /proc/config.gz \| grep -E 'CONFIG_COMPAT=\|BINFMT_MISC'` (需要内核开 `IKCONFIG_PROC`) | 无法移植 |
| 能拿到设备内核的源码和配置 | GKI 设备看 `uname -r` 里的 tag 和 build number | 无法编译 `tango32.ko` |
| system 镜像带 32 位库 | 镜像里有 `bin/app_process32`, `lib/libc.so`, `lib/libandroid_runtime.so` | 纯 64 位底包缺少 32 位 bionic, ART 和 framework 原生库, 无法用本方法 |
| 能修改并刷入 system 镜像 | Bootloader 已解锁, 能自己构建 GSI 或 ROM | 无法移植 |
| 有以 root 和独立 SELinux 域运行的 init 服务 | 用于加载模块, 改 sysctl | 自己在策略里定义一个域; 参考实现用 [herald-core](https://github.com/CharlesGool/herald-core) |
| Tango 集成包 | 从 [Amanieu Systems](https://www.amanieusystems.com/) 获取, 含 `prebuilts/tango_translator`, `kernel-module/`, AOSP 补丁 (含 binfmt rc 和 sepolicy), `EULA.txt` | 无法移植 |

## 步骤 1: 翻译器和 binfmt_misc

1. 把 `tango_translator` 装到 `/system_ext/bin/` (一加装在 `/system/bin/`, 位置不重要), 模式 0755, SELinux 标签 **必须**是 `system_lib_file`: 各个域把它当解释器映射执行, 用 `system_file` 会被拒绝.
2. 在 `early-init` 挂载 `binfmt_misc` 并写入注册串. 注册串, 掩码和 `POCF` 标志照集成包原样复制, 小米, 一加和参考实现逐字相同.
3. 用属性 (参考实现为 `tango.enabled`) 在运行时开关 `/proc/sys/fs/binfmt_misc/tango_translator`, 方便排错.
4. 开发阶段把翻译器日志目录放在 `/metadata` 下: 它在 first-stage 挂载, 开机循环时也能保留.

参考: [001.core](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/001.core/README.md).

## 步骤 2: SELinux

1. 把集成包 AOSP 补丁中的 `sepolicy/system/{public,private}/tango.te` 和 `te_macros` 加进 system 策略. 二进制底包没有策略源码时, 展开成 CIL 追加到 `plat_sepolicy.cil`; 追加后 `plat_sepolicy_and_mapping.sha256` 不再匹配, init 开机时会在设备上重编.
2. 宏 `tango_translated` 要覆盖所有会运行 32 位代码的域: `zygote`, `app_zygote`, `webview_zygote`, `appdomain`, `crash_dump`, 以及 32 位自检的域.
3. 给 `/dev/tango32`, 翻译器路径, 日志目录和 `tango.*` 属性加标签.
4. 开机后看 `dmesg | grep avc` 补规则. 参考实现补了 `zygote` 读 `proc_security`.

对 `appdomain` 开 `execmem` 违反 AOSP 的 neverallow, 会降低所有 app 的加固; 这是 Tango 官方策略本身的要求.

参考: [002.sepolicy](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/002.sepolicy/README.md).

## 步骤 3: tango32 内核模块

1. **必须**用设备内核的同一份源码和配置编译 `tango32.ko`. GKI 开了 `TRIM_UNUSED_KSYMS`, 源码或配置不同会导致 `Unknown symbol` 或 `disagrees about version of symbol`.
2. GKI 设备: 从 `uname -r` 得到 tag 和 build number, 用 Kleaf 编出内核和模块, 核对自编 `Image` 的 UTS_RELEASE 和 .ko 的 `vermagic` 都与设备一致. 具体命令见[参考实现的编译方法](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/003.kernel-module/README.md#编译-tango32ko).
3. 在第一个 32 位进程启动之前加载模块. 厂商集成把它放在 vendor 模块列表里由 first-stage init 加载; 只改 system 时, 在 `post-fs-data` 用 `exec_start` 运行的 root 服务加载即可. 参考实现中 init 自己在 `early-init` 的 `insmod` 没有加载成功, 原因未查明.
4. 用 ueventd 把 `/dev/tango32` 设为 0666.

模块为 GPL-2.0. 分发含 `tango32.ko` 的镜像时需要提供源码.

参考: [003.kernel-module](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/003.kernel-module/README.md).

## 步骤 4: 32 位 GPU 库

纯 64 位设备的 vendor 通常只有 64 位 GPU 驱动. 32 位 app 进程使用 HWUI, 没有 32 位 EGL, GLES, Vulkan 和 gralloc/mapper 就画不出界面.

1. 从**同 SoC** 且出厂支持 32 位 app 的设备固件中取 32 位库, 优先选 32 位与目标设备 64 位驱动版本相同或相近的固件.
2. 用 `readelf` 的依赖和字符串中 `dlopen` 的库名算出闭包, 只取需要的文件.
3. vendor 分区受 AVB 保护, 不能直接加文件. 把库放进 system, 在 `post-fs` 用 init 的 `mount` 叠成只读 overlay 盖到 `/vendor/lib`, lowerdir 中 system 的目录在前.
4. 标签按 `plat_file_contexts` 和原厂 `vendor_file_contexts` 计算, 与原厂 64 位同名文件的实际标签核对.

32 位与 64 位驱动版本不同时, 与内核 KGSL 的兼容性需确认.

参考: [004.vendor-lib32](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/004.vendor-lib32/README.md).

## 步骤 5: 声明 32 位 ABI 和 32 位 zygote

1. 在 system `build.prop` 显式写 `ro.product.cpu.abilist`, `abilist32`, `abilist64`. 先确认这些属性在 vendor, odm, product 中有没有被后加载的值覆盖: init 按 system, vendor, odm, product 的顺序加载, 后加载的 `ro.*` 生效.
2. **不要**改 `ro.zygote`. 改成 `zygote64_32` 会被 vendor 或 odm 的值覆盖; 即使生效, 原版 `init.zygote64_32.rc` 也会带入 32 位自检和 `onrestart restart zygote`.
3. 自己定义 32 位 zygote 服务, 命令行抄原版 `zygote_secondary`:
   - 服务名**禁止**用 `zygote_secondary`: 原版 `init.rc` 的 `on zygote-start` 无条件启动它, `disabled` 挡不住. 小米用 `zygote_mi_secondary`, 一加用 `zygote_tango`.
   - socket 名**必须**是 `zygote_secondary`, system_server 按 socket 名连接.
   - 去掉 `onrestart restart zygote`, 32 位 zygote 崩溃时只影响 32 位 app.
   - 加 `disabled`, 在 `zygote-start` 按条件启动.
4. Android 14 及以上: 设 `ro.dalvik.vm.enable_uffd_gc=false`, 写在实际生效的那个 `build.prop` 里. Tango 不支持 userfaultfd, 32 位 ART 只能用 CC GC, 而 odrefresh 默认按 CMC 编 32 位启动镜像. 一加 12 保持 `true` 也能运行, 所以这一项不是必需, 但可以避免启动镜像不匹配.

不修改 framework 时, 32 位 zygote 必须在开机时启动: `abilist32` 一声明, system_server 开机完成时就会连接它. 小米和一加的按需启动依赖 framework 补丁.

参考: [005.zygote32](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/005.zygote32/README.md).

## 步骤 6: 处理声明 32 位 ABI 引发的问题

### 32 位自检

`abilist32` 非空后, 开机早期会同步运行 32 位 BoringSSL 自检, vendor 的那个通常带 `reboot_on_failure`. 出厂不支持 32 位的 vendor 往往没有 `boringssl_self_test32`, 结果是开机循环, `sys.boot.reason.last` 为 `reboot,boringssl-self-check-failed`.

1. 扫描原厂 vendor 和 odm 的全部 init rc, 找出带 `reboot_on_failure` 或 `critical`, 程序是 32 位或不存在, 又会因 `abilist32` 触发的服务. 参考实现的 [check_vendor32.py](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/005.zygote32/check_vendor32.py) 在构建时做这件事.
2. 在 `/system/etc/init` 先定义同名服务, 换成 64 位程序. `/system/etc/init` 比 `/vendor/etc/init` 先解析, vendor 的同名定义没有 `override` 时被丢弃. 替身**禁止**用 `/system/bin/true`, init 没有到 `toolbox_exec` 的域转换.

### 调度器的 32 位任务检查

`tango32` 转发 compat 系统调用时会临时置位任务的 `TIF_32BIT`. 调度器若检查 "32 位任务只能在 32 位核上运行", 在没有 32 位核的 CPU 上就会出错.

参考实现中联想的高通 WALT 模块 `sched-walt.ko` 会因此 `BUG()`, 表现为 32 位 zygote 运行几十秒后整机重启. 一加 12 的同一模块在 32 位核掩码为空时跳过检查, 不受影响. 排查方法:

1. 高通设备检查 `/proc/sys/walt/panic_on_walt_bug` 是否存在. 存在时反汇编 `sched-walt.ko` 的 `android_rvh_set_task_cpu`, 看 `TIF_32BIT` 检查前有没有空掩码判断.
2. 没有判断时, 在启动 32 位 zygote 前写 `0` 到该 sysctl (同时关闭了 WALT 的其他 BUG 检查), 或者给模块补空掩码判断后重打 `vendor_boot`.
3. 其他调度器 (如联发科) 是否有同类检查需确认.

细节见[调研记录](research.md#真因-walt-32-位任务自检).

## 验证

分两阶段刷机, 先不声明 32 位 ABI, 开机风险最小:

1. 阶段 1, 只做步骤 1 到 4. 开机后:

   ```bash
   adb shell cat /proc/sys/fs/binfmt_misc/tango_translator   # enabled
   adb shell ls -lZ /dev/tango32                             # crw-rw-rw-, tango32_device
   adb shell /system/bin/app_process32                       # 经翻译运行, 打印用法提示
   adb shell dmesg | grep -iE 'tango|avc'
   ```

2. 阶段 2, 加上步骤 5 和 6. 开机后:

   ```bash
   adb shell getprop sys.boot.reason.last
   adb shell getprop ro.product.cpu.abilist                  # 含 armeabi-v7a
   adb shell ps -A | grep zygote                             # 64 位和 32 位 zygote 都在
   ```

   再安装一个 32 位 APK, 测试使用 GL 和 Vulkan 的界面.

开发阶段建议加一个开机保险: 在 `post-fs-data` 检查模块和上次开机是否稳定, 不满足时撤掉 32 位 ABI 按纯 64 位开机, 并保存 `/sys/fs/pstore`. 参考实现的 `tango-guard.sh` 可以照搬, 见 [005.zygote32](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/005.zygote32/README.md#开机保险).

## 排错

| 现象 | 常见原因 |
| --- | --- |
| 开机循环, `reboot,boringssl-self-check-failed` | vendor 32 位自检程序不存在, 见步骤 6 |
| 卡开机动画, system_server 反复崩溃 | 声明了 `abilist32` 但 32 位 zygote 没起来; 检查 `ro.zygote` 和服务定义 |
| 32 位 zygote 反复 SIGILL | `/dev/tango32` 不存在, 模块没加载 |
| `read barrier state mismatch` | 32 位启动镜像按 CMC 编, 见步骤 5 第 4 条 |
| 回退纯 64 位时仍整机重启, `updatable_crashing` | 32 位 zygote 服务名用了 `zygote_secondary` |
| 32 位 zygote 运行后几十秒整机重启 | 调度器的 32 位任务检查, 见步骤 6; 看 pstore 的 `console-ramoops` |
| 32 位 app 能启动但没有界面 | 缺 32 位 GPU 库, 或 overlay 没挂上 |
| `insmod` 报 `Unknown symbol` | 模块与设备内核的源码或配置不一致 |

## 授权

- Tango 是 Amanieu Systems 的专有软件, 受集成包中的 EULA 约束. 本仓库不包含翻译器, 集成包和 `tango32.ko`, 移植者需要自己获取并接受 EULA. 分发含翻译器的镜像前, 建议向 Amanieu Systems 书面确认.
- 从其他厂商固件取出的 32 位库是专有文件, **禁止**提交到公开仓库.

详见[第三方声明](THIRD_PARTY_NOTICES.md).
