---
name: research-en
description: "Tango research on TB321FU: final approach, comparison with Xiaomi 14 Ultra and OnePlus, root cause of the whole-device reboot, QEMU tests"
metadata:
  version: "1.0.0"
  updated: "2026-11-05"
  lang: "en"
---

# Tango Research Notes

This document records the technical basis for taking Tango on the TB321FU (Snapdragon 8 Gen 3, Lenovo Android 15 vendor) from not booting to running 32-bit APKs correctly: the final approach, a comparison with vendor integrations on the same platform, the root cause of the whole-device reboot, and the QEMU test results. The real-device behavior of each version is recorded chronologically in [HISTORY of tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/HISTORY.md#真机测试经过); for the implementation, see that repository.

V883 (2026-10-31) boots on a real device with all 5 parts, and 32-bit APKs run correctly.

## Final Approach

| Area | Approach | Source |
| --- | --- | --- |
| Translator | Tango v2.1.0 `tango_translator` installed in `/system_ext/bin`, `binfmt_misc` registered in `early-init` (`POCF`) | Official integration package; same as Xiaomi and OnePlus |
| Kernel module | `tango32.ko` built for the device's GKI 6.1.112, installed in `/system/etc/tango/`, loaded in `post-fs-data` by the herald domain; ueventd sets `/dev/tango32` to 0666 | Official source; Xiaomi and OnePlus load it earlier, in the vendor module stage, and this project's timing is early enough |
| SELinux | Official `tango.te` expanded, plus `tango_log_file` and `zygote` reading `proc_security` | Official; same as Xiaomi and OnePlus |
| 32-bit GPU stack | Xiaomi 14 Ultra's 32-bit Adreno `V@0762.4` and gralloc/mapper, overlaid onto `/vendor/lib` at boot | Xiaomi firmware |
| ABI declaration | `ro.product.cpu.abilist*` written explicitly in the system `build.prop` | Same as Xiaomi and OnePlus |
| 32-bit zygote | `ro.zygote` unchanged, own service `tango_zygote32` (socket name `zygote_secondary`, `disabled`, no `onrestart restart zygote`), started in `zygote-start` after the boot guard allows it | Xiaomi `zygote_mi_secondary`, OnePlus `zygote_tango` |
| Vendor 32-bit self-test | `tango-boringssl32.rc` defines a same-named service first, replaced by a 64-bit program | Specific to the Lenovo vendor |
| ART GC | `ro.dalvik.vm.enable_uffd_gc=false` (system and product) | Same as Xiaomi; OnePlus uses `true`, not required |
| Whole-device reboot fix | Write `0` to `/proc/sys/walt/panic_on_walt_bug` before allowing 32-bit | This project |

## Xiaomi 14 Ultra

The comparison target is xiaomi.eu OS2.0.213 (Android 15); OS3 does the same. Xiaomi HyperOS ships Tango on the 8 Gen 3.

| Item | Xiaomi | This project |
| --- | --- | --- |
| Translator | `/system_ext/bin/tango_translator` v2.0.7, label `system_lib_file` | v2.1.0, same path and label |
| binfmt registration | `system_ext/etc/init/mi-32bit-trans.rc`, `on early-init && property:ro.vendor.mi_fake_32bit_support=true`; registration string and `POCF` flag byte-identical | Same |
| Kernel module | `vendor_dlkm/lib/modules/tango32.ko`, listed in `modules.load`, loaded by init as an ordinary vendor module; `chmod 0666 /dev/tango32` at `late-fs` | `/system/etc/tango/`, loaded in `post-fs-data` |
| Module code | Imports 31 symbols, identical to this project's .ko (same source); built for Xiaomi kernel `6.1.68-android14-11` | Built for GKI `6.1.112-android14-11` |
| `ro.zygote` | Stays `zygote64`; custom `zygote_mi_secondary` (`--socket-name=zygote_secondary`, `class main` + `disabled`), plus lazy start via `sys.mi_zygote32.start` (the framework has the `MI_FAKE_32BIT_SUPPORTED_ABIS` patch) | Stays `zygote64`, custom `tango_zygote32` |
| ABI declaration | `ro.system.product.cpu.abilist=arm64-v8a,armeabi-v7a,armeabi` in the system `build.prop` | `ro.product.cpu.abilist*` in the system `build.prop` |
| 32-bit BoringSSL self-test | The vendor rc is `import boringssl_self_test.${ro.zygote}.rc`, which runs only the 64-bit one with `zygote64` | The Lenovo vendor rc looks only at `abilist32`, so it must be taken over |
| ART GC | product `build.prop` sets `ro.dalvik.vm.enable_uffd_gc=false` | Same |
| SELinux | Same as official `tango.te`; plus `allowx app_zygote tango32_device ioctl 0x74a3-0x74a7` | Official rules plus `tango_log_file`, `proc_security` |
| Pre-translation | system has 237 pre-translated `.tango` files; `tango_pretranslator` pre-translates when an app is installed | None (requires modifying framework source) |
| 32-bit GPU stack | vendor/lib ships the 32-bit Adreno `V@0762.4` | The same set of 30 files taken from Xiaomi |

What Xiaomi has and this project cannot do: framework-side lazy start and install-time pre-translation, both of which need source.

## OnePlus 13T

The OnePlus 13T is a Snapdragon 8 Elite (SM8750), a different platform. The package compared is a conversion package of the China-market phone flashed with the India OxygenOS 16.0.3.501: super is OxygenOS, odm and my_region are China-market ColorOS.

- There is no Tango in OxygenOS: no `app_process32`, no translator, and `abilist32` is empty.
- The parts shared by both sides show OnePlus's approach:
  - `tango32.ko` is in `modules.load` of the first-stage ramdisk of vendor_boot, loaded by first-stage init. vermagic `6.6.89-android15-8-o`; it imports 30 symbols, and differs from this project's .ko only by the 6.6 kernel's `system_cpucaps` replacement, so it is the same source.
  - The switch property is in odm `build.prop`: `ro.vendor.oplus.tango_binfmt_misc_support=1`.
  - The translator is at `/system/bin/tango_translator`, with the same label `system_lib_file`.
  - The 32-bit zygote (`system_ext/etc/init/init.otranslator.zygote.rc`) does not change `ro.zygote`; it defines a custom service `zygote_tango` (`--socket-name=zygote_secondary`, `class main` + `disabled`, no `onrestart restart zygote`), controlled by the persistent property `persist.sys.tango_zygote32.start=1`.
- Conclusion: neither Xiaomi nor OnePlus uses the stock service name `zygote_secondary`, because the stock `init.rc` starts it unconditionally in `on zygote-start`.

## OnePlus 12

The comparison target is the OnePlus 12 China-market ColorOS PJD110_15.0.0.860(CN01), Android 15, 2025-09 patch, which has the same SM8650 as the TB321FU.

| Item | OnePlus 12 | Xiaomi 14 Ultra | This project |
| --- | --- | --- | --- |
| Kernel | GKI `6.1.118-android14-11` | `6.1.68-android14-11` | GKI `6.1.112-android14-11` |
| Translator | v2.0.7 | v2.0.7 (not the same file as OnePlus's) | v2.1.0 |
| `tango32.ko` | vendor_boot first-stage ramdisk; imported symbols same as this project | vendor_dlkm | `/system/etc/tango/`, `post-fs-data` |
| `/dev/tango32` | vendor `ueventd.rc` 0666 | `late-fs` chmod | system `ueventd.rc` 0666 |
| binfmt registration | `on late-fs && ro.vendor.oplus.tango_allow32=true`, same registration string | `early-init` | `early-init` |
| ABI declaration | system, product and vendor all write `abilist32`, odm overrides it to empty; what takes effect is decided by the framework's `QCOM_TANGO_SUPPORTED_ABIS` | system `build.prop` | system `build.prop` |
| `ro.zygote` | vendor writes `zygote64_32`, odm sets it back to `zygote64` | `zygote64` | `zygote64` |
| 32-bit zygote start timing | On demand: framework `TangoPackage` sets `persist.sys.tango_zygote32.start` only after finding an installed 32-bit app, and starting a 32-bit app starts it and waits on the spot | Lazy start | `zygote-start` at boot |
| ART GC | `enable_uffd_gc=true` | `false` | `false` |
| 32-bit GPU stack | vendor/lib ships it; 32-bit and 64-bit Adreno are both `V@0762.34` | 32-bit 0762.4, 64-bit 0762.30 to 0762.36 | 32-bit 0762.4, 64-bit 0762.28 |
| Vendor 32-bit self-test | The rc imports with `${ro.zygote}`, no takeover needed | Same as OnePlus | Must be taken over |
| Pre-translation | No `.tango` files | 237 | None |

Conclusions:

1. Neither vendor starts the 32-bit zygote unconditionally at boot. Without modifying the framework, as soon as `abilist32` is declared, system_server must connect to the 32-bit zygote when boot completes (V77 crashed because it could not connect), so this project still starts it in `zygote-start`.
2. `enable_uffd_gc=false` is not required (OnePlus runs with `true`), but this project keeps `false`.
3. The translator version is not the deciding difference. On QEMU, v2.0.7 also hangs on the multi-threaded fatal path (see [QEMU tests](#qemu-tests)).
4. The OnePlus 12 vendor/lib is a complete 32-bit library set from the same platform with matching 32/64-bit versions, and could serve as an alternative source for the 32-bit GPU stack (not tried).

## Root Cause: WALT 32-bit Task Check

All versions from V68 to V882 with the 32-bit zygote rebooted the whole device 8 to 28 seconds after the 32-bit zygote preload or the first 32-bit app process started. V882's console-ramoops pointed to a `BUG()` in `walt.c`. llvm-objdump was used to disassemble `android_rvh_set_task_cpu` in the `sched-walt.ko` of Lenovo and of the OnePlus 12:

- **Lenovo** (`6.1.78-android14-11`): when bit 22 (`TIF_32BIT`) of the task's `thread_info.flags` is 1 and the target CPU is not in `system_32bit_el0_cpumask()`, it reads `sysctl_panic_on_walt_bug`. When its high bits equal the magic number `0x4544de00`: bit 4 prints `WALT-BUG selecting non 32 bit cpu`, and bit 1 runs `walt_task_dump` and `BUG()`. The default value is `0x4544de1b`, with both bits on.
- **OnePlus 12** (`6.1.118-android14-11`): the same code first checks `system_32bit_el0_cpumask()`, and when the mask is empty (no 32-bit cores) it skips the whole check.

The 8 Gen 3 has no AArch32 cores, so the 32-bit core mask is always empty. When `tango32.ko` forwards a compat ioctl it must set `TIF_32BIT` (the driver relies on `in_compat_syscall()` to determine the argument layout); the task sleeps during this period and, once woken, is migrated, and Lenovo's WALT triggers `BUG()`. This explains why QEMU (`-cpu max`, all cores support 32-bit, and no WALT module) cannot reproduce it.

Fix: the boot guard writes `0` to `/proc/sys/walt/panic_on_walt_bug` before allowing 32-bit; the magic number no longer matches and the whole set of WALT-BUG checks is skipped. The cost is that WALT's other BUG checks are also disabled. A cleaner approach is to patch `sched-walt.ko` with an empty-mask test as OnePlus has and repack `vendor_boot`; this has not been done.

## Other Problems Found Early

These problems were fixed before the root cause was located, and all are built into the current implementation:

| Problem | Symptom | Fix |
| --- | --- | --- |
| Lenovo vendor's `boringssl_self_test32_vendor` program is missing and has `reboot_on_failure` | V68 boot loop, `reboot,boringssl-self-check-failed` | Taken over by `tango-boringssl32.rc` |
| `ro.zygote=zygote64_32` written in system is overridden by the vendor's `zygote64` | V77 the 32-bit zygote did not start, system_server could not connect to `zygote_secondary` | Do not change `ro.zygote`, ship own service |
| init's `insmod` in `early-init` did not load successfully | V78 no `/dev/tango32`, all 32-bit library CFI checks hit SIGILL | Changed to load in `post-fs-data` by the herald domain |
| odrefresh compiled the 32-bit boot image for CMC, and 32-bit ART can only use CC | V79 `read barrier state mismatch` | `enable_uffd_gc=false` |
| The stock `init.rc` unconditionally runs `start zygote_secondary` | V881 on fallback it was pulled up and aborted repeatedly, triggering a `updatable_crashing` whole-device reboot | Service renamed to `tango_zygote32` |

## QEMU Tests

Environment: `qemu-system-aarch64 -M virt -cpu max -smp 4 -m 1024`, with the same GKI Image, `tango32.ko` and `tango_translator` as the device. The binfmt registration string must be written as literal `\x7f\x45...` text, which the kernel unescapes; writing real binary truncates at the first `\0`, and 32-bit programs would be run natively as AArch32.

- 32-bit static busybox workloads of concurrency, fork, directory traversal, signals, networking and mixed 32/64-bit runs (about 40 minutes): the 19 serial logs contain no Oops, BUG, WARNING, panic, RCU stall or softlockup.
- Fatal paths of the self-built bionic static 32-bit multi-threaded program `thr32`: the kernel gives no warning, but the translator in user space hangs (all threads stuck in the 64-bit `futex_wait`) or misreports SIGSEGV as SIGABRT. The native AArch32 control is normal. With 8 threads it does not hang.

| Scenario (32 threads) | v2.1.0 (7 rounds) | v2.0.7 (20 rounds) | Native AArch32 (20 rounds) |
| --- | --- | --- | --- |
| Busy main thread SIGSEGV | 3 normal, 3 misreported SIGABRT, 1 hang | 9 normal, 11 misreported SIGABRT | 20 normal |
| Busy abort | 6 normal, 1 hang | 19 normal, 1 hang | 20 normal |
| exit_group | 7 normal | 20 normal | 20 normal |
| Multi-threaded parent fork then exec | 7 normal | 18 normal, 2 misreported SIGABRT | 19 normal, 1 hang (QEMU's own problem) |

QEMU does not cover: the `compat_ioctl` of device drivers such as binder, kgsl and dmabuf heap, Lenovo's vendor_dlkm modules and vendor hooks, SELinux, and a real CPU without AArch32.

## Possible Improvements

None of these block use:

- Patch `sched-walt.ko` with an empty-mask test, restoring WALT's other checks.
- Start the 32-bit zygote on demand, which requires modifying the framework's `ZygoteProcess`.
- Pre-translation, which requires modifying the framework and installd source.
- Change `TANGO_DEBUG` to `off` once stable.
