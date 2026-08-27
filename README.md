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

- `schedule`: 1 vez por día (12:00 UTC ≈ 08:00/09:00 Chile según DST).
- `workflow_dispatch`: manual, o vía cron-job.org para mayor frecuencia.

## Debug

- En falla, el log queda como **artifact** privado del run.
- Opcionalmente se commitea a `latest_failure/` **redactado** (sin tokens firmados).
