---
name: project-history-es
description: Registros históricos
metadata:
  version: "1.0.0"
  lang: "es"
---

# Registros históricos

## Multilingüe

[简体中文](../HISTORY.md) | [English](../en/HISTORY.md) | **Español**

## Documentación

- Descripción general del proyecto: [README](README.md)

- Justificación del diseño: [DESIGN](DESIGN.md)

- Estado del proyecto: [LOG](LOG.md)
- Registros históricos: [HISTORY](HISTORY.md)
- Historial de cambios: [CHANGELOG](CHANGELOG.md)

- Avisos de terceros: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## Registros históricos

El desarrollo de las pruebas en dispositivo real (V68 a V883) está en [el HISTORY de tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch/blob/main/doc/HISTORY.md#真机测试经过).

### Cambios de organización

- 2026-10-31: los 5 módulos pasan de `mods/tango/` a `patches/tango/` de Mods.
- 2026-11-01: los 5 parches se fusionan en un único parche, `required/general/tango`, de 5 partes, versión de parche V1.0.
- Principios de 2026-11 (fecha sin comprobar): el nombre del runtime pasa de omk a herald (`heraldctl`, dominio herald).
- 2026-11-05: separación en un repositorio privado independiente, tb321fu-tango.
- 2026-11-05: se renombra a tango-porting-guide y se hace público, con el enfoque de guía general de portado.
- 2026-11-05: el parche pasa al repositorio independiente tb321fu-tango-patch; este repositorio conserva solo la documentación.
