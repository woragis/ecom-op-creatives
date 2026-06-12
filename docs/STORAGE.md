# Storage — persistência de runs, JSONs e mídia

## Arquitetura

```text
┌─────────────────────────────────────┐
│  PostgreSQL (./data/postgres)       │
│  creative_runs, pipeline_steps      │
│  output_json (JSONB) ← IA results   │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  ./storage/runs/{runId}/            │
│  mídia + artifacts/ (JSON mirror)   │
└─────────────────────────────────────┘
```

## Por run — layout no disco

```text
storage/runs/{uuid}/
├── input/                    # uploads (persona, product, intro)
├── artifacts/                # mirror JSON por step (Fase B)
│   ├── 01-research.json
│   ├── 02-hooks.json
│   ├── ...
│   └── run-meta.json
├── narration.mp3
├── scene_*.png
├── scene_*.mp4
├── captions.srt
├── manifest.json
├── draft.mp4
├── final.mp4
└── thumbnail.jpg
```

URLs públicas: `GET /media/runs/{id}/{file}`

## Postgres — o que fica no DB

| Tabela | Conteúdo |
|--------|----------|
| `creative_runs` | status, providers, hook, input_assets |
| `pipeline_steps` | `output_json`, `error_message`, `provider_used`, timestamps |

Colunas planejadas para uso pleno: `input_json`, `provider_used`.

## Docker — persistência no host

| Path | Conteúdo |
|------|----------|
| `./data/postgres` | banco (bind mount) |
| `./storage` | mídia + artifacts |
| `./exports` | bundles exportados (futuro) |

**Nunca** `docker compose down -v` em produção — apaga o Postgres se ainda estiver em volume nomeado.

## Fases

| Fase | Entrega | Status |
|------|---------|--------|
| A | Bind mount Postgres em `./data/postgres` | implementado |
| B | Artifact mirror em `artifacts/` | implementado |
| D | `PAUSE_BEFORE_VIDEO` + `POST /continue` | implementado |
| E | `GET /export` zip por run | planejado |
| F | S3/MinIO (substituir `storage.Local`) | planejado |

## Backup manual

```bash
# Mídia + artifacts
cp -r storage/runs/{run-id} exports/{run-id}

# Dump DB
docker compose exec postgres pg_dump -U creatives creatives > data/backups/creatives.sql
```
