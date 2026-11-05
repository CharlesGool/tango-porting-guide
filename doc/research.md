---
name: research
description: "Tango 在 TB321FU 上的调研: 最终做法, 与小米 14 Ultra 和一加的对照, 整机重启真因, QEMU 测试"
metadata:
  version: "1.0.0"
  updated: "2026-11-05"
  lang: "zh-CN"
---

# Tango 调研记录

本文记录 Tango 在 TB321FU (骁龙 8 Gen 3, 联想 Android 15 vendor) 上从不能开机到 32 位 APK 正常运行的技术依据: 最终做法, 与同平台厂商集成的对照, 整机重启的真因和 QEMU 测试结果. 各版本的真机现象按时间记录在 [HISTORY](HISTORY.md#真机测试经过).

V883 (2026-10-31) 真机带全部 5 个部分开机, 32 位 APK 正常运行.

## 最终做法

| 环节 | 做法 | 来源 |
| --- | --- | --- |
| 翻译器 | Tango v2.1.0 `tango_translator` 装在 `/system_ext/bin`, `early-init` 注册 `binfmt_misc` (`POCF`) | 官方集成包; 小米和一加相同 |
| 内核模块 | `tango32.ko` 按设备 GKI 6.1.112 编译, 装在 `/system/etc/tango/`, `post-fs-data` 由 herald 域加载; ueventd 给 `/dev/tango32` 0666 | 官方源码; 小米和一加在 vendor 模块阶段更早加载, 本项目的时机足够早 |
| SELinux | 官方 `tango.te` 展开, 加 `tango_log_file` 和 `zygote` 读 `proc_security` | 官方; 小米和一加相同 |
| 32 位 GPU 栈 | 小米 14 Ultra 的 32 位 Adreno `V@0762.4` 和 gralloc/mapper, 开机 overlay 到 `/vendor/lib` | 小米固件 |
| ABI 声明 | system `build.prop` 显式写 `ro.product.cpu.abilist*` | 小米和一加相同 |
| 32 位 zygote | 不改 `ro.zygote`, 自带服务 `tango_zygote32` (socket 名 `zygote_secondary`, `disabled`, 无 `onrestart restart zygote`), 开机保险放行后在 `zygote-start` 启动 | 小米 `zygote_mi_secondary`, 一加 `zygote_tango` |
| vendor 32 位自检 | `tango-boringssl32.rc` 先定义同名服务, 换成 64 位程序 | 联想 vendor 特有 |
| ART GC | `ro.dalvik.vm.enable_uffd_gc=false` (system 和 product) | 小米相同; 一加为 `true`, 不是必需 |
| 整机重启修复 | 放行 32 位前写 `0` 到 `/proc/sys/walt/panic_on_walt_bug` | 本项目 |

## 小米 14 Ultra

对照对象是 xiaomi.eu OS2.0.213 (Android 15), OS3 的做法相同. 小米 HyperOS 在 8 Gen 3 上自带 Tango.

| 项目 | 小米 | 本项目 |
| --- | --- | --- |
| 翻译器 | `/system_ext/bin/tango_translator` v2.0.7, 标签 `system_lib_file` | v2.1.0, 同路径同标签 |
| binfmt 注册 | `system_ext/etc/init/mi-32bit-trans.rc`, `on early-init && property:ro.vendor.mi_fake_32bit_support=true`; 注册串和 `POCF` 标志逐字相同 | 相同 |
| 内核模块 | `vendor_dlkm/lib/modules/tango32.ko`, 在 `modules.load` 中, 由 init 当普通 vendor 模块加载; `late-fs` 时 `chmod 0666 /dev/tango32` | `/system/etc/tango/`, `post-fs-data` 加载 |
| 模块代码 | 导入 31 个符号, 与本项目的 .ko 完全一致 (同一份源码); 按小米内核 `6.1.68-android14-11` 编译 | 按 GKI `6.1.112-android14-11` 编译 |
| `ro.zygote` | 保持 `zygote64`; 自定义 `zygote_mi_secondary` (`--socket-name=zygote_secondary`, `class main` + `disabled`), 另有 `sys.mi_zygote32.start` 懒加载 (framework 有 `MI_FAKE_32BIT_SUPPORTED_ABIS` 补丁) | 保持 `zygote64`, 自定义 `tango_zygote32` |
| ABI 声明 | system `build.prop` 的 `ro.system.product.cpu.abilist=arm64-v8a,armeabi-v7a,armeabi` | system `build.prop` 的 `ro.product.cpu.abilist*` |
| 32 位 BoringSSL 自检 | vendor rc 是 `import boringssl_self_test.${ro.zygote}.rc`, `zygote64` 时只跑 64 位 | 联想 vendor rc 只看 `abilist32`, 必须接管 |
| ART GC | product `build.prop` 设 `ro.dalvik.vm.enable_uffd_gc=false` | 相同 |
| SELinux | 与官方 `tango.te` 相同; 另有 `allowx app_zygote tango32_device ioctl 0x74a3-0x74a7` | 官方规则加 `tango_log_file`, `proc_security` |
| 预翻译 | system 有 237 个 `.tango` 预翻译件; `tango_pretranslator` 在装 app 时预翻译 | 没有 (需要修改 framework 源码) |
| 32 位 GPU 栈 | vendor/lib 自带 32 位 Adreno `V@0762.4` | 取自小米的同一套 30 个文件 |

小米有而本项目做不了的: framework 侧懒加载和装机预翻译, 都需要源码.

## 一加 13T

一加 13T 是骁龙 8 Elite (SM8750), 不同平台. 对照的包是国行机刷印度 OxygenOS 16.0.3.501 的转换包: super 是 OxygenOS, odm 和 my_region 是国行 ColorOS.

- OxygenOS 里没有 Tango: 没有 `app_process32`, 没有翻译器, `abilist32` 为空.
- 两边共用的部分能看到一加的做法:
  - `tango32.ko` 在 vendor_boot 第一阶段 ramdisk 的 `modules.load` 中, 由 first-stage init 加载. vermagic `6.6.89-android15-8-o`; 导入 30 个符号, 与本项目的 .ko 只差 6.6 内核的 `system_cpucaps` 替换, 是同一份源码.
  - 开关属性在 odm `build.prop`: `ro.vendor.oplus.tango_binfmt_misc_support=1`.
  - 翻译器在 `/system/bin/tango_translator`, 标签同为 `system_lib_file`.
  - 32 位 zygote (`system_ext/etc/init/init.otranslator.zygote.rc`) 不改 `ro.zygote`, 自定义服务 `zygote_tango` (`--socket-name=zygote_secondary`, `class main` + `disabled`, 没有 `onrestart restart zygote`), 由持久化属性 `persist.sys.tango_zygote32.start=1` 控制.
- 结论: 小米和一加都不用原版服务名 `zygote_secondary`, 因为原版 `init.rc` 的 `on zygote-start` 会无条件启动它.

## 一加 12

对照对象是一加 12 国行 ColorOS PJD110_15.0.0.860(CN01), Android 15, 2025-09 补丁, 与 TB321FU 同为 SM8650.

| 项目 | 一加 12 | 小米 14 Ultra | 本项目 |
| --- | --- | --- | --- |
| 内核 | GKI `6.1.118-android14-11` | `6.1.68-android14-11` | GKI `6.1.112-android14-11` |
| 翻译器 | v2.0.7 | v2.0.7 (与一加不是同一个文件) | v2.1.0 |
| `tango32.ko` | vendor_boot 第一阶段 ramdisk; 导入符号与本项目相同 | vendor_dlkm | `/system/etc/tango/`, `post-fs-data` |
| `/dev/tango32` | vendor `ueventd.rc` 0666 | `late-fs` chmod | system `ueventd.rc` 0666 |
| binfmt 注册 | `on late-fs && ro.vendor.oplus.tango_allow32=true`, 注册串相同 | `early-init` | `early-init` |
| ABI 声明 | system, product, vendor 都写 `abilist32`, odm 又覆盖为空; 实际生效的由 framework 的 `QCOM_TANGO_SUPPORTED_ABIS` 决定 | system `build.prop` | system `build.prop` |
| `ro.zygote` | vendor 写 `zygote64_32`, odm 盖回 `zygote64` | `zygote64` | `zygote64` |
| 32 位 zygote 启动时机 | 按需: framework `TangoPackage` 发现已装 32 位应用才置 `persist.sys.tango_zygote32.start`, 启动 32 位 app 时现起现等 | 懒加载 | 开机 `zygote-start` |
| ART GC | `enable_uffd_gc=true` | `false` | `false` |
| 32 位 GPU 栈 | vendor/lib 自带, 32 位和 64 位 Adreno 都是 `V@0762.34` | 32 位 0762.4, 64 位 0762.30 到 0762.36 | 32 位 0762.4, 64 位 0762.28 |
| vendor 32 位自检 | rc 用 `${ro.zygote}` import, 不用接管 | 同一加 | 必须接管 |
| 预翻译 | 没有 `.tango` 文件 | 237 个 | 没有 |

结论:

1. 两家都不在开机时无条件启动 32 位 zygote. 不改 framework 时, `abilist32` 一声明, system_server 开机完成时就要连接 32 位 zygote (V77 因连不上崩溃), 所以本项目仍在 `zygote-start` 启动.
2. `enable_uffd_gc=false` 不是必需 (一加为 `true` 也能运行), 但本项目保持 `false`.
3. 翻译器版本不是决定性差异. QEMU 上 v2.0.7 也有多线程致命路径的挂死 (见 [QEMU 测试](#qemu-测试)).
4. 一加 12 的 vendor/lib 是同平台, 32/64 位版本一致的完整 32 位库, 可以作为 32 位 GPU 栈的替代来源 (未试).

## 真因: WALT 32 位任务自检

V68 到 V882 带 32 位 zygote 的版本都在 32 位 zygote 预加载或第一个 32 位 app 进程启动后 8 到 28 秒内整机重启. V882 的 console-ramoops 指向 `walt.c` 中的 `BUG()`. 用 llvm-objdump 反汇编联想和一加 12 `sched-walt.ko` 的 `android_rvh_set_task_cpu`:

- **联想** (`6.1.78-android14-11`): 任务 `thread_info.flags` 的 bit 22 (`TIF_32BIT`) 为 1, 且目标 CPU 不在 `system_32bit_el0_cpumask()` 中时, 读取 `sysctl_panic_on_walt_bug`. 其高位等于魔数 `0x4544de00` 时: bit 4 打印 `WALT-BUG selecting non 32 bit cpu`, bit 1 执行 `walt_task_dump` 和 `BUG()`. 默认值 `0x4544de1b`, 两位都开.
- **一加 12** (`6.1.118-android14-11`): 同一段代码前先检查 `system_32bit_el0_cpumask()`, 掩码为空 (没有 32 位核) 时跳过整条检查.

8 Gen 3 没有 AArch32 核, 32 位核掩码永远为空. `tango32.ko` 转发 compat ioctl 时必须置位 `TIF_32BIT` (驱动靠 `in_compat_syscall()` 判断参数布局), 任务在此期间睡眠后被唤醒, 迁移, 联想的 WALT 就触发 `BUG()`. 这解释了 QEMU (`-cpu max` 所有核支持 32 位, 也没有 WALT 模块) 无法复现.

修法: 开机保险在放行 32 位前写 `0` 到 `/proc/sys/walt/panic_on_walt_bug`, 魔数不再匹配, 整套 WALT-BUG 检查跳过. 代价是 WALT 的其他 BUG 检查也被关闭. 更干净的做法是像一加那样给 `sched-walt.ko` 补空掩码判断并重打 `vendor_boot`, 没有做.

## 早期排查的其他问题

这些问题在定位真因前已修复, 都写进了现在的实现:

| 问题 | 现象 | 修法 |
| --- | --- | --- |
| 联想 vendor 的 `boringssl_self_test32_vendor` 程序不存在且 `reboot_on_failure` | V68 开机循环, `reboot,boringssl-self-check-failed` | `tango-boringssl32.rc` 接管 |
| `ro.zygote=zygote64_32` 写在 system 被 vendor 的 `zygote64` 覆盖 | V77 32 位 zygote 没起, system_server 连不上 `zygote_secondary` | 不改 `ro.zygote`, 自带服务 |
| init 在 `early-init` 的 `insmod` 没有加载成功 | V78 没有 `/dev/tango32`, 32 位库 CFI 检查全部 SIGILL | 改为 `post-fs-data` 由 herald 域加载 |
| odrefresh 按 CMC 编 32 位启动镜像, 32 位 ART 只能用 CC | V79 `read barrier state mismatch` | `enable_uffd_gc=false` |
| 原版 `init.rc` 无条件 `start zygote_secondary` | V881 回退时被拉起反复 abort, 触发 `updatable_crashing` 整机重启 | 服务改名 `tango_zygote32` |

## QEMU 测试

环境: `qemu-system-aarch64 -M virt -cpu max -smp 4 -m 1024`, 与设备相同的 GKI Image, `tango32.ko` 和 `tango_translator`. binfmt 注册串要写字面的 `\x7f\x45...` 文本, 由内核解转义; 写真二进制会在第一个 `\0` 截断, 32 位程序会被原生 AArch32 执行.

- 32 位静态 busybox 的并发, fork, 目录遍历, 信号, 网络和 32/64 位混跑负载 (约 40 分钟): 19 份串口日志中没有 Oops, BUG, WARNING, panic, RCU stall 或 softlockup.
- 自编 bionic 静态 32 位多线程程序 `thr32` 的致命路径: 内核没有告警, 但翻译器用户态会挂死 (所有线程停在 64 位 `futex_wait`) 或把 SIGSEGV 误报为 SIGABRT. 原生 AArch32 对照正常. 8 线程时不挂死.

| 场景 (32 线程) | v2.1.0 (7 轮) | v2.0.7 (20 轮) | 原生 AArch32 (20 轮) |
| --- | --- | --- | --- |
| 忙时主线程 SIGSEGV | 3 正常, 3 误报 SIGABRT, 1 挂死 | 9 正常, 11 误报 SIGABRT | 20 正常 |
| 忙时 abort | 6 正常, 1 挂死 | 19 正常, 1 挂死 | 20 正常 |
| exit_group | 7 正常 | 20 正常 | 20 正常 |
| 多线程父进程 fork 再 exec | 7 正常 | 18 正常, 2 误报 SIGABRT | 19 正常, 1 挂死 (QEMU 自身问题) |

QEMU 覆盖不到: binder, kgsl, dmabuf heap 等设备驱动的 `compat_ioctl`, 联想 vendor_dlkm 模块和 vendor hook, SELinux, 没有 AArch32 的真实 CPU.

## 可改进项

以下都不阻塞使用:

- 给 `sched-walt.ko` 补空掩码判断, 恢复 WALT 的其他检查.
- 按需启动 32 位 zygote, 需要修改 framework 的 `ZygoteProcess`.
- 预翻译, 需要修改 framework 和 installd 源码.
- 稳定后把 `TANGO_DEBUG` 改为 `off`.
