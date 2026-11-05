---
name: porting-guide-en
description: "Steps for porting the Tango binary translator to Android devices without AArch32 to run 32-bit APKs"
metadata:
  version: "1.0.0"
  updated: "2026-11-05"
  lang: "en"
---

# Tango Porting Guide

This guide explains how to let 32-bit APKs install and run without root on arm64 Android devices whose CPU lacks AArch32, using the Tango binary translator. The official way of using the Tango integration package is to patch the AOSP source (reference device: Pixel 6); this guide targets the case where the source is unavailable or you do not want to rebuild it, and rewrites the compiled system image directly, leaving the framework and vendor partitions untouched. Every step maps to files in the [TB321FU reference implementation](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/README.md), which is verified on a real Legion Y700 Gen 3 (Snapdragon 8 Gen 3, Android 15). Other devices and Android versions are unverified; items marked "to be confirmed" must be tested on the target device.

## Principle

Tango is process-level translation, not a native bridge:

1. The kernel's `binfmt_misc` hands every 32-bit ARM ELF to the static arm64 program `tango_translator`.
2. The translator translates ARM32 instructions to ARM64 in user space; the JIT needs `execmem`.
3. System calls that the kernel must handle with 32-bit semantics (compat ioctl and the like) are provided to the translator by the kernel module `tango32` through `/dev/tango32`.
4. 32-bit apps are forked from a 32-bit zygote; the 32-bit zygote itself is also a translated process.

So porting means: install the translator and the kernel module, add the SELinux policy, add the 32-bit user-space libraries, declare the 32-bit ABI and start the 32-bit zygote, and then handle the boot problems caused by declaring the 32-bit ABI.

## Prerequisites

| Condition | How to check | If not met |
| --- | --- | --- |
| The CPU has no AArch32 | Devices with AArch32 do not need Tango | No porting needed |
| Kernel ≥ 5.6 with `CONFIG_COMPAT` and `CONFIG_BINFMT_MISC` enabled | `adb shell uname -r`; `adb shell zcat /proc/config.gz \| grep -E 'CONFIG_COMPAT=\|BINFMT_MISC'` (requires the kernel to enable `IKCONFIG_PROC`) | Cannot be ported |
| Source and config of the device kernel are available | For GKI devices, look at the tag and build number in `uname -r` | Cannot build `tango32.ko` |
| The system image ships 32-bit libraries | The image contains `bin/app_process32`, `lib/libc.so`, `lib/libandroid_runtime.so` | A pure 64-bit base lacks the 32-bit bionic, ART and framework native libraries; this method cannot be used |
| The system image can be modified and flashed | Bootloader unlocked, and you can build a GSI or ROM yourself | Cannot be ported |
| An init service that runs as root in its own SELinux domain | Used to load the module and change sysctls | Define a domain in the policy yourself; the reference implementation uses [herald-core](https://github.com/CharlesGool/herald-core) |
| The Tango integration package | Obtain it from [Amanieu Systems](https://www.amanieusystems.com/); it contains `prebuilts/tango_translator`, `kernel-module/`, the AOSP patch (including the binfmt rc and sepolicy) and `EULA.txt` | Cannot be ported |

## Step 1: Translator and binfmt_misc

1. Install `tango_translator` to `/system_ext/bin/` (OnePlus installs it in `/system/bin/`; the location does not matter), mode 0755, and its SELinux label **MUST** be `system_lib_file`: domains map it as an interpreter, and `system_file` would be denied.
2. Mount `binfmt_misc` in `early-init` and write the registration string. Copy the registration string, mask and `POCF` flag verbatim from the integration package; Xiaomi, OnePlus and the reference implementation are byte-identical.
3. Use a property (`tango.enabled` in the reference implementation) to toggle `/proc/sys/fs/binfmt_misc/tango_translator` at runtime, which eases troubleshooting.
4. During development, put the translator log directory under `/metadata`: it is mounted in first-stage and survives boot loops.

Reference: [001.core](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/001.core/README.md).

## Step 2: SELinux

1. Add `sepolicy/system/{public,private}/tango.te` and `te_macros` from the integration package's AOSP patch to the system policy. If the binary base has no policy source, expand them into CIL and append to `plat_sepolicy.cil`; after appending, `plat_sepolicy_and_mapping.sha256` no longer matches, and init recompiles the policy on the device at boot.
2. The macro `tango_translated` must cover every domain that runs 32-bit code: `zygote`, `app_zygote`, `webview_zygote`, `appdomain`, `crash_dump`, and the domains of the 32-bit self-test.
3. Label `/dev/tango32`, the translator path, the log directory and the `tango.*` properties.
4. After boot, check `dmesg | grep avc` and add rules. The reference implementation added `zygote` reading `proc_security`.

Granting `execmem` to `appdomain` violates an AOSP neverallow and lowers the hardening of all apps; this is a requirement of Tango's official policy itself.

Reference: [002.sepolicy](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/002.sepolicy/README.md).

## Step 3: tango32 Kernel Module

1. **MUST** build `tango32.ko` from the same source and config as the device kernel. GKI enables `TRIM_UNUSED_KSYMS`; a different source or config causes `Unknown symbol` or `disagrees about version of symbol`.
2. GKI devices: get the tag and build number from `uname -r`, build the kernel and modules with Kleaf, and check that the UTS_RELEASE of your built `Image` and the `vermagic` of the .ko both match the device. For the exact commands, see the [reference implementation's build method](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/003.kernel-module/README.md#编译-tango32ko).
3. Load the module before the first 32-bit process starts. Vendor integrations put it in the vendor module list, loaded by first-stage init; when only system is modified, it is enough to load it in `post-fs-data` from a root service run with `exec_start`. In the reference implementation, init's own `insmod` in `early-init` did not load it successfully, and the cause was not found.
4. Use ueventd to set `/dev/tango32` to 0666.

The module is GPL-2.0. Source must be provided when distributing an image that contains `tango32.ko`.

Reference: [003.kernel-module](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/003.kernel-module/README.md).

## Step 4: 32-bit GPU Libraries

The vendor partition of a pure 64-bit device usually has only 64-bit GPU drivers. A 32-bit app process uses HWUI, and without 32-bit EGL, GLES, Vulkan and gralloc/mapper it cannot draw its UI.

1. Take the 32-bit libraries from the firmware of a device with the **same SoC** that supported 32-bit apps out of the box; prefer firmware whose 32-bit driver version is the same as or close to the target device's 64-bit driver.
2. Compute the closure from the dependencies shown by `readelf` and the library names in `dlopen` strings, and take only the files needed.
3. The vendor partition is protected by AVB, so files cannot be added directly. Put the libraries in system and, in `post-fs`, use init's `mount` to stack a read-only overlay over `/vendor/lib`, with the system directory first in lowerdir.
4. Compute labels from `plat_file_contexts` and the stock `vendor_file_contexts`, and check them against the actual labels of the same-named stock 64-bit files.

When the 32-bit and 64-bit driver versions differ, compatibility with the kernel KGSL is to be confirmed.

Reference: [004.vendor-lib32](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/004.vendor-lib32/README.md).

## Step 5: Declare the 32-bit ABI and the 32-bit zygote

1. Write `ro.product.cpu.abilist`, `abilist32` and `abilist64` explicitly in the system `build.prop`. First check whether these properties are overridden by later-loaded values in vendor, odm or product: init loads in the order system, vendor, odm, product, and the later `ro.*` value wins.
2. **Do not** change `ro.zygote`. A change to `zygote64_32` is overridden by the vendor or odm value; even if it took effect, the stock `init.zygote64_32.rc` would bring in the 32-bit self-test and `onrestart restart zygote`.
3. Define your own 32-bit zygote service, copying the command line from the stock `zygote_secondary`:
   - The service name **MUST NOT** be `zygote_secondary`: the stock `init.rc` starts it unconditionally in `on zygote-start`, and `disabled` cannot stop it. Xiaomi uses `zygote_mi_secondary`, OnePlus uses `zygote_tango`.
   - The socket name **MUST** be `zygote_secondary`, because system_server connects by socket name.
   - Remove `onrestart restart zygote`, so a 32-bit zygote crash affects only 32-bit apps.
   - Add `disabled`, and start it conditionally in `zygote-start`.
4. Android 14 and above: set `ro.dalvik.vm.enable_uffd_gc=false` in the `build.prop` that actually takes effect. Tango does not support userfaultfd, so 32-bit ART can only use the CC GC, while odrefresh compiles the 32-bit boot image for CMC by default. OnePlus 12 keeps `true` and still runs, so this item is not required, but it avoids a boot image mismatch.

Without modifying the framework, the 32-bit zygote must start at boot: as soon as `abilist32` is declared, system_server connects to it when boot completes. The on-demand start of Xiaomi and OnePlus depends on framework patches.

Reference: [005.zygote32](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/005.zygote32/README.md).

## Step 6: Handle the Problems Caused by Declaring the 32-bit ABI

### 32-bit Self-test

Once `abilist32` is non-empty, a 32-bit BoringSSL self-test runs synchronously early in boot, and the vendor's usually carries `reboot_on_failure`. A vendor that did not support 32-bit out of the box often has no `boringssl_self_test32`, resulting in a boot loop with `sys.boot.reason.last` equal to `reboot,boringssl-self-check-failed`.

1. Scan all init rc files in the stock vendor and odm for services that carry `reboot_on_failure` or `critical`, whose program is 32-bit or missing, and that are triggered by `abilist32`. The reference implementation's [check_vendor32.py](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/005.zygote32/check_vendor32.py) does this at build time.
2. Define a same-named service first in `/system/etc/init`, replaced by a 64-bit program. `/system/etc/init` is parsed before `/vendor/etc/init`, and a same-named vendor definition without `override` is discarded. The stand-in **MUST NOT** be `/system/bin/true`: init has no domain transition to `toolbox_exec`.

### Scheduler 32-bit Task Check

When `tango32` forwards a compat system call, it temporarily sets the task's `TIF_32BIT`. If the scheduler checks that "32-bit tasks may only run on 32-bit cores", it errors on a CPU that has no 32-bit cores.

In the reference implementation, Lenovo's Qualcomm WALT module `sched-walt.ko` hits `BUG()` because of this, showing up as a whole-device reboot a few tens of seconds after the 32-bit zygote starts. The same module on the OnePlus 12 skips the check when the 32-bit core mask is empty, so it is unaffected. How to investigate:

1. On Qualcomm devices, check whether `/proc/sys/walt/panic_on_walt_bug` exists. If it does, disassemble `android_rvh_set_task_cpu` in `sched-walt.ko` and see whether an empty-mask test precedes the `TIF_32BIT` check.
2. If there is no such test, write `0` to that sysctl before starting the 32-bit zygote (this also disables WALT's other BUG checks), or patch the module with an empty-mask test and repack `vendor_boot`.
3. Whether other schedulers (such as MediaTek's) have a similar check is to be confirmed.

For details, see the [research notes](research.md#root-cause-walt-32-bit-task-check).

## Verification

Flash in two stages; not declaring the 32-bit ABI first carries the least boot risk:

1. Stage 1, steps 1 to 4 only. After boot:

   ```bash
   adb shell cat /proc/sys/fs/binfmt_misc/tango_translator   # enabled
   adb shell ls -lZ /dev/tango32                             # crw-rw-rw-, tango32_device
   adb shell /system/bin/app_process32                       # runs via translation, prints a usage hint
   adb shell dmesg | grep -iE 'tango|avc'
   ```

2. Stage 2, adding steps 5 and 6. After boot:

   ```bash
   adb shell getprop sys.boot.reason.last
   adb shell getprop ro.product.cpu.abilist                  # contains armeabi-v7a
   adb shell ps -A | grep zygote                             # both 64-bit and 32-bit zygote are present
   ```

   Then install a 32-bit APK and test a UI that uses GL and Vulkan.

During development, a boot guard is recommended: in `post-fs-data`, check that the module and the previous boot are stable, and if not, withdraw the 32-bit ABI to boot as pure 64-bit and save `/sys/fs/pstore`. The reference implementation's `tango-guard.sh` can be copied as is; see [005.zygote32](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/005.zygote32/README.md#开机保险).

## Troubleshooting

| Symptom | Common cause |
| --- | --- |
| Boot loop, `reboot,boringssl-self-check-failed` | The vendor 32-bit self-test program is missing; see step 6 |
| Stuck at the boot animation, system_server crashes repeatedly | `abilist32` is declared but the 32-bit zygote did not start; check `ro.zygote` and the service definition |
| 32-bit zygote repeatedly SIGILL | `/dev/tango32` does not exist, the module is not loaded |
| `read barrier state mismatch` | The 32-bit boot image was compiled for CMC; see step 5 item 4 |
| Whole-device reboot even after falling back to pure 64-bit, `updatable_crashing` | The 32-bit zygote service is named `zygote_secondary` |
| Whole-device reboot a few tens of seconds after the 32-bit zygote runs | The scheduler's 32-bit task check; see step 6; look at `console-ramoops` in pstore |
| 32-bit app starts but has no UI | The 32-bit GPU libraries are missing, or the overlay is not mounted |
| `insmod` reports `Unknown symbol` | The module's source or config differs from the device kernel's |

## Licensing

- Tango is proprietary software of Amanieu Systems, governed by the EULA in the integration package. This repository does not include the translator, the integration package or `tango32.ko`; porters must obtain them and accept the EULA themselves. Before distributing an image that contains the translator, written confirmation from Amanieu Systems is recommended.
- 32-bit libraries taken from other vendors' firmware are proprietary files and **MUST NOT** be committed to a public repository.

For details, see the [third-party notices](THIRD_PARTY_NOTICES.md).
