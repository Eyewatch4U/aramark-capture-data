# aramark-capture-data

Captura automática del snapshot de **Meltwater** para el **Monitor Aramark**
(NSGroup / Eyewatch Intelligence Unit) e ingesta directa al Cloudflare Worker.

## Arquitectura

```
Meltwater (shareable report, 4 tabs)
  -> GitHub Actions + Playwright (runner con navegador real, IP no bloqueada)
     baja y valida el .gz (señal > 0)
  -> POST /ingest (JSON completo del snapshot, header x-sync-token)
  -> Worker corre extractMeltwater() y escribe KV latest_aramark
  -> Dashboard
```

Se usa `POST /ingest` (JSON completo) en lugar de `/update-url` porque Meltwater
bloquea con 502 las IPs de datacenter (Cloudflare/Make/Azure). El runner de
GitHub ya baja el `.gz` para validar la señal, así que enviar el JSON en vez de
la URL firmada no cuesta nada extra y elimina el bloqueo y la carrera por
expiración de la STS URL.

## Reporte (4 tabs)

ARAMARK · Benchmark Chile · Benchmark Internacional · Industria Minera

## Secrets del repo (Settings → Secrets and variables → Actions)

| Secret | Valor |
|---|---|
| `MELTWATER_URL` | URL pública del snapshot de Aramark |
| `MELTWATER_PASSWORD` | contraseña del reporte |
| `WORKER_INGEST_URL` | `https://aramarkmonitor.reports-dca.workers.dev/ingest` |
| `SYNC_TOKEN` | token de sincronización del Worker |

## Disparo

- Disparo: `workflow_dispatch` vía cron-job.org cada 30 min (:05 y :35). Sin `schedule` nativo, para no duplicar con el cron externo.
- `workflow_dispatch`: manual, o vía cron-job.org para mayor frecuencia.

## Debug

- En falla, el log queda como **artifact** privado del run.
- Opcionalmente se commitea a `latest_failure/` **redactado** (sin tokens firmados).

## Nota Cloudflare Bot Fight Mode

El zone del Worker Aramark tiene Bot Fight Mode activo y rechaza clientes sin
fingerprint TLS de navegador (403 error 1010). Por eso el POST `/ingest` usa
`curl_cffi` con `impersonate="chrome"` (ver `requirements.txt`). Un User-Agent
de navegador NO alcanza: el bloqueo es a nivel TLS (JA3), no de headers.
