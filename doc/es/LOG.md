---
name: project-log-es
description: Errores, limitaciones, decisiones y traspaso del proyecto
metadata:
  version: "1.0.0"
  lang: "es"
---

# Registro

## Multilingüe

[简体中文](../LOG.md) | [English](../en/LOG.md) | **Español**

## Documentación

- Descripción general del proyecto: [README](README.md)

- Justificación del diseño: [DESIGN](DESIGN.md)

- Estado del proyecto: [LOG](LOG.md)
- Registros históricos: [HISTORY](HISTORY.md)
- Historial de cambios: [CHANGELOG](CHANGELOG.md)

- Avisos de terceros: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## Registros

- [Registros históricos](HISTORY.md)
- [Historial de cambios](CHANGELOG.md)

## Errores

Ninguno por ahora. Los errores conocidos de la implementación de referencia están en [tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/LOG.md#缺陷).

## Limitaciones

- La guía solo se ha verificado en el TB321FU (Snapdragon 8 Gen 3, Android 15); los pasos para otros dispositivos y versiones se han deducido de las integraciones oficiales de Xiaomi y OnePlus.
- Tango es software propietario, por lo que este proyecto no puede proporcionar el traductor ni imágenes listas para flashear.

## Decisiones

| Fecha | Decisión | Estado | Motivo |
| --- | --- | --- | --- |
| 2026-10-31 | No modificar `ro.zygote`; incluir un servicio propio de zygote de 32 bits | Aceptada | El `ro.zygote` de vendor sobrescribe el valor de system, y el rc original introduce la autocomprobación de 32 bits y el reinicio en cadena. |
| 2026-10-31 | `ro.dalvik.vm.enable_uffd_gc=false` | Aceptada | Tango no admite userfaultfd, por lo que el ART de 32 bits solo puede usar el GC CC. |
| 2026-10-31 | Escribir `panic_on_walt_bug=0` como salvaguarda de arranque para corregir el reinicio del equipo | Aceptada | Modificar `sched-walt.ko` obliga a reempaquetar `vendor_boot`; el sysctl se ha verificado como eficaz en el dispositivo real. |
| 2026-10-31 | Usar las bibliotecas GPU de 32 bits del OnePlus 12 | Rechazada | La causa real es la autocomprobación de WALT, y las bibliotecas actuales funcionan correctamente en el dispositivo real. |
| 2026-11-01 | Fusionar 5 parches independientes en un único parche de 5 partes | Aceptada | Las partes tienen que usarse juntas para funcionar. |
| 2026-11-05 | Separar en un repositorio independiente, como única fuente del parche, y mantener en Mods una copia sincronizada | Aceptada | Coincide con la organización de herald-core y facilita el mantenimiento y la publicación por separado. |
| 2026-11-05 | Renombrar el repositorio a tango-porting-guide y hacerlo público, con el enfoque de guía general de portado | Aceptada | El usuario pidió publicar cómo portar Tango; el nombre no incluye el del dispositivo. |
| 2026-11-05 | Mover el parche del TB321FU al repositorio público independiente tb321fu-tango-patch; este repositorio conserva solo la documentación | Aceptada | El parche tiene una sola fuente y la guía está dirigida a todos los dispositivos. |

## Traspaso

Consulte el traspaso vigente en el [LOG.md en chino simplificado](../LOG.md#交接).
