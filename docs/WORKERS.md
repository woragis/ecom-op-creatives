# Workers e filas — ecom-op-creatives

Arquitetura de workers assíncronos e RabbitMQ para produção.

---

## Visão geral

```text
Go API
   │ publish
   ▼
RabbitMQ ──┬── pipeline.research    → worker-llm
           ├── pipeline.llm          → worker-llm
           ├── pipeline.audio        → worker-media
           ├── pipeline.image        → worker-media
           ├── pipeline.video        → worker-media
           ├── pipeline.render       → worker-render (Node)
           └── pipeline.supervisor   → worker-llm

Cada fila tem DLQ: pipeline.{name}.dlq
```

---

## Por que RabbitMQ (não Redis puro, não Kafka como fila)

| Requisito | RabbitMQ |
|-----------|----------|
| Filas persistentes | Sim |
| Dead Letter Queue | Sim |
| Retry + backoff | Sim |
| Concorrência por fila | Sim (prefetch) |
| Management UI | Sim |
| Maturidade produção | Décadas |

**PostgreSQL** = fonte da verdade do estado.  
**RabbitMQ** = dispatch de jobs.  
**Kafka** (fase 3+) = event log para analytics, não substitui RabbitMQ.

---

## Workers

### worker-llm (Go)

**Filas:** `pipeline.research`, `pipeline.llm`, `pipeline.supervisor`

**Responsabilidades:**
- Executar agentes (research, hooks, script, director, prompter, supervisor)
- Serper + fetch + LLM synthesis (research)
- Publicar próximo step na fila correta após conclusão

**Concorrência:** alta (20–50 goroutines)

---

### worker-media (Go)

**Filas:** `pipeline.audio`, `pipeline.image`, `pipeline.video`, postprocess FFmpeg

**Responsabilidades:**
- ElevenLabs TTS
- Image APIs (Ideogram, Recraft, etc.)
- Video providers plugáveis (Kling, Runway, Luma, Veo)
- FFmpeg pós-process (loudness, thumbnail, validação)

**Concorrência:**
- audio/image: média (10)
- video: **baixa (2–5)** — caro, lento, rate limits

---

### worker-render (Node)

**Fila:** `pipeline.render`

**Responsabilidades:**
- Consumir Render Manifest JSON
- Remotion render → MP4 9:16
- Upload S3
- Notificar API / update Postgres

**Concorrência:** baixa (2–3) — CPU/RAM pesado (Chromium)

Ver [RENDER.md](./RENDER.md).

---

## Fluxo de um job

```text
1. Worker consome msg da fila
2. UPDATE pipeline_steps SET status=running, started_at=now()
3. Executa trabalho (com heartbeat a cada 30s se longo)
4. Salva output_json + artifacts no S3
5. UPDATE status=done, completed_at=now()
6. Publica próximo step na fila apropriada
7. ACK msg RabbitMQ
```

---

## Resiliência

### Retry

```text
Falha → retry 1 (30s)
      → retry 2 (2min)
      → retry 3 (10min)
      → DLQ + status=failed no Postgres
```

### Stale jobs

Cron no Go API (ou worker dedicado):

```text
Steps status=running AND started_at < now() - 30min
  → marca failed
  → opcional: re-enqueue (attempt_count < max)
```

### Idempotência

Workers devem ser idempotentes: reprocessar o mesmo step com mesmo input produz mesmo output (ou detecta já done e ACK).

Use `step_id` + `attempt_count` como chave.

### Continuar após falha

Estado no Postgres permite:

```text
User clica "Reprocessar step 8"
  → API re-enfileira step 8
  → worker continua pipeline normalmente
```

Fila persistente + estado no banco = pipeline recuperável.

---

## Mensagem RabbitMQ

```json
{
  "creativeRunId": "uuid",
  "stepId": "uuid",
  "stepType": "video",
  "attempt": 1,
  "payload": {}
}
```

Payload mínimo — input completo vem do Postgres (`input_json`).

---

## Layout backend

```text
backend/
├── server/           # Go API
├── worker-llm/
│   ├── cmd/worker/main.go
│   └── internal/...
├── worker-media/
│   ├── cmd/worker/main.go
│   └── internal/
│       ├── audio/elevenlabs/
│       ├── image/
│       ├── video/
│       │   ├── provider.go      # interface
│       │   ├── kling/
│       │   ├── runway/
│       │   ├── luma/
│       │   └── veo/
│       └── ffmpeg/
└── worker-render/
    ├── package.json
    ├── src/worker.ts
    └── remotion/
```

---

## Variáveis de ambiente

```env
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
DATABASE_URL=postgres://...

# Concorrência por worker
WORKER_LLM_CONCURRENCY=30
WORKER_MEDIA_CONCURRENCY=10
WORKER_VIDEO_CONCURRENCY=3
WORKER_RENDER_CONCURRENCY=2

# Heartbeat
WORKER_HEARTBEAT_INTERVAL_SEC=30
WORKER_STALE_THRESHOLD_MIN=30
```

---

## Docker Compose (workers)

```text
services:
  api
  worker-llm
  worker-media
  worker-render
  postgres
  rabbitmq
  minio
```

---

*Ver também: [PIPELINE.md](./PIPELINE.md), [backend/docs/adr/0002-rabbitmq-pipeline.md](../backend/docs/adr/0002-rabbitmq-pipeline.md)*
