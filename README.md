# AutoFirma en Bazzite — Instalación vía contenedor Distrobox

Manual para instalar AutoFirma en Bazzite (y en general cualquier distro
inmutable basada en Fedora, como Silverblue o Kinoite) usando un contenedor
Distrobox con Ubuntu, instalando el paquete `.deb` **oficial y nativo** del
Gobierno de España dentro del contenedor.

## Por qué este método y no Flatpak

- **Paquete 100% oficial y nativo**, el mismo `.deb` que publica el Gobierno
  para cualquier Ubuntu/Debian — sin avisos de "no verificado" de Flathub.
- **Menos pasos frágiles.** La vía Flatpak necesita usar un almacén NSS como
  "puente" obligatorio entre AutoFirma y el navegador, y dar permisos
  cruzados entre apps con `flatpak override`, porque Flatpak aísla cada
  aplicación de las demás incluso con acceso general al sistema de archivos.
  Con un contenedor, ese aislamiento no existe, y la configuración es más
  directa.
- Validado con doble verificación: firma real comprobada con `pdfsig` y en
  [valide.redsara.es](https://valide.redsara.es/), y también con el test
  automático independiente del Ministerio de Trabajo.

## Contenido

- [`Manual_AutoFirma_Bazzite_Distrobox.md`](./Manual_AutoFirma_Bazzite_Distrobox.md) —
  guía completa paso a paso, pensada para alguien sin conocimientos previos:
  desde crear el contenedor Distrobox, instalar el paquete oficial, exportarlo
  al menú de aplicaciones del host, configurar el almacén de certificados NSS,
  hasta la verificación final firmando y validando un documento real. Incluye
  una tabla comparativa con la vía Flatpak.

## Resumen rápido

1. Crear el contenedor: `distrobox create --name autofirma-box --image ubuntu:24.04`
2. Instalar el `.deb` oficial de AutoFirma dentro del contenedor
3. Exportarlo al menú de aplicaciones del host: `distrobox-export --app autofirma`
4. Crear el almacén NSS con contraseña en blanco e importar certificado
   personal + CAs de la FNMT
5. Confiar en el certificado de comunicación de AutoFirma desde Firefox y
   Brave, ya en el host
6. Verificar firmando desde el navegador y validando en
   [valide.redsara.es](https://valide.redsara.es/valide/firmar/ejecutar.html)

Ver [`Manual_AutoFirma_Bazzite_Distrobox.md`](./Manual_AutoFirma_Bazzite_Distrobox.md)
para todos los detalles y solución de problemas.

## Repo relacionado

Para CachyOS/Arch Linux (compilación desde código fuente), ver
[`csr79a/autofirma-cachyos-manual`](https://github.com/csr79a/autofirma-cachyos-manual).
