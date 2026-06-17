# Polza.ai Aggregator Integration Plan

## Overview

Add Polza.ai as a first-class provider in Odysseus, with:
- One-click setup from the "Add API Models" dropdown
- Provider-aware model picker showing sub-providers (AtlasCloud, Cloudflare, OpenAI, etc.) with context window, max tokens, and pricing
- Per-message cost display in rubles (₽), captured from the `cost_rub` field in each response
- Auto-refresh of model inventory (same as other providers) — the `/v1/models` endpoint is free
- Zero impact on existing Custom URL / local endpoint behavior

## What problem are we solving?

Today, when Polza.ai is added as a Custom URL endpoint, Odysseus floods it with dozens of `GET /v1/models` requests:

```
data/logs/app.log (real log lines):
16:20:28 — GET https://polza.ai/api/v1/models "HTTP/1.1 200 OK"
16:20:36 — GET https://polza.ai/api/v1/models "HTTP/1.1 200 OK"
16:20:36 — GET https://polza.ai/api/v1/models "HTTP/1.1 200 OK"
16:20:50 — GET https://polza.ai/api/v1/models "HTTP/1.1 200 OK"
...
16:22:00 — GET https://polza.ai/api/v1/models "HTTP/1.1 200 OK"
16:33:30 — GET https://polza.ai/api/v1/models "HTTP/1.1 200 OK"
```

These come from three sources:
1. **Port scanning** (`model_discovery.py`) — scans 27 ports on every discovered host
2. **Model picker** (`llm_core.py:list_model_ids()`) — probes on every open
3. **Context detection** (`model_context.py:_query_context_length()`) — probes per model

Additionally:
- There's no way to select a specific sub-provider (AtlasCloud vs Cloudflare)
- The `cost_rub` field returned by Polza.ai in every response is unused (not displayed)
- The rich provider data (`providers[]` with context length, max tokens, pricing) from `/v1/models?include_providers=true` is unused

## How Polza.ai's API works (verified with real curl tests)

### Model listing (verified)
```bash
GET https://polza.ai/api/v1/models?type=chat&include_providers=true
Authorization: Bearer <API_KEY>
```

Returns (real response — verified):
```json
{
  "data": [
    {
      "id": "z-ai/glm-5.2",
      "name": "Z.ai: GLM 5.2",
      "type": "chat",
      "top_provider": { "name": "AtlasCloud", "context_length": 202752, "pricing": {...} },
      "providers": [
        {
          "name": "AtlasCloud",
          "context_length": 202752,
          "max_completion_tokens": 202752,
          "pricing": {
            "prompt_per_million": "130.52928000",
            "completion_per_million": "410.23488000",
            "currency": "RUB"
          },
          "supports_native_web_search": true
        },
        {
          "name": "Cloudflare",
          "context_length": 262144,
          "max_completion_tokens": 262144,
          "pricing": { "prompt_per_million": "130.52928000", ... }
        }
      ]
    }
  ]
}
```

Key fields per provider: `name`, `context_length`, `max_completion_tokens`, `pricing.prompt_per_million`, `pricing.completion_per_million`, `pricing.currency`.

### Per-request cost (verified)
Every chat completions response includes `cost_rub` in the `usage` block:
```json
{
  "usage": {
    "prompt_tokens": 13,
    "completion_tokens": 10,
    "total_tokens": 23,
    "cost_rub": 0.00579923,
    "cost": 0.00579923
  },
  "provider": "openrouter"
}
```

`cost_rub` is the **actual cost of this specific request** in rubles, including prompt + completion tokens.

### Provider selection (verified: `@provider=` alias DOES NOT WORK)
Must use `extra_body`:
```json
{
  "model": "z-ai/glm-5.2",
  "messages": [...],
  "provider": {"only": ["AtlasCloud"]}
}
```

---

## What we want to achieve

### 1. Pricing info in the model picker (at selection time)
When browsing models, the user sees **estimated** pricing per provider so they can make an informed choice:
```
▼ z-ai/glm-5.2
  └─ AtlasCloud   202K ctx · 32K max
                  130.53 / 410.23 ₽    ← per-million-token pricing from catalog
```

### 2. Actual cost under each message (after the response)
After sending a message and receiving a response, the **actual** cost of that specific exchange is shown:
```
User: "What is Rust?"
Assistant: "Rust is a systems programming language..."
          13→10 tok · 0.0058 ₽    ← actual cost_rub from the response
```

These two serve different purposes: catalog pricing for pre-selection comparison, `cost_rub` for post-facto tracking of actual spend.

---

## Fixes (in implementation order)

---

### Fix 1 — Stop port scanning aggregator hosts

**File**: `src/model_discovery.py`
**Lines**: ~190-225 (`discover_models()`)
**Lines changed**: ~25 added

**Problem**: `ModelDiscovery.discover_models()` scans 27 ports on every discovered host. Aggregator hosts don't need port scanning — they expose models through their own `/v1/models` API.

**Solution**: Before building the scan target list, query the DB for endpoints with `endpoint_kind IN ('api', 'proxy')`, extract their hostnames, and exclude them from scanning.

**Exact code** (add after `targets = [...]` on line 202, before `seen_models = set()`):

```python
# Skip hosts belonging to configured api/proxy endpoints (aggregators).
# These serve model lists through their own /v1/models API, not port scanning.
try:
    from core.database import SessionLocal, ModelEndpoint
    from urllib.parse import urlparse as _urlparse
    db = SessionLocal()
    try:
        skip_hosts = {
            _urlparse(ep.base_url).hostname
            for ep in db.query(ModelEndpoint).filter(
                ModelEndpoint.is_enabled == True,
                ModelEndpoint.endpoint_kind.in_(["api", "proxy"])
            ).all()
            if ep.base_url
        }
    finally:
        db.close()
except Exception:
    skip_hosts = set()

targets = [(h, p) for h, p in targets if h not in skip_hosts]
```

**Result**: Port scanning skips aggregator hosts. Local endpoints (Ollama, vLLM, LM Studio) continue scanning unchanged.

---

### Fix 2 — Use aggregator's `/v1/models` for model inventory (auto-refresh)

**File**: `routes/model_routes.py` (background refresh logic)
**Lines changed**: ~30 added

**Problem**: The auto-refresh subsystem already calls `GET /v1/models` for auto-mode endpoints, but only extracts flat `data[].id` strings. For aggregators, we need the complete response with `providers[]` arrays (name, context_length, max_completion_tokens, pricing).

**Solution**: When refreshing an endpoint with `kind = "api"` or `"proxy"`, call `GET /v1/models?type=chat&include_providers=true` and store the full response JSON in `cached_models`.

**What changes**: The existing refresh worker in [`routes/model_routes.py`](routes/model_routes.py) detects `endpoint_kind` and:
1. Appends `?type=chat&include_providers=true` to the models URL
2. Stores the full `r.text` (JSON string) instead of just `[id1, id2, ...]`

**Result**: Aggregator model inventory auto-refreshes like any other endpoint. One call per refresh cycle — no probing on model picker open.

---

### Fix 3 — Skip per-request probes in `list_model_ids()` and `_resolve_model()`

**Files**: `src/llm_core.py:1305-1350`, `src/ai_interaction.py:69-155`
**Lines changed**: ~15

**Problem**: Two functions probe `/v1/models` unnecessarily:
1. [`list_model_ids()`](src/llm_core.py:1305) — when model picker opens (if cache empty)
2. [`_resolve_model()`](src/ai_interaction.py:69) — on every chat message

For aggregators, `cached_models` is populated by auto-refresh (Fix 2). These probes are redundant.

**Change A** — in `list_model_ids()`, after line 1319 (`return list(ANTHROPIC_MODELS)`) and before the `try:` block:

```python
from src.model_context import _configured_endpoint_kind
if _configured_endpoint_kind(base_chat_url) in ("api", "proxy"):
    return cached or []
```

**Change B** — in `_resolve_model()`, inside the `else:` block at line 123:

```python
else:
    from src.model_context import _configured_endpoint_kind
    if _configured_endpoint_kind(base) in ("api", "proxy"):
        # Aggregator: trust model exists (populated by auto-refresh)
        model_ids = json.loads(ep.cached_models or "[]")
        if isinstance(model_ids, dict) and "data" in model_ids:
            model_ids = [m["id"] for m in model_ids.get("data", [])]
    else:
        # Existing probe code (lines 125-141)
        ...
```

**Result**: No `/v1/models` probes on model picker open or chat dispatch for aggregator endpoints.

---

### Fix 4 — Enhanced model picker with provider sub-list

**File**: `static/js/modelPicker.js`
**Lines changed**: ~80

**What the user sees** (catalog pricing for selection):

```
┌──────────────────────────────────────────┐
│ 🔍 Search models...                      │
│                                          │
│ ▼ z-ai/glm-5.2                           │
│   └─ AtlasCloud   202K ctx · 32K max     │  ← clickable
│      prompt 130.53 ₽/M                   │
│      compl  410.23 ₽/M                   │
│                                          │
│ ▼ openai/gpt-4o                          │
│   └─ OpenAI       128K ctx · 16K max     │
│      prompt   7.50 ₽/M                   │
│      compl   22.50 ₽/M                   │
│                                          │
│ ▼ anthropic/claude-opus-4-6              │
│   ├─ Anthropic    200K ctx · 32K max     │
│   │  prompt  45.00 ₽/M                   │
│   │  compl  135.00 ₽/M                   │
│   └─ Amazon Bedrock  200K ctx · 32K max  │
│      prompt  42.00 ₽/M                   │
│      compl  126.00 ₽/M                   │
└──────────────────────────────────────────┘
```

**Implementation**:
- Parse `cached_models` JSON for endpoints with `data-aggregator="true"`
- If JSON has `data[]` with `providers[]`: show expandable groups
- Each provider row shows: context_length, max_completion_tokens, prompt/completion pricing
- Clicking a provider: stores `{model, provider}` preference for Fix 6

---

### Fix 5 — Provider routing via `extra_body`

**File**: `src/llm_core.py`
**Lines**: ~1734-1761 (stream_llm) + ~1620-1629 (llm_call)
**Lines changed**: ~20

**Solution**: When user selects a sub-provider in the model picker, store the preference and inject it as `extra_body` in chat completions requests.

**Exact code** in `stream_llm()` OpenAI-compatible path (around line 1736):

```python
# After building base payload — for aggregator endpoints, add provider selection
provider_pref = _get_provider_preference(model)
if provider_pref:
    payload["provider"] = {"only": [provider_pref]}
```

Helper at module level:
```python
_provider_preferences: Dict[str, str] = {}  # model_id -> provider_name

def set_provider_preference(model_id: str, provider_name: str):
    _provider_preferences[model_id] = provider_name

def _get_provider_preference(model_id: str) -> Optional[str]:
    return _provider_preferences.get(model_id)
```

**Result**: Request body includes `provider: {only: ["AtlasCloud"]}` when user selected that provider.

---

### Fix 6 — Add Polza.ai as a preset provider

**Files**: `static/index.html`, `static/js/admin.js`
**Lines changed**: 4

**Change A** — in [`static/index.html:2154`](static/index.html:2154), add:

```html
<option value="https://polza.ai/api/v1" data-logo="polza" data-aggregator="true">Polza.ai</option>
```

**Change B** — in [`static/js/admin.js:1058`](static/js/admin.js:1058):

```javascript
if (provider.value && /polza\.ai/i.test(provider.value)) {
    fd.set('require_models', 'true');
}
```

**Result**: Polza.ai in dropdown. Auto-refresh, `endpoint_kind = "api"`, provider-aware model picker.

---

### Fix 7 — Display `cost_rub` under each message

**Files**: `src/llm_core.py` (backend capture) + `static/js/chatRenderer.js` (frontend display)
**Lines changed**: ~30

**What the user sees** (actual cost after each response):

```
┌──────────────────────────────────────────┐
│ User:                                    │
│ What is Rust?                             │
├──────────────────────────────────────────┤
│ Assistant:                                │
│ Rust is a systems programming language    │
│ focused on safety, speed, and concurrency │
│ without a garbage collector...            │
│                                           │
│  13→10 tok · 0.0058 ₽                    │  ← actual cost from cost_rub
└──────────────────────────────────────────┘
```

**Backend change** — in [`stream_llm()` lines 2099-2119](src/llm_core.py:2099), the OpenAI-compatible streaming SSE handler already captures `usage` blocks and emits `type: "usage"` SSE events. Add `cost_rub`:

```python
# Current code (line 2101-2103):
if "usage" in j and not _delta_has_output:
    u = j["usage"] or {}
    _usage_data = {"input_tokens": u.get("prompt_tokens", 0),
                   "output_tokens": u.get("completion_tokens", 0)}

# Add cost_rub extraction:
if "usage" in j and not _delta_has_output:
    u = j["usage"] or {}
    _usage_data = {
        "input_tokens": u.get("prompt_tokens", 0),
        "output_tokens": u.get("completion_tokens", 0),
    }
    # Aggregator endpoints include actual cost in the usage block
    if "cost_rub" in u:
        _usage_data["cost_rub"] = float(u["cost_rub"])
    elif "cost" in u:
        _usage_data["cost_rub"] = float(u["cost"])
    # ... existing timings handling unchanged ...
```

**Frontend change** — in `static/js/chatRenderer.js` (where `type: "usage"` SSE events are handled):

```javascript
if (data.type === "usage") {
    const usage = data.data;
    let footer = `${usage.input_tokens}→${usage.output_tokens} tok`;
    if (usage.cost_rub) {
        footer += ` · ${usage.cost_rub.toFixed(4)} ₽`;
    }
    // Update message footer
    updateMessageFooter(currentMessageId, footer);
}
```

**Note**: `cost_rub` is only present for Polza.ai and similar aggregator responses. For local endpoints, OpenAI, Anthropic, etc., the field is absent — the footer shows just token counts, unchanged.

---

## Complete flow

### Setup (admin does once):
1. Settings → Model Endpoints → **Add API Models** → **Polza.ai**
2. Enter API key → **Add**
3. `GET /v1/models?type=chat&include_providers=true` fetches full catalog
4. `cached_models` stores complete JSON with providers/pricing/context
5. Auto-refresh updates periodically

### Daily use:
1. Open model picker → models from cache (no probe)
2. Expand `z-ai/glm-5.2` → see AtlasCloud (202K, 32K max, 130.53 ₽/M prompt) vs Cloudflare (262K, 32K max)
3. Click AtlasCloud
4. Send message → `POST /v1/chat/completions` includes `provider: {only: ["AtlasCloud"]}`
5. Response streams back → `cost_rub: 0.00579923` captured from usage block
6. Under message: `13→10 tok · 0.0058 ₽`

---

## Summary

| # | File | Lines | What changes |
|---|---|---|---|
| 1 | `src/model_discovery.py:190-225` | +25 | Skip port scanning of aggregator hosts |
| 2 | `routes/model_routes.py` | +30 | Auto-refresh via `/v1/models?include_providers=true`, store full JSON |
| 3a | `src/llm_core.py:1305-1350` | +5 | Skip probe in `list_model_ids()` for aggregators |
| 3b | `src/ai_interaction.py:69-155` | +10 | Skip probe in `_resolve_model()` for aggregators |
| 4 | `static/js/modelPicker.js` | +80 | Provider sub-list: context, max tokens, pricing per million |
| 5 | `src/llm_core.py:1734-1761` | +20 | `extra_body` provider routing |
| 6a | `static/index.html:2154` | +1 | Polza.ai option in provider dropdown |
| 6b | `static/js/admin.js:1058` | +3 | Trigger model fetch for aggregator |
| 7a | `src/llm_core.py:2099-2119` | +5 | Capture `cost_rub` in streaming SSE usage event |
| 7b | `static/js/chatRenderer.js` | +25 | Display cost under each message |

**Total: ~205 lines across 9 files. Zero database migrations. Zero new tables.**

## Two kinds of pricing — different purposes

| Pricing | Where from | When shown | Purpose |
|---|---|---|---|
| Catalog pricing (₽/M) | `GET /v1/models?include_providers=true` | Model picker — at selection | Compare providers before sending |
| Actual cost (₽) | `cost_rub` in response `usage` | Under each message — after response | Track real spend per exchange |

## What stays unchanged

- Custom URL endpoints: no behavior change
- Local endpoints (Ollama, vLLM, LM Studio): continue port scanning
- OpenAI, Anthropic, Groq, etc. presets: continue working as before
- `cost_rub` display: only appears when the field is present (aggregator endpoints). Existing endpoints see no change.
- All other functionality: completely unaffected
