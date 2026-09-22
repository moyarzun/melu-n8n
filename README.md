# n8n en este host

Despliegue local de [n8n](https://n8n.io) (automatización de flujos) en
Docker, expuesto a la red Tailscale del operador mediante Funnel para
acceso desde cualquier dispositivo de la tailnet.

## URL pública

- **Editor**: `https://melu-n8n.tailef0a9a.ts.net/n8n/`
- Nodo Tailscale: `melu-n8n` (debe estar `active` para que Funnel entregue
  tráfico; ahora figura `offline` en `tailscale status`).
- Funnel sirve `443` y mapea la ruta al contenedor n8n local.

## Estado actual (a fecha del último commit)

- `docker-compose.yml` define un único servicio `n8n` con
  `n8nio/n8n:latest`, puerto `5678`, volumen `./data` para persistir
  workflows / credenciales / clave de cifrado.
- Imagen **sin pinear a versión específica**: cualquier `docker compose
  pull && up -d` trae la última `:latest`, que puede cambiar entre
  invocaciones. La actualización reproducible requiere pinear a un tag
  (p.ej. `1.x.y` o `1.x`).
- Variables de entorno presentes:
  - `GENERIC_TIMEZONE=UTC`
  - `N8N_PORT=5678`
  - `N8N_EDITOR_BASE_URL=http://your-pi-ip:5678` (placeholder; debe
    actualizarse a `https://melu-n8n.tailef0a9a.ts.net/n8n/` o los
    webhooks / OAuth callbacks de n8n apuntarán mal).
  - `N8N_PROTOCOL=http`
- Sin subpath configurado: hoy n8n se monta en `/`. Para servirlo bajo
  `/n8n/` hay que setear `N8N_PATH=/n8n/` y asegurar que Funnel mapee
  `/n8n/*` al puerto `5678` (no al `/` raíz, donde hoy escucha el
  caddy muerto).
- Volumen `data/` (~4.6 MB, último acceso 26-ago) ignorado por git
  (incluye `database.sqlite` con workflows y `config` con
  `encryptionKey`).
- Reverse proxy local (caddy en `127.0.0.1:8080`) **muerto**: las
  rutas Tailscale Funnel que apuntaban a él no responden. Reemplazado
  por mapeo directo a n8n:5678 en el último plan Funnel.

## Cómo se opera

```bash
# Levantar (imagen + contenedor)
docker compose pull
docker compose up -d

# Bajar (preservando data/)
docker compose down

# Inspeccionar logs
docker compose logs -f n8n

# Validar el compose sin desplegar (usado por el test runner del loop)
docker compose config -q
```

## Riesgos que el plan debe contemplar

1. **`data/` no versionado** — borrar el volumen destruye workflows y
   credenciales cifradas con la `encryptionKey` local. Hacer backup
   antes de cualquier operación destructiva sobre el volumen.
2. **Imagen `:latest` no reproducible** — pinear a tag semver
   (recomendado: el último estable publicado, por ejemplo
   `1.118.1` o el que diga `docker run --rm n8nio/n8n:<tag> n8n
   --version` en la imagen recién bajada).
3. **`N8N_EDITOR_BASE_URL` debe cambiar** al URL Funnel, o los links
   que n8n genera (webhooks, OAuth, invitaciones) apuntarán al
   placeholder.
4. **`N8N_PATH=/n8n/`** + **Funnel re-ruteando `/n8n/*` → 5678**:
   tienen que estar sincronizados. Si Funnel sigue mapeando `/` al
   5678 y n8n también está en `/`, el subpath `/n8n/` va a
   404.
5. **Nodo Tailscale offline**: hasta que `melu-n8n` vuelva a estar
   `active` en `tailscale status`, el funnel no entrega tráfico y el
   smoke test externo (URL pública) no es ejecutable. El último
   checkpoint del plan lo deja pendiente de la reconexión manual del
   operador.