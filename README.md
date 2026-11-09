# GCP-Calculator-Skill

Skill que convierte requerimientos de infraestructura (texto, Excel, PDF, inventarios VMware/RVTools)
en un **plan JSON** para el motor de cotización de la calculadora oficial de Google Cloud.
La skill nunca calcula precios: el precio lo calcula Google.

| Carpeta | Para qué |
|---|---|
| `cotizar-gcp-nl/` | Skill para Claude (salida en español) |
| `cotizar-gcp-nl-en/` | Skill para Claude (salida en inglés) |
| `gemini/cotizar-gcp-nl/` | Empaque para Gemini (conocimiento en `.txt`/`.md`) |

Cada carpeta lleva `SKILL.md` + `knowledge/`. Para Claude: comprime la carpeta en `.zip` y súbela en
Configuración → Capacidades → Skills.

## Licencia

Propietaria, todos los derechos reservados (Información Localizada S.A.S). Visible al público; uso interno y por autorizados. Ver [LICENSE](LICENSE).
