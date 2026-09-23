# Email Classifier Workflow (Jev + OpenRouter)

Workflow n8n que clasifica correos electr\u00f3nicos usando Jev, el modelo de
decisiones estructuradas de TypeSafe accesible v\u00eda OpenRouter.

## Topolog\u00eda

```
Webhook (/email-classifier)
  \u2192 Extract Email Fields (Set)
    \u2192 Build Jev Body (Code node: arma el body JSON con $input.first().json)
      \u2192 Jev Classify (HTTP Request \u2192 https://openrouter.ai/api/alpha/decisions)
        \u2192 Parse Jev (Set: extrae is_spam, urgency, category, model)
          \u2192 Switch Route (5 ramas: drop_quarantine|urgent|work|personal|finance)
            \u2192 Stub: <Route> (webhook.site placeholder)
              \u2192 Respond OK
```

## Endpoints

- **Webhook production**: `https://melu-n8n.tailef0a9a.ts.net/n8n/webhook/email-classifier` (POST JSON)
- **Webhook local** (test): `http://localhost:5678/webhook/email-classifier`
- **Workflow ID**: `awBerTT0gLmPB3LM`
- **Versi\u00f3n actual**: 11 (counter guardado en `email-classifier.json`)

## Input shape

```json
{
  "subject": "URGENT: Wire $5000",
  "body": "Send $5000 to acct 123-456 ASAP. Confirm by EOD.",
  "from": "[email protected]",
  "headers": {}        // opcional
}
```

## Output shape (de Respond OK)

```json
{
  "status": "received",
  "route": "drop_quarantine",
  "urgency": 0,
  "is_spam": "spam",
  "category": "finance"
}
```

`route` es uno de: `drop_quarantine`, `urgent`, `work`, `personal`, `finance`.

## Jev schema

```json
{
  "model": "typesafe/jev-1.13",
  "state": {"context": "Subject: ... | From: ... | Body: ..."},
  "questions": {
    "q1": {
      "type": "choice",                         // spam classification
      "options": ["spam", "not_spam"],
      "criteria": {"spam": {"weight": 1.0}, "not_spam": {"weight": 1.0}},
      "instructions": "..."
    },
    "q2": {
      "type": "score", min: 0, max: 1,         // urgency 0-1
      "criteria": [{"weight": 1.0}]
    },
    "q3": {
      "type": "choice",                         // category
      "options": ["work", "personal", "finance", "other"],
      "criteria": {"work": {"weight":1.0}, "personal": {"weight":1.0}, "finance": {"weight":1.0}, "other": {"weight":1.0}}
    }
  }
}
```

Respuesta:
```json
{
  "answers": {
    "q1": {"choice": "spam", "probabilities": {"spam": 0.91, "not_spam": 0.09}, "confidence": 0.83},
    "q2": {"score": 0.7, ...},
    "q3": {"choice": "finance", ...}
  },
  "usage": {"input_tokens": 365, "output_tokens": 34, "cost": 0.0000153}
}
```

## Credenciales necesarias

- **OPENROUTER_API_KEY**: actualmente hardcodeada en el header del nodo `Jev Classify`. Para producci\u00f3n, mover a HTTP Header Auth credential en n8n (`n8n_manage_credentials`).
- **N8N_API_KEY** (JWT): usado por `n8n-mcp` y la API p\u00fablica para crear/editar el workflow. Configurado en `~/.config/opencode/secrets/n8n-mcp.env`.

## Routing

| Condici\u00f3n (en orden) | Stub | Acci\u00f3n recomendada (producci\u00f3n) |
|---|---|---|
| `is_spam == "spam"` | `Stub: Quarantine (webhook.site)` | Email a quarantine mailbox / Slack #security |
| `urgency >= 0.7 AND is_spam != "spam"` | `Stub: Urgent (webhook.site)` | Slack #urgent + push notification |
| `category == "work"` | `Stub: Work (webhook.site)` | Notion DB / ClickUp task |
| `category == "personal"` | `Stub: Personal (webhook.site)` | Archive (low priority) |
| `category == "finance"` | `Stub: Finance (webhook.site)` | Notion DB / 1Password / spreadsheet |

Las URLs `webhook.site/unique-{route}-id` son **placeholders**. Reemplazar con
endpoints reales (Slack incoming webhook, Notion API, Postgres INSERT, etc.).

## Costo operativo

~$0.000015 USD por clasificaci\u00f3n (~$67 USD por 1k clasificaciones). Free tier
de OpenRouter (5 USD de cr\u00e9dito) alcanza para ~330k clasificaciones antes
de pagar.

## External smoke test

Para ejecutar el smoke test desde fuera del host, el operador necesita:

1. **Activar el nodo `melu-n8n` en Tailscale** (est\u00e1 offline al \u00faltimo commit; verificar con `tailscale status`).
2. **Verificar que el daemon** `tailscaled` est\u00e9 corriendo (`sudo systemctl status tailscaled`).
3. **Confirmar que el Funnel sigue configurado** (`sudo tailscale funnel status` debe listar `https://melu-n8n.tailef0a9a.ts.net`).
4. **Probar**:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"subject":"URGENT: Wire $5000","body":"Send ASAP","from":"[email protected]"}' \
  https://melu-n8n.tailef0a9a.ts.net/n8n/webhook/email-classifier
```

Respuesta esperada (HTTP 200):

```json
{
  "status": "received",
  "route": "drop_quarantine",
  "urgency": 0,
  "is_spam": "spam",
  "category": "finance"
}
```

5. **Inspeccionar logs del webhook.site** (URLs reales) o revisar executions
   en n8n (`/api/v1/executions?workflowId=awBerTT0gLmPB3LM`) para confirmar
   que el stub correcto se ejecut\u00f3.

## Limitaciones conocidas

- **Jev schema picky**: los criterios deben tener el shape correcto (`{"spam": {"weight":1.0}}` no `{"default": "spam"}`).
- **OPENROUTER_API_KEY hardcodeada**: para producci\u00f3n mover a HTTP Header Auth credential.
- **Stub URLs son placeholders**: requieren reemplazo antes de usar en producci\u00f3n.
- **Jev free tier**: 5 USD de cr\u00e9dito. Si el volumen crece, switch a plan pago o BYOK.

## Reproducibilidad

```bash
# 1. Cargar OPENROUTER_API_KEY en env (o usar hardcodeada en el workflow)
export OPENROUTER_API_KEY="sk-or-v1-..."

# 2. Activar n8n (si no est\u00e1 corriendo)
cd /home/moyarzun/n8n && docker compose up -d

# 3. Aplicar el workflow via REST API o n8n-mcp
# (workflow JSON en workflows/email-classifier.json)

# 4. Test local
curl -X POST -H "Content-Type: application/json" \
  -d '{"subject":"Test","body":"Body","from":"[email protected]"}' \
  http://localhost:5678/webhook/email-classifier
```
