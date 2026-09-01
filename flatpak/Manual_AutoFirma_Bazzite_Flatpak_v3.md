# Manual: AutoFirma en Bazzite (Firefox/Brave Flatpak)

Guía completa para dejar AutoFirma funcionando de punta a punta: comunicación con el navegador, firma sin buscar el certificado manualmente, firma/validación invocada desde la web, y validación correcta en Okular.

**Versión 3 — corregida y ampliada tras una segunda depuración real en producción (septiembre 2026): URLs de descarga actualizadas, aviso sobre nombres de CA que pueden no coincidir en Firefox, y aclaración sobre cuándo la sección 4 no es necesaria.**

**Entorno de referencia:**
- Sistema: Bazzite (Fedora Atomic/inmutable)
- Firefox: Flatpak (`org.mozilla.firefox`)
- Brave: Flatpak (`com.brave.Browser`)
- AutoFirma: Flatpak (`es.gob.afirma`) — **no usar la versión de Homebrew, es exclusiva de macOS**
- Certificado personal: FNMT, formato `.pfx`/`.p12`

---

## 0. Requisitos previos

Instala Homebrew si no lo tienes (en sistemas inmutables es la forma más limpia de añadir herramientas CLI sin tocar la imagen base):

```bash
brew install nss poppler
```

Esto da acceso a `certutil`, `pk12util` (gestión de certificados NSS) y `pdfsig` (diagnóstico de firmas en PDF).

> **Nota:** en algunas instalaciones de Bazzite, `pdfsig` ya viene incluido en el sistema base (`/usr/bin/pdfsig`), y solo faltará instalar `nss` para tener `certutil` y `pk12util`. Comprueba primero qué tienes ya disponible antes de instalar nada:

```bash
which certutil pk12util pdfsig
```

Si falta alguno, instala solo lo necesario con `brew install nss` y/o `brew install poppler`.

---

## 1. Instalar AutoFirma

```bash
flatpak install flathub es.gob.afirma
```

Ábrelo una vez (desde el menú de aplicaciones, o `flatpak run es.gob.afirma`) para que genere su certificado de comunicación local. Se guarda en:

```
~/.afirma/Autofirma/Autofirma_ROOT.cer
~/.afirma/Autofirma/autofirma.pfx
```

Comprueba que se generaron correctamente:

```bash
ls -la ~/.afirma/Autofirma/
```

---

## 2. Confiar en el certificado de AutoFirma desde los navegadores

Esto permite que la web pueda "hablar" con AutoFirma cuando pulsas firmar.

**Firefox:**
`about:preferences#privacy` → Certificados → Ver certificados → pestaña **Autoridades** → Importar → selecciona `~/.afirma/Autofirma/Autofirma_ROOT.cer` → marca **"Confiar en esta CA para identificar sitios web"**.

**Brave:**
`brave://settings/security` → Certificados locales → Instalados por ti → **Certificados de confianza** → Importar → mismo archivo → confiar para sitios web.

Comprueba que funciona en https://valide.redsara.es/valide/firmar/ejecutar.html (botón Firmar): debe abrirse AutoFirma sin error de conexión.

---

## 3. Almacén NSS compartido: `~/.pki/nssdb`

⚠️ **Esta sección es más importante de lo que parece.** `~/.pki/nssdb` no es solo para que Okular valide firmas — **AutoFirma también lo usa como almacén "compartido" cuando se invoca desde el navegador** (protocolo `afirma://`, botón "Firmar"/"Validar" en una web). Si esta base de datos falla, AutoFirma abierto manualmente seguirá funcionando bien, pero **firmar o validar desde una web fallará silenciosamente** mostrando "No se han encontrado certificados válidos en el almacén", aunque el certificado esté perfectamente importado en el perfil de Firefox.

> 💡 En la práctica, completar bien esta sección suele ser suficiente para que AutoFirma también muestre tu certificado automáticamente al abrirlo **manualmente** (sin buscar ningún archivo), sin necesidad de la sección 4. Ver la nota al principio de la sección 4 para más detalle.

### 🔴 REGLA CRÍTICA: la contraseña de esta base de datos debe quedar EN BLANCO

AutoFirma, cuando la abre automáticamente desde el flujo web, **no tiene forma de pedirte una contraseña interactivamente**. Si la base de datos tiene contraseña, el intento de apertura automática falla con `java.io.IOException: load failed` y AutoFirma muestra el almacén como vacío, aunque los certificados sigan ahí dentro.

```bash
mkdir -p ~/.pki/nssdb
certutil -N -d sql:$HOME/.pki/nssdb
```

Cuando te pida la contraseña, **pulsa Enter dos veces, sin escribir nada**. No pongas ninguna contraseña, aunque el asistente lo sugiera como opcional.

> Alternativa más rápida (evita el prompt interactivo): `certutil -N -d sql:$HOME/.pki/nssdb --empty-password`

### Importa tu certificado personal aquí también

```bash
pk12util -i ~/Documentos/TU_CERTIFICADO.pfx -d sql:$HOME/.pki/nssdb
```
(te pedirá solo la contraseña del propio `.pfx`, no la del almacén, ya que está en blanco)

### Importa las CA de la FNMT

Si ya tienes Firefox configurado con el certificado (sección 4), puedes intentar copiar las CA directamente desde ahí en vez de descargarlas de nuevo. Localiza primero tu perfil real de Firefox (ver paso 4.1 más abajo) y comprueba qué CA de la FNMT tienes realmente importadas, porque **el nombre puede no coincidir exactamente** con el de ejemplo:

```bash
certutil -L -d sql:$HOME/.var/app/org.mozilla.firefox/config/mozilla/firefox/TU_PERFIL | grep -i fnmt
```

> ⚠️ **Aviso real de depuración:** en la práctica, Firefox puede tener importada solo una de las dos CA (normalmente "AC FNMT Usuarios - FNMT-RCM"), y no la CA raíz. Si tras el `grep` anterior falta alguna, no te preocupes — pasa directamente a la alternativa de descarga de más abajo para la que falte.

Si el nombre coincide, cópiala así (ajusta el nombre exacto que haya devuelto el `grep`):

```bash
certutil -L -n "AC FNMT Usuarios - FNMT-RCM" -a -d sql:$HOME/.var/app/org.mozilla.firefox/config/mozilla/firefox/TU_PERFIL > /tmp/ac_fnmt_usuarios.crt
certutil -A -n "AC FNMT Usuarios" -t "TC,C,C" -i /tmp/ac_fnmt_usuarios.crt -d sql:$HOME/.pki/nssdb
rm /tmp/ac_fnmt_usuarios.crt
```

**Para la CA que falte (o si prefieres partir de cero con ambas), descárgala directamente de la web oficial de la FNMT.** Verifica primero las URLs vigentes en https://www.sede.fnmt.gob.es/descargas/certificados-raiz-de-la-fnmt, ya que pueden cambiar. A fecha de esta revisión (septiembre 2026), las URLs directas vigentes son:

```bash
mkdir -p ~/Descargas
wget "https://www.sede.fnmt.gob.es/documents/10445900/10526749/AC_Raiz_FNMT-RCM_SHA256.cer" -O ~/Descargas/AC_Raiz_FNMT-RCM.cer
wget "https://www.sede.fnmt.gob.es/documents/10445900/10526749/AC_FNMT_Usuarios.cer" -O ~/Descargas/AC_FNMT_Usuarios.cer

certutil -A -n "AC Raiz FNMT-RCM" -t "TC,C,C" -i ~/Descargas/AC_Raiz_FNMT-RCM.cer -d sql:$HOME/.pki/nssdb
certutil -A -n "AC FNMT Usuarios" -t "TC,C,C" -i ~/Descargas/AC_FNMT_Usuarios.cer -d sql:$HOME/.pki/nssdb
```

> Si alguna de estas URLs da error 404, confirma la vigente en la página oficial enlazada arriba antes de seguir — verifica también con el comando `file` (o simplemente comprobando el tamaño del archivo descargado, unos cientos de bytes a un par de KB) que lo descargado es realmente un certificado y no una página de error.

### Verifica el resultado final

```bash
certutil -L -d sql:$HOME/.pki/nssdb
```

Deberías ver tu certificado personal (`u,u,u`) y las dos CA de la FNMT (`CT,C,C`).

Okular ya debería apuntar aquí por defecto (compruébalo en *Configurar Okular → Configuración de motores → PDF → Firmas digitales → Base de datos de certificados*; si no, indícalo a mano: `/home/TU_USUARIO/.pki/nssdb`).

### Si ya la creaste con contraseña por error

No hace falta recordar la contraseña — es más rápido recrearla:

```bash
mv ~/.pki/nssdb ~/.pki/nssdb.old_$(date +%s)
mkdir -p ~/.pki/nssdb
certutil -N -d sql:$HOME/.pki/nssdb --empty-password
```
Y repite la importación del certificado personal y las CA como se indica arriba.

---

## 4. Que AutoFirma use tu certificado sin tener que buscarlo cada vez (abierto manualmente)

> 💡 **Antes de hacer nada aquí: prueba primero si ya lo tienes resuelto.** Con la sección 3 bien completada (almacén NSS compartido con contraseña en blanco, certificado y CAs importados), abre AutoFirma manualmente e intenta firmar un documento de prueba. En la práctica, muchas instalaciones actuales de AutoFirma ya detectan y usan automáticamente el almacén NSS también en el uso manual, sin necesitar nada de esta sección. Si tu certificado ya aparece en la lista sin buscarlo, **puedes saltarte toda la sección 4** y pasar directamente a la sección 5.

Esta sección cubre el flujo de **abrir AutoFirma directamente** (Preferencias → Almacenes de claves → Ver contenido), para el caso en que el paso anterior no fuera suficiente. Es un problema distinto al de la sección 3.

La causa raíz: **Flatpak aísla `~/.var/app/<app>` entre aplicaciones incluso con permiso `filesystems=host`** — AutoFirma no puede leer los datos de Firefox salvo que se le dé acceso explícito a esa carpeta concreta.

**4.1. Localiza el perfil real de Firefox (Flatpak):**

```bash
find ~/.var/app/org.mozilla.firefox -iname "profiles.ini" 2>/dev/null
```

Abre ese `profiles.ini` (con `cat` o un editor) y busca la sección `[InstallXXXXXXXX]` — el valor de `Default=` ahí es el perfil que Firefox **realmente** usa. No te fíes solo del flag `Default=1` clásico de una sección `[ProfileN]`, puede apuntar al perfil equivocado en instalaciones Flatpak.

**4.2. Importa tu certificado personal directamente en ese perfil real:**

```bash
pk12util -i ~/Documentos/TU_CERTIFICADO.pfx \
  -d sql:$HOME/.var/app/org.mozilla.firefox/config/mozilla/firefox/TU_PERFIL
```

Verifica:

```bash
certutil -L -d sql:$HOME/.var/app/org.mozilla.firefox/config/mozilla/firefox/TU_PERFIL
```

**4.3. (Opcional, no imprescindible) Corrige `profiles.ini`** para que el flag `Default=1` clásico coincida con el perfil que de verdad usa Firefox (algunas herramientas antiguas solo entienden ese formato). Si todo te funciona ya sin hacer esto, no hace falta tocarlo:

```bash
cp ~/.var/app/org.mozilla.firefox/config/mozilla/firefox/profiles.ini ~/.var/app/org.mozilla.firefox/config/mozilla/firefox/profiles.ini.backup
# edita a mano o con sed para que Default=1 quede bajo el [ProfileN] correcto
```

**4.4. El paso decisivo — da permiso explícito a AutoFirma para leer la carpeta de Firefox:**

```bash
flatpak override --user --filesystem=~/.var/app/org.mozilla.firefox:ro es.gob.afirma
flatpak kill es.gob.afirma
```

(Es de solo lectura, y solo afecta a AutoFirma — no toca nada más).

**4.5. Configura AutoFirma:**

Abre AutoFirma → Herramientas/Preferencias → pestaña **Almacenes de claves**:
- Almacén por defecto: **NSS**
- Marca: **"Usar también en las llamadas a Autofirma desde el navegador"**
- Aplicar → Aceptar

Reabre AutoFirma, intenta firmar: tu certificado debería aparecer directo en la lista, sin buscar ningún archivo.

---

## 5. Diagnóstico si algo sigue sin funcionar

Si tras las secciones 3 y 4 algo sigue fallando, capturar el log en tiempo real es la forma más rápida de ver la causa exacta:

```bash
journalctl --user -f
```

Déjalo corriendo, repite la acción que falla (abrir AutoFirma, o pulsar "Firmar"/"Validar" en una web), y busca líneas con `es.gob.afirma.desktop`, `Exception`, `ADVERTENCIA` o `GRAVE`. Busca en concreto:

- `Detectado directorio del almacen NSS de claves: ~/.pki/nssdb` → confirma que ese flujo usa el almacén compartido de la sección 3, no el perfil de Firefox.
- `No se ha podido abrir el almacen ... IOException: load failed` → problema de contraseña en `~/.pki/nssdb` (ver sección 3).
- Referencias a `~/.var/app/org.mozilla.firefox` con "no existe" → falta el `flatpak override` de la sección 4.

---

## 6. Verificación final

1. Firma un documento de prueba en AutoFirma (tanto abierto manualmente como desde una web).
2. Confirma que "Emisor del certificado" muestra **AC FNMT Usuarios**.
3. Abre el resultado en Okular.

> **Nota práctica:** si al descargar o guardar el PDF de prueba el archivo se queda sin extensión (por ejemplo `firma-prueba` en vez de `firma-prueba.pdf`), los comandos de este manual (`pdfsig`, etc.) funcionan igual siempre que uses el nombre de archivo exacto que tengas — no hace falta renombrarlo. Si un comando dice "No such file or directory", comprueba el nombre real con `ls -la ~/Documentos/ | grep -i firma` y, si tienes dudas de qué tipo de archivo es, `file ~/Documentos/NOMBRE_EXACTO`.

### ⚠️ Sobre el aviso de Okular — **confirmado real y sin importancia**

Okular puede mostrar en el panel de firmas: **"La firma es criptográficamente no válida"** y **"Todavía no se ha verificado el certificado"**, incluso con todo correctamente configurado según este manual. Esto se confirmó como un **falso positivo de Okular/Poppler** con las cadenas de certificados de la FNMT (no depende de tu configuración, no se ha encontrado forma de solucionarlo desde el lado del usuario).

**Cómo confirmar que el documento SÍ es válido, ignorando a Okular:**

```bash
pdfsig -nssdir sql:$HOME/.pki/nssdb ~/RUTA/AL/ARCHIVO.pdf
```
Debe decir `Signature Validation: Signature is Valid.` (el apartado `Certificate Validation` puede seguir marcando error aunque la firma sea válida — es el mismo bug).

Y, para la confirmación oficial y definitiva:

https://valide.redsara.es/valide/firmar/ejecutar.html → **Validar Firma** → sube el PDF.

Si dice **"Firma válida"**, el documento es legalmente correcto y válido para presentar ante cualquier administración, **sin importar lo que diga Okular**.

### Comprobación adicional (opcional)

También puedes usar el test automático independiente del Ministerio de Trabajo, que comprueba navegador, sistema operativo y hace un test de firma real en los dos formatos estándar (CAdES y XAdES):

https://expinterweb.mites.gob.es/scriptAutofirmaTest/

---

## Resumen ultra-corto (para memoria rápida)

1. `flatpak install flathub es.gob.afirma`
2. Importar `Autofirma_ROOT.cer` como autoridad de confianza en Firefox y Brave
3. Crear `~/.pki/nssdb` **con contraseña EN BLANCO** (crítico) e importar ahí tu certificado personal + las CA de la FNMT — este almacén lo usan **tanto Okular como AutoFirma cuando firma/valida desde la web**. Verifica los nombres reales de las CA en Firefox antes de copiarlas; si falta alguna, descárgala de la web oficial de la FNMT.
4. Prueba primero si AutoFirma abierto manualmente ya detecta tu certificado solo con el paso 3 — si es así, sáltate el resto de esta sección. Si no, importa tu `.pfx` personal también en el perfil **real** de Firefox Flatpak y aplica `flatpak override --user --filesystem=~/.var/app/org.mozilla.firefox:ro es.gob.afirma`
5. AutoFirma → Preferencias → Almacenes de claves → NSS + "usar también desde el navegador"
6. Si algo falla, `journalctl --user -f` mientras repites la acción, para ver el error exacto
7. Si Okular dice "firma no válida" pese a todo, confía en `pdfsig` y en https://valide.redsara.es/ — es un bug conocido de Okular, no un problema real
