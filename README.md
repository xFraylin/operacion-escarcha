# Operación Escarcha — Laboratorio forense (estilo eCDFP)

Un laboratorio de forense digital **original, completo y gratuito**: imágenes de
disco reales, capturas de red, hives de registro, artefactos de Windows,
sistema de archivos NTFS con borrados y particiones eliminadas, evidencia
volátil, 50 preguntas tipo examen y un corrector interactivo offline.

Se resuelve entero con herramientas libres (The Sleuth Kit, TestDisk, PhotoRec,
Wireshark, hivex, EZ Tools…). No necesitas licencias ni hardware especial.

> **Aviso.** Este proyecto **no está afiliado, patrocinado ni respaldado por INE
> ni eLearnSecurity**. "eCDFP" es marca de sus respectivos titulares y aquí se
> menciona solo de forma descriptiva. **No contiene material del examen real**:
> el caso, la evidencia y las preguntas son 100% originales y generados desde
> cero. Si preparas una certificación, respeta el NDA que hayas firmado.

---

## Qué hay dentro

| Módulo | Contenido |
|---|---|
| **Imágenes forenses** | Disco `NVL-WKS-0472` en **E01 / RAW / DD** (GPT, partición eliminada), **USB** FAT32, **tarjeta SD** |
| **Red** | 6 PCAPs: navegación, fake antivirus, descarga + C2, DNS, SMB, exfiltración FTP |
| **Registro** | 8 hives REGF: `SYSTEM`, `SOFTWARE`, `SAM`, `SECURITY`, `DEFAULT`, `NTUSER.DAT`, `UsrClass.dat`, `Amcache.hve` |
| **Artefactos Windows** | LNK, Chrome/Firefox (History, Cookies, Downloads, Cache), Scheduled Tasks, historial PowerShell, Event Logs |
| **Sistema de archivos** | `$MFT`, `$LogFile`, `$UsnJrnl`, archivos y carpetas eliminados, Papelera `$I/$R`, **ADS**, **file slack**, partición eliminada |
| **Volátil** | Imagen de RAM, `pagefile.sys`, `hiberfil.sys`, salidas de live response |
| **Evaluación** | 50 preguntas (4 opciones), clave con **ruta forense** de cada una, 20 flags `NVL{...}` |
| **Extra** | `CORRECTOR_AUTOEVALUACION.html` — corrector interactivo, offline, sin instalación |

### Escenarios cubiertos
Fake antivirus · infección y persistencia · uso de USB · acceso a recurso SMB ·
robo de documentos · exfiltración de datos · PowerShell · borrado de archivos ·
anti-forense · recuperación de particiones · carving · análisis de línea de tiempo.

---

## El caso

`NVL-IR-2025-1187` — Una empresa de logística marítima detecta la fuga de su
manifiesto confidencial de clientes. Se incauta la estación de un empleado
descontento. Dos hilos se entrelazan y hay que **separarlos**: una infección por
scareware que hace mucho ruido, y la exfiltración real, deliberada y manual.
El analista debe reconstruir la línea de tiempo y probar qué pasó de verdad.

Empieza por [`Operacion_Escarcha/00_README_Escenario.md`](Operacion_Escarcha/00_README_Escenario.md).

---

## Inicio rápido (Linux)

```bash
# 1) Dependencias (Debian/Ubuntu; en Kali casi todo viene ya)
sudo apt install -y sleuthkit ewf-tools testdisk ntfs-3g \
     wireshark tshark bulk-extractor sqlite3 libhivex-bin yara

# 2) Clona el repo
git clone https://github.com/<TU-USUARIO>/operacion-escarcha.git
cd operacion-escarcha

# 3) Descarga el paquete completo (imágenes grandes) desde Releases
#    y descomprímelo, o regenera RAW/DD desde el E01 incluido:
bash scripts/rebuild_from_e01.sh

# 4) Mira las particiones (verás C: y un hueco: la partición borrada)
mmls Operacion_Escarcha/01_Imagenes_Forenses/NVL-WKS-0472.raw
```

Guía completa de herramientas: [`SETUP.md`](SETUP.md) · Cómo investigar el caso: [`GUIA_DE_ANALISIS.md`](GUIA_DE_ANALISIS.md).

### Cómo trabajarlo

> **¿Nuevo en forense o no sabes por dónde empezar?** Lee la
> **[Guía de análisis](GUIA_DE_ANALISIS.md)**: explica de qué va el caso, la
> metodología fase por fase y qué herramienta usar para cada artefacto. Sin spoilers.
1. Lee el briefing (`00_README_Escenario.md`). Verifica los hashes.
2. Analiza la evidencia y responde [`PREGUNTAS.md`](Operacion_Escarcha/PREGUNTAS.md).
3. Autoevalúate abriendo **`CORRECTOR_AUTOEVALUACION.html`** en tu navegador.
4. Solo al terminar, abre `RESPUESTAS_Y_RUTA_FORENSE.md` para ver la ruta forense
   de cada pregunta.

---

## Descargas

- **Repositorio** (este): documentación, PCAPs, hives, artefactos, scripts,
  corrector y la imagen **E01** (1,5 MB). Suficiente para casi todo el lab.
- **[Releases](../../releases)**: paquete completo (~2,7 MB comprimido) con
  `RAW`/`DD`, imagen de **USB**, **tarjeta SD** y evidencia **volátil**.

Los `.raw`/`.dd` no se versionan (640 MB cada uno); se regeneran del E01:
```bash
ewfexport -u -t NVL-WKS-0472 -f raw NVL-WKS-0472.E01
```

---

## Publicar tu propia copia

¿Quieres alojarlo en tu cuenta o crear una variante? Un solo comando:

```bash
gh auth login          # una vez (GitHub CLI)
./publish.sh           # crea el repo, hace el push y publica el Release
```
Opciones: `./publish.sh -n mi-repo -p` (privado) · `-z ruta/al/Lab.zip`.

---

## Notas de fidelidad (lo que es y lo que no)

Se prioriza la honestidad técnica sobre aparentar completitud:

**Auténtico y verificado** — GPT y partición eliminada (recuperable con TestDisk),
NTFS real con `$MFT`/`$LogFile` nativos, archivos y carpetas borrados, Papelera
`$I/$R`, ADS, file slack inyectado en un clúster real, hives REGF válidos
(parseables con Registry Explorer/RECmd/hivex), `$UsnJrnl` con registros
`USN_RECORD_V2`, LNK reales, SQLite de navegadores, PCAPs diseccionables en
Wireshark, E01 verificado con `ewfverify`.

**Con matices, documentados en cada README:**

| Formato | Situación | Cobertura equivalente |
|---|---|---|
| Prefetch, Jump Lists | sin escritor open-source fiable | UserAssist + Amcache + LNK + RecentDocs |
| EVTX binario | ídem | export CSV estilo EvtxECmd |
| `SRUDB.dat`, BITS | ESE, no forjable | Chrome History + PCAP |
| `AD1` | formato propietario de FTK | E01 + RAW/DD cubren todo |
| Imagen de RAM | análisis de **cadenas** (strings/grep/bulk_extractor/YARA), **no** estructuras de kernel | Volatility `pslist` **no** funcionará; procesos y conexiones se entregan en `LiveResponse/` |
| `hiberfil.sys` | comprimido con Xpress, no forjable | placeholder documentado |

---

## Seguridad

**No hay malware real en este repositorio.** El "fake antivirus", el "RAT" y el
C2 existen solo como *huella forense*: nombres de archivo, claves de registro,
entradas de log, balizas en el PCAP y binarios inertes tipo placeholder (texto
plano con una cabecera `MZ`). Nada es ejecutable ni funcional.

Aun así, algunos antivirus pueden marcar heurísticamente los placeholders o los
PCAPs con dominios simulados. Es un falso positivo. Analiza el lab en una VM si
te da tranquilidad, como buena práctica general.

---

## Reproducir o crear variantes

En [`Operacion_Escarcha/05_Scripts/`](Operacion_Escarcha/05_Scripts/) están los
generadores. Cambiando las variables (empresa, sospechoso, IPs, fechas, seriales)
se regenera un caso distinto — útil para no reconocer las respuestas de memoria o
para montar un lab a tus alumnos.

---

## Licencia

- **Scripts y código** (`05_Scripts/`, corrector HTML): [MIT](LICENSE).
- **Contenido del lab** (caso, evidencia, preguntas, documentación):
  [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/deed.es).

Úsalo, modifícalo y compártelo. Si lo usas para formación o lo publicas
modificado, cita el origen y mantén la misma licencia.

---

## Créditos

Laboratorio diseñado y construido de forma colaborativa con Claude (Anthropic),
verificando cada artefacto con herramientas forenses reales (The Sleuth Kit,
libewf, hivex, scapy, TestDisk). Nació como preparación personal para el eCDFP y
se publica para que cualquiera pueda practicar forense digital, tenga o no
intención de certificarse.

Si te sirve, deja una estrella ⭐ y compártelo con quien esté aprendiendo.

---

<details>
<summary><b>English summary</b></summary>

**Operación Escarcha** is a free, fully original digital forensics lab in the
style of a practical DFIR exam. It ships real disk images (E01/RAW/DD with a
deleted GPT partition), a USB and SD card image, 6 PCAPs, 8 valid registry
hives, Windows artifacts (LNK, browser SQLite, scheduled tasks, PowerShell
history, event logs), NTFS internals (`$MFT`, `$LogFile`, `$UsnJrnl`, deleted
files/folders, Recycle Bin, ADS, file slack), volatile evidence (RAM image,
pagefile, live response output), 50 exam-style questions with a full forensic
walkthrough answer key, 20 `NVL{...}` flags, and an offline interactive grader.

Solvable end-to-end with free tools on Linux (The Sleuth Kit, TestDisk, PhotoRec,
Wireshark, hivex, EZ Tools via .NET). The case narrative is in Spanish; artifacts,
paths, and network data are language-neutral.

**Not affiliated with INE / eLearnSecurity. Contains no real exam material** —
everything is original. **No real malware**: all "malicious" components exist
only as inert forensic traces.

See [SETUP.md](SETUP.md) to get started. Content CC BY-SA 4.0, scripts MIT.
</details>
