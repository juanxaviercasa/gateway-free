# Guía para ChatGPT / Asistente en la otra máquina: cómo funciona este gateway, cómo crear todas las claves y cómo configurar modelos open source y sin censura

> **Para la IA (ChatGPT / Codex / agente en otra máquina):**
> Este documento explica la arquitectura real de este proyecto (**FreeLLMAPI**), cómo está desplegado actualmente y el procedimiento exacto (endpoints, comandos CLI, formatos de importación y enlaces reales del código) para ayudar al usuario a:
> 1. Crear y registrar todas las API keys gratuitas posibles.
> 2. Conectar modelos **open source** y modelos locales **sin censura** (Ollama, `llama.cpp`, LM Studio, vLLM).
> 3. Poblar y activar todos los modelos, cadenas de reserva y clientes (Claude Code, Codex, OpenAI SDK, etc.).
>
> **Regla de seguridad:** Nunca guardes ni subas a Git archivos `.env` reales, `ENCRYPTION_KEY`, contraseñas, tokens de sesión (`freellmapi_dashboard_token`), claves `freellmapi-...` ni bases `*.db`.

---

## 1. Cómo funciona este proyecto y estado actual del despliegue

Este repositorio es una copia limpia de **FreeLLMAPI** (`tashfeenahmed/freellmapi`), un router/gateway unificado que expone:
- **Superficie OpenAI:** `/v1/chat/completions`, `/v1/responses`, `/v1/completions`, `/v1/embeddings`, `/v1/images/generations`, `/v1/videos/generations`, `/v1/audio/speech`, `/v1/audio/transcriptions`, `/v1/models`
- **Superficie Anthropic (Claude Code):** `/v1/messages` (la URL base para Claude Code es la raíz del servidor **sin** `/v1`)
- **Superficie Gemini nativa:** `/v1beta`
- **Superficie Ollama y MCP:** `/api/*` y `/mcp`

### Despliegue activo actual
- **URL pública del gateway (VPS):** `https://llm.forjavia.site`
- **Arquitectura en el VPS:**
  - Contenedor Docker `freellmapi` (`ghcr.io/tashfeenahmed/freellmapi:latest`, multi-arch ARM64/AMD64) corriendo 24/7 en loopback `127.0.0.1:3011->3001/tcp` (el puerto `3001` del host ya estaba usado por otro servicio, por eso se publicó en `3011`).
  - Publicado por HTTPS mediante **Cloudflare Tunnel** hacia `https://llm.forjavia.site`, con `TRUST_PROXY=1`, `ALLOWED_HOSTS=llm.forjavia.site` y `DASHBOARD_ORIGINS=https://llm.forjavia.site`.
  - La plantilla sanitizada de ese despliegue está en [`deploy/vps/compose.yml`](deploy/vps/compose.yml) y [`deploy/vps/.env.example`](deploy/vps/.env.example).
- **Almacenamiento de claves:** Todas las API keys de proveedores se cifran en reposo con AES-256-GCM en SQLite (`/app/server/data/freeapi.db`) usando `ENCRYPTION_KEY`.
- **Sincronización de catálogo:** El servidor sincroniza automáticamente el catálogo firmado de modelos desde `api.freellmapi.co` cada 12 horas.

---

## 2. Arquitectura recomendada para la otra máquina (Open Source y modelos sin censura)

Es clave entender esta distinción antes de configurar nada:

1. **Proveedores gratuitos en la nube:**
   - Sirven modelos open-weight (Llama, Qwen, DeepSeek, Mistral, Gemma, GLM, etc.) gratis con límites diarios/por minuto.
   - **Nota sobre censura:** aunque el modelo sea open-weight, cada API en la nube aplica sus propias políticas de moderación/uso y filtros del proveedor.
2. **Modelos 100% open source y sin censura real (ej. pesos *abliterated*, *Dolphin*, *Uncensored* GGUF):**
   - Deben ejecutarse **en hardware propio** (en esa otra máquina con GPU/CPU usando **Ollama**, **`llama.cpp` (`llama-server`)**, **LM Studio** o **vLLM**) y conectarse a FreeLLMAPI mediante el adaptador **`custom` (OpenAI-compatible)**.

### Elige una de estas dos topologías según dónde corran los modelos locales

- **Topología 1 — Gateway local en la otra máquina (la más simple si usas Ollama / `llama.cpp` / LM Studio en esa misma PC):**
  - Ejecutas FreeLLMAPI directamente en esa máquina (`npm install && npm run build && node server/dist/index.js` o Docker local en `http://localhost:3001`).
  - Ahí puedes mezclar **en el mismo router**:
    - Todas las API keys gratuitas en la nube, **más**
    - Tus modelos locales sin censura en `http://127.0.0.1:11434/v1` (Ollama), `http://127.0.0.1:8080/v1` (`llama-server`) o `http://127.0.0.1:1234/v1` (LM Studio).
- **Topología 2 — Conectar esa máquina al gateway del VPS (`https://llm.forjavia.site`):**
  - El VPS en la nube **no puede alcanzar** una IP privada `127.0.0.1` o `192.168.x.x` de tu otra máquina directamente.
  - Si quieres que `https://llm.forjavia.site` también enrute hacia tus modelos locales de esa otra máquina, debes exponer el endpoint compatible con OpenAI de esa máquina por **HTTPS** mediante un túnel seguro (por ejemplo **Cloudflare Tunnel**, Tailscale Funnel o un túnel SSH inverso con token Bearer) y luego añadir esa URL pública en `https://llm.forjavia.site/keys` como **Custom (OpenAI-compatible)**.

> **Importante sobre protección SSRF:** `server/src/lib/url-guard.ts` valida las URLs de proveedores `custom`. Para servidores locales en la misma máquina (`127.0.0.1` / `localhost`), FreeLLMAPI permite conectar endpoints locales en instalaciones locales; detrás de un VPS remoto, usa siempre una URL HTTPS propia con autenticación Bearer.

---

## 3. Lista completa de proveedores y enlaces para conseguir claves y tokens

Todos los `platform` IDs y URLs están verificados contra [`client/src/components/keys/shared.tsx`](client/src/components/keys/shared.tsx) y [`docs/en/providers/01-supported-platforms.md`](docs/en/providers/01-supported-platforms.md).

### A. Proveedores SIN clave (*Keyless*) — activar primero
No requieren tarjeta ni registro obligatorio para empezar (aunque `aihorde` mejora su cola si creas una clave gratuita):
- `kilo` — Kilo Gateway (`https://app.kilo.ai`) — rutas `:free` anónimas (200 req/h por IP)
- `ovh` — OVH AI Endpoints (`https://endpoints.ai.cloud.ovh.net`) — nivel anónimo (2 req/min por IP/modelo)
- `aihorde` — AI Horde (`https://aihorde.net/register`) — trabajadores comunitarios voluntarios (lento, sin streaming; funciona con clave anónima `0000000000` o clave registrada para mayor prioridad)
- `halx` (`https://hal-x.ai`) y `aisense` (`https://www.aisense.no/free-public-api-decide-api-endpoint`) — solo enrutamiento *Decision* (sin clave)

### B. Proveedores principales gratuitos recomendados (alta cuota / modelos open-weight)
Crea cuenta y genera API key en cada uno:
1. `groq` — Groq: `https://console.groq.com/keys`
2. `cerebras` — Cerebras: `https://cloud.cerebras.ai`
3. `google` — Google AI Studio: `https://aistudio.google.com/apikey`
4. `openrouter` — OpenRouter (modelos `:free`): `https://openrouter.ai/keys`
5. `nvidia` — NVIDIA NIM: `https://build.nvidia.com/settings/api-keys`
6. `mistral` — Mistral: `https://console.mistral.ai/api-keys/`
7. `cloudflare` — Cloudflare Workers AI: `https://dash.cloudflare.com` *(formato obligatorio: `ACCOUNT_ID:API_TOKEN`)*
8. `cohere` — Cohere: `https://dashboard.cohere.com/api-keys`
9. `huggingface` — HuggingFace Router: `https://huggingface.co/settings/tokens`
10. `zhipu` — Zhipu AI (Z.ai): `https://z.ai/manage-apikey/apikey-list`
11. `ollama` — Ollama Cloud: `https://ollama.com/settings/keys`

### C. Gateways y routers secundarios con capa gratuita
Útiles para multiplicar el pool de tokens/peticiones diarias:
- `requesty` (`https://www.requesty.ai`) — 200 req/día en modelos gratuitos
- `routeway` (`https://routeway.ai`)
- `orcarouter` (`https://www.orcarouter.ai`)
- `unorouter` (`https://unorouter.com`)
- `xkiro` (`https://xkiro.com`) — 5M tokens/día en modelos gratuitos
- `iteracompute` (`https://iteracompute.com/free/`) — 30M tokens/mes compartidos
- `powertokens` (`https://www.powertokens.ai`)
- `longcat` (`https://longcat.chat/platform`) — acepta registro por email fuera de China
- `sealion` (`https://sea-lion.ai`) — 10 RPM gratuitos
- `pollinations` (`https://enter.pollinations.ai`)
- `llm7` (`https://llm7.io`)
- `onomeo` (`https://onomeo.com/dashboard`)
- `apinex` (`https://apinex.bond/keys`)
- `zro` (`https://zro.moonmath.ai/api-keys`)
- `silvr` (`https://dashboard.silvrai.net`)
- `novai` (`https://aiapi-pro.com`) — chat + video
- `electronhub` (`https://app.electronhub.ai`)
- `experiential` (`https://platform.experientiallabs.ai`)
- `router9` (`https://www.router9.com`)
- `septor` (`https://septorlabs.com/dashboard`)
- `clod` (`https://app.clod.io`)
- `lucidity` (`https://composite.lucidity.sh`)
- `airforce` (`https://api.airforce`)
- `dreamprompting` (`https://dreamprompting.com`)
- `blockrun` (`https://user.blockrun.ai/dashboard/keys`)
- `moondream` (`https://moondream.ai/c/cloud/api-keys`)
- `speka` (`https://speka.me/dashboard/keys`)
- `kenari` (`https://kenari.id`)
- `kiosapi` (`https://kiosapi.id/dashboard/keys`)
- `typhoon` (`https://playground.opentyphoon.ai`)
- `plugsky` (`https://plugsky.com/dashboard`)
- `inferbase` (`https://inferbase.ai`)
- `simplellm` (`https://app.simplellm.eu`)
- `llmtr` (`https://llmtr.com`)
- `gizmo` (`https://gizmoplatforms.com/developers`)
- `jamai` (`https://cloud.jamaibase.com`)
- `nabh` (`https://app.nabh.cloud`)
- `varosity` (`https://varosity.ai/app/keys/api-keys`)
- `radeon` (`https://developer.amd.com.cn/radeon/tokenfactory`)
- `agnes` (`https://platform.agnes-ai.com`)
- `bazaarlink` (`https://bazaarlink.ai`)
- `ainative` (`https://ainative.studio`)
- `aion` (`https://www.aionlabs.ai`)
- `navy` (`https://api.navy`)
- `nara` (`https://router.bynara.id`)
- `anyapi` (`https://anyapi.ai`)
- `freeinference` (`https://freeinference.org`) *(nota: proveedor de investigación que registra prompts)*
- `freemodel` (`https://freemodel.in/dashboard`) *(verificación Telegram)*
- `blaze` (`https://blazeapi.org/dashboard`) *(verificación Discord)*
- `logfare` (`https://logfare.ai`)
- `bai` (`https://b.ai`)

### D. Proveedores de media / voz / búsqueda (opcionales)
- **Imagen / TTS / Audio:** `siliconflow` (`https://siliconflow.com`), `rime` (`https://app.rime.ai`), `typecast` (`https://studio.typecast.ai/developers/api`), `cartesia` (`https://play.cartesia.ai/keys`), `speechify` (`https://platform.speechify.ai`), `freetts` (`https://freetts.org/dashboard?section=api-keys`), `freeai` (`https://free.ai/account/#api-keys`)

### E. Proveedores NO recomendados inicialmente (restricciones)
- **Requieren verificación de identidad real en China:** `modelscope`, `qianfan`, `volcengine`, `xfyun`
- **Requieren tarjeta o saldo prepago:** `opencode`, `reka`, `sail`
- **Ocultos / sin registro nuevo disponible:** `aclide`, `waterfall`

---

## 4. Cómo debe proceder ChatGPT para cargar TODAS las claves y llenar los modelos

Cuando el usuario haya generado sus claves (o mientras las va pegando), la IA puede automatizar la carga por **3 vías oficiales**:

### Vía 1 — Importación masiva con plantilla `.env` (la más rápida)
1. Copia [`deploy/templates/keys-import.env.example`](deploy/templates/keys-import.env.example) a un archivo temporal fuera del repo (por ejemplo `mis-claves.env`).
2. Rellena únicamente las variables de los proveedores que el usuario haya creado.
   - Los prefijos válidos están definidos en `server/src/lib/key-parser.ts` (`PREFIX_MAP`).
   - Para `cloudflare`, el valor es `ACCOUNT_ID:API_TOKEN`.
   - Para endpoints open-source / sin censura locales (`Ollama`, `llama.cpp`, `LM Studio`, `vLLM`), usa parejas:
     ```env
     CUSTOM_1_BASE_URL=http://127.0.0.1:11434/v1
     CUSTOM_1_KEY=no-key
     CUSTOM_1_MODELS=dolphin-llama3:8b,qwen2.5:14b-TOOLS
     ```
     *(Sufijos soportados en `CUSTOM_*_MODELS`: `-TOOLS` y `-VISION`.)*
3. En el panel web (`Claves → Importar`) o vía `POST /api/keys/import`, sube el archivo.

### Vía 2 — CLI oficial de FreeLLMAPI (desde esta carpeta o `npx freellmapi`)
El CLI permite añadir, listar y probar claves contra el gateway local o contra `https://llm.forjavia.site`:

1. Obtén un **token de sesión del dashboard** (`FREELLMAPI_DASHBOARD_TOKEN`):
   - Desde el navegador (localStorage `freellmapi_dashboard_token`), o vía API:
     ```bash
     curl -sS -X POST https://llm.forjavia.site/api/auth/login \
       -H "Content-Type: application/json" \
       -d '{"email":"<TU_EMAIL>","password":"<TU_PASSWORD>"}'
     ```
2. Exporta la URL y el token de sesión en tu terminal y añade claves:
   ```bash
   export FREELLMAPI_URL="https://llm.forjavia.site"
   export FREELLMAPI_DASHBOARD_TOKEN="<TOKEN_DE_SESION>"

   # Proveedores sin clave (keyless):
   npx freellmapi keys add kilo
   npx freellmapi keys add ovh
   npx freellmapi keys add aihorde

   # Proveedores con clave:
   npx freellmapi keys add groq --key "<GROQ_KEY>"
   npx freellmapi keys add cerebras --key "<CEREBRAS_KEY>"
   npx freellmapi keys add openrouter --key "<OPENROUTER_KEY>"
   npx freellmapi keys add nvidia --key "<NVIDIA_KEY>"
   npx freellmapi keys add google --key "<GOOGLE_KEY>"
   npx freellmapi keys add mistral --key "<MISTRAL_KEY>"
   npx freellmapi keys add cloudflare --key "<ACCOUNT_ID>:<API_TOKEN>"

   # Verificar salud y contar modelos habilitados por clave:
   npx freellmapi keys list
   npx freellmapi keys test groq
   ```
   *(En PowerShell usa `$env:FREELLMAPI_URL = "..."` y `$env:FREELLMAPI_DASHBOARD_TOKEN = "..."`, o ejecuta `node cli/dist/index.js ...` directamente).*

### Vía 3 — API REST del panel para endpoints `custom` (modelos open source / sin censura) y descubrimiento de modelos
Con el header `Authorization: Bearer <TOKEN_DE_SESION>`:

1. **Detectar automáticamente los modelos** de un servidor OpenAI-compatible (Ollama, `llama-server`, LM Studio, vLLM):
   ```bash
   curl -sS -X POST https://llm.forjavia.site/api/keys/custom/discover-models \
     -H "Authorization: Bearer <TOKEN_DE_SESION>" \
     -H "Content-Type: application/json" \
     -d '{"baseUrl":"https://tu-endpoint-local-o-tunel/v1","apiKey":"opcional"}'
   ```
2. **Registrar el endpoint personalizado y sus modelos** (con etiqueta de grupo `groupLabel` para poder invocarlo aparte, por ejemplo `#uncensored`):
   ```bash
   curl -sS -X POST https://llm.forjavia.site/api/keys/custom \
     -H "Authorization: Bearer <TOKEN_DE_SESION>" \
     -H "Content-Type: application/json" \
     -d '{
       "baseUrl": "https://tu-endpoint-local-o-tunel/v1",
       "apiKey": "no-key",
       "label": "Mi GPU Local Sin Censura",
       "groupLabel": "uncensored",
       "models": [
         { "model": "dolphin-llama3:8b", "supportsTools": true, "kind": "chat" }
       ]
     }'
   ```
   Después podrás llamar a ese grupo explícitamente como `custom:dolphin-llama3:8b#uncensored`, o dejarlo dentro de tu cadena de reserva.
3. **Descubrir modelos en proveedores integrados** (como `openrouter`, `groq`, `nvidia`, `huggingface`, etc.):
   - Desde la UI: página **Modelos** / **Claves** → activar los modelos deseados y probar salud de claves.
   - Si agregas claves vía API/CLI, FreeLLMAPI habilita y sincroniza el catálogo automáticamente, y puedes crear perfiles nombrados en **Cadenas de reserva** (por ejemplo `auto:coding`, `auto:uncensored`) desde la página **Modelos**.

---

## 5. Cómo conectar los clientes y agentes a `https://llm.forjavia.site`

Usa tu **clave unificada** (`freellmapi-...`, visible en la parte superior de **Claves**; NO confundirla con el token de sesión del dashboard):

- **Claude Code:**
  ```bash
  npx freellmapi setup-claude --url https://llm.forjavia.site --api-key <UNIFIED_KEY>
  ```
  *(Nota: Claude Code usa la URL raíz sin `/v1` porque añade `/v1/messages` por su cuenta).*
- **OpenAI SDK / LangChain / scripts en Python o Node:**
  - `base_url = "https://llm.forjavia.site/v1"`
  - `api_key = "<UNIFIED_KEY>"`
  - `model = "auto"` (o `"auto:fast"`, `"auto:smartest"`, `"auto:<perfil>"`, o el ID exacto de un modelo).
- **Otros agentes compatibles** (`codex`, `opencode`, `cline`, `aider`, `qwen`, `roo`, `goose`, `continue`, etc.):
  ```bash
  npx freellmapi list
  npx freellmapi setup-<agente> --url https://llm.forjavia.site --api-key <UNIFIED_KEY>
  ```
  Documentación completa en [`docs/en/clients/02-supported-agents.md`](docs/en/clients/02-supported-agents.md) y [`docs/en/clients/01-agent-clients.md`](docs/en/clients/01-agent-clients.md).
