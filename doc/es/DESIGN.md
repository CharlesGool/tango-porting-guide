---
name: project-design-es
description: Arquitectura y restricciones de diseño del proyecto
metadata:
  version: "1.0.0"
  lang: "es"
---

# tango-porting-guide — Diseño

## Multilingüe

[简体中文](../DESIGN.md) | [English](../en/DESIGN.md) | **Español**

## Documentación

- Descripción general del proyecto: [README](README.md)

- Justificación del diseño: [DESIGN](DESIGN.md)

- Estado del proyecto: [LOG](LOG.md)
- Registros históricos: [HISTORY](HISTORY.md)
- Historial de cambios: [CHANGELOG](CHANGELOG.md)

- Avisos de terceros: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## Objetivos de diseño

- [x] Explicar los pasos generales para portar Tango a una imagen de system de Android ya compilada, sin modificar el código fuente.
- [x] Que cada paso corresponda a una implementación de referencia verificada en un dispositivo real ([tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch), TB321FU, Android 15).
- [x] Aportar la comparación con las integraciones oficiales de los fabricantes y la causa real del reinicio del equipo, como fundamento de los pasos.
- [ ] Verificar la guía en un segundo dispositivo o en otra versión de Android.
- No es objetivo: proporcionar imágenes listas para flashear ni código de parches para ningún dispositivo.

## Arquitectura

Este proyecto contiene solo documentación:

| Archivo | Contenido |
| --- | --- |
| [porting-guide.md](porting-guide.md) | Guía general de portado: requisitos previos, 6 pasos, verificación y solución de problemas; cada paso enlaza con la parte correspondiente de la implementación de referencia |
| [research.md](research.md) | Notas de investigación: enfoque final, comparación con Xiaomi y OnePlus, causa real, pruebas con QEMU |

| Directorio de primer nivel | Responsabilidad |
| --- | --- |
| `doc/` | Documentación del proyecto |

La implementación de referencia está en el repositorio independiente [tb321fu-tango-patch](https://github.com/CharlesGool/tb321fu-tango-patch). Este proyecto no tiene código, texto de interfaz ni artefactos de compilación.

## Restricciones de diseño

- El repositorio **MUST NOT** incluir Tango, `tango32.ko` ni otros archivos propietarios de fabricantes.
- El contenido de la guía que no se haya verificado en un dispositivo real **MUST** marcarse como por confirmar.
- Cada paso de la guía **MUST** poder localizarse en un archivo de la implementación de referencia; cuando cambie una ruta de la implementación de referencia, **MUST** actualizarse el enlace.
