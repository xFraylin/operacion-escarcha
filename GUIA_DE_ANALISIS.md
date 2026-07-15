# Guía de análisis — cómo trabajar este laboratorio

Esto **no** es un manual de instalación (eso está en [`SETUP.md`](SETUP.md)).
Esto es el *cómo se investiga*: qué es cada pieza de evidencia, qué historia
cuenta, con qué herramienta se abre y en qué orden conviene atacarla.

**No hay spoilers.** No se revela ninguna respuesta: se enseña el método.

---

## 1. De qué va esto

Te entregan la estación de trabajo incautada de un empleado y varias capturas de
red. La empresa sospecha que se filtró un documento confidencial. Tu trabajo es
el de un analista forense real:

> Reconstruir **qué pasó, quién lo hizo, cuándo, y demostrarlo con artefactos.**

El caso tiene una trampa deliberada, y es la lección más importante del lab:
**hay dos hilos entrelazados**. Uno hace mucho ruido y llama la atención. El otro
es el que de verdad importa. Un analista con prisa culpa al ruido y cierra el
caso mal. Tu trabajo es separarlos y demostrar cuál es cuál.

Regla de oro del lab: **ningún dato se acepta porque "tiene sentido"**. Cada
afirmación tuya debe apoyarse en un artefacto que puedas señalar con el dedo.

### Las reglas del juego
- Responde las 50 preguntas de `PREGUNTAS.md` analizando la evidencia.
- Ve capturando los flags `NVL{...}` que aparecen escondidos en los artefactos.
- Autoevalúate con `CORRECTOR_AUTOEVALUACION.html` (offline, en tu navegador).
- **No abras `RESPUESTAS_Y_RUTA_FORENSE.md` hasta terminar.** Ahí está la ruta
  forense de cada pregunta: es para aprender del fallo, no para evitarlo.

---

## 2. La metodología (el orden importa)

Un caso no se ataca abriendo archivos al azar. Este es el flujo profesional, y el
que te recomiendo seguir aquí:

```
1. Verificar integridad   →  ¿la evidencia es la que dicen que es?
2. Reconocimiento         →  ¿qué hay? discos, particiones, huecos
3. Perfilar el sistema    →  ¿qué máquina es? quién la usa? qué red?
4. Ejecución              →  ¿qué programas corrieron?
5. Actividad de usuario   →  ¿qué archivos abrió, dónde navegó?
6. Dispositivos           →  ¿qué se conectó?
7. Red                    →  ¿con quién habló la máquina?
8. Borrado y anti-forense →  ¿qué intentó ocultar?
9. Recuperación           →  ¿qué puedo rescatar de lo destruido?
10. Volátil               →  ¿qué estaba vivo en el momento de la captura?
11. Timeline              →  juntar todo en una sola línea temporal
```

Cada fase alimenta a la siguiente. Un nombre de archivo que sale en la fase 5 te
dice qué buscar en la 7. Una IP de la fase 7 te dice qué buscar en la 10.

**Consejo:** ten abierto un documento de notas desde el minuto uno. Anota cada
hallazgo con **la fuente exacta** (artefacto + ruta + timestamp). Ese documento
*es* tu informe, y responder las preguntas será casi automático.

---

## 3. Fase por fase

### Fase 1 · Verificar la integridad

Antes de tocar nada: demuestra que trabajas sobre la evidencia correcta y que no
la has alterado. En un caso real, esto es lo primero que ataca la defensa.

```bash
ewfverify NVL-WKS-0472.E01          # ¿el hash interno cuadra?
sha256sum NVL-WKS-0472.E01
grep -A3 'NVL-WKS-0472.E01' 04_Hashes/HASHES_completo.txt
```
Trabaja siempre en **solo lectura** o sobre copias. Nunca montes en modo escritura.

### Fase 2 · Reconocimiento del disco

¿Cuántas particiones hay? ¿Coincide lo que ves con el tamaño del disco? Un hueco
grande sin asignar **es una pista**, no un detalle.

```bash
mmls NVL-WKS-0472.raw     # tabla de particiones + espacio no asignado
fsstat -o <offset> NVL-WKS-0472.raw    # tipo de FS, etiqueta, cluster, serial
```
Anota los **offsets** de cada partición: los vas a usar en todos los comandos
siguientes (`-o` en sectores).

### Fase 3 · Perfilar el sistema (registro)

Aquí sale la identidad de la máquina. Los hives están en la imagen
(`Windows/System32/config/`) y también sueltos en `03_Windows/Registry/`.

Qué buscar y dónde:

| Dato | Hive | Clave |
|---|---|---|
| Hostname | SYSTEM | `ControlSet001\Control\ComputerName\ComputerName` |
| Zona horaria | SYSTEM | `...\Control\TimeZoneInformation` |
| IP / Gateway / DNS / dominio | SYSTEM | `...\Services\Tcpip\Parameters\Interfaces\{GUID}` |
| Versión de Windows, propietario | SOFTWARE | `Microsoft\Windows NT\CurrentVersion` |
| Persistencia (autoarranque) | SOFTWARE | `Microsoft\Windows\CurrentVersion\Run` |
| Usuarios locales | SAM | `SAM\Domains\Account\Users\Names` |

```bash
hivexsh 03_Windows/Registry/SYSTEM      # navegación interactiva: cd, ls, lsval
hivexget 03_Windows/Registry/SYSTEM '\ControlSet001\Control\ComputerName\ComputerName' ComputerName
```
Con .NET: `RECmd`. Sin .NET: `hivexsh`, `regipy`, RegRipper.

**La zona horaria es crítica.** Si la ignoras, toda tu línea temporal quedará
desplazada y sacarás conclusiones erróneas sobre el orden de los hechos.

### Fase 4 · ¿Qué programas se ejecutaron?

La ejecución de un binario deja rastro en varios sitios a la vez. Que coincidan
es lo que convierte una sospecha en prueba.

| Artefacto | Qué prueba | Dónde |
|---|---|---|
| **UserAssist** | ejecución **por el usuario** desde el escritorio, con contador y última vez | NTUSER.DAT |
| **Amcache** | presencia/ejecución de binarios, rutas, tamaños | Amcache.hve |
| **Scheduled Tasks** | ejecución programada / persistencia | `Windows\System32\Tasks\` |
| **Run keys** | ejecución en cada arranque | SOFTWARE |

Ojo: **UserAssist está cifrado en ROT13**. Registry Explorer y RECmd lo decodifican
solos; a mano, aplica ROT13 al nombre del valor.

### Fase 5 · Actividad del usuario y archivos

Aquí es donde se demuestra la intención. No basta con que un archivo exista: hay
que probar que **alguien lo abrió**, y **desde dónde**.

| Artefacto | Qué cuenta |
|---|---|
| **LNK** (`...\Recent\`) | qué archivo se abrió, su **ruta original**, incluso si el origen ya no existe (¡USB!) |
| **RecentDocs** | documentos recientes, ordenados por `MRUListEx` (el primer índice = el más reciente) |
| **ShellBags** (UsrClass.dat) | qué **carpetas** se navegaron, incluso ya borradas |
| **TypedPaths** | rutas escritas a mano en la barra del Explorador |
| **RunMRU** | comandos escritos en el cuadro *Ejecutar* |
| **Navegador** | historial, descargas, cookies, caché |

```bash
# LNK
dotnet LECmd.dll -f archivo.lnk        # o: python3 -m pylnk3 parse archivo.lnk
# Navegador (SQLite puro)
sqlite3 History "SELECT datetime(last_visit_time/1000000-11644473600,'unixepoch'), url FROM urls;"
sqlite3 History "SELECT target_path, tab_url FROM downloads;"
```

**El detalle clave de los LNK:** guardan la ruta del *target*. Si un LNK apunta a
una unidad extraíble, has probado que el archivo se abrió **desde ese dispositivo**,
aunque el dispositivo ya no esté. Eso es oro en un caso de robo de datos.

### Fase 6 · Dispositivos USB

Un USB deja una cadena de rastros que hay que **correlacionar**:

1. `SYSTEM\...\Enum\USBSTOR` → fabricante, modelo y **número de serie**.
2. `USBSTOR\...\Properties\{83da6326-...}` → primera conexión, última conexión.
3. `SYSTEM\MountedDevices` → **qué letra de unidad** se le asignó.
4. Event log `20001` → instalación del driver = momento de la conexión.
5. ShellBags / LNK con esa letra → **qué se hizo** con él.

La cadena completa (serial → letra → archivos abiertos en esa letra) es lo que
convierte "hubo un USB" en "este USB se llevó estos archivos".

### Fase 7 · Red

Seis PCAPs separados por escenario. Ábrelos en Wireshark o tira de `tshark`.

```bash
# Consultas y respuestas DNS: ¿qué dominios resolvió y a qué IP?
tshark -r PCAP04_dns.pcap -Y "dns.flags.response==1" -T fields -e dns.qry.name -e dns.a

# Peticiones HTTP: ¿qué se pidió y a qué host?
tshark -r PCAP03_*.pcap -Y http.request -T fields -e http.host -e http.request.uri

# Extraer archivos transferidos por HTTP
tshark -r PCAP03_*.pcap --export-objects http,./salida_http

# FTP: comandos en claro (USER/PASS/STOR)
tshark -r PCAP06_*.pcap -Y ftp -T fields -e ftp.request.command -e ftp.request.arg

# Seguir una conversación completa
tshark -r captura.pcap -q -z follow,tcp,ascii,0
```
En Wireshark, botón derecho → **Follow → TCP Stream** es tu mejor amigo.

Pregúntate siempre: ¿esta conexión la inició **un programa** o **una persona**?
La respuesta cambia por completo la conclusión del caso.

### Fase 8 · Borrado y anti-forense

Lo que alguien intenta borrar suele ser lo más interesante del caso.

| Artefacto | Qué revela | Herramienta |
|---|---|---|
| **Papelera** `$I`/`$R` | nombre y ruta **original**, tamaño y **fecha de borrado** | `RBCmd` |
| **$MFT** | entradas de archivos borrados; `$SI` vs `$FN` delata *timestomping* | `MFTECmd`, `istat` |
| **$UsnJrnl** | el diario: crear, renombrar, borrar… en orden | `MFTECmd -f 'UsnJrnl_$J'` |
| **ADS** | datos escondidos en flujos alternos | `fls -r` (muestra `archivo:stream`) |
| **File slack** | restos de archivos anteriores en el espacio sobrante del clúster | `blkls -s` |
| **Event 1102** | alguien limpió el log de seguridad | Event logs |

```bash
fls -rd -o <offset> imagen.raw          # archivos borrados (marcados con *)
icat -o <offset> imagen.raw <inodo>     # extraer uno
tsk_recover -e -o <offset> imagen.raw ./recuperado/
blkls -s -o <offset> imagen.raw | strings | less    # file slack
```

**Sobre ADS:** en `fls -r` un flujo alterno aparece como `archivo.txt:nombre_stream`
con su propio identificador. Se extrae con `icat` usando ese id completo. Es un
escondite clásico y muchos analistas ni lo miran.

### Fase 9 · Recuperación

Hay una partición eliminada. Dos caminos, y conviene entender la diferencia:

```bash
testdisk imagen.raw     # Analyse → Quick Search → Deeper Search
                        # RECONSTRUYE la tabla: recuperas la partición con sus
                        # nombres de archivo y su estructura intacta.

photorec imagen.raw     # CARVING por firma: rescata el contenido aunque la tabla
                        # esté destruida, pero PIERDES los nombres originales.
```
Regla: **TestDisk primero** (si la estructura sobrevive, conservas metadatos).
**PhotoRec después**, para lo que TestDisk no alcance.

¿Recuperaste bien un archivo? Calcula su SHA256 y compáralo con
`04_Hashes/HASHES_completo.txt`. Si coincide bit a bit, tu recuperación es
íntegra y defendible ante un tribunal.

### Fase 10 · Evidencia volátil

Lo que estaba vivo cuando se capturó la máquina: procesos, conexiones abiertas,
credenciales en memoria, portapapeles, caché DNS.

```bash
strings memdump.raw | grep -Ei "ESTABLISHED|PASS |http://"
strings pagefile.sys | grep -Ei "STOR|\.xlsx"
bulk_extractor -o salida/ memdump.raw     # extrae emails, IPs, URLs, tarjetas...
```
Mira también `06_Volatil/LiveResponse/`: son las salidas de los comandos que se
ejecutaron en la máquina viva (`tasklist`, `netstat`, `net use`…).

> Nota: esta imagen de RAM está pensada para análisis de **cadenas y patrones**;
> no tiene estructuras de kernel, así que Volatility `pslist` no funcionará. Está
> explicado en `06_Volatil/README_VOLATIL.md`.

### Fase 11 · La línea de tiempo

Aquí es donde el caso deja de ser una lista de hallazgos y se convierte en una
**historia demostrable**.

```bash
# Timeline del sistema de archivos con TSK
fls -r -m C: -o <offset> imagen.raw > bodyfile
mactime -b bodyfile -d 2025-11-01..2025-11-09 > timeline.csv

# Supertimeline (si tienes plaso)
log2timeline.py --storage-file caso.plaso imagen.raw
psort.py -o dynamic -w timeline.csv caso.plaso
```

Luego **funde a mano** los eventos de red, registro y memoria en esa línea. Cuando
veas la secuencia completa en orden, el caso se explica solo — y ahí es cuando
distingues el ruido del hecho real.

---

## 4. Chuleta: artefacto → herramienta

| Quiero saber… | Miro… | Con… |
|---|---|---|
| Qué máquina es | SYSTEM / SOFTWARE | `hivexsh`, RECmd |
| Quién usó el equipo | SAM, Event 4624 | RECmd, EvtxECmd |
| Qué programas corrieron | UserAssist, Amcache | RECmd |
| Qué archivos se abrieron | LNK, RecentDocs, JumpLists | LECmd, RECmd |
| Dónde navegó | ShellBags, TypedPaths | SBECmd, RECmd |
| Qué USB se conectó | USBSTOR, MountedDevices, Evt 20001 | RECmd |
| Qué se borró | Papelera, $MFT, $UsnJrnl | RBCmd, MFTECmd, `fls -rd` |
| Qué se ocultó | ADS, file slack | `fls -r`, `blkls -s` |
| Con quién habló | PCAPs | Wireshark, `tshark` |
| Qué estaba vivo | RAM, LiveResponse | `strings`, `bulk_extractor` |
| Qué hay en el hueco del disco | partición borrada | `testdisk`, `photorec` |
| Cuándo pasó todo | timeline | `mactime`, plaso |

---

## 5. Errores típicos (los que este lab castiga a propósito)

1. **Ignorar la zona horaria.** Tu timeline queda desplazada y el orden de los
   hechos deja de tener sentido.
2. **Confundir "existe" con "se ejecutó/se abrió".** Un archivo en disco no prueba
   nada por sí solo. Necesitas UserAssist, un LNK, un Prefetch…
3. **Quedarte en el primer hallazgo llamativo.** Lo ruidoso rara vez es lo grave.
4. **Mirar un solo artefacto por conclusión.** Toda afirmación fuerte se apoya en
   dos o tres fuentes independientes que coinciden.
5. **Olvidar lo no asignado.** Espacio libre, slack, papelera y particiones
   borradas son la mitad del caso.
6. **Atribuir a un programa lo que hizo una persona** (o al revés). Es *la*
   trampa central de este laboratorio.

---

## 6. Cuando termines

1. Pasa tus respuestas al **corrector** (`CORRECTOR_AUTOEVALUACION.html`) y mira
   el desglose por secciones: te dice exactamente en qué área flojeas.
2. Abre `RESPUESTAS_Y_RUTA_FORENSE.md` y compara **tu ruta** con la propuesta.
   Que hayas acertado importa menos que si llegaste por el camino correcto: en un
   caso real, adivinar no vale.
3. Escribe un informe de una página: qué pasó, quién, cuándo y **qué artefacto lo
   demuestra**. Ese ejercicio es el que de verdad te prepara.

---

¿Estás empezando en forense? No pasa nada si te atascas. Sigue el orden de las
fases, apunta todo con su fuente, y deja que la evidencia hable. Suerte 🧊
