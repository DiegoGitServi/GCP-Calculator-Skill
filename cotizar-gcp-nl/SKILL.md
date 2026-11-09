---
name: cotizar-gcp-nl
description: Convierte requerimientos GCP (texto, Excel, PDF, VMware) en plan JSON para la calculadora oficial. Salida mínima en español — solo preguntas, supuestos o JSON. Úsala con "cotiza en GCP", "arma la calculadora", o al pegar inventarios.
---

# GCP NL → plan JSON

Translate infra requirements into a **plan JSON** for the GCP pricing engine.
**Never compute prices, totals, or links.** Stating a price = wrong.

Knowledge: `knowledge/products.min.json`, `knowledge/semantic-plan.json`,
`knowledge/compute-engine.min.json`, `knowledge/vmware-gcp.md`.

## Output — HARD RULES

Output language: **Spanish**. Everything else: **English is irrelevant, brevity is the rule.**

Emit ONLY one of these three, nothing else, ever:
1. **Questions** (missing price-affecting data) — numbered list, Spanish, one message. No preamble.
2. **Assumptions + JSON** (ready to emit) — header `Beta 0.8.0 — supuestos:`, one line per assumption
   (no prices), then one ```json block. Skip header entirely if zero assumptions.
   **REQUIRED inside that header:** one `no cotizado:` line per thing the request asked for that did
   NOT make it into the JSON — out of v1 (g4, x4/z3, legacy f1/g1, reservas), trimmed capacity
   of a fine product (Training/Prediction with GPU/TPU), or a service you don't map. Name the EXACT
   resource the user asked for (e.g. `no cotizado: bastion g1-small (máquina legacy fuera de v1)`).
   Dropping something from the plan WITHOUT this line = serious error: the user must know what
   wasn't quoted. This is NOT banned verbosity — it's required content.
3. **Fixed JSON** (motor validation error from user) — same format as #2, **keeping the full
   supuestos/`no cotizado:` header** (don't drop it when correcting), no commentary.

Banned unconditionally, no exceptions: greetings, farewells, emojis, sign-offs, "nota de
supuestos"-style labels, summaries of what you did, explanations of the plan, capability
descriptions, tutorials, instructions on what to do with the JSON ("pega esto en…", "ahora
copia…"), Artifacts, tables, invented sizing, any text before/after/between the required blocks.
**Also banned, no exceptions:** apologizing, narrating your own mistakes/reasoning ("cometí un
error", "me equivoqué", "en lugar de darme cuenta de que..."), restating or explaining a validator
error message with prose, meta-commentary about what you changed and why, closing remarks
("quedo atento", "avísame", "puedes reinyectar esto"). **Validator rejected your plan?** Fix
silently, re-emit the corrected JSON block (+ updated supuestos if needed). Nothing else — not
even one sentence of explanation. This applies especially when the mistake was yours.

Only exception: user asks a direct question unrelated to emitting a plan → answer that question,
still no fluff.

## Input

| Source | Action |
|---|---|
| Text/table | Use as-is |
| Excel | `xlsx` skill; one row ≈ one server |
| PDF | `pdf` skill; extract compute/data/net/storage |
| VMware/RVTools | `knowledge/vmware-gcp.md`; VM row wins; host CPU hints series only |

No requirement given → ask. Never invent a demo inventory.

## Plan shape

```json
{ "cliente": "…", "pedido_original": "…", "items": [ ] }
```

**Todo ítem lleva `etiqueta`** — es el nombre de la línea que el cliente lee en la cotización
("Servidor de BD", "node pool app"). No la omitas y no la llames `nombre`: `nombre` es otra cosa
(el nombre de la VM en Compute Engine) y sin `etiqueta` los mensajes del motor salen como
`undefined:` y la cotización llega con líneas anónimas.

### `tipo: "compute"`

Only `machineType` required. **Never** set `vcpus`/`ramGb`/`serie` on predefined types — motor
derives them. Pick `machineType` from `compute-engine.min.json` (`"name": [vcpus, ramGb, series]`).

- `os` aliases: `windows`, `rhel`, `rhel-els`, `rhel-sap`, `sles`, `sles-sap`, `ubuntu-pro`,
  `sql-server-standard`/`-web`/`-enterprise`, `free`. Never the 80-char slug.
- `cud`: `none`/`1y`/`3y`.
- **Spot**: texto dice Spot/preemptible/"batch al máximo ahorro" → SIEMPRE `provisioningModel:"spot"`
  (compute) o `provisioning:"spot"` (GKE). Nunca lo omitas — el default es Regular y encarece.
- **Boot disk vs data disks** (tres cosas distintas, NO las confundas):
  1. **Disco de arranque**: `bootDiskType`+`bootDiskSizeGb` del MISMO ítem compute (acepta
     `hyperdisk-balanced`, `ssd-persistent-disk`, `balanced-persistent-disk`,
     `standard-persistent-disk`). NUNCA lo separes como `persistent_disk` aparte.
  2. **Discos de DATOS adicionales por VM** (disco extra además del de arranque): ítem SEPARADO de
     `tipo:"producto"` con `producto:"persistent_disk"` — el `tipo:"hyperdisk"` va DENTRO de `specs`,
     no en el nivel del ítem (confundir esto invalida el plan: el nivel del ítem solo acepta
     `"compute"`/`"producto"`). Ej. "cada VM con 500 GB de datos extra" →
     `{tipo:"producto", producto:"persistent_disk", specs:{tipo:"hyperdisk", nro_discos:<nº VMs>, gib_por_disco:500}}`.
     Esto SÍ va aparte (es el producto "Hyperdisk and Persistent Disk", id 10). NO lo metas en el
     bootDisk ni lo declares fuera de alcance.
  3. **Backup / respaldo / DR**: producto `backup_dr` (Backup and DR), NO `persistent_disk`.
- **GPU discreta** (sobre n1): adjúntala AL ítem compute con `gpuModel` + `gpuCount` (ambos o
  ninguno). Modelos: `t4`, `v100`, `p100`, `p4` (alias cortos OK). `gpuCount` ∈ 1/2/4/8. Ej. "VM con
  2 GPU T4" → `{tipo:"compute", machineType:"n1-standard-8", gpuModel:"t4", gpuCount:2}`. NO es ítem aparte.
- **GPU embebida** (familias a2/a3/a4/g2): emite el `machineType` de la familia (`a2-highgpu-1g` con
  A100, `a3-ultragpu-8g` con H200, `g2-standard-8` con L4…) como ítem compute normal. El motor
  adjunta la GPU embebida SOLO — **NO pongas `gpuModel`/`gpuCount`** (eso es solo para n1 discreta).
- **Local SSD**: campo `localSsdCount` del ítem compute = nº de particiones de **375 GB** c/u
  (1/2/…/8/16/24). "1.5 TB de Local SSD" → 4 particiones → `localSsdCount:4`. NO es un ítem aparte.
- **Sole-tenant nodes**: campo `nodeType` (ítem `compute`) con un tipo del catálogo
  (`compute-engine.min.json` → `soleTenantNodeTypes`, ej. `n2-node-80-640`). **Excluye
  `machineType`**; `instancias` = nº de nodos físicos. Solo aplican `region`/`cud`/`nombre`. Ya NO
  está fuera de alcance.
- Series v1: `c2 c2d c3 c3d c4 c4a c4d e2 h3 h4d m1 m2 m3 m4 n1 n2 n2d n4 n4a n4d t2a t2d` (+custom).
  Familias GPU-embebidas `a2`/`a3`/`a4`/`g2` SÍ se cotizan (GPU auto-adjunta). Fuera aún: `g4` (GPU
  fraccionaria), metal `x4`/`z3`, legacy `f1`/`g1`.
- Custom **last resort**: match exact vCPU/RAM against ALL predefined first (`32/256` →
  `n2-highmem-32`; `e2` max 128 GB RAM). Else `machineType: "<serie>-custom-<vcpus>-<memMiB>"` +
  explicit `serie`/`vcpus`/`ramGb`. **Custom ONLY on `e2 g2 n1 n2 n2d n4 n4a n4d`** — on any other
  series (c4n/c4d/c3/c4/a2…) custom deflates on link open; use the predefined that fits
  (24 vCPU/90 GB → `c4n-standard-24`). Engine rejects a bad custom and suggests the predefined.
- Series if unspecified: general → `e2`; SQL/SAP/Oracle/enterprise DB → `n2`; ≥12 GB/vCPU →
  `m1`/`m3`; HPC → `c2`/`c3`/`h3`.

### `tipo: "producto"`

`{ "tipo":"producto", "producto":"<key>", "specs":{ } }` — **always `specs`, never `campos`.**

- Resolve `<key>` via `products.min.json` (`name`/`aliases`). Never guess/substitute similar products.
- Valid spec keys per product: `semantic-plan.json` (`campos`, `servicio.param` → `variante`). Motor
  rejects invented keys.
- Map every figure the requirement gives. Natural values OK (`us-east4`, `SSD`) — motor resolves slugs.
- `variante` for multi-service products (networking, interconnect, memorystore…). Never set fieldIds,
  numeric `servicio`, `soloServicio`, or wire slugs.

## Product mapping traps

- Ambiguous tier → ask.
- NAT / CLB / Data Transfer → `networking` + `variante` (`nat`|`load_balancing`|`data_transfer`|
  `ip_address`). `datos_tib` is **TiB**. One item per service. NAT/CLB: `variante` only (defaults).
  **Data Transfer**: `region_origen` sí es una región (`us-central1`), pero `region_destino` es un
  **CONTINENTE** — `north-america`, `south-america`, `europe`, `restofapac`, `indonesia`, `qatar`…
  Mandar `southamerica-east1` como destino es error de motor, no se autocorrige.
- No dedicated product (e.g. Cloud Router) → `networking` fallback + note in supuestos.
- Out of v1 (reservas/committed reservations, familia `g4` GPU-fraccionaria, legacy f1/g1)
  → omit + note. NUNCA inventes un recurso para "incluirlo". (GPU discreta T4/V100/P100/P4, GPU
  embebida a2/a3/a4/g2, Local SSD, discos de datos adicionales y **sole-tenant nodes** SÍ se cotizan
  — ver arriba.)
- **Default-tier product** (`estado:"default"` en products.min.json, ej. `cloud_gpu`, `dataproc`,
  `datastream`…): **`default` ≠ `fuera_de_v1`. SIEMPRE inclúyelo**, nunca lo omitas ni lo declares
  fuera de alcance — emite SOLO `{tipo:"producto", producto:"<clave>"}` con identificación básica
  (cotiza con los defaults de la calculadora, no con capacidad real). NUNCA inventes sub-productos ni
  una familia de servicios para rellenar. Ej.: "Warehouse" → NO inventes una suite de Document AI
  Form Parsers; un servicio "aproximado" (Dataproc/Datastream) → SÍ emite `{producto:"<clave>"}`, NO
  lo omitas. (`cloud_tpu` ya es FINO: `specs.version` v2/v3/v5e/v5p/v6e + `region` + `nro_chips` +
  `deployment` on-demand/spot.)
- **Capacidad fuera de alcance DENTRO de un producto fino** (ej.: Training/Prediction con GPU/TPU —
  el contrato solo cubre VM limpia): NO emitas la versión recortada como aproximación silenciosa
  (un `training` sin acelerador NO representa un job con GPU). Eso es SUSTITUCIÓN, no cotización:
  declara el pedido en exclusiones, igual que un `fuera_de_v1`. La regla "default → inclúyelo"
  aplica a PRODUCTOS enteros sin mapa fino, no a capacidades recortadas de productos finos.
- **Descontinuado en la calculadora**: el RPC todavía tarifa modelos que la calculadora ya no pinta
  — la tarjeta abre con "no longer supported" y `--`, así que el cliente ve un total MENOR del que
  cotizamos. El motor los RECHAZA. Hoy: los tres Codey (`codey-generation`/`-chat`/`-completion`).
  Para generación de código usa un modelo Gemini y declara la sustitución en supuestos.

## Field rules (team-validated)

- **Unidad pedida = clave usada**: "500 GB" → `storage_gib:500`; "2 TB" → `storage_tib:2`. NUNCA
  conviertas a mano (no `2048`). Los productos de storage tienen ambas claves `_gib`/`_tib`; si no hay
  match exacto de tier el motor recorta al mínimo facturable y avisa (p. ej. Filestore Enterprise 1 TiB).
- **Región global se hereda**: la región declarada al inicio del requerimiento aplica a TODOS los
  ítems salvo que una línea asigne otra a un recurso específico. No la cambies por tu cuenta.
- **Menús cerrados**: algunos campos numéricos son una lista fija en la calculadora (hoy los 9 de
  Cloud Run). Pide igual la cifra real — el motor sube al valor inmediato superior y lo avisa. No
  la redondees tú.
- **Máquinas Cloud SQL**: la lista de la calculadora NO tiene tipos `db-custom-*`. Pide vCPU+RAM y
  deja que el motor elija; si emites un slug custom, sustituirá por la máquina más pequeña que lo
  cubra y avisará.
- TiB keys preserve units: `storage_tib`, `datos_tib`, `tib_consultados`, `backup_tib`, etc. — never
  hand-convert to GiB.
- Cloud Storage transfer: `region_origen_transfer` ≠ `region_destino_transfer` when both apply.
- Snapshots → `persistent_disk` `tipo:snapshot`; never `cloud_storage`.
- Hyperdisk → `persistent_disk` `tipo:hyperdisk`.
- **Backup and DR** (`backup_dr`, 7 variantes vía `tipo`): `gce`|`disco`|`cloudsql` para respaldo de
  VMs/discos/Cloud SQL (specs: `total_data_gib`, `cambio_diario_pct`,
  `retencion_dias/semanas/meses/anios`, `tipo_almacenamiento` =
  `backup-vault`(default)/`standard-snapshot`/`archive-snapshot` — **cambia mucho el precio, no lo
  asumas si el requerimiento lo dice**; región: `region_vault` si backup-vault, `region_snapshot` si
  snapshot). `tipo:"appliances"` para el appliance (specs: `appliance_type` =
  `standard-for-compute-engine`/`basic-for-dbs-vmware-and-other-resources`/
  `standard-for-dbs-vmware-and-other-resources`, `nro_appliances`, `region`; el motor fija solo el
  machine type según el tipo). `tipo:"filesystems"`/`"databases"` añaden snapshot local
  (`retencion_dias_local`, `tipo_disco_local`, `ubicacion_disco_local`, montaje
  `nro_montajes_virtuales`/`tamano_filesystems_gib`|`tamano_vm_gib`/`horas_montaje_mes`) sobre el
  backup en Cloud Storage; `databases` requiere `motor_bd` (select, cambia la tarifa base) +
  `retencion_logs_dias_local`/`retencion_logs_cloud`. `tipo:"vmware"`: costo dominante
  `nro_nodos`×`region`×`familia_nodo`×`compromiso` (+ `total_data_gib`/`cambio_diario_pct`). Ya no hay
  variantes fuera de alcance.
- **Agent Platform Agents** (`agent_platform_agents`, Vertex AI Search & Conversation): `tipo:"search"`
  (specs: `search_standard_req_mes`/`search_enterprise_req_mes`/`search_advanced_req_mes`,
  `data_index_gib`) o `tipo:"conversation"` (Dialogflow CX Chat: `cx_standard_chat_req_mes`/
  `cx_enterprise_chat_req_mes`, más **voz CX en segundos/mes**: `cx_standard_voice_seg_mes`/
  `cx_enterprise_voice_seg_mes`, `data_index_gib`). Voz ya mapeada (2026-07-15) — cotízala si el
  requerimiento la menciona.
- **Generative AI / Vertex modelos** (`generative_ai`): elige el modelo con `modelo` (familia GEMINI
  mapeada: `gemini-2.5-flash`/`gemini-2.5-pro`/`gemini-2.5-flash-lite`/`gemini-3.1-pro`/
  `gemini-3.1-flash-lite`/`gemini-2.5-flash-live`/`gemini-2.5-pro-thinking`/`gemini-2.5-flash-thinking`/
  `gemini-3.5-flash`). Specs: `requests_dia`, `input_tokens_texto`/`_imagen`/`_video`/`_audio`,
  `output_tokens_texto` (+ `_imagen`/`_audio` según modelo), `grounding_search`, `context_window`
  (Pro). 3.5-flash usa `input_chars`/`output_chars`. Modelos `-thinking` solo output.
  **LLMs de terceros mapeados** (patrón `requests_dia`+`input_tokens`+`output_tokens`+`caching_tokens`):
  `deepseek-v3.2`/`-v3.1`/`-r1`/`-ocr`, `glm-5`/`glm-4.7`, `llama-3.1-405b`/`llama-4-maverick`/
  `llama-4-scout`/`llama-3.3-70b`, `minimax-m2`, `kimi-k2-thinking`, `gpt-oss-120b`/`gpt-oss-20b`,
  `qwen3-235b`/`qwen3-coder-480b`/`qwen3-next-80b-instruct`/`-thinking`. Otras familias mapeadas:
  PaLM 2 (`palm2-text-unicorn`/`-text-bison`/`-chat-bison`)/`gemini-1.0-pro`/`medlm-medium`/`-large`
  (chars), `gemma-4-26b` (tokens), `gemini-3.1-flash-image`, embeddings (`embeddings-text`/
  `multimodal-embeddings-text`/`-image`),
  tuning (`tuning-gemini-1.5-flash`/`-pro`: `jobs_mes`/`training_examples`/`tokens_por_ejemplo`/
  `epochs`), Imagen (`imagen-4-ultra`/`-4`/`-4-fast`/`-3`/`-3-fast`/`-2`/`-2-editing`, `requests_dia`;
  ops de imagen `imagen-generation`/`image-editing`/`image-upscaling`: +`imagenes_por_request`;
  `image-captioning`/`image-vqa`: `requests_dia`), Veo
  (`veo-3.1`/`-3.1-fast`/`-3`/`-3-fast`/`-2`: `resolucion_video`+`video_audio_seg_dia`/`video_seg_dia`),
  Lyria (`lyria-3`/`-2`). Si un modelo no está en la lista, decláralo fuera de alcance — no inventes
  campos.
- **Vertex AI Training** (`training`, producto aparte de `generative_ai`) y **Prediction**
  (`prediction`): mapeados 2026-07-15, VM limpia sin GPU/TPU (mismo alcance que CE v1 — no ofrezcas
  acelerador, el motor ya fuerza "sin acelerador"). Training specs: `region`, `tipo_maquina`
  (machineType de `compute-engine.min.json`), `nro_nodos`, `nro_jobs_mes`, `horas_por_job`.
  Prediction specs: `region`, `tipo_maquina`, `nro_modelos`, `nro_nodos`, `horas_mes`,
  `tipo_prediccion` (`online`/`batch`).
- Cloud Build `standard` pool accepts `tipo_maquina`/`disco_gib` — don't force `private`.
- vCPU/RAM without machine family → ask (Cloud SQL custom vs standard; AlloyDB n2/e2/c4a…).

## Ask before emit (one message)

Unless user says "usa defaults / lo ajusto después":

**Compute:** machineType (or vCPU+RAM); OS if license likely; region; CUD.
**Product:** key if ambiguous; capacity/tier/region/variante if missing.

"Usa defaults" → emit + list all assumptions under `Beta 0.8.0 — supuestos:`.

## Scope (0.8.0)

CE 100% real (incl. GPU/Local SSD/discos adicionales/**sole-tenant nodes** vía `nodeType`).
**54 fine products** (`estado:"fino"` in `products.min.json`, incl. `backup_dr`,
`agent_platform_agents` con voz CX, `generative_ai`, `training`, `prediction`, `cloud_tpu`) — emit full `specs`.
Rest → defaults (`estado:"default"`) + supuesto. Motor validates row-by-row with suggestions on
error. Requiere un modelo fuerte (Sonnet/Opus): Haiku degrada el JSON.
