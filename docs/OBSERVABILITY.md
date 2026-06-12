# Observability — logging e monitoramento

Plano de evolução do logging para runs de pipeline pagos (Runway, OpenAI, etc.).

## Estado atual

Monitoramento: `docker compose logs -f worker-pipeline` + UI `/runs/{id}` + arquivos em `storage/runs/{id}/`.

## Fase C — Logging estruturado (`log/slog`) ✅

Implementado em `internal/platform/applog` + `internal/platform/runctx` — JSON por default, `run_id` / `step` / `step_id` propagados via context no executor.

### Cobertura por integração

| Integração | Campos logados |
|------------|----------------|
| LLM (OpenAI) | `system_preview`, `user_preview`, `prompt_tokens`, `completion_tokens`, `duration_ms` |
| Serper | `query`, `results`, `duration_ms` |
| OpenAI TTS / DALL·E / Whisper | `operation`, previews, `duration_ms`, tamanho de saída |
| Vídeo (Runway etc.) | `job_id`, submit/poll, `duration_ms`, erros de provider |
| Remotion | slog no Go + JSONL em `storage/runs/{id}/logs/render.log` |
| FFmpeg | loudnorm/thumbnail start/complete, `duration_ms`, stderr resumido em falha |
| RabbitMQ | publish/dequeue com `queue`, `run_id`, `step_id`, `attempt` |
| Artefatos | `artifact written` com `path` e `bytes` |

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
