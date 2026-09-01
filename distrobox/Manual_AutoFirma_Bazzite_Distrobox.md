# Manual: AutoFirma en Bazzite mediante Distrobox (contenedor)

Guía completa, paso a paso, para instalar y dejar funcionando AutoFirma en Bazzite usando un **contenedor Distrobox con Ubuntu** en vez de la versión Flatpak. Este método instala el paquete `.deb` **oficial y nativo** del Gobierno de España dentro de un contenedor, evitando los problemas de aislamiento entre aplicaciones (sandboxing) que tiene la vía Flatpak.

**Versión 1 — validada en producción sobre una VM de Bazzite (septiembre 2026).**

**Entorno de referencia:**
- Sistema host: Bazzite (Fedora Atomic/inmutable)
- Contenedor: Distrobox con imagen `ubuntu:24.04`
- AutoFirma: paquete `.deb` oficial (versión 1.9), instalado de forma nativa dentro del contenedor — **no Flatpak**
- Firefox y Brave: instalados en el **host** (Bazzite), no dentro del contenedor
- Certificado personal: FNMT, formato `.pfx`/`.p12`

---

## ¿Por qué un contenedor en vez de Flatpak?

En Bazzite (sistema inmutable), tanto AutoFirma como Firefox suelen instalarse como Flatpak. El problema es que Flatpak **aísla cada aplicación de las demás**, incluso cuando ambas tienen acceso general al sistema de archivos del usuario. Esto obliga a rodeos (un almacén de certificados "puente" compartido, permisos explícitos entre apps) para que AutoFirma y el navegador puedan hablarse.

Con un contenedor Distrobox, AutoFirma se instala como el paquete nativo `.deb` que publica el Gobierno, dentro de un sistema Ubuntu normal sin ese aislamiento extra. Esto simplifica la configuración y evita esa categoría de problemas desde el principio.

**Requisito previo:** Distrobox ya instalado en tu Bazzite (viene preinstalado en la mayoría de imágenes de Bazzite; si no lo tienes, instálalo con `rpm-ostree install distrobox` y reinicia).

---

## 1. Crear el contenedor con Ubuntu

Abre una terminal en el **host** (Bazzite, fuera de cualquier contenedor) y ejecuta:

```bash
distrobox create --name autofirma-box --image ubuntu:24.04
```

Esto descarga la imagen de Ubuntu 24.04 (puede tardar unos minutos la primera vez, según tu conexión) y crea el contenedor `autofirma-box`.

Cuando termine, entra en él:

```bash
distrobox enter autofirma-box
```

El prompt de tu terminal cambiará para indicar que estás dentro del contenedor (normalmente aparece un icono de caja 📦 seguido del nombre del contenedor y del usuario, por ejemplo `📦[cesar@autofirma-box ~]$`).

Puedes confirmarlo con:

```bash
whoami && hostname
```

---

## 2. Instalar dependencias dentro del contenedor

Sigue **dentro del contenedor** (no en el host). Actualiza la lista de paquetes e instala las herramientas necesarias:

```bash
sudo apt update
sudo apt install -y default-jre unzip wget
```

- `default-jre`: entorno de ejecución de Java, necesario para AutoFirma.
- `unzip`: para descomprimir el instalador descargado.
- `wget`: para descargar archivos desde la terminal.

(Nota: al instalar el paquete `.deb` de AutoFirma en el siguiente paso, `apt` instalará automáticamente `libnss3-tools`, que aporta `certutil` y `pk12util` — las herramientas de gestión de certificados NSS que usaremos más adelante. No hace falta instalarlo aparte.)

---

## 3. Descargar e instalar AutoFirma (paquete oficial)

Sigue dentro del contenedor. Ve a tu carpeta personal y descarga el ZIP oficial para Debian/Ubuntu desde la página de descargas del Gobierno:

```bash
cd ~
wget "https://firmaelectronica.gob.es/content/dam/firmaelectronica/descargas-software/autofirma19/Autofirma_Linux_Debian.zip" -O Autofirma_Linux_Debian.zip
```

> ⚠️ **Esta URL puede cambiar con el tiempo.** Si el comando anterior falla o descarga un archivo muy pequeño (unos pocos KB en vez de varias decenas de MB), verifica el enlace vigente en la página oficial: https://firmaelectronica.gob.es/descargas (apartado "Para Linux" → "Versión ... para Debian Linux").

Descomprime el ZIP:

```bash
unzip Autofirma_Linux_Debian.zip
```

Esto extrae un archivo `.deb` (por ejemplo `autofirma_1_9.deb`; el nombre exacto depende de la versión vigente). Compruébalo:

```bash
ls -la *.deb
```

Instálalo con `apt`, que resuelve automáticamente las dependencias (incluyendo `libnss3-tools`):

```bash
sudo apt install -y ./autofirma_1_9.deb
```

(Sustituye el nombre del archivo por el que hayas obtenido si es distinto.)

Durante la instalación verás bastante texto informativo en pantalla — es normal. AutoFirma genera automáticamente su propio certificado de comunicación con el navegador. Puede aparecer un aviso como:

```
ADVERTENCIA: No se ha detectado un perfil de Mozilla Firefox en el que instalar el certificado
```

**Esto es esperado y no es un error.** Ocurre porque Firefox no está instalado dentro de este contenedor (vive en el host, Bazzite). Lo resolveremos manualmente en el paso 6.

---

## 4. Exportar AutoFirma al menú de aplicaciones del host

Uno de los puntos fuertes de Distrobox: puedes hacer que una aplicación instalada dentro del contenedor aparezca como una app normal en el menú de aplicaciones del host, sin que el usuario tenga que entrar manualmente al contenedor cada vez.

Primero, confirma el nombre del ejecutable dentro del contenedor:

```bash
which autofirma
```

Debería mostrar algo como `/usr/bin/autofirma`.

Ahora expórtalo al host:

```bash
distrobox-export --app autofirma
```

Deberías ver un mensaje como:

```
Application autofirma successfully exported.
OK!
autofirma will appear in your applications list in a few seconds.
```

En unos segundos, AutoFirma aparecerá en el menú de aplicaciones de Bazzite como si fuera una app nativa más. Internamente, al abrirlo, el sistema ejecuta `distrobox enter autofirma-box -- autofirma` de forma transparente para ti.

---

## 5. Crear el almacén de certificados NSS dentro del contenedor

Sigue dentro del contenedor. AutoFirma necesita un almacén de certificados NSS donde tener tu certificado personal FNMT y las autoridades de certificación (CA) que lo emiten. Como el contenedor no tiene el aislamiento de Flatpak, este almacén no necesita ser "compartido" con nada externo — es simplemente el almacén NSS normal del sistema del contenedor.

### 🔴 Regla importante: contraseña en blanco

Igual que en la instalación por Flatpak, la contraseña de este almacén debe quedar en blanco, porque AutoFirma no tiene forma de pedírtela de forma interactiva cuando se invoca desde el navegador.

Crea la carpeta y la base de datos con contraseña vacía en un solo paso:

```bash
mkdir -p ~/.pki/nssdb
certutil -N -d sql:$HOME/.pki/nssdb --empty-password
```

El flag `--empty-password` crea la base de datos directamente sin contraseña, sin necesidad de pulsar Enter dos veces de forma interactiva.

Verifica que se creó correctamente (debe salir una lista vacía, sin errores):

```bash
certutil -L -d sql:$HOME/.pki/nssdb
```

---

## 6. Importar tu certificado personal FNMT

Distrobox comparte automáticamente tu carpeta personal (`$HOME`) entre el host y el contenedor. Esto significa que si tienes tu certificado `.pfx` guardado en `~/Documentos/` en Bazzite, ya lo verás también desde dentro del contenedor sin copiar nada.

Comprueba que lo ves (ajusta el nombre de archivo al tuyo):

```bash
ls -la ~/Documentos/ | grep -i pfx
```

Impórtalo al almacén NSS que acabas de crear:

```bash
pk12util -i ~/Documentos/TU_CERTIFICADO.pfx -d sql:$HOME/.pki/nssdb
```

Te pedirá la **contraseña del propio `.pfx`** (la que le pusiste al exportarlo/descargarlo de la FNMT) — no la del almacén, que ya dejamos en blanco.

---

## 7. Importar las CA (autoridades de certificación) de la FNMT

Como no hay Firefox instalado dentro de este contenedor del que copiar las CA (a diferencia de la vía Flatpak, donde se pueden copiar del perfil de Firefox), las descargamos directamente de la web oficial de la FNMT.

> ⚠️ Verifica primero que las URLs siguen vigentes en: https://www.sede.fnmt.gob.es/descargas/certificados-raiz-de-la-fnmt (pueden cambiar con el tiempo).

Descarga ambos certificados:

```bash
mkdir -p ~/Descargas
wget "https://www.sede.fnmt.gob.es/documents/10445900/10526749/AC_Raiz_FNMT-RCM_SHA256.cer" -O ~/Descargas/AC_Raiz_FNMT-RCM.cer
wget "https://www.sede.fnmt.gob.es/documents/10445900/10526749/AC_FNMT_Usuarios.cer" -O ~/Descargas/AC_FNMT_Usuarios.cer
```

Impórtalos al almacén NSS:

```bash
certutil -A -n "AC Raiz FNMT-RCM" -t "TC,C,C" -i ~/Descargas/AC_Raiz_FNMT-RCM.cer -d sql:$HOME/.pki/nssdb
certutil -A -n "AC FNMT Usuarios" -t "TC,C,C" -i ~/Descargas/AC_FNMT_Usuarios.cer -d sql:$HOME/.pki/nssdb
```

### Verifica el resultado final

```bash
certutil -L -d sql:$HOME/.pki/nssdb
```

Deberías ver tres entradas:
- Tu certificado personal, con confianza `u,u,u`
- `AC Raiz FNMT-RCM - FNMT-RCM`, con confianza `CT,C,C`
- `AC FNMT Usuarios - FNMT-RCM`, con confianza `CT,C,C`

---

## 8. Configurar AutoFirma para usar el almacén NSS

Abre AutoFirma (desde el menú de aplicaciones del host, gracias al paso 4, o ejecutando `autofirma` dentro del contenedor).

Ve a **Herramientas/Preferencias → pestaña "Almacenes de claves"** y confirma que el almacén por defecto sea **NSS**. Si no lo está, selecciónalo y aplica los cambios.

Tu certificado personal debería aparecer automáticamente en la lista, sin necesidad de buscar ningún archivo manualmente.

---

## 9. Confiar en el certificado de comunicación de AutoFirma desde los navegadores del host

Este es el paso que conecta ambos "mundos": el navegador vive en el host (Bazzite), AutoFirma vive dentro del contenedor. Para que la web pueda invocar a AutoFirma al pulsar un botón de "Firmar", el navegador necesita confiar en el certificado de comunicación que generó AutoFirma.

**9.1. Localiza el certificado dentro del contenedor:**

```bash
find / -iname "AutoFirma_ROOT*" 2>/dev/null
```

Debería aparecer, entre otras rutas, `/usr/lib/Autofirma/Autofirma_ROOT.cer`.

**9.2. Cópialo a tu carpeta personal**, que sí es visible desde el host (a diferencia de `/usr/lib`, que es exclusiva del contenedor):

```bash
cp /usr/lib/Autofirma/Autofirma_ROOT.cer ~/Descargas/Autofirma_ROOT.cer
```

**9.3. Sal del contenedor** y vuelve al host:

```bash
exit
```

Tu prompt debería volver a la normalidad (sin el icono 📦). Confirma que el archivo es visible desde el host:

```bash
ls -la ~/Descargas/Autofirma_ROOT.cer
```

**9.4. Importa el certificado en Firefox** (ya en el host):

`about:preferences#privacy` → Certificados → Ver certificados → pestaña **Autoridades** → Importar → selecciona `~/Descargas/Autofirma_ROOT.cer` → marca **"Confiar en esta CA para identificar sitios web"**.

**9.5. Importa el certificado en Brave** (ya en el host):

`brave://settings/security` → Certificados locales → Instalados por ti → **Certificados de confianza** → Importar → mismo archivo → confiar para sitios web.

---

## 10. Verificación: firmar desde el navegador

Con todo configurado, prueba que el navegador puede invocar a AutoFirma. Ve a:

https://valide.redsara.es/valide/firmar/ejecutar.html

Pulsa el botón **Firmar**. Debería abrirse AutoFirma automáticamente (a través del contenedor, de forma transparente) sin error de conexión, y tu certificado debería aparecer ya seleccionado sin tener que buscarlo.

Si en vez de eso aparece un error de protocolo no reconocido o AutoFirma no se abre, revisa que el paso 4 (`distrobox-export --app autofirma`) se completó correctamente, y que el archivo `.desktop` generado esté registrado como manejador del protocolo `afirma://` en el sistema.

---

## 11. Verificación adicional: test automático de requisitos

Como comprobación extra e independiente, puedes usar la página de test automático del Ministerio de Trabajo:

https://expinterweb.mites.gob.es/scriptAutofirmaTest/

Esta página comprueba automáticamente: navegador compatible, sistema operativo detectado, versión del script de AutoFirma, y realiza un test de firma real en los dos formatos estándar (CAdES y XAdES). Todos los indicadores deberían mostrar **"Correcto"** en verde.

---

## 12. Verificación final: firmar y validar un documento real

1. **Firma un documento de prueba** con AutoFirma (tanto desde el navegador como abriendo AutoFirma directamente).

2. **Verifica la firma con `pdfsig`**, ejecutándolo dentro del contenedor (o llamándolo desde el host a través de `distrobox enter`):

```bash
distrobox enter autofirma-box -- pdfsig -nssdir sql:$HOME/.pki/nssdb ~/RUTA/AL/ARCHIVO.pdf
```

Busca la línea:
```
Signature Validation: Signature is Valid.
```

> Nota: el apartado `Certificate Validation` puede seguir mostrando un aviso de error incluso con la firma perfectamente válida — es un bug conocido de las librerías de validación de certificados con las cadenas FNMT, no indica un problema real (el mismo comportamiento que se observa en Okular con la vía Flatpak). Confía en `Signature Validation`, no en `Certificate Validation`.

3. **Confirmación oficial y definitiva:**

https://valide.redsara.es/valide/firmar/ejecutar.html → **Validar Firma** → sube el PDF.

Si dice **"Firma válida"**, el documento es legalmente correcto y válido para presentar ante cualquier administración española.

---

## Resumen ultra-corto (para memoria rápida)

1. `distrobox create --name autofirma-box --image ubuntu:24.04` y `distrobox enter autofirma-box`
2. Dentro del contenedor: `sudo apt update && sudo apt install -y default-jre unzip wget`
3. Descargar el ZIP oficial de AutoFirma para Debian desde https://firmaelectronica.gob.es/descargas, descomprimir e instalar con `sudo apt install -y ./autofirma_X_Y.deb`
4. `distrobox-export --app autofirma` para que aparezca en el menú de aplicaciones del host
5. Crear el almacén NSS del contenedor con contraseña en blanco: `certutil -N -d sql:$HOME/.pki/nssdb --empty-password`
6. Importar tu `.pfx` personal (accesible directamente porque Distrobox comparte el `$HOME`): `pk12util -i ~/Documentos/TU_CERTIFICADO.pfx -d sql:$HOME/.pki/nssdb`
7. Descargar e importar las dos CA de la FNMT desde https://www.sede.fnmt.gob.es/descargas/certificados-raiz-de-la-fnmt
8. En AutoFirma → Preferencias → Almacenes de claves → confirmar NSS como almacén por defecto
9. Copiar `Autofirma_ROOT.cer` del contenedor (`/usr/lib/Autofirma/`) a `~/Descargas/` (carpeta compartida) e importarlo como autoridad de confianza en Firefox y Brave, ya en el host
10. Probar en https://valide.redsara.es/valide/firmar/ejecutar.html (botón Firmar)
11. Comprobación adicional en https://expinterweb.mites.gob.es/scriptAutofirmaTest/
12. Firmar un documento real, verificar con `pdfsig` y validación oficial en https://valide.redsara.es/

---

## Diferencias clave frente a la vía Flatpak

| Aspecto | Flatpak | Distrobox (este manual) |
|---|---|---|
| Origen del paquete | Flathub (`es.gob.afirma`) | `.deb` oficial del Gobierno |
| Aislamiento entre apps | Sí (requiere `flatpak override` y almacén "puente") | No (contenedor comparte el sistema internamente) |
| Almacén NSS | `~/.pki/nssdb`, usado como puente obligatorio para el flujo web | `~/.pki/nssdb` dentro del contenedor, almacén normal sin rol de "puente" |
| Acceso al perfil de Firefox | Necesario para uso manual (sección 4 del manual Flatpak) | No necesario — no hay Firefox dentro del contenedor |
| Exportar al menú de apps | Automático (Flatpak) | Manual, con `distrobox-export --app` |
| Mantenimiento | Actualizaciones automáticas vía Flathub | Manual (actualizar el `.deb` dentro del contenedor cuando salga nueva versión) |
