# Spike: integraci\u00f3n Jev / OpenRouter con n8n

Estado del entorno antes de tocar el workflow de clasificaci\u00f3n de correos.

## Supuestos validados

| Supuesto | M\u00e9todo | Resultado |
|---|---|---|
| (a) Tools `n8n_*` del MCP activos en esta sesi\u00f3n | `tools/list` JSON-RPC contra el stdio MCP server (cargado con `~/.config/opencode/secrets/n8n-mcp.env`) | 21 tools `n8n_*` disponibles: `n8n_create_workflow`, `n8n_get_workflow`, `n8n_update_full_workflow`, `n8n_update_partial_workflow`, `n8n_delete_workflow`, `n8n_test_workflow`, `n8n_list_workflows`, `n8n_executions`, `n8n_health_check`, `n8n_audit_instance`, `n8n_manage_credentials`, `n8n_manage_datatable`, `n8n_manage_folders`, `n8n_deploy_template`, `n8n_evaluate`, etc. |
| (b) n8n local responde | `curl http://localhost:5678/healthz` | `200 OK` con `{"status":"ok"}` |
| (c) `OPENROUTER_API_KEY` autentica | `GET /api/v1/auth/key` con la key de `~/.local/share/opencode/auth.json` (provider `openrouter`) | `200 OK`; key con `limit: 5` (free tier), `is_free_tier: true` |
| (d) `/api/alpha/decisions` existe y devuelve esquema con preguntas estructuradas | `POST /api/alpha/decisions` con `model=typesafe/jev-1.13` | OK con `state: object`, `questions.q1.type: noul\|choice\|score`; retorna `{"model":"typesafe/jev-1.13-20260917","answers":{...}}` con `usage.input_tokens`, `cost` (~0.000012 USD/request) |
| (e) Modelo `jev` disponible en OpenRouter | `GET /api/v1/models/typesafe/jev-1.13` | **404** — el slug NO est\u00e1 en `/models`, pero **s\u00ed** funciona v\u00eda `/api/alpha/decisions`. Provider = `TypeSafe`, versionado como `typesafe/jev-1.13-20260917`. |

## Auth: dos keys distintas para dos servicios

- `~/.local/share/opencode/auth.json` -> `openrouter` key (`sk-or-v1-...`)
  -> `https://openrouter.ai/api/v1/*` (incluido `/api/alpha/decisions`)
- `~/.config/opencode/secrets/n8n-mcp.env` -> `N8N_API_KEY` (JWT 267 chars)
  -> `https://melu-n8n.tailef0a9a.ts.net/n8n/api/v1/*` (n8n public API)

**Para el workflow**: el nodo HTTP Request al clasificador necesita `OPENROUTER_API_KEY` (la primera). El JWT (`N8N_API_KEY`) es para los tools MCP de n8n. Hay que crear una credencial HTTP Header Auth nueva en n8n con la OPENROUTER_API_KEY.

## Schema confirmado para /api/alpha/decisions

```json
{
  "model": "typesafe/jev-1.13",
  "state": {"context": "..."},          // required, object (no string)
  "questions": {
    "q1": {
      "type": "noul",                   // "noul" | "choice" | "score"
      "prompt": "...",                  // required
      "instructions": "...",            // required
      // noul: no extra fields
      // choice: + "options": [...], "criteria": {"default": "<option>"}
      // score:  + "min": 0, "max": 1, "criteria": [{"weight": 1.0}]
    }
  }
}
```

Respuesta de ejemplo (`noul`, "Is the sky blue?"):
```json
{
  "model": "typesafe/jev-1.13-20260917",
  "answers": {
    "q1": {"type": "noul", "noul": 0.43}
  },
  "usage": {"input_tokens": 274, "output_tokens": 21, "cost": 0.000011508},
  "id": "gen-dec-1790194024-OUbPfsuuB9qOR9vgAwsK",
  "provider": "TypeSafe"
}
```

## Costo

~$0.000012 USD por request (~$83 USD por 1k requests). Free tier de OpenRouter tiene $5 de cr\u00e9dito; alcanza para ~400k clasificaciones antes de pagar.

## Decisiones derivadas para los pr\u00f3ximos checkpoints

- **CP2 (trigger)**: Webhook en `https://melu-n8n.tailef0a9a.ts.net/n8n/webhook/email-classifier` con body JSON `{from, subject, body, headers}`. NO IMAP (necesita servidor real y credentials; el objetivo del plan lo dejaba como optativo).
- **CP3 (clasificador)**: HTTP Request node con `method=POST`, `url=https://openrouter.ai/api/alpha/decisions`, body con **tres preguntas**: `q1: is_spam` (choice, options=[spam, not_spam], default=not_spam), `q2: urgency` (score, 0-1, criteria=[{weight:1.0}]), `q3: category` (choice, options=[work, personal, finance, other], default=other).
- **Credenciales n8n**: agregar HTTP Header Auth credential en n8n con `Authorization: Bearer sk-or-v1-...` apuntando a OpenRouter.
- **Estructura de respuesta**: `output.answers.q1.choice` (spam|not_spam), `q2.score` (0-1), `q3.choice` (work|personal|finance|other). Cablear al Set para que downstream reciba `$json.is_spam`, `$json.urgency`, `$json.category`.
