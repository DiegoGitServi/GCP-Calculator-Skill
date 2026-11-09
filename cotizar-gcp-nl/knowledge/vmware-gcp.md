# VMware → GCP — interpretar RVTools e inventarios vSphere

Guía determinista para convertir un export de VMware (RVTools `.xlsx`/`.csv`, o tabla copiada de
vCenter) en ítems `compute` del plan. Regla de oro: **la fila de la VM manda** (columnas
`CPUs`/`Memory` de la VM); el hardware del host (modelo de procesador) solo orienta la **elección
de serie**, nunca cambia las specs de la VM.

## 1. Detectar el formato

- RVTools: pestañas `vInfo` (una fila por VM), `vCPU`, `vMemory`, `vDisk`, `vPartition`,
  `vHost` (hosts físicos ESXi).
- Señales inequívocas: columnas `VM`, `Powerstate`, `CPUs`, `Memory`, `Provisioned MiB`,
  `In Use MiB`, `OS according to the configuration file`.
- Fuente de verdad por VM = `vInfo`. `vHost` solo aporta el modelo de CPU del host (§4).
- `vDisk`/`vPartition`: discos individuales por VM — usar solo si piden separar discos de datos.

## 2. Columnas vInfo → plan

| Columna RVTools | Uso |
|---|---|
| `VM` | `etiqueta` |
| `Powerstate` | `poweredOff` → cotizar igual y ANOTAR en supuestos (o preguntar si se excluyen) |
| `CPUs` | vCPUs de la VM (**ya es el total**; NO multiplicar por sockets) |
| `Sockets` × `Cores p/s` | solo si falta `CPUs`: vCPUs = sockets × cores |
| `Memory` | RAM en **MiB** → GB = MiB ÷ 1024 (redondear hacia arriba) |
| `Provisioned MiB` | almacenamiento aprovisionado de la VM → `bootDiskSizeGb` de la MISMA VM = MiB ÷ 1024 (redondear hacia arriba). Si el inventario distingue disco de sistema vs discos de datos, el de sistema es el boot disk y los de DATOS van como ítem `persistent_disk` aparte (ver abajo). |
| `In Use MiB` | ignorar para cotizar (es consumo, no capacidad); usar solo si piden explícitamente "pagar lo usado" |
| `OS according to the configuration file` (o `…VMware Tools`) | → alias de `os` (§3) |
| `Host` / `Cluster` / `Resource pool` / `Datacenter` | ignorar (no afectan el precio en GCP) |

**El disco de arranque va SIEMPRE dentro del ítem compute** (`bootDiskType` + `bootDiskSizeGb`),
NUNCA como un `persistent_disk`/Hyperdisk separado. Si el requerimiento dice "disco Hyperdisk
Balanced de 150 GiB" para una VM, eso es `bootDiskType:"hyperdisk-balanced"`, `bootDiskSizeGb:150`
en esa misma VM — no un recurso de almacenamiento aparte (separarlo duplica el cobro y deja la VM
con el disco default de 10 GiB).

**Discos de DATOS adicionales** (los que van ADEMÁS del disco de arranque, p. ej. "cada servidor con
un disco de datos de 500 GB", o un RVTools que lista varios discos por VM): SÍ van como un ítem
`persistent_disk` separado — `{tipo:"hyperdisk", nro_discos:<nº total de discos>, gib_por_disco:<GB c/u>}`.
NO están fuera de alcance (son el producto "Hyperdisk and Persistent Disk", id 10). Agrupa discos del
mismo tamaño en un solo ítem con `nro_discos`.

## 3. SO de VMware → alias `os` del plan

| Texto del SO (contiene) | alias `os` |
|---|---|
| `Microsoft Windows Server …` | `windows` |
| `Microsoft Windows 10/11 …` | `windows` (anotar: SO de escritorio; en GCP aplica licencia Server) |
| `Red Hat Enterprise Linux …` | `rhel` (si la carga es SAP → `rhel-sap`) |
| `SUSE Linux Enterprise …` | `sles` (SAP → `sles-sap`) |
| `Ubuntu …` | `free` (Ubuntu estándar es gratis; `ubuntu-pro` SOLO si piden Pro) |
| `CentOS` / `Rocky` / `Alma` / `Debian` / `Oracle Linux` / `FreeBSD` / `Photon` / `Other … Linux` | `free` |
| VM o SO menciona `SQL Server` | `sql-server-standard` (preguntar edición: web/standard/enterprise) |

- Oracle Database sobre Linux: la licencia Oracle es BYOL (la calculadora no la cotiza) → `os`
  según el SO real (`rhel`/`free`) y ANOTAR "licencia Oracle no incluida".
- SO vacío o irreconocible → preguntar; si dijeron "usa defaults" → `free` + supuesto anotado.

## 4. Procesadores — descomponer el modelo y elegir serie

Formato típico del host: `2 x Intel(R) Xeon(R) Gold 6248R CPU @ 3.00GHz` o
`AMD EPYC 7543 32-Core Processor`.

- Cores físicos del host = sockets × cores por socket; con Hyper-Threading, hilos = cores × 2.
- **1 vCPU de GCP = 1 hilo (hyper-thread).** Las vCPUs de cada VM ya vienen en `CPUs`; el modelo
  del host NO cambia ese número.
- Ratio de sobresuscripción vCPU:pCPU del cluster: **ignorarlo** — se cotiza lo asignado a la VM.
- La frecuencia (GHz) no se traduce a un campo; solo orienta serie (cargas >3.5 GHz → C/H).
- El modelo del host SÍ orienta la serie GCP equivalente-o-superior:

| Procesador del host | Generación | Serie GCP sugerida |
|---|---|---|
| Xeon E5-26xx v3/v4 (Haswell/Broadwell) o anterior | pre-2017 | `e2` (costo) o `n1` |
| Xeon Scalable 1ª gen — Bronze/Silver/Gold/Platinum **x1xx** (Skylake) | 2017 | `e2` / `n1` / `n2` |
| Xeon Scalable 2ª gen — **x2xx** (Cascade Lake, ej. Gold 6248R) | 2019 | `n2` |
| Xeon Scalable 3ª gen — **x3xx** (Ice Lake) | 2021 | `n2` |
| Xeon Scalable 4ª gen — **x4xx** (Sapphire Rapids) | 2023 | `c3` |
| Xeon Scalable 5ª gen — **x5xx** (Emerald Rapids) o superior | 2024+ | `n4` / `c4` |
| AMD EPYC **7xx2** (Rome) / **7xx3** (Milan) | 2019/2021 | `n2d` (alternativas `t2d`/`c2d`) |
| AMD EPYC **9xx4** (Genoa) | 2023 | `c3d` |
| AMD EPYC **9xx5** (Turin) | 2024+ | `c4d` / `n4d` |

- Sin modelo de host ni preferencia del usuario: default `e2` (menor costo) y anotarlo.
- El dígito clave en Xeon Scalable es el SEGUNDO del modelo (62**4**8 → no; 6**2**48 → 2ª gen).
- Cargas críticas de cómputo/HPC → sugerir `c2`/`c3`/`h3` y preguntar.

## 5. Elegir machineType (con serie decidida)

1. Buscar en `compute-engine.min.json` la máquina MÁS PEQUEÑA de la serie con
   vCPU ≥ `CPUs` de la VM y RAM ≥ `Memory` en GB.
2. Ratio RAM/vCPU de la VM > 8 → `-highmem-` o series `m1`/`m3`/`m4` (memoria extrema).
   Ratio < 2 → `-highcpu-`.
3. Si la predefinida desperdicia mucho (>50% de RAM sobrante), considerar custom:
   `machineType: "<serie>-custom-<vcpus>-<memMiB>"` **con `serie`, `vcpus` y `ramGb` explícitos**
   (solo series que GCP permite: e2, n2, n2d, n4…).
4. Anotar en supuestos TODO redondeo (VM 3 vCPU/6 GB → `e2-standard-4` = 4 vCPU/16 GB).

## 6. Qué preguntar (UNA sola vez, para todo el inventario)

- Región destino y CUD (`none`/`1y`/`3y`).
- ¿Incluir VMs `poweredOff`?
- Serie preferida (solo si no hay modelo de host que la sugiera).
- Discos de datos: ¿sumados al disco de arranque de cada VM, o cotizados aparte como
  `persistent_disk`?
