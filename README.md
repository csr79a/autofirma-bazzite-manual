# AutoFirma en Bazzite — Dos métodos de instalación

Manuales para instalar AutoFirma en Bazzite (y en general cualquier distro
inmutable basada en Fedora, como Silverblue o Kinoite), con dos enfoques
distintos y completamente probados. Elige el que mejor se adapte a ti.

## Métodos disponibles

- **[`flatpak/`](./flatpak/)** — instalación vía Flatpak (`es.gob.afirma`
  desde Flathub). Actualizaciones automáticas, pero requiere más pasos de
  configuración por el sandboxing entre aplicaciones de Flatpak.
- **[`distrobox/`](./distrobox/)** — instalación del paquete `.deb` oficial
  dentro de un contenedor Distrobox con Ubuntu. Paquete 100% nativo, menos
  pasos frágiles, pero el mantenimiento (actualizar a una versión nueva) es
  manual.

## ¿Cuál elegir?

| Aspecto | Flatpak | Distrobox |
|---|---|---|
| Origen del paquete | Flathub (`es.gob.afirma`) | `.deb` oficial del Gobierno |
| Actualizaciones | Automáticas | Manuales (repetir instalación del `.deb`) |
| Aislamiento entre apps | Sí (requiere `flatpak override` y almacén NSS como "puente") | No |
| Complejidad de configuración | Media-alta (varios puntos de fallo posibles) | Media (más directo) |
| Instalación inicial | Un solo comando (`flatpak install`) | Varios pasos (crear contenedor, instalar `.deb`, exportar) |

Ambos métodos están completamente documentados, probados de punta a punta
(firma real verificada con `pdfsig` y en
[valide.redsara.es](https://valide.redsara.es/), además del test
independiente del Ministerio de Trabajo), y llevan a un resultado
funcionalmente idéntico: AutoFirma firmando y validando correctamente desde
el navegador y de forma manual.

## Repo relacionado

Para CachyOS/Arch Linux (compilación desde código fuente), ver
[`csr79a/autofirma-cachyos-manual`](https://github.com/csr79a/autofirma-cachyos-manual).
