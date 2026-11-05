---
name: porting-guide-es
description: "Pasos para portar el traductor binario Tango a dispositivos Android sin AArch32 y ejecutar APK de 32 bits"
metadata:
  version: "1.0.0"
  updated: "2026-11-05"
  lang: "es"
---

# Guía de portado de Tango

Este documento explica cómo usar el traductor binario Tango para instalar y ejecutar APK de 32 bits sin root en dispositivos Android arm64 cuya CPU no admite AArch32. El método oficial del paquete de integración de Tango consiste en aplicar parches al código fuente de AOSP (dispositivo de referencia: Pixel 6); este documento está pensado para quien no puede o no quiere recompilar el código fuente: reescribe directamente la imagen de system ya compilada y no modifica el framework ni las particiones de vendor. Cada paso corresponde a archivos de la [implementación de referencia del TB321FU](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/README.md), verificada en un dispositivo real, el Lenovo Legion Y700 de tercera generación (Snapdragon 8 Gen 3, Android 15). Los demás dispositivos y versiones de Android no están verificados; los puntos marcados como "por confirmar" deben probarse en el dispositivo de destino.

## Principio de funcionamiento

Tango traduce a nivel de proceso; no es un native bridge:

1. El `binfmt_misc` del kernel entrega cada ELF ARM de 32 bits al programa arm64 estático `tango_translator`.
2. El traductor convierte en espacio de usuario las instrucciones ARM32 a ARM64; el JIT necesita `execmem`.
3. Las llamadas al sistema que el kernel debe tratar con semántica de 32 bits (compat ioctl, etc.) se las proporciona al traductor el módulo del kernel `tango32` a través de `/dev/tango32`.
4. Las apps de 32 bits se crean con fork desde un zygote de 32 bits; ese zygote de 32 bits es, a su vez, un proceso traducido.

Por tanto, portar consiste en: instalar el traductor y el módulo del kernel, completar la política SELinux, añadir las bibliotecas de espacio de usuario de 32 bits, declarar la ABI de 32 bits e iniciar el zygote de 32 bits, y resolver después los problemas de arranque que provoca declarar la ABI de 32 bits.

## Requisitos previos

| Condición | Cómo comprobarla | Si no se cumple |
| --- | --- | --- |
| La CPU no tiene AArch32 | Los dispositivos con AArch32 no necesitan Tango | No hace falta portar |
| Kernel 5.6 o superior, con `CONFIG_COMPAT` y `CONFIG_BINFMT_MISC` activados | `adb shell uname -r`; `adb shell zcat /proc/config.gz \| grep -E 'CONFIG_COMPAT=\|BINFMT_MISC'` (requiere `IKCONFIG_PROC` en el kernel) | No se puede portar |
| Acceso al código fuente y a la configuración del kernel del dispositivo | En dispositivos GKI, mire el tag y el build number en `uname -r` | No se puede compilar `tango32.ko` |
| La imagen de system incluye bibliotecas de 32 bits | La imagen contiene `bin/app_process32`, `lib/libc.so`, `lib/libandroid_runtime.so` | Una imagen base puramente de 64 bits carece de bionic, ART y bibliotecas nativas del framework de 32 bits, y este método no sirve |
| Se puede modificar y flashear la imagen de system | Bootloader desbloqueado, con posibilidad de construir un GSI o una ROM propios | No se puede portar |
| Servicio de init que se ejecute como root y en un dominio SELinux independiente | Sirve para cargar el módulo y modificar sysctl | Defina usted mismo un dominio en la política; la implementación de referencia usa [herald-core](https://github.com/CharlesGool/herald-core) |
| Paquete de integración de Tango | Se obtiene de [Amanieu Systems](https://www.amanieusystems.com/); contiene `prebuilts/tango_translator`, `kernel-module/`, los parches de AOSP (con el rc de binfmt y sepolicy) y `EULA.txt` | No se puede portar |

## Paso 1: traductor y binfmt_misc

1. Instale `tango_translator` en `/system_ext/bin/` (OnePlus lo instala en `/system/bin/`; la ubicación no importa), con modo 0755 y etiqueta SELinux **MUST** `system_lib_file`: cada dominio lo mapea para ejecutarlo como intérprete, y con `system_file` se deniega.
2. En `early-init`, monte `binfmt_misc` y escriba la cadena de registro. La cadena de registro, la máscara y el indicador `POCF` se copian tal cual del paquete de integración; Xiaomi, OnePlus y la implementación de referencia son idénticos carácter por carácter.
3. Use una propiedad (en la implementación de referencia, `tango.enabled`) para activar y desactivar `/proc/sys/fs/binfmt_misc/tango_translator` en tiempo de ejecución, lo que facilita la depuración.
4. Durante el desarrollo, ponga el directorio de registros del traductor bajo `/metadata`: se monta en first-stage y se conserva incluso en un bucle de arranque.

Referencia: [001.core](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/001.core/README.md).

## Paso 2: SELinux

1. Añada a la política de system los archivos `sepolicy/system/{public,private}/tango.te` y `te_macros` del parche de AOSP del paquete de integración. Si la imagen binaria base no tiene el código fuente de la política, expándalos a CIL y añádalos a `plat_sepolicy.cil`; tras añadirlos, `plat_sepolicy_and_mapping.sha256` ya no coincide y init recompila la política en el dispositivo durante el arranque.
2. La macro `tango_translated` debe cubrir todos los dominios que ejecuten código de 32 bits: `zygote`, `app_zygote`, `webview_zygote`, `appdomain`, `crash_dump` y los dominios de la autocomprobación de 32 bits.
3. Etiquete `/dev/tango32`, la ruta del traductor, el directorio de registros y las propiedades `tango.*`.
4. Tras el arranque, revise `dmesg | grep avc` para añadir reglas. La implementación de referencia añadió la lectura de `proc_security` por `zygote`.

Habilitar `execmem` para `appdomain` infringe un neverallow de AOSP y reduce el endurecimiento de todas las apps; es un requisito de la propia política oficial de Tango.

Referencia: [002.sepolicy](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/002.sepolicy/README.md).

## Paso 3: módulo del kernel tango32

1. **MUST** compilar `tango32.ko` con el mismo código fuente y la misma configuración que el kernel del dispositivo. GKI activa `TRIM_UNUSED_KSYMS`, y un código fuente o una configuración distintos provocan `Unknown symbol` o `disagrees about version of symbol`.
2. Dispositivos GKI: obtenga el tag y el build number de `uname -r`, compile el kernel y los módulos con Kleaf, y compruebe que el UTS_RELEASE del `Image` compilado y el `vermagic` del .ko coinciden con los del dispositivo. Los comandos concretos están en el [método de compilación de la implementación de referencia](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/003.kernel-module/README.md#编译-tango32ko).
3. Cargue el módulo antes de que arranque el primer proceso de 32 bits. Las integraciones de los fabricantes lo incluyen en la lista de módulos de vendor, que carga init en first-stage; si solo se modifica system, basta con cargarlo en `post-fs-data` con un servicio root ejecutado mediante `exec_start`. En la implementación de referencia, el `insmod` del propio init en `early-init` no consiguió cargarlo, y no se ha averiguado el motivo.
4. Con ueventd, ponga `/dev/tango32` en modo 0666.

El módulo es GPL-2.0. Al distribuir una imagen que contenga `tango32.ko` hay que ofrecer el código fuente.

Referencia: [003.kernel-module](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/003.kernel-module/README.md).

## Paso 4: bibliotecas GPU de 32 bits

El vendor de los dispositivos puramente de 64 bits suele tener solo controladores GPU de 64 bits. Los procesos de apps de 32 bits usan HWUI, y sin EGL, GLES, Vulkan y gralloc/mapper de 32 bits no pueden dibujar la interfaz.

1. Tome las bibliotecas de 32 bits del firmware de un dispositivo con el **mismo SoC** que admita apps de 32 bits de fábrica; elija preferentemente un firmware cuyo controlador de 32 bits tenga la misma versión que el de 64 bits del dispositivo de destino, o una muy cercana.
2. Calcule la clausura con las dependencias de `readelf` y los nombres de bibliotecas de `dlopen` que aparecen en las cadenas, y tome solo los archivos necesarios.
3. La partición vendor está protegida por AVB y no admite añadir archivos directamente. Ponga las bibliotecas en system y, en `post-fs`, superpóngalas como overlay de solo lectura sobre `/vendor/lib` con el `mount` de init, con el directorio de system primero en lowerdir.
4. Calcule las etiquetas según `plat_file_contexts` y el `vendor_file_contexts` original, y contrástelas con la etiqueta real de los archivos homónimos de 64 bits originales.

Cuando las versiones de los controladores de 32 y 64 bits difieren, la compatibilidad con el KGSL del kernel está por confirmar.

Referencia: [004.vendor-lib32](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/004.vendor-lib32/README.md).

## Paso 5: declarar la ABI de 32 bits y el zygote de 32 bits

1. Escriba explícitamente en el `build.prop` de system `ro.product.cpu.abilist`, `abilist32` y `abilist64`. Antes, confirme si vendor, odm o product los sobrescriben con valores cargados después: init carga en el orden system, vendor, odm, product, y los `ro.*` cargados después prevalecen.
2. **No** modifique `ro.zygote`. Si lo cambia a `zygote64_32`, el valor de vendor u odm lo sobrescribe; y aunque surtiera efecto, el `init.zygote64_32.rc` original introduce la autocomprobación de 32 bits y `onrestart restart zygote`.
3. Defina su propio servicio de zygote de 32 bits, copiando la línea de comandos del `zygote_secondary` original:
   - El nombre del servicio **MUST NOT** ser `zygote_secondary`: el `on zygote-start` del `init.rc` original lo inicia incondicionalmente, y `disabled` no lo impide. Xiaomi usa `zygote_mi_secondary` y OnePlus, `zygote_tango`.
   - El nombre del socket **MUST** ser `zygote_secondary`; system_server se conecta por el nombre del socket.
   - Quite `onrestart restart zygote`, de modo que un fallo del zygote de 32 bits solo afecte a las apps de 32 bits.
   - Añada `disabled` y arránquelo condicionalmente en `zygote-start`.
4. Android 14 y posteriores: establezca `ro.dalvik.vm.enable_uffd_gc=false`, escrito en el `build.prop` que realmente esté en vigor. Tango no admite userfaultfd, por lo que el ART de 32 bits solo puede usar el GC CC, mientras que odrefresh compila por defecto la imagen de arranque de 32 bits para CMC. El OnePlus 12 funciona con el valor en `true`, así que este punto no es imprescindible, pero evita que la imagen de arranque no coincida.

Si no se modifica el framework, el zygote de 32 bits debe iniciarse durante el arranque: en cuanto se declara `abilist32`, system_server se conecta a él al terminar el arranque. El arranque bajo demanda de Xiaomi y OnePlus depende de parches del framework.

Referencia: [005.zygote32](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/005.zygote32/README.md).

## Paso 6: resolver los problemas que causa declarar la ABI de 32 bits

### Autocomprobación de 32 bits

Cuando `abilist32` deja de estar vacío, al principio del arranque se ejecuta de forma síncrona la autocomprobación de BoringSSL de 32 bits; la de vendor suele llevar `reboot_on_failure`. Los vendor que no admiten 32 bits de fábrica suelen carecer de `boringssl_self_test32`, lo que produce un bucle de arranque con `sys.boot.reason.last` igual a `reboot,boringssl-self-check-failed`.

1. Examine todos los rc de init originales de vendor y odm para encontrar los servicios que lleven `reboot_on_failure` o `critical`, cuyo programa sea de 32 bits o no exista, y que se activen por `abilist32`. El [check_vendor32.py](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/005.zygote32/check_vendor32.py) de la implementación de referencia hace esto en tiempo de compilación.
2. Defina antes, en `/system/etc/init`, un servicio con el mismo nombre que ejecute un programa de 64 bits. `/system/etc/init` se analiza antes que `/vendor/etc/init`, y la definición homónima de vendor se descarta si no lleva `override`. El sustituto **MUST NOT** ser `/system/bin/true`: init no tiene una transición de dominio hacia `toolbox_exec`.

### Comprobación de tareas de 32 bits del planificador

Al reenviar llamadas al sistema compat, `tango32` activa temporalmente `TIF_32BIT` en la tarea. Si el planificador comprueba que "las tareas de 32 bits solo pueden ejecutarse en núcleos de 32 bits", en una CPU sin núcleos de 32 bits se produce un error.

En la implementación de referencia, el módulo WALT de Qualcomm de Lenovo, `sched-walt.ko`, ejecuta `BUG()` por este motivo, lo que se manifiesta como un reinicio del equipo unas decenas de segundos después de iniciar el zygote de 32 bits. El mismo módulo del OnePlus 12 omite la comprobación cuando la máscara de núcleos de 32 bits está vacía, y no se ve afectado. Método de diagnóstico:

1. En dispositivos Qualcomm, compruebe si existe `/proc/sys/walt/panic_on_walt_bug`. Si existe, desensamble `android_rvh_set_task_cpu` de `sched-walt.ko` y mire si hay una comprobación de máscara vacía antes de la comprobación de `TIF_32BIT`.
2. Si no la hay, escriba `0` en ese sysctl antes de iniciar el zygote de 32 bits (esto desactiva también el resto de comprobaciones BUG de WALT), o añada al módulo la comprobación de máscara vacía y reempaquete `vendor_boot`.
3. Está por confirmar si otros planificadores (por ejemplo, el de MediaTek) tienen comprobaciones similares.

Los detalles están en las [notas de investigación](research.md#causa-raíz-comprobación-de-tareas-de-32-bits-en-walt).

## Verificación

Flashee en dos fases; empezar sin declarar la ABI de 32 bits es lo que menos riesgo de arranque tiene:

1. Fase 1, solo los pasos 1 a 4. Tras el arranque:

   ```bash
   adb shell cat /proc/sys/fs/binfmt_misc/tango_translator   # enabled
   adb shell ls -lZ /dev/tango32                             # crw-rw-rw-, tango32_device
   adb shell /system/bin/app_process32                       # se ejecuta traducido e imprime el mensaje de uso
   adb shell dmesg | grep -iE 'tango|avc'
   ```

2. Fase 2, añadiendo los pasos 5 y 6. Tras el arranque:

   ```bash
   adb shell getprop sys.boot.reason.last
   adb shell getprop ro.product.cpu.abilist                  # incluye armeabi-v7a
   adb shell ps -A | grep zygote                             # están los zygote de 64 y de 32 bits
   ```

   Después, instale un APK de 32 bits y pruebe una interfaz que use GL y Vulkan.

Durante el desarrollo se recomienda añadir una salvaguarda de arranque: en `post-fs-data`, comprobar si el módulo y el arranque anterior son estables y, si no lo son, retirar la ABI de 32 bits y arrancar como puramente de 64 bits, guardando `/sys/fs/pstore`. El `tango-guard.sh` de la implementación de referencia se puede copiar tal cual; véase [005.zygote32](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/parts/005.zygote32/README.md#开机保险).

## Solución de problemas

| Síntoma | Causa habitual |
| --- | --- |
| Bucle de arranque, `reboot,boringssl-self-check-failed` | El programa de autocomprobación de 32 bits de vendor no existe; véase el paso 6 |
| Se queda en la animación de arranque, system_server se bloquea repetidamente | Se declaró `abilist32` pero el zygote de 32 bits no arrancó; compruebe `ro.zygote` y la definición del servicio |
| El zygote de 32 bits recibe SIGILL repetidamente | `/dev/tango32` no existe, el módulo no se cargó |
| `read barrier state mismatch` | La imagen de arranque de 32 bits se compiló para CMC; véase el punto 4 del paso 5 |
| Al volver a puramente de 64 bits sigue reiniciándose el equipo, `updatable_crashing` | El servicio del zygote de 32 bits se llamó `zygote_secondary` |
| El equipo se reinicia unas decenas de segundos después de que se ejecute el zygote de 32 bits | Comprobación de tareas de 32 bits del planificador; véase el paso 6 y mire `console-ramoops` en pstore |
| Las apps de 32 bits arrancan pero no tienen interfaz | Faltan las bibliotecas GPU de 32 bits, o el overlay no se montó |
| `insmod` informa `Unknown symbol` | El módulo no coincide con el código fuente o la configuración del kernel del dispositivo |

## Autorización

- Tango es software propietario de Amanieu Systems y está sujeto al EULA del paquete de integración. Este repositorio no incluye el traductor, el paquete de integración ni `tango32.ko`; quien porte debe obtenerlos y aceptar el EULA por su cuenta. Antes de distribuir una imagen que contenga el traductor, se recomienda pedir confirmación por escrito a Amanieu Systems.
- Las bibliotecas de 32 bits extraídas del firmware de otros fabricantes son archivos propietarios y **MUST NOT** subirse a un repositorio público.

Más detalles en los [avisos de terceros](THIRD_PARTY_NOTICES.md).
