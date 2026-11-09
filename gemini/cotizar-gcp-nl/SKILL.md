---
name: cotizar-gcp-nl
description: Convierte requerimientos GCP (texto, Excel, PDF, VMware) en plan JSON para la calculadora oficial. Salida mínima en español — solo preguntas, supuestos o JSON. Úsala con "cotiza en GCP", "arma la calculadora", o al pegar inventarios.
---

# GCP NL → plan JSON

Translate infra requirements into a **plan JSON** for the GCP pricing engine.
**Never compute prices, totals, or links.** Stating a price = wrong.

Knowledge: `knowledge/products.min.txt`, `knowledge/semantic-plan.txt`,
`knowledge/compute-engine.min.txt`, `knowledge/vmware-gcp.md`.

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
derives them. Pick `machineType` from `compute-engine.min.txt` (`"name": [vcpus, ramGb, series]`).

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
  (`compute-engine.min.txt` → `soleTenantNodeTypes`, ej. `n2-node-80-640`). **Excluye
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

- Resolve `<key>` via `products.min.txt` (`name`/`aliases`). Never guess/substitute similar products.
- Valid spec keys per product: `semantic-plan.txt` (`campos`, `servicio.param` → `variante`). Motor
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
- **Default-tier product** (`estado:"default"` en products.min.txt, ej. `cloud_gpu`, `dataproc`,
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
  (machineType de `compute-engine.min.txt`), `nro_nodos`, `nro_jobs_mes`, `horas_por_job`.
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
**54 fine products** (`estado:"fino"` in `products.min.txt`, incl. `backup_dr`,
`agent_platform_agents` con voz CX, `generative_ai`, `training`, `prediction`, `cloud_tpu`) — emit full `specs`.
Rest → defaults (`estado:"default"`) + supuesto. Motor validates row-by-row with suggestions on
error. Requiere un modelo fuerte (Sonnet/Opus): Haiku degrada el JSON.

## SPECS VÁLIDOS POR PRODUCTO (OBLIGATORIO — copia estos nombres EXACTOS)

Los nombres de `specs` NO se inventan ni se traducen: usa solo los de esta lista. El motor rechaza
(`Plan inválido`) cualquier otro. Si falta un dato, omite el spec (usa default) y anótalo en supuestos.
Unidades van en el nombre (`_gib`, `_tib`, `_millones`, `_ms`): convierte antes de emitir
(ej. 3600 requests = `requests_millones`: 0.0036; 10k operaciones = `ops_clase_a_millones`: 0.01).
Productos con "variante" → emite `"variante": "<nombre>"` DENTRO de `specs` (nunca como campo del ítem),
junto con solo los specs de esa variante. Ej.:
`"specs": {"variante":"nat","tipo_nat":"public-nat","vms_asignadas":1,"datos_procesados_gib":10,"ips_usadas":1}`.

- `kubernetes_engine`: nodos, tipo_maquina, disco_tipo, disco_gib, region, cud, sistema_operativo, provisioning, edicion
- `alloydb`: region, serie_maquina, vcpu_primario, alta_disponibilidad, read_pool_n2, read_pool_n2_vcpu, read_pool_c4a, read_pool_c4a_vcpu, read_pool_c4, read_pool_c4_vcpu, storage_gib, storage_tib, backup_multiplo, cud
- `memorystore`: variante "redis" → region, tier, capacidad_gib, replicas, horas_mes, cud | variante "cluster" → region, node_type, shards, replicas_por_shard, horas_mes | variante "memcached" → region, instancias, nodos_por_instancia, vcpu_por_nodo, memoria_por_nodo_gib, horas_mes | variante "valkey" → region, node_type, shards, replicas_por_shard, horas_mes
- `cloud_vpn`: region, tuneles
- `cloud_interconnect`: variante "partner" → vlan_50mbps, vlan_100mbps, vlan_200mbps, vlan_300mbps, vlan_400mbps, vlan_500mbps, vlan_1gbps, vlan_2gbps, vlan_5gbps, vlan_10gbps, vlan_20gbps, vlan_50gbps, datos_tib, ubicacion | variante "dedicated" → circuitos_10g, circuitos_100g, vlan_dedicated_hasta_10gbps, vlan_dedicated_hasta_20gbps, vlan_dedicated_hasta_50gbps, datos_tib, ubicacion
- `networking`: variante "ip_address" → region, ips_sin_uso, ips_en_uso, ips_en_uso_spot | variante "data_transfer" → tier_red, datos_tib, region_origen, region_destino | variante "nat" → tipo_nat, vms_asignadas, datos_procesados_gib, ips_usadas | variante "load_balancing" → alcance, region, reglas_reenvio, reglas_reenvio_global, datos_entrada_gib, datos_salida_gib, proxy_region, proxy_instancias, proxy_datos_gib
- `cloud_storage`: tipo_ubicacion, region, clase, storage_gib, storage_tib, ops_clase_a_millones, ops_clase_b_millones, transfer_interno_gib, transfer_interno_tib, region_origen_transfer, region_destino_transfer
- `artifact_registry`: storage_gib, transfer_gb, transfer_origen_destino
- `cloud_armor`: recursos_protegidos, inspeccion_clb_gib, inspeccion_nlb_gib, tier, requests_global_millones, requests_regional_millones, policies, rules
- `secret_manager`: versiones_activas, ops_acceso, notificaciones_rotacion
- `cloud_kms`: nivel_proteccion, versiones_aes256, versiones_hmac, versiones_rsa_ec, ops_aes256, ops_hmac, ops_rsa_ec, versiones_hmac_hsm, versiones_aes256_hsm, versiones_rsa2048_hsm, versiones_rsa3072_hsm, versiones_rsa4096_hsm, versiones_ecp256_hsm, versiones_ecp384_hsm, ops_aes256_hsm, ops_hmac_hsm, ops_rsa2048_hsm, ops_rsa3072_hsm, ops_rsa4096_hsm, ops_ecp256_hsm, ops_pkcs1_hsm, ops_auto_key_hsm, versiones_simetricas_external, versiones_asimetricas_external, ops_simetricas_external, ops_asimetricas_external, fee_hsm_single_tenant, fee_cse_key_service
- `cloud_dns`: consultas_millones, consultas_routing_millones, zonas
- `cloud_logging`: storage_gib, retencion_meses
- `cloud_monitoring`: variante "monitoring" → conoce_volumen_mib, volumen_mib, num_metricas, datapoints_por_hora, num_recursos | variante "prometheus" → conoce_samples, samples_por_segundo, series_unicas, intervalo_muestreo | variante "trace" → trace_spans, uptime_checks, nro_regiones, frecuencia
- `cloud_build`: region, tipo_pool, tipo_maquina, build_minutos, disco_gib
- `cloud_sql`: variante "mysql" → region, edicion, tipo_maquina, instancias, alta_disponibilidad, storage_gib, storage_type, backup_gib, cud | variante "postgres" → region, edicion, tipo_maquina, instancias, alta_disponibilidad, storage_gib, storage_type, backup_gib, cud | variante "sqlserver" → region, edicion, tipo_maquina, instancias, alta_disponibilidad, storage_gib, storage_type, backup_gib, cud
- `persistent_disk`: variante "hyperdisk" → region, nro_discos, gib_por_disco, iops_provisionadas, throughput_mbps, nro_discos_throughput, gib_por_disco_throughput, throughput_mbps_throughput, nro_discos_extreme, gib_por_disco_extreme, iops_extreme, nro_discos_ml, gib_por_disco_ml, throughput_mbps_ml, nro_discos_ha, gib_por_disco_ha, iops_ha, throughput_mbps_ha | variante "snapshot" → region, snapshot_storage_gib, snapshot_multiregional_gib | variante "persistent" → region, snapshot_storage_gib, snapshot_multiregional_gib
- `backup_dr`: variante "gce" → total_data_gib, cambio_diario_pct, retencion_dias, retencion_semanas, retencion_meses, retencion_anios, tipo_almacenamiento, ubicacion_tipo, region_vault, region_snapshot | variante "appliances" → region, appliance_type, nro_appliances | variante "disco" → total_data_gib, cambio_diario_pct, retencion_dias, retencion_semanas, retencion_meses, retencion_anios, tipo_almacenamiento, ubicacion_tipo, region_snapshot, region_vault | variante "cloudsql" → total_data_gib, cambio_diario_pct, retencion_dias, retencion_semanas, retencion_meses, retencion_anios, tipo_almacenamiento, ubicacion_tipo, region_snapshot, region_vault | variante "filesystems" → total_data_gib, cambio_diario_pct, retencion_dias_local, tipo_disco_local, ubicacion_disco_local, retencion_dias, retencion_semanas, retencion_meses, retencion_anios, tipo_almacenamiento, ubicacion_tipo, region_snapshot, nro_montajes_virtuales, tamano_filesystems_gib, horas_montaje_mes | variante "databases" → motor_bd, total_data_gib, cambio_diario_pct, retencion_dias_local, retencion_logs_dias_local, tipo_disco_local, ubicacion_disco_local, retencion_dias, retencion_semanas, retencion_meses, retencion_anios, retencion_logs_cloud, tipo_almacenamiento, ubicacion_tipo, region_snapshot, cross_regional, nro_montajes_virtuales, tamano_vm_gib, horas_montaje_mes | variante "vmware" → total_data_gib, cambio_diario_pct, nro_nodos, region, familia_nodo, compromiso
- `agent_platform`: region, nro_vms, tipo_disco, disco_gib, horas_mes
- `gemini_code_assist`: edicion, asientos, compromiso
- `agent_platform_agents`: variante "search" → search_standard_req_mes, search_enterprise_req_mes, search_advanced_req_mes, data_index_gib | variante "conversation" → cx_standard_chat_req_mes, cx_enterprise_chat_req_mes, cx_standard_voice_seg_mes, cx_enterprise_voice_seg_mes, data_index_gib
- `datastream`: region, cdc_volumen_gib_mes, backfill_volumen_gib_mes
- `text_to_speech`: tipo_voz, caracteres_millones_mes
- `cloud_vision`: label_detection_miles_img, text_detection_miles_img, safe_search_miles_img, facial_detection_miles_img, facial_celebrity_miles_img, landmark_detection_miles_img, logo_detection_miles_img, image_properties_miles_img, web_detection_miles_img, document_text_detection_miles_img, crop_hints_miles_img, object_localization_miles_img
- `firebase_app_hosting`: region, requests_miles_mes, cpu_por_instancia, memoria_gib_por_instancia, tipo_asignacion_cpu, tiempo_ejecucion_ms, concurrencia_por_instancia, nro_recursos, instancia_horas_mes, bandwidth_sin_cache_gib, bandwidth_con_cache_gib, build_pool_tipo, build_machine_tipo, build_minutos_mes, artifact_storage_gib, logging_retencion_meses, logging_storage_gib, secret_versiones_activas, secret_operaciones_acceso, secret_rotaciones
- `application_integration`: ejecuciones_mes, data_integraciones_gib_mes, nodos_conexion_google, nodos_conexion_terceros, data_nodos_conexion_gib_mes
- `support_services`: service_level, gcp_spend, workspace_spend, tam_level, nro_tams
- `generative_ai`: variante "gemini-3.5-flash" → requests_dia, input_chars, output_chars, input_imagenes, input_video_seg, input_audio_seg, grounding_search | variante "gemini-3.1-pro" → context_window, requests_dia, input_tokens_texto, input_tokens_imagen, input_tokens_video, input_tokens_audio, output_tokens_texto, output_tokens_imagen, grounding_search | variante "gemini-3.1-flash-lite" → requests_dia, input_tokens_texto, input_tokens_imagen, input_tokens_video, input_tokens_audio, output_tokens_texto, grounding_search | variante "gemini-2.5-pro" → context_window, requests_dia, input_tokens_texto, input_tokens_imagen, input_tokens_video, input_tokens_audio, output_tokens_texto, grounding_search | variante "gemini-2.5-flash" → requests_dia, input_tokens_texto, input_tokens_imagen, input_tokens_video, input_tokens_audio, output_tokens_texto, output_tokens_imagen, grounding_search | variante "gemini-2.5-flash-lite" → requests_dia, input_tokens_texto, input_tokens_imagen, input_tokens_video, input_tokens_audio, output_tokens_texto, grounding_search | variante "gemini-2.5-flash-live" → requests_dia, input_tokens_texto, input_tokens_imagen, input_tokens_video, input_tokens_audio, output_tokens_texto, output_tokens_audio | variante "gemini-2.5-pro-thinking" → context_window, requests_dia, output_tokens_texto | variante "gemini-2.5-flash-thinking" → context_window, requests_dia, output_tokens_request | variante "deepseek-v3.2" → requests_dia, input_tokens, output_tokens, caching_tokens | variante "deepseek-v3.1" → requests_dia, input_tokens, output_tokens, caching_tokens | variante "deepseek-r1" → requests_dia, input_tokens, output_tokens | variante "deepseek-ocr" → requests_dia, input_tokens, output_tokens | variante "glm-5" → requests_dia, input_tokens, output_tokens, caching_tokens | variante "glm-4.7" → requests_dia, input_tokens, output_tokens, caching_tokens | variante "llama-3.1-405b" → requests_dia, input_tokens, output_tokens | variante "llama-4-maverick" → requests_dia, input_tokens, output_tokens | variante "llama-4-scout" → requests_dia, input_tokens, output_tokens | variante "llama-3.3-70b" → requests_dia, input_tokens, output_tokens | variante "minimax-m2" → requests_dia, input_tokens, output_tokens, caching_tokens | variante "kimi-k2-thinking" → requests_dia, input_tokens, output_tokens, caching_tokens | variante "gpt-oss-120b" → requests_dia, input_tokens, output_tokens | variante "gpt-oss-20b" → requests_dia, input_tokens, output_tokens, caching_tokens | variante "qwen3-235b" → requests_dia, input_tokens, output_tokens | variante "qwen3-coder-480b" → requests_dia, input_tokens, output_tokens, caching_tokens | variante "qwen3-next-80b-instruct" → requests_dia, input_tokens, output_tokens | variante "qwen3-next-80b-thinking" → requests_dia, input_tokens, output_tokens | variante "gemini-1.0-pro" → requests_dia, input_chars, output_chars | variante "palm2-text-unicorn" → requests_dia, input_chars, output_chars | variante "palm2-text-bison" → requests_dia, input_chars, output_chars | variante "palm2-chat-bison" → requests_dia, input_chars, output_chars | variante "medlm-medium" → requests_dia, input_chars, output_chars | variante "medlm-large" → requests_dia, input_chars, output_chars | variante "gemma-4-26b" → requests_dia, input_tokens, output_tokens, caching_tokens | variante "gemini-3.1-flash-image" → requests_dia, input_tokens_texto, input_tokens_imagen, output_tokens_texto, output_tokens_imagen, grounding_search | variante "embeddings-text" → requests_dia, input_chars | variante "multimodal-embeddings-text" → requests_dia, input_chars | variante "multimodal-embeddings-image" → requests_dia | variante "codey-generation" → requests_dia, input_chars, output_chars | variante "codey-chat" → requests_dia, input_chars, output_chars | variante "codey-completion" → requests_dia, input_chars, output_chars | variante "tuning-gemini-1.5-flash" → jobs_mes, training_examples, tokens_por_ejemplo, epochs | variante "tuning-gemini-1.5-pro" → jobs_mes, training_examples, tokens_por_ejemplo, epochs | variante "imagen-4-ultra" → requests_dia | variante "imagen-4" → requests_dia | variante "imagen-4-fast" → requests_dia | variante "imagen-3" → requests_dia | variante "imagen-3-fast" → requests_dia | variante "imagen-2" → requests_dia | variante "imagen-2-editing" → requests_dia | variante "imagen-generation" → requests_dia, imagenes_por_request | variante "image-editing" → requests_dia, imagenes_por_request | variante "image-upscaling" → requests_dia, imagenes_por_request | variante "image-captioning" → requests_dia | variante "image-vqa" → requests_dia | variante "veo-3.1" → resolucion_video, video_audio_seg_dia, video_seg_dia | variante "veo-3.1-fast" → resolucion_video, video_audio_seg_dia, video_seg_dia | variante "veo-3" → resolucion_video, video_audio_seg_dia, video_seg_dia | variante "veo-3-fast" → resolucion_video, video_audio_seg_dia, video_seg_dia | variante "veo-2" → video_seg_dia, advanced_controls_seg_dia | variante "lyria-3" → requests_dia, output_audio, output_clip_audio | variante "lyria-2" → requests_dia, music_generation
- `training`: region, tipo_maquina, nro_nodos, nro_jobs_mes, horas_por_job, acelerador_tipo, acelerador_modelo, acelerador_nro_cores_tpu
- `prediction`: region, tipo_maquina, nro_modelos, nro_nodos, horas_mes, tipo_prediccion, acelerador_tipo, acelerador_modelo, acelerador_nro_por_nodo, acelerador_nro_cores_tpu
- `cloud_tpu`: version, region, nro_chips, deployment
- `bigquery`: variante "ondemand" → region, tib_consultados, tib_storage | variante "editions" → region, edition, max_slots, max_slots_custom, baseline_slots, commitment, slot_commitments, utilizacion_autoscale_pct, storage_activo_tib, storage_largo_plazo_tib, storage_fisico_activo_gib, storage_fisico_largo_plazo_gib, streaming_inserts_gib, streaming_reads_tib, storage_write_api_gib
- `composer`: region, tamano, horas_mes, nro_schedulers, vcpu_scheduler, memoria_scheduler_gib, storage_scheduler_gib, nro_workers, vcpu_worker, memoria_worker_gib, storage_worker_gib, vcpu_webserver, memoria_webserver_gib, storage_webserver_gib, nro_triggerers, vcpu_triggerer, memoria_triggerer_gib, storage_triggerer_gib, db_storage_gib
- `cloud_spanner`: edicion, configuracion, location, unidad_computo, nro_nodos, processing_units, ssd_storage_gib, backup_gib, cud
- `firestore`: tipo_ubicacion, region, lecturas_mes, escrituras_mes, deletes_mes, lecturas_dia, escrituras_dia, deletes_dia, storage_gib
- `cloud_bigtable`: region, nro_nodos, tipo_storage, storage_tib, storage_gib, backup_tib, backup_gib
- `cloud_cdn`: cache_transfer_gib, cache_lookup_millones, egress_asia_pacifico_gib, egress_china_gib, egress_europa_gib, egress_norteamerica_gib, egress_oceania_gib, egress_sudamerica_gib, egress_otros_destinos_gib, cache_fill_regional_gib, cache_fill_interregion_gib
- `cloud_pubsub`: datos_publicados_gib, tipo_entrega, topic_retention_dias, nro_suscripciones, nro_snapshots
- `cloud_dataflow`: tipo, region, job_horas_mes, nro_workers, worker_machine_type, streaming_engine, storage_gib, storage_tipo
- `app_engine`: variante "standard" → region, instance_class, transfer_gib, nro_instancias | variante "flexible" → region, vcpu_por_hora, memoria_gib_por_hora, disco_gib, transfer_gib
- `cloud_run`: region, use_case, min_instancias, requests_millones, tiempo_ejecucion_ms, concurrencia, vcpu, memoria_gib, egress_gib, nro_recursos, gpu_modelo, gpu_count
- `apigee`: tipo_entorno, nro_environments, nro_regiones, proxy_deployments, api_calls_estandar_millones, api_calls_extensibles_millones, analytics_addon, api_security_addon
- `filestore_advanced`: region, nro_instancias, tier, capacidad_tib, capacidad_gib, capacity_band, agregar_backup, backup_storage_gib, region_backup
- `document_ai`: variante "general" → ocr_millares, form_parser_millares, otros_millares | variante "identity" → fr_licencia_conduccion_millares, fr_documento_identidad_millares, fr_pasaporte_millares, us_licencia_conduccion_millares, us_pasaporte_millares | variante "lending" → formulario_1040_millares, formulario_1120_millares, formulario_1120s_millares, extracto_bancario_millares, extracto_hipoteca_millares, volante_pago_millares, formulario_ssa1099_millares, formulario_w2_millares, formulario_w9_millares, clasificador_documentos_prestamo_millares | variante "procurement" → factura_millares, recibo_gasto_millares, factura_servicios_millares, clasificador_documentos_compras_millares | variante "workbench" → custom_extractor_millares, splitter_millares, classifier_millares | variante "warehouse" → documentos_indexados_millares, api_calls_millares
- `dataproc`: variante "serverless" → region, horas_mes, interactive, vcpus, memoria_por_vcpu_gib, shuffle_storage_gib_por_vcpu, utilizacion_pct | variante "gce_gke" → region, nro_nodos, horas_mes
- `speech_to_text`: minutos, modelo, registro_datos, lote_dinamico
- `cloud_gpu`: region, modelo_gpu, numero_maquinas, tipo_maquina, cud
- `vmware_engine`: region, nro_nodos, familia_nodo, tipo_nodo, tipo_maquina, cud, protegido
- `bare_metal`: region, configuracion_servidor, plazo_compromiso, tipo_almacenamiento_1, cantidad_almacenamiento_1, tipo_almacenamiento_2, cantidad_almacenamiento_2, tipo_almacenamiento_3, cantidad_almacenamiento_3, interconnect_gbps, soporte_sap_hana
- `kafka`: region, ancho_banda_productor_mib_s, ancho_banda_consumidor_mib_s, retencion_dias, utilizacion_vcpu_pct, particiones_totales, num_replicas, ram_gib_por_vcpu, usar_replica_local, cud
- `managed_lustre`: region, nro_instancias, tier_rendimiento, tamano_gib, duracion_meses
- `cloud_run_functions`: version, region, memoria, solicitudes_millones_mes, duracion_promedio_ms, transferencia_gib_por_solicitud, instancias_minimas, concurrencia_por_instancia
- `media_cdn`: egress_asia_pacifico_gib, egress_china_corea_gib, egress_europa_gib, egress_norteamerica_gib, egress_oceania_gib, egress_latam_gib, egress_india_gib, egress_medio_oriente_gib, egress_otros_destinos_gib, solicitudes_http_millones_mes
- `dns_armor`: cargas_activas_por_hora, consultas_externas_por_hora
- `gemini_enterprise`: variante "business" → asientos | variante "standard" → asientos, storage_gib, agent_platform_compute, agent_platform_storage, agent_platform_memory, org_armor_tokens, armor_tokens_premium, armor_tokens_subscription, image_generation_imagen3, image_generation_imagen4, video_generation_veo2, video_generation_veo3_fast, grounding_gemini25pro_google_search, grounding_1p3p_data, grounding_google_search, grounding_enterprise_web, grounding_large_text_embedding, g36_flash_text_input_global_tokens, g36_flash_text_input_caching_global, g36_flash_text_output_global_tokens, g36_flash_audio_input_global_tokens, g36_flash_audio_input_caching_global, g36_flash_image_input_global_tokens, g36_flash_image_input_caching_global, g36_flash_video_input_global_tokens, g36_flash_video_input_caching_global, g36_flash_text_input_regional_tokens, g36_flash_text_input_caching_regional, g36_flash_text_output_regional_tokens, g36_flash_audio_input_regional_tokens, g36_flash_audio_input_caching_regional, g36_flash_image_input_regional_tokens, g36_flash_image_input_caching_regional, g36_flash_video_input_regional_tokens, g36_flash_video_input_caching_regional, g35_flash_text_input_global_tokens, g35_flash_text_input_caching_global, g35_flash_text_output_global_tokens, g35_flash_audio_input_global_tokens, g35_flash_audio_caching_global, g35_flash_image_input_global_tokens, g35_flash_image_caching_global, g35_flash_video_input_global_tokens, g35_flash_text_input_regional_tokens, g35_flash_text_input_caching_regional, g35_flash_text_output_regional_tokens, g35_flash_audio_input_regional_tokens, g35_flash_audio_caching_regional, g35_flash_image_input_regional_tokens, g35_flash_image_caching_regional, g35_flash_video_input_regional_tokens, g35_flash_video_caching_regional, g35_flash_lite_text_input_global_tokens, g35_flash_lite_text_input_caching_global, g35_flash_lite_text_output_global_tokens, g35_flash_lite_audio_input_global_tokens, g35_flash_lite_audio_input_caching_global, g35_flash_lite_image_input_global_tokens, g35_flash_lite_image_input_caching_global, g35_flash_lite_video_input_global_tokens, g35_flash_lite_video_input_caching_global, g35_flash_lite_text_input_regional_tokens, g35_flash_lite_text_input_caching_regional, g35_flash_lite_text_output_regional_tokens, g35_flash_lite_audio_input_regional_tokens, g35_flash_lite_audio_input_caching_regional, g35_flash_lite_image_input_regional_tokens, g35_flash_lite_image_input_caching_regional, g35_flash_lite_video_input_regional_tokens, g35_flash_lite_video_input_caching_regional, g31_flash_image_text_input_global_tokens, g31_flash_image_text_output_global_tokens, g31_flash_image_image_input_images, g31_flash_image_image_input_tokens, g31_flash_image_image_caching, g31_flash_image_image_output_images, g31_flash_lite_text_input_global_tokens, g31_flash_lite_text_input_regional_tokens, g31_flash_lite_text_caching_global, g31_flash_lite_text_caching_regional, g31_flash_lite_text_output_global_tokens, g31_flash_lite_text_output_regional_tokens, g31_flash_lite_image_input_images, g30_pro_text_input_tokens, g30_pro_text_caching, g30_pro_text_output_tokens, g30_pro_audio_input_audio_seconds, g30_pro_audio_caching, g30_pro_image_input_images, g30_pro_image_caching, g30_pro_image_output_images, g30_pro_video_input_video_seconds, g30_pro_video_caching, g30_pro_text_input_long_tokens, g30_pro_text_caching_long, g30_pro_text_output_long_tokens, g30_pro_audio_input_long_audio_seconds, g30_pro_audio_caching_long, g30_pro_image_input_long_images, g30_pro_image_caching_long, g30_pro_image_output_long_images, g30_pro_video_input_long_video_seconds, g30_pro_video_caching_long, g30_flash_text_input_tokens, g30_flash_text_caching, g30_flash_text_output_tokens, g30_flash_audio_input_tokens, g30_flash_audio_caching, g30_flash_video_input_tokens, g30_flash_video_caching, g25_pro_text_input_tokens, g25_pro_text_caching, g25_pro_text_output_tokens, g25_pro_thinking_text_output_tokens, g25_pro_audio_input_audio_seconds, g25_pro_audio_caching, g25_pro_image_input_images, g25_pro_image_caching, g25_pro_video_input_video_seconds, g25_pro_video_caching, g25_pro_text_input_long_tokens, g25_pro_text_caching_long, g25_pro_text_output_long_tokens, g25_pro_thinking_text_output_long_tokens, g25_pro_audio_input_long_audio_seconds, g25_pro_audio_caching_long, g25_pro_image_input_long_images, g25_pro_image_caching_long, g25_pro_video_input_long_video_seconds, g25_pro_video_caching_long, g25_flash_text_input_tokens, g25_flash_live_text_input_tokens, g25_flash_text_caching, g25_flash_text_output_tokens, g25_flash_live_text_output_tokens, g25_flash_text_output_thinking_on_tokens, g25_flash_thinking_text_output_tokens, g25_flash_audio_input_ga_tokens, g25_flash_live_audio_input_audio_seconds, g25_flash_audio_caching, g25_flash_live_audio_output_av2a_audio_seconds, g25_flash_live_audio_output_audio_seconds, g25_flash_image_input_images, g25_flash_image_caching_ga, g25_flash_image_output_images, g25_flash_video_input_ga_tokens, g25_flash_video_caching_ga, g25_flash_text_input_long_tokens, g25_flash_text_caching_long, g25_flash_text_output_long_tokens, g25_flash_text_output_thinking_on_long_tokens, g25_flash_thinking_text_output_long_tokens, g25_flash_audio_input_long_tokens, g25_flash_audio_caching_long, g25_flash_image_input_long_images, g25_flash_image_caching_long, g25_flash_video_input_long_tokens, g25_flash_video_caching_long, g25_flash_lite_text_input_tokens, g25_flash_lite_text_caching, g25_flash_lite_text_output_tokens, g25_flash_lite_thinking_text_output_tokens, g25_flash_lite_image_input_images | variante "plus" → asientos, storage_gib, agent_platform_compute, agent_platform_storage, agent_platform_memory, org_armor_tokens, armor_tokens_premium, armor_tokens_subscription, image_generation_imagen3, image_generation_imagen4, video_generation_veo2, video_generation_veo3_fast, grounding_gemini25pro_google_search, grounding_1p3p_data, grounding_google_search, grounding_enterprise_web, grounding_large_text_embedding, g36_flash_text_input_global_tokens, g36_flash_text_input_caching_global, g36_flash_text_output_global_tokens, g36_flash_audio_input_global_tokens, g36_flash_audio_input_caching_global, g36_flash_image_input_global_tokens, g36_flash_image_input_caching_global, g36_flash_video_input_global_tokens, g36_flash_video_input_caching_global, g36_flash_text_input_regional_tokens, g36_flash_text_input_caching_regional, g36_flash_text_output_regional_tokens, g36_flash_audio_input_regional_tokens, g36_flash_audio_input_caching_regional, g36_flash_image_input_regional_tokens, g36_flash_image_input_caching_regional, g36_flash_video_input_regional_tokens, g36_flash_video_input_caching_regional, g35_flash_text_input_global_tokens, g35_flash_text_input_caching_global, g35_flash_text_output_global_tokens, g35_flash_audio_input_global_tokens, g35_flash_audio_caching_global, g35_flash_image_input_global_tokens, g35_flash_image_caching_global, g35_flash_video_input_global_tokens, g35_flash_text_input_regional_tokens, g35_flash_text_input_caching_regional, g35_flash_text_output_regional_tokens, g35_flash_audio_input_regional_tokens, g35_flash_audio_caching_regional, g35_flash_image_input_regional_tokens, g35_flash_image_caching_regional, g35_flash_video_input_regional_tokens, g35_flash_video_caching_regional, g35_flash_lite_text_input_global_tokens, g35_flash_lite_text_input_caching_global, g35_flash_lite_text_output_global_tokens, g35_flash_lite_audio_input_global_tokens, g35_flash_lite_audio_input_caching_global, g35_flash_lite_image_input_global_tokens, g35_flash_lite_image_input_caching_global, g35_flash_lite_video_input_global_tokens, g35_flash_lite_video_input_caching_global, g35_flash_lite_text_input_regional_tokens, g35_flash_lite_text_input_caching_regional, g35_flash_lite_text_output_regional_tokens, g35_flash_lite_audio_input_regional_tokens, g35_flash_lite_audio_input_caching_regional, g35_flash_lite_image_input_regional_tokens, g35_flash_lite_image_input_caching_regional, g35_flash_lite_video_input_regional_tokens, g35_flash_lite_video_input_caching_regional, g31_flash_image_text_input_global_tokens, g31_flash_image_text_output_global_tokens, g31_flash_image_image_input_images, g31_flash_image_image_input_tokens, g31_flash_image_image_caching, g31_flash_image_image_output_images, g31_flash_lite_text_input_global_tokens, g31_flash_lite_text_input_regional_tokens, g31_flash_lite_text_caching_global, g31_flash_lite_text_caching_regional, g31_flash_lite_text_output_global_tokens, g31_flash_lite_text_output_regional_tokens, g31_flash_lite_image_input_images, g30_pro_text_input_tokens, g30_pro_text_caching, g30_pro_text_output_tokens, g30_pro_audio_input_audio_seconds, g30_pro_audio_caching, g30_pro_image_input_images, g30_pro_image_caching, g30_pro_image_output_images, g30_pro_video_input_video_seconds, g30_pro_video_caching, g30_pro_text_input_long_tokens, g30_pro_text_caching_long, g30_pro_text_output_long_tokens, g30_pro_audio_input_long_audio_seconds, g30_pro_audio_caching_long, g30_pro_image_input_long_images, g30_pro_image_caching_long, g30_pro_image_output_long_images, g30_pro_video_input_long_video_seconds, g30_pro_video_caching_long, g30_flash_text_input_tokens, g30_flash_text_caching, g30_flash_text_output_tokens, g30_flash_audio_input_tokens, g30_flash_audio_caching, g30_flash_video_input_tokens, g30_flash_video_caching, g25_pro_text_input_tokens, g25_pro_text_caching, g25_pro_text_output_tokens, g25_pro_thinking_text_output_tokens, g25_pro_audio_input_audio_seconds, g25_pro_audio_caching, g25_pro_image_input_images, g25_pro_image_caching, g25_pro_video_input_video_seconds, g25_pro_video_caching, g25_pro_text_input_long_tokens, g25_pro_text_caching_long, g25_pro_text_output_long_tokens, g25_pro_thinking_text_output_long_tokens, g25_pro_audio_input_long_audio_seconds, g25_pro_audio_caching_long, g25_pro_image_input_long_images, g25_pro_image_caching_long, g25_pro_video_input_long_video_seconds, g25_pro_video_caching_long, g25_flash_text_input_tokens, g25_flash_live_text_input_tokens, g25_flash_text_caching, g25_flash_text_output_tokens, g25_flash_live_text_output_tokens, g25_flash_text_output_thinking_on_tokens, g25_flash_thinking_text_output_tokens, g25_flash_audio_input_ga_tokens, g25_flash_live_audio_input_audio_seconds, g25_flash_audio_caching, g25_flash_live_audio_output_av2a_audio_seconds, g25_flash_live_audio_output_audio_seconds, g25_flash_image_input_images, g25_flash_image_caching_ga, g25_flash_image_output_images, g25_flash_video_input_ga_tokens, g25_flash_video_caching_ga, g25_flash_text_input_long_tokens, g25_flash_text_caching_long, g25_flash_text_output_long_tokens, g25_flash_text_output_thinking_on_long_tokens, g25_flash_thinking_text_output_long_tokens, g25_flash_audio_input_long_tokens, g25_flash_audio_caching_long, g25_flash_image_input_long_images, g25_flash_image_caching_long, g25_flash_video_input_long_tokens, g25_flash_video_caching_long, g25_flash_lite_text_input_tokens, g25_flash_lite_text_caching, g25_flash_lite_text_output_tokens, g25_flash_lite_thinking_text_output_tokens, g25_flash_lite_image_input_images
- `recaptcha`: assessments_mes
- `workload_manager`: evaluaciones_recursos_mes
- `parameter_manager`: llamadas_acceso_mes, versiones_almacenadas
- `video_intelligence`: etiquetas_min, tomas_min, contenido_explicito_min, transcripcion_min, seguimiento_objetos_min, texto_min, logos_min, rostros_min, personas_min, celebridades_min, etiquetas_streaming_min, tomas_streaming_min, contenido_explicito_streaming_min, seguimiento_objetos_streaming_min
- `cloud_translation`: variante "basic" → caracteres_traduccion_mes, caracteres_deteccion_mes, paginas_documentos_mes | variante "advanced" → caracteres_deteccion_mes, caracteres_traduccion_mes, caracteres_traduccion_lote_mes, caracteres_xlsx_mes, caracteres_xlsx_lote_mes, caracteres_romanizar_mes, paginas_documentos_mes
- `maps_platform`: variante "maps" → mapas_dinamicos_mes, street_view_dinamico_mes, mapas_inmersivos_mes, street_view_estatico_mes, mapas_estaticos_mes, aerial_view_mes, tiles_2d_mes, tiles_street_view_mes, tiles_3d_fotorrealistas_mes, elevacion_mes | variante "routes" → navigation_sdk_mes, optimizacion_un_vehiculo_mes, optimizacion_flota_mes, roads_carretera_cercana_mes, roads_ruta_recorrida_mes, matriz_rutas_essentials_mes, matriz_rutas_pro_mes, matriz_rutas_enterprise_mes, rutas_essentials_mes, rutas_pro_mes, rutas_enterprise_mes | variante "places" → validacion_direcciones_pro_mes, validacion_direcciones_enterprise_mes, autocomplete_mes, geocoding_mes, geolocalizacion_mes, nearby_search_pro_mes, nearby_search_enterprise_mes, nearby_search_enterprise_atmosphere_mes, place_details_essentials_mes, place_details_pro_mes, place_details_enterprise_mes, place_details_enterprise_atmosphere_mes, places_aggregate_mes, place_photos_mes, text_search_pro_mes, text_search_enterprise_mes, text_search_enterprise_atmosphere_mes, maps_grounding_lite_mes, zona_horaria_mes, places_ui_kit_query_mes, places_ui_kit_autocomplete_sesion_mes, places_ui_kit_pro_mes | variante "environment" → calidad_aire_mes, polen_mes, solar_building_insights_mes, solar_data_layers_mes, clima_mes
- `cloud_healthcare`: region, almacenamiento_estructurado_gib, almacenamiento_standard_gib, almacenamiento_nearline_gib, almacenamiento_coldline_gib, almacenamiento_archive_gib, consentimientos_gestionados, politicas_consentimiento_admin, requests_estandar_mes, requests_complejas_mes, requests_multi_operacion_mes, requests_avanzadas_mes, export_lote_gib, transcodificacion_dicom_gib
- `netapp_volumes`: variante "standard" → region, capacidad_pool_tib, datos_backup_gib | variante "premium" → region, capacidad_pool_tib, datos_backup_gib | variante "extreme" → region, capacidad_pool_tib, datos_backup_gib

### Reglas anti-default (los defaults de la calculadora INFLAN el total: un spec omitido NO es 0)
- `networking` variante `ip_address`: SIEMPRE emite `ips_sin_uso` (0 salvo que pidan IPs reservadas
  sin asignar) y `ips_en_uso`. Omitir `ips_sin_uso` cotiza 20 IPs sin uso (~$146/mes de más).
- `networking` variante `nat`: SIEMPRE emite `tipo_nat` ("public-nat"), `vms_asignadas` (nº de
  VMs/gateways que pidan), `datos_procesados_gib` e `ips_usadas`. Omitirlos cotiza el default (~14 VMs).
- Regla general: todo recurso que el usuario dimensiona (cantidades, GiB, nº) va explícito; omite solo
  lo que el usuario no mencionó y no es una cantidad.
