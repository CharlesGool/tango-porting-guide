# tango-porting-guide

<!-- Descomenta la siguiente línea cuando el proyecto tenga un icono -->
<!-- <img align="right" src="../resources/logo.png" width="160" alt="tango-porting-guide"> -->

[简体中文](../../README.md) | [English](../en/README.md) | **Español**

Los dispositivos Android cuya CPU no admite AArch32 no pueden ejecutar APK de 32 bits; este proyecto explica cómo portar el traductor binario Tango a una imagen de system ya compilada, sin modificar el código fuente, para ejecutar APK de 32 bits sin root, e incluye una implementación de referencia verificada en un dispositivo real.

[![License](https://img.shields.io/badge/License-GPL--3.0--or--later-orange)](../../LICENSE)

## Documentación

- Descripción general del proyecto: [README](README.md)

- Justificación del diseño: [DESIGN](DESIGN.md)

- Estado del proyecto: [LOG](LOG.md)
- Registros históricos: [HISTORY](HISTORY.md)
- Historial de cambios: [CHANGELOG](CHANGELOG.md)

- Avisos de terceros: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## Introducción

| Contenido | Descripción |
| --- | --- |
| [Guía de portado](porting-guide.md) | Comprobación de requisitos previos, 6 pasos de portado, verificación por fases y tabla de solución de problemas |
| [Notas de investigación](research.md) | Comparación punto por punto con las integraciones oficiales del Xiaomi 14 Ultra, el OnePlus 12 y el OnePlus 13T; la causa real del reinicio del equipo al declarar la ABI de 32 bits (comprobación de tareas de 32 bits del planificador WALT de Qualcomm); pruebas con QEMU |
| [Implementación de referencia](https://github.com/CharlesGool/tb321fu-tango-patch) | Parche completo para el Lenovo Legion Y700 de tercera generación TB321FU (Snapdragon 8 Gen 3, Android 15); en V883 (2026-10-31) los APK de 32 bits funcionan correctamente en el dispositivo real |

[Tango](https://www.amanieusystems.com/) es el traductor binario de ARM32 a ARM64 de Amanieu Systems. Su método oficial de integración consiste en aplicar parches al código fuente de AOSP; el método de este proyecto solo reescribe la imagen de system y sirve para GSI y para ROM sin acceso al código fuente.

Los dispositivos y versiones de Android distintos de la implementación de referencia no están verificados. Los problemas conocidos de la implementación de referencia están en [el LOG de tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/LOG.md#缺陷).

## Requisitos

- Dispositivo arm64 cuya CPU no tenga AArch32; kernel 5.6 o superior con `CONFIG_COMPAT` y `CONFIG_BINFMT_MISC` activados
- Acceso al código fuente y a la configuración del kernel del dispositivo, para compilar `tango32.ko`
- Imagen de system con bibliotecas de 32 bits (`app_process32`, bionic de 32 bits y bibliotecas nativas del framework de 32 bits)
- Bootloader desbloqueado, con posibilidad de modificar y flashear la imagen de system
- Paquete de integración de Tango, obtenido de Amanieu Systems y con su EULA aceptado

El método de comprobación de cada punto está en [los requisitos previos de la guía de portado](porting-guide.md#requisitos-previos).

## Instalación

Este proyecto es una guía con una implementación de referencia; no requiere instalación.

### Instalación rápida

```bash
git clone https://github.com/CharlesGool/tango-porting-guide.git
```

### Instalación normal

La implementación de referencia está en el repositorio independiente [tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch), un parche de [TB321FU-GSI-Mods](https://github.com/CharlesGool/TB321FU-GSI-Mods). Los parámetros de compilación y el modo de uso están en ese repositorio.

```bash
git clone https://github.com/CharlesGool/tb321fu-tango-patch.git
```

## Uso

1. Compruebe el dispositivo de destino según los [requisitos previos](porting-guide.md#requisitos-previos).
2. Reescriba la imagen de system siguiendo los pasos 1 a 6 de la [guía de portado](porting-guide.md), comparando cada paso con la parte correspondiente de la [implementación de referencia](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/patch/README.md).
3. Flashee primero la fase 1 (sin declarar la ABI de 32 bits) para verificar el traductor y el módulo del kernel, y después la fase 2; véase [Verificación](porting-guide.md#verificación).
4. Si el equipo no arranca o se reinicia, consulte la [tabla de solución de problemas](porting-guide.md#solución-de-problemas).

## Actualización

Nada por ahora.

## Desinstalación

No hace falta desinstalar nada: basta con borrar el directorio clonado. Para quitar Tango de un dispositivo hay que volver a flashear una imagen de system que no lo incluya.

## Agradecimientos

- [Tango](https://www.amanieusystems.com/) de Amanieu Systems: origen del traductor, del módulo del kernel `tango32`, del registro binfmt y de la política SELinux
- Las integraciones oficiales de Tango del Xiaomi 14 Ultra y el OnePlus 12: referencia para el zygote de 32 bits y el enfoque del GC de ART

## Licencia

Este proyecto se distribuye bajo la GNU General Public License v3.0 o posterior, identificador SPDX `GPL-3.0-or-later`; el texto completo está en [LICENSE](../../LICENSE). Este proyecto no incluye Tango, el módulo `tango32` ni las bibliotecas de 32 bits de otros fabricantes; véase [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md).
