---
name: research-es
description: "Investigación de Tango en el TB321FU: enfoque final, comparación con el Xiaomi 14 Ultra y OnePlus, causa real del reinicio del equipo, pruebas con QEMU"
metadata:
  version: "1.0.0"
  updated: "2026-11-05"
  lang: "es"
---

# Notas de investigación de Tango

Este documento recoge el fundamento técnico por el que Tango pasó, en el TB321FU (Snapdragon 8 Gen 3, vendor Android 15 de Lenovo), de no arrancar a ejecutar correctamente APK de 32 bits: el enfoque final, la comparación con las integraciones de fabricantes de la misma plataforma, la causa real del reinicio del equipo y los resultados de las pruebas con QEMU. Los síntomas en dispositivo real de cada versión se registran cronológicamente en [el HISTORY de tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/HISTORY.md#真机测试经过); la implementación está en ese repositorio.

V883 (2026-10-31) arrancó en el dispositivo real con las 5 partes y los APK de 32 bits funcionan correctamente.

## Enfoque final

| Aspecto | Enfoque | Origen |
| --- | --- | --- |
| Traductor | `tango_translator` de Tango v2.1.0 instalado en `/system_ext/bin`; `binfmt_misc` (`POCF`) registrado en `early-init` | Paquete de integración oficial; igual en Xiaomi y OnePlus |
| Módulo del kernel | `tango32.ko` compilado para el GKI 6.1.112 del dispositivo, instalado en `/system/etc/tango/` y cargado en `post-fs-data` por el dominio herald; ueventd da 0666 a `/dev/tango32` | Código fuente oficial; Xiaomi y OnePlus lo cargan antes, en la fase de módulos de vendor, y la temporización de este proyecto es suficientemente temprana |
| SELinux | `tango.te` oficial expandido, con `tango_log_file` y la lectura de `proc_security` por `zygote` | Oficial; igual en Xiaomi y OnePlus |
| Pila GPU de 32 bits | Adreno de 32 bits `V@0762.4` y gralloc/mapper del Xiaomi 14 Ultra, superpuestos como overlay sobre `/vendor/lib` durante el arranque | Firmware de Xiaomi |
| Declaración de ABI | `ro.product.cpu.abilist*` escritos explícitamente en el `build.prop` de system | Igual en Xiaomi y OnePlus |
| Zygote de 32 bits | No se modifica `ro.zygote`; servicio propio `tango_zygote32` (nombre de socket `zygote_secondary`, `disabled`, sin `onrestart restart zygote`), iniciado en `zygote-start` tras el visto bueno de la salvaguarda de arranque | Xiaomi `zygote_mi_secondary`, OnePlus `zygote_tango` |
| Autocomprobación de 32 bits de vendor | `tango-boringssl32.rc` define antes el servicio homónimo, sustituido por un programa de 64 bits | Específico del vendor de Lenovo |
| GC de ART | `ro.dalvik.vm.enable_uffd_gc=false` (system y product) | Igual en Xiaomi; en OnePlus es `true`, no imprescindible |
| Corrección del reinicio del equipo | Escribir `0` en `/proc/sys/walt/panic_on_walt_bug` antes de dar paso a los 32 bits | Este proyecto |

## Xiaomi 14 Ultra

El objeto de comparación es xiaomi.eu OS2.0.213 (Android 15); OS3 funciona igual. Xiaomi HyperOS incluye Tango de serie en el 8 Gen 3.

| Aspecto | Xiaomi | Este proyecto |
| --- | --- | --- |
| Traductor | `/system_ext/bin/tango_translator` v2.0.7, etiqueta `system_lib_file` | v2.1.0, misma ruta y misma etiqueta |
| Registro de binfmt | `system_ext/etc/init/mi-32bit-trans.rc`, `on early-init && property:ro.vendor.mi_fake_32bit_support=true`; la cadena de registro y el indicador `POCF` son idénticos carácter por carácter | Igual |
| Módulo del kernel | `vendor_dlkm/lib/modules/tango32.ko`, en `modules.load`, cargado por init como un módulo de vendor normal; `chmod 0666 /dev/tango32` en `late-fs` | `/system/etc/tango/`, cargado en `post-fs-data` |
| Código del módulo | Importa 31 símbolos, totalmente idénticos al .ko de este proyecto (mismo código fuente); compilado para el kernel de Xiaomi `6.1.68-android14-11` | Compilado para GKI `6.1.112-android14-11` |
| `ro.zygote` | Se mantiene `zygote64`; `zygote_mi_secondary` propio (`--socket-name=zygote_secondary`, `class main` + `disabled`), y además arranque diferido con `sys.mi_zygote32.start` (el framework tiene el parche `MI_FAKE_32BIT_SUPPORTED_ABIS`) | Se mantiene `zygote64`, `tango_zygote32` propio |
| Declaración de ABI | `ro.system.product.cpu.abilist=arm64-v8a,armeabi-v7a,armeabi` en el `build.prop` de system | `ro.product.cpu.abilist*` en el `build.prop` de system |
| Autocomprobación de BoringSSL de 32 bits | El rc de vendor es `import boringssl_self_test.${ro.zygote}.rc`; con `zygote64` solo se ejecuta la de 64 bits | El rc de vendor de Lenovo solo mira `abilist32`, hay que tomar el control |
| GC de ART | `ro.dalvik.vm.enable_uffd_gc=false` en el `build.prop` de product | Igual |
| SELinux | Igual que el `tango.te` oficial; además `allowx app_zygote tango32_device ioctl 0x74a3-0x74a7` | Reglas oficiales más `tango_log_file` y `proc_security` |
| Pretraducción | system tiene 237 archivos pretraducidos `.tango`; `tango_pretranslator` pretraduce al instalar una app | No hay (requiere modificar el código fuente del framework) |
| Pila GPU de 32 bits | vendor/lib incluye el Adreno de 32 bits `V@0762.4` | Los mismos 30 archivos tomados de Xiaomi |

Lo que Xiaomi tiene y este proyecto no puede hacer: el arranque diferido en el lado del framework y la pretraducción al instalar, que requieren el código fuente.

## OnePlus 13T

El OnePlus 13T usa el Snapdragon 8 Elite (SM8750), una plataforma distinta. El paquete usado para la comparación es una conversión de un equipo de la versión de China flasheado con OxygenOS 16.0.3.501 de la India: super es OxygenOS, y odm y my_region son ColorOS de la versión de China.

- OxygenOS no contiene Tango: no hay `app_process32`, no hay traductor y `abilist32` está vacío.
- En las partes comunes a ambos se ve el enfoque de OnePlus:
  - `tango32.ko` está en `modules.load` del ramdisk de la primera etapa de vendor_boot y lo carga init en first-stage. vermagic `6.6.89-android15-8-o`; importa 30 símbolos, y solo difiere del .ko de este proyecto en la sustitución de `system_cpucaps` del kernel 6.6, es decir, es el mismo código fuente.
  - La propiedad de activación está en el `build.prop` de odm: `ro.vendor.oplus.tango_binfmt_misc_support=1`.
  - El traductor está en `/system/bin/tango_translator`, con la misma etiqueta `system_lib_file`.
  - El zygote de 32 bits (`system_ext/etc/init/init.otranslator.zygote.rc`) no modifica `ro.zygote` y define el servicio propio `zygote_tango` (`--socket-name=zygote_secondary`, `class main` + `disabled`, sin `onrestart restart zygote`), controlado por la propiedad persistente `persist.sys.tango_zygote32.start=1`.
- Conclusión: ni Xiaomi ni OnePlus usan el nombre de servicio original `zygote_secondary`, porque el `on zygote-start` del `init.rc` original lo inicia incondicionalmente.

## OnePlus 12

El objeto de comparación es el OnePlus 12 de la versión de China con ColorOS PJD110_15.0.0.860(CN01), Android 15, parche de 2025-09, con el mismo SM8650 que el TB321FU.

| Aspecto | OnePlus 12 | Xiaomi 14 Ultra | Este proyecto |
| --- | --- | --- | --- |
| Kernel | GKI `6.1.118-android14-11` | `6.1.68-android14-11` | GKI `6.1.112-android14-11` |
| Traductor | v2.0.7 | v2.0.7 (no es el mismo archivo que el de OnePlus) | v2.1.0 |
| `tango32.ko` | Ramdisk de la primera etapa de vendor_boot; los símbolos importados son los mismos que en este proyecto | vendor_dlkm | `/system/etc/tango/`, `post-fs-data` |
| `/dev/tango32` | `ueventd.rc` de vendor, 0666 | chmod en `late-fs` | `ueventd.rc` de system, 0666 |
| Registro de binfmt | `on late-fs && ro.vendor.oplus.tango_allow32=true`, misma cadena de registro | `early-init` | `early-init` |
| Declaración de ABI | system, product y vendor escriben `abilist32`, y odm lo vuelve a sobrescribir con vacío; el valor realmente efectivo lo decide `QCOM_TANGO_SUPPORTED_ABIS` del framework | `build.prop` de system | `build.prop` de system |
| `ro.zygote` | vendor escribe `zygote64_32`, odm lo vuelve a poner en `zygote64` | `zygote64` | `zygote64` |
| Momento de inicio del zygote de 32 bits | Bajo demanda: `TangoPackage` del framework solo establece `persist.sys.tango_zygote32.start` cuando detecta una app de 32 bits instalada, y se inicia y espera en el momento de lanzar una app de 32 bits | Arranque diferido | `zygote-start` durante el arranque |
| GC de ART | `enable_uffd_gc=true` | `false` | `false` |
| Pila GPU de 32 bits | vendor/lib incluye la propia; el Adreno de 32 y de 64 bits son `V@0762.34` | 0762.4 de 32 bits, 0762.30 a 0762.36 de 64 bits | 0762.4 de 32 bits, 0762.28 de 64 bits |
| Autocomprobación de 32 bits de vendor | El rc importa con `${ro.zygote}`, no hace falta tomar el control | Igual que OnePlus | Hay que tomar el control |
| Pretraducción | No hay archivos `.tango` | 237 | No hay |

Conclusiones:

1. Ninguno de los dos inicia incondicionalmente el zygote de 32 bits durante el arranque. Sin modificar el framework, en cuanto se declara `abilist32`, system_server tiene que conectarse al zygote de 32 bits al terminar el arranque (V77 se bloqueó por no poder conectarse), así que este proyecto lo sigue iniciando en `zygote-start`.
2. `enable_uffd_gc=false` no es imprescindible (OnePlus funciona con `true`), pero este proyecto mantiene `false`.
3. La versión del traductor no es la diferencia decisiva. En QEMU, v2.0.7 también se cuelga en la ruta fatal multihilo (véase [Pruebas con QEMU](#pruebas-con-qemu)).
4. El vendor/lib del OnePlus 12 es un conjunto completo de bibliotecas de 32 bits de la misma plataforma, con versiones de 32 y 64 bits coincidentes, y puede servir como fuente alternativa de la pila GPU de 32 bits (sin probar).

## Causa raíz: comprobación de tareas de 32 bits en WALT

Las versiones V68 a V882 con zygote de 32 bits reiniciaban el equipo entre 8 y 28 segundos después de la precarga del zygote de 32 bits o del arranque del primer proceso de app de 32 bits. El console-ramoops de V882 apuntaba a un `BUG()` en `walt.c`. Se desensambló con llvm-objdump `android_rvh_set_task_cpu` de los `sched-walt.ko` de Lenovo y del OnePlus 12:

- **Lenovo** (`6.1.78-android14-11`): cuando el bit 22 (`TIF_32BIT`) de `thread_info.flags` de la tarea es 1 y la CPU de destino no está en `system_32bit_el0_cpumask()`, se lee `sysctl_panic_on_walt_bug`. Si sus bits altos son iguales al número mágico `0x4544de00`: el bit 4 imprime `WALT-BUG selecting non 32 bit cpu`, y el bit 1 ejecuta `walt_task_dump` y `BUG()`. El valor por defecto es `0x4544de1b`, con ambos bits activados.
- **OnePlus 12** (`6.1.118-android14-11`): el mismo fragmento de código comprueba antes `system_32bit_el0_cpumask()`, y si la máscara está vacía (no hay núcleos de 32 bits) omite toda la comprobación.

El 8 Gen 3 no tiene núcleos AArch32, así que la máscara de núcleos de 32 bits está siempre vacía. Al reenviar un ioctl compat, `tango32.ko` debe activar `TIF_32BIT` (el controlador determina la disposición de los parámetros con `in_compat_syscall()`); si durante ese tiempo la tarea duerme, la despiertan y la migran, el WALT de Lenovo dispara `BUG()`. Esto explica por qué no se puede reproducir en QEMU (`-cpu max`: todos los núcleos admiten 32 bits, y tampoco hay módulo WALT).

Corrección: la salvaguarda de arranque escribe `0` en `/proc/sys/walt/panic_on_walt_bug` antes de dar paso a los 32 bits; el número mágico deja de coincidir y se omite todo el conjunto de comprobaciones WALT-BUG. El coste es que también se desactivan las demás comprobaciones BUG de WALT. Una solución más limpia sería, como hace OnePlus, añadir a `sched-walt.ko` la comprobación de máscara vacía y reempaquetar `vendor_boot`; no se ha hecho.

## Otros problemas de la investigación inicial

Estos problemas se corrigieron antes de localizar la causa real y están todos incorporados en la implementación actual:

| Problema | Síntoma | Corrección |
| --- | --- | --- |
| El programa `boringssl_self_test32_vendor` de vendor de Lenovo no existe y lleva `reboot_on_failure` | V68, bucle de arranque, `reboot,boringssl-self-check-failed` | `tango-boringssl32.rc` toma el control |
| `ro.zygote=zygote64_32` escrito en system se sobrescribe con el `zygote64` de vendor | V77, el zygote de 32 bits no arrancó y system_server no puede conectarse a `zygote_secondary` | No modificar `ro.zygote`, servicio propio |
| El `insmod` de init en `early-init` no consiguió cargar | V78, no hay `/dev/tango32`, y todas las comprobaciones CFI de las bibliotecas de 32 bits dan SIGILL | Pasa a cargarse en `post-fs-data` por el dominio herald |
| odrefresh compila la imagen de arranque de 32 bits para CMC, y el ART de 32 bits solo puede usar CC | V79, `read barrier state mismatch` | `enable_uffd_gc=false` |
| El `init.rc` original hace `start zygote_secondary` incondicionalmente | V881, al volver atrás se inicia y aborta repetidamente, lo que dispara el reinicio del equipo por `updatable_crashing` | Se renombra el servicio a `tango_zygote32` |

## Pruebas con QEMU

Entorno: `qemu-system-aarch64 -M virt -cpu max -smp 4 -m 1024`, con la misma Image GKI que el dispositivo, `tango32.ko` y `tango_translator`. La cadena de registro de binfmt hay que escribirla como texto literal `\x7f\x45...`, que el kernel desescapa; si se escribe binario real, se trunca en el primer `\0` y los programas de 32 bits se ejecutan de forma nativa en AArch32.

- Cargas de busybox estático de 32 bits con concurrencia, fork, recorrido de directorios, señales, red y ejecución mezclada de 32 y 64 bits (unos 40 minutos): en 19 registros de puerto serie no hubo Oops, BUG, WARNING, panic, RCU stall ni softlockup.
- Ruta fatal del programa multihilo estático de 32 bits `thr32`, compilado con bionic propio: el kernel no emite avisos, pero el espacio de usuario del traductor se cuelga (todos los hilos detenidos en `futex_wait` de 64 bits) o informa por error SIGSEGV como SIGABRT. El control con AArch32 nativo funciona bien. Con 8 hilos no se cuelga.

| Escenario (32 hilos) | v2.1.0 (7 rondas) | v2.0.7 (20 rondas) | AArch32 nativo (20 rondas) |
| --- | --- | --- | --- |
| SIGSEGV del hilo principal en ocupado | 3 normales, 3 SIGABRT por error, 1 colgado | 9 normales, 11 SIGABRT por error | 20 normales |
| abort en ocupado | 6 normales, 1 colgado | 19 normales, 1 colgado | 20 normales |
| exit_group | 7 normales | 20 normales | 20 normales |
| Proceso padre multihilo con fork y luego exec | 7 normales | 18 normales, 2 SIGABRT por error | 19 normales, 1 colgado (problema del propio QEMU) |

Lo que QEMU no cubre: el `compat_ioctl` de controladores de dispositivo como binder, kgsl y dmabuf heap, los módulos vendor_dlkm y vendor hooks de Lenovo, SELinux, y una CPU real sin AArch32.

## Posibles mejoras

Ninguna impide el uso:

- Añadir a `sched-walt.ko` la comprobación de máscara vacía, para restablecer las demás comprobaciones de WALT.
- Iniciar el zygote de 32 bits bajo demanda; requiere modificar `ZygoteProcess` del framework.
- Pretraducción; requiere modificar el código fuente del framework y de installd.
- Cuando sea estable, cambiar `TANGO_DEBUG` a `off`.
