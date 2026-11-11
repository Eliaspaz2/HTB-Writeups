[README.md](https://github.com/user-attachments/files/34694023/README.md)
# 🔍 HTB Sherlock — Tracer

![Plataforma](https://img.shields.io/badge/Plataforma-Hack%20The%20Box-9FEF00?style=flat-square&logo=hackthebox&logoColor=black)
![Categoría](https://img.shields.io/badge/Categoría-DFIR-blue?style=flat-square)
![Dificultad](https://img.shields.io/badge/Dificultad-Easy-green?style=flat-square)
![SO](https://img.shields.io/badge/Entorno-Kali%20Linux-557C94?style=flat-square&logo=kalilinux&logoColor=white)

Writeup de mi resolución del Sherlock **Tracer** de Hack The Box, un laboratorio de **análisis forense en Windows** enfocado en detectar el uso de **PsExec** para movimiento lateral, usando logs de eventos (EVTX) y archivos Prefetch.

> ⚠️ Este repositorio tiene fines **educativos**. Documenta el proceso de investigación, los errores que encontré en el camino y cómo los resolví.

---

## 📑 Índice

1. [Escenario](#-escenario)
2. [Herramientas utilizadas](#-herramientas-utilizadas)
3. [Preparación del entorno](#-preparación-del-entorno)
4. [Análisis inicial con Chainsaw](#-análisis-inicial-con-chainsaw)
5. [Análisis de Prefetch](#-análisis-de-prefetch)
6. [Resolución de las preguntas](#-resolución-de-las-preguntas)
7. [Problemas encontrados y soluciones](#-problemas-encontrados-y-soluciones)
8. [Mapeo MITRE ATT&CK](#-mapeo-mitre-attck)
9. [Conclusiones y aprendizajes](#-conclusiones-y-aprendizajes)

---

## 📖 Escenario

> Un analista SOC junior reportó múltiples alertas que indicaban la presencia de **PsExec** en una estación de trabajo. Las alertas fueron verificadas y escaladas a Tier II. Como **Incident Responder**, se realizó el triage del endpoint en busca de artefactos de interés para responder las preguntas y reportar el incidente.

**Artefacto entregado:** `tracer.zip`, que contiene una recolección parcial de un sistema Windows (el host víctima es **Forela-Wkstn002**):

```
C/
├── $Extend/
│   ├── $J          ← USN Journal
│   └── $Max
└── Windows/
    ├── prefetch/   ← Archivos Prefetch (.pf)
    └── System32/
        └── winevt/
            └── logs/   ← Logs de eventos (.evtx)
```

---

## 🧰 Herramientas utilizadas

| Herramienta | Uso |
|---|---|
| [Chainsaw](https://github.com/WithSecureLabs/chainsaw) | Búsqueda y detección sobre logs EVTX (hunt / search) |
| [Sigma Rules](https://github.com/SigmaHQ/sigma) | Reglas de detección usadas por Chainsaw |
| [WinPrefetchView](https://www.nirsoft.net/utils/win_prefetch_view.html) (NirSoft) | Visualización de archivos Prefetch |
| `sccainfo` (libscca-utils) | Parseo nativo en Linux de Prefetch comprimidos de Windows 10/11 |
| Wine | Ejecutar herramientas de Windows en Kali |

---

## ⚙️ Preparación del entorno

### 1. Instalar Chainsaw

```bash
apt install chainsaw
```

> 💡 El paquete de `apt` instala **solo el binario**. Los *mappings* y las reglas no quedan en el directorio de trabajo, así que hay que descargarlos aparte (ver paso 2).

### 2. Descargar mappings, reglas de Chainsaw y reglas Sigma

```bash
mkdir -p /home/kali/Tools && cd /home/kali/Tools
git clone https://github.com/WithSecureLabs/chainsaw.git
git clone https://github.com/SigmaHQ/sigma.git
```

Estructura resultante:

```
/home/kali/Tools/
├── chainsaw/   ← contiene mappings/ y rules/
└── sigma/      ← contiene rules/
```

![Clonado de repositorios](images/01-clone-repos.png)

### 3. Instalar herramientas para Prefetch

```bash
# Wine para ejecutar WinPrefetchView
apt install wine

# Parser nativo de Prefetch (soporta compresión de Windows 10)
apt install libscca-utils
```

WinPrefetchView se descarga desde la web de NirSoft (versión **x64**) y se descomprime:

```bash
mkdir /home/kali/Tools/winprefetchview
unzip ~/Downloads/winprefetchview-x64.zip -d /home/kali/Tools/winprefetchview/
```

---

## 🪓 Análisis inicial con Chainsaw

Con todo preparado, ejecuté un **hunt** sobre todos los logs usando las reglas de Sigma y las de Chainsaw:

```bash
cd /home/kali/Tools

chainsaw hunt /home/kali/htb_labs_blue/tracer/Tracer/C/Windows/System32/winevt/logs/ \
  -s sigma/rules/ \
  --mapping chainsaw/mappings/sigma-event-logs-all.yml \
  -r chainsaw/rules/
```

| Parámetro | Descripción |
|---|---|
| `-s` | Directorio de reglas Sigma |
| `--mapping` | Archivo que traduce los campos de Sigma a los campos de los EVTX |
| `-r` | Reglas nativas de Chainsaw |

Chainsaw cargó **3147 reglas** y procesó los **153 archivos EVTX** (214 MiB) en unos 3 minutos. Los resultados se agrupan por categoría (Account Tampering, Antivirus, etc.):

![Chainsaw hunt](images/02-chainsaw-hunt.png)

Entre las detecciones aparecen varias relacionadas con PsExec en el host **Forela-Wkstn002**:

- **PsExec Service File Creation** (Sysmon, Event ID 11): creación de `C:\Windows\PSEXESVC.exe`
- **PsExec Service Installation** (System, Event ID 7045): instalación del servicio `PSEXESVC`
- **PsExec Default Named Pipe** (Sysmon, Event ID 17): creación del pipe `\PSEXESVC`
- **PSEXEC Remote Execution File Artefact** (Sysmon, Event ID 11): creación de Key Files `PSEXEC-*.key`

---

## 🗂️ Análisis de Prefetch

El Prefetch registra cuántas veces se ejecutó un binario y sus últimas 8 ejecuciones, lo que lo hace ideal para contar las ejecuciones de PsExec.

### Intento 1: WinPrefetchView con Wine

```bash
cd /home/kali/Tools/winprefetchview
wine WinPrefetchView.exe
```

En **Options → Advanced Options** se apunta a la carpeta Prefetch del laboratorio (en Wine la raíz de Linux es la unidad `Z:`):

```
Z:\home\kali\htb_labs_blue\tracer\Tracer\C\Windows\prefetch
```

![WinPrefetchView en Wine](images/03-winprefetchview-wine.png)

❌ **Problema:** el archivo `PSEXESVC.EXE-AD70946C.pf` aparece, pero las columnas **Process EXE**, **Run Counter** y **Last Run Time** quedan vacías o en 0.

**Causa:** en Windows 10/11 los Prefetch están **comprimidos** (formato MAM / Xpress Huffman). WinPrefetchView usa la API de Windows `RtlDecompressBufferEx` para descomprimirlos, y **Wine no la implementa**.

### Intento 2: `sccainfo` (nativo en Linux) ✅

```bash
sccainfo /home/kali/htb_labs_blue/tracer/Tracer/C/Windows/prefetch/PSEXESVC.EXE-AD70946C.pf
```

`sccainfo` sí descomprime el Prefetch y muestra el **run count**, los **últimos 8 tiempos de ejecución (en UTC)** y los **archivos referenciados** por el proceso:

![sccainfo run count](images/04-sccainfo-runcount.png)

| # | Last run time (UTC) |
|---|---|
| 1 | 2023-09-07 12:10:03 |
| 2 | 2023-09-07 12:09:09 |
| 3 | 2023-09-07 12:08:54 |
| 4 | 2023-09-07 12:08:23 |
| **5** | **2023-09-07 12:06:54** |
| 6 | 2023-09-07 11:57:53 |
| 7 | 2023-09-07 11:57:43 |
| 8 | 2023-09-07 11:55:44 |

En la lista de archivos referenciados aparecen los **Key Files** que deja PsExec en `C:\Windows\`, con el host de origen en el nombre:

```
\WINDOWS\PSEXEC-FORELA-WKSTN001-CAD5E7EF.KEY
\WINDOWS\PSEXEC-FORELA-WKSTN001-89A517EE.KEY
\WINDOWS\PSEXEC-FORELA-WKSTN001-415385DF.KEY
\WINDOWS\PSEXEC-FORELA-WKSTN001-C3E84A44.KEY
\WINDOWS\PSEXEC-FORELA-WKSTN001-95F03CFE.KEY
...
```

![sccainfo key files](images/05-sccainfo-keyfiles.png)

---

## ❓ Resolución de las preguntas

### Q1 — ¿Cuántas veces ejecutó PsExec el atacante en el sistema?

Cada ejecución de PsExec contra un equipo remoto instala y ejecuta el servicio `PSEXESVC.exe` en la víctima. El campo **Run count** del Prefetch `PSEXESVC.EXE-AD70946C.pf` (ver captura de `sccainfo` arriba) indica cuántas veces corrió.

**Respuesta:** `9`

---

### Q2 — ¿Cuál es el nombre del binario de servicio que deja PsExec para ejecutar comandos remotos?

PsExec copia un binario al share `ADMIN$`, lo registra como servicio, ejecuta el comando y luego se elimina. Por eso en la víctima no aparece `PsExec64.exe`, sino el servicio. Se confirma con el **Executable filename** del Prefetch (`PSEXESVC.EXE`), el evento **7045** y el Sysmon **11** (`TargetFilename: C:\Windows\PSEXESVC.exe`).

**Respuesta:** `psexesvc.exe`

---

### Q3 — Timestamp de la 5ª última ejecución del servicio de PsExec

`sccainfo` lista los **Last run times** del más reciente al más antiguo, ya en **UTC**. El 5º es `Sep 07, 2023 12:06:54 UTC` (ver tabla de la sección anterior).

**Respuesta:** `07/09/2023 12:06:54`

> 💡 Cuidado con las zonas horarias: WinPrefetchView muestra los tiempos en **hora local** del equipo de análisis, mientras que `sccainfo` y Sysmon (`UtcTime`) usan **UTC**. Siempre conviene normalizar a UTC antes de cruzar artefactos.

---

### Q4 — Hostname de la estación desde la que el atacante se movió lateralmente

PsExec incluye el **nombre del equipo de origen** en los artefactos que crea en la víctima:

- Key Files: `PSEXEC-FORELA-WKSTN001-<ID>.key` (vistos en el Prefetch)
- Named pipes: `\PSEXESVC-FORELA-WKSTN001-<PID>-stdin/stdout/stderr`

Como la víctima es `Forela-Wkstn002`, el origen del movimiento lateral es la otra estación.

**Respuesta:** `Forela-Wkstn001`

---

### Q5 — Nombre completo del Key File dejado por la 5ª última instancia

Buscando en los resultados del hunt la detección **PSEXEC Remote Execution File Artefact** (Sysmon 11) con la marca de tiempo de la 5ª ejecución (12:06:5x UTC), aparece el evento con **Record ID `159599`**:

![Key File de la 5ta instancia](images/06-q5-keyfile.png)

`TargetFilename: C:\Windows\PSEXEC-FORELA-WKSTN001-95F03CFE.key`

**Respuesta:** `PSEXEC-FORELA-WKSTN001-95F03CFE.key`

---

### Q6 — Timestamp de creación del Key File en disco

Con el EventRecordID se puede ver el evento completo usando una expresión *tau* (`-t`):

```bash
chainsaw search -t 'Event.System.EventRecordID: =159599' \
  /home/kali/htb_labs_blue/tracer/Tracer/C/Windows/System32/winevt/logs
```

> ⚠️ Los nombres de campo distinguen mayúsculas: es `Event.System`, **no** `Event.system`. Con minúscula, Chainsaw devuelve `Found 0 hits`.

![EventRecordID 159599](images/07-q6-eventrecordid.png)

El evento es un Sysmon **Event ID 11** (FileCreate) en `Forela-Wkstn002`, con `SystemTime: 2023-09-07T12:06:55.064Z` y `CreationUtcTime: 2023-09-07 12:06:55.054`.

**Respuesta:** `07/09/2023 12:06:55`

---

### Q7 — Nombre completo del Named Pipe terminado en "stderr" de la 5ª última instancia

PsExec crea named pipes para redirigir `stdin`, `stdout` y `stderr` entre los dos equipos (Sysmon **Event ID 17**, PipeCreated). Se buscan por palabra clave:

```bash
chainsaw search 'stderr' /home/kali/htb_labs_blue/tracer/Tracer/C/Windows/System32/winevt/logs
```

Se elige el que coincide con la 5ª instancia: `SystemTime 2023-09-07T12:06:55.084Z`, **EventRecordID `159603`**, inmediatamente después de la creación del Key File (`159599`).

![Named pipe stderr](images/08-q7-namedpipe.png)

**Respuesta:** `\PSEXESVC-FORELA-WKSTN001-3056-stderr`

---

### 📋 Resumen de respuestas

| # | Pregunta | Respuesta |
|---|---|---|
| 1 | Cantidad de ejecuciones de PsExec | `9` |
| 2 | Binario de servicio dejado por PsExec | `psexesvc.exe` |
| 3 | Timestamp de la 5ª última ejecución | `07/09/2023 12:06:54` |
| 4 | Host de origen del movimiento lateral | `Forela-Wkstn001` |
| 5 | Key File de la 5ª instancia | `PSEXEC-FORELA-WKSTN001-95F03CFE.key` |
| 6 | Creación del Key File | `07/09/2023 12:06:55` |
| 7 | Named pipe `stderr` de la 5ª instancia | `\PSEXESVC-FORELA-WKSTN001-3056-stderr` |

### 🕒 Línea de tiempo de la 5ª instancia (UTC)

| Hora | Evento | Fuente |
|---|---|---|
| 12:06:54.928 | Ejecución de `PSEXESVC.EXE` | Prefetch (run time #5) |
| 12:06:55.054 | Creación de `PSEXEC-FORELA-WKSTN001-95F03CFE.key` | Sysmon 11 · Record 159599 |
| 12:06:55.084 | Creación del pipe `\PSEXESVC-FORELA-WKSTN001-3056-stderr` | Sysmon 17 · Record 159603 |

---

## 🛠️ Problemas encontrados y soluciones

| Error | Causa | Solución |
|---|---|---|
| `Specified event log path is invalid - rules/` | Las carpetas `rules/`, `sigma/` y `mappings/` no existían en el directorio actual (apt solo instala el binario) | Clonar los repos de Chainsaw y Sigma y usar rutas correctas |
| `Specified event log path is invalid - .../winevt/Logs/` | Linux distingue mayúsculas: la carpeta real es `logs` | Verificar con `find ... -iname "*.evtx"` |
| `ShellExecuteEx failed: File not found` (Wine) | WinPrefetchView no estaba descargado; Wine no lo trae | Descargar el zip x64 de NirSoft y descomprimirlo |
| WinPrefetchView sin Run Counter ni Process EXE | Prefetch de Win10 comprimido; Wine no implementa `RtlDecompressBufferEx` | Usar `sccainfo` o WinPrefetchView en un Windows real |
| `sccainfo: no such file` | La carpeta real es `prefetch` (minúscula), no `Prefetch` | `find ... -iname "PSEXESVC*.pf"` |
| `chainsaw search -t` → `Found 0 hits` | Campo mal escrito: `Event.system` | Usar `Event.System.EventRecordID` |

> 💡 **Lección principal:** cuando trabajás con evidencia de Windows en Linux, **nunca asumas la capitalización de las rutas**. Usá `find -iname` para ubicar los archivos.

---

## 🎯 Mapeo MITRE ATT&CK

| Táctica | Técnica | Evidencia |
|---|---|---|
| Lateral Movement | [T1021.002 — SMB/Windows Admin Shares](https://attack.mitre.org/techniques/T1021/002/) | Copia de `PSEXESVC.exe` vía `ADMIN$` |
| Execution | [T1569.002 — Service Execution](https://attack.mitre.org/techniques/T1569/002/) | Instalación del servicio `PSEXESVC` (Event ID 7045) |
| Lateral Movement | [T1570 — Lateral Tool Transfer](https://attack.mitre.org/techniques/T1570/) | Binario transferido desde `Forela-Wkstn001` |

---

## 🧠 Conclusiones y aprendizajes

- **PsExec deja muchos rastros en la víctima:** binario `PSEXESVC.exe`, servicio (7045), Key Files `PSEXEC-<HOST>-<ID>.key`, named pipes `\PSEXESVC-<HOST>-<PID>-stdin/stdout/stderr` y su entrada en Prefetch.
- **Los artefactos se complementan:** el Prefetch da el conteo y los tiempos; los EVTX (Sysmon + System) dan el detalle y permiten reconstruir la línea de tiempo al milisegundo.
- **Event IDs clave para detectar PsExec:**
  - `7045` (System): instalación de servicio
  - `1` (Sysmon): creación de proceso
  - `11` (Sysmon): creación de archivo
  - `17` / `18` (Sysmon): creación / conexión de named pipe
- **Chainsaw** es muy útil para triage rápido con `hunt` y para pivotear con `search -t` sobre campos específicos.
- El Prefetch comprimido de Windows 10+ **no se puede leer con Wine**; hay que usar herramientas que implementen la descompresión (como `sccainfo`) o un Windows real.

---

## 📚 Referencias

- [Hack The Box — Tracer Sherlock](https://app.hackthebox.com/sherlocks/Tracer)
- [HTB Blog — How to detect PsExec and lateral movements](https://www.hackthebox.com/blog/how-to-detect-psexec-and-lateral-movements)
- [Chainsaw — WithSecure Labs](https://github.com/WithSecureLabs/chainsaw)
- [SigmaHQ](https://github.com/SigmaHQ/sigma)
- [libscca (sccainfo)](https://github.com/libyal/libscca)

---

<p align="center">Hecho por <b>[TU NOMBRE / USUARIO]</b> · Estudiante de la Universidad Siglo 21</p>
