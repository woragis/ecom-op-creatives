# Observability — logging e monitoramento

Plano de evolução do logging para runs de pipeline pagos (Runway, OpenAI, etc.).

## Estado atual (baseline)

| Existe | Não existe |
|--------|------------|
| `log.Printf` no startup API/worker | Logs estruturados (JSON) |
| Uma linha por step no worker | `run_id` / `step_id` em todos os logs |
| `pipeline_steps.error_message` no Postgres | Log de prompts, latência, custo |
| | Arquivo de log persistente |
| | Correlação API ↔ worker |

Monitoramento hoje: `docker compose logs -f worker-pipeline` + UI `/runs/{id}`.

## Fase C — Logging estruturado (`log/slog`) ✅

Implementado em `internal/platform/applog` — JSON por default, campos `run_id` e `step`.

## Detalhes — Logging estruturado (`log/slog`)

### Formato

```json
{
  "time": "2026-06-12T15:00:00Z",
  "level": "INFO",
  "run_id": "uuid",
  "step": "video",
  "provider": "runway",
  "duration_ms": 45000,
  "msg": "step completed"
}
```

### Eventos

| Evento | Nível |
|--------|-------|
| step started / completed / failed | INFO / ERROR |
| provider API call (sem body completo) | DEBUG |
| render/ffmpeg stderr resumido | WARN |

### Env

| Variável | Default | Descrição |
|----------|---------|-----------|
| `LOG_LEVEL` | `info` | debug, info, warn, error |
| `LOG_FORMAT` | `json` | json ou text |

### Docker

Stdout vai para `docker compose logs`. Opcional futuro: bind mount `./logs:/var/log/creatives`.

## Fases futuras

- **Fase E:** export bundle com `meta.json` + logs do run
- **Fase F:** integração Datadog/Loki (produção)
- Métricas: duração por step, taxa de falha por provider, custo estimado

## Referências

- [STORAGE.md](./STORAGE.md) — onde ficam JSONs e mídia
- [TESTING.md](./TESTING.md) — checklist pré-run pago
