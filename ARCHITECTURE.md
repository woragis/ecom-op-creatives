# Arquitetura — ecom-op-creatives

Documento vivo. Descreve a arquitetura da plataforma de criativos para e-commerce.

---

## Índice

1. [Visão geral](#1-visão-geral)
2. [Runtimes](#2-runtimes)
3. [Pipeline](#3-pipeline)
4. [Estado e filas](#4-estado-e-filas)
5. [Storage](#5-storage)
6. [Formato de saída](#6-formato-de-saída)
7. [Decisões fechadas](#7-decisões-fechadas)
8. [Fases de entrega](#8-fases-de-entrega)

---

## 1. Visão geral

```text
Frontend (Next.js)
       │
       ▼
Go API (server/)              ← CRUD, auth, pipeline state, enqueue
       │
       ├─ PostgreSQL            ← fonte da verdade (runs, steps, JSON editável)
       ├─ RabbitMQ              ← dispatch de jobs por etapa
       └─ S3/MinIO              ← mídia (mp3, mp4, png)
              │
              ▼
       Workers
       ├─ worker-llm (Go)      ← agentes LLM + Serper + supervisor
       ├─ worker-media (Go)    ← ElevenLabs, imagem, vídeo (multi-provider)
       └─ worker-render (Node)  ← Remotion UGC 9:16 + FFmpeg pós-process
```

**Princípio central:** PostgreSQL é o dono do estado. RabbitMQ apenas dispara trabalho. Se a fila falhar, o estado no banco permite re-enfileirar e continuar de qualquer step.

---

## 2. Runtimes

| Runtime | Responsabilidade |
|---------|------------------|
| **Go API** | HTTP, auth, CRUD, state machine, enqueue |
| **Go worker-llm** | Research, hooks, script, director, prompter, supervisor |
| **Go worker-media** | ElevenLabs, image APIs, video providers |
| **Node worker-render** | Remotion composition + FFmpeg post-process |
| **Next.js** | Dashboard, editor por step, provider picker |

Padrão de camadas no Go (herdado do Lingo):

```text
handler → service → repository
```

Ver [backend/docs/ARCHITECTURE.md](./backend/docs/ARCHITECTURE.md).

---

## 3. Pipeline

Cada **CreativeRun** é uma execução do pipeline para um produto + gancho.

```text
1.  research        Serper + LLM → dores, linguagem, público
2.  hooks           LLM → ganchos rankeados
3.  script          LLM → roteiro com timing e narração
4.  director        LLM → câmera, transição, música, ritmo
5.  prompter        LLM → prompts finais imagem/vídeo
6.  voice           ElevenLabs → narração mp3
7.  image           API imagem → assets (quando necessário)
8.  video           Provider escolhido → clips mp4
9.  subtitles       Whisper/Deepgram → SRT/word timing
10. render          Remotion → vídeo final 9:16
11. postprocess     FFmpeg → loudness, thumbnail, validação
12. supervisor      LLM → QA score, approved/rejected
```

Detalhes: [docs/PIPELINE.md](./docs/PIPELINE.md)

**Reprocessamento parcial:** editar `output_json` de um step invalida steps downstream e re-enfileira a partir do próximo.

---

## 4. Estado e filas

### PostgreSQL (fonte da verdade)

```text
products
campaigns
creative_runs
pipeline_steps     ← status, input_json, output_json, attempt_count
artifacts            ← S3 URLs, tipo, metadata
api_costs            ← tracking de custo por provider
```

### RabbitMQ (filas por tipo)

| Fila | Worker | Concorrência |
|------|--------|--------------|
| `pipeline.research` | worker-llm | média |
| `pipeline.llm` | worker-llm | alta |
| `pipeline.audio` | worker-media | média |
| `pipeline.image` | worker-media | média |
| `pipeline.video` | worker-media | baixa |
| `pipeline.render` | worker-render | baixa |
| `pipeline.supervisor` | worker-llm | alta |

Cada fila tem DLQ (Dead Letter Queue) para jobs que falham após retries.

Detalhes: [docs/WORKERS.md](./docs/WORKERS.md)

### Kafka (fase 3+)

Event log para analytics e feedback loop (métricas pós-publicação → Hook Agent). Não substitui RabbitMQ.

---

## 5. Storage

| Tipo | Onde |
|------|------|
| Metadados, JSON, estado | PostgreSQL |
| Áudio, vídeo, imagens | S3/MinIO |
| Prompts versionados | Repo (backend) + opcional DB |

Convenção de paths S3:

```text
{orgId}/{creativeRunId}/{stepType}/{artifactId}.{ext}
```

---

## 6. Formato de saída

Quase 100% dos criativos:

| Propriedade | Valor |
|-------------|-------|
| Aspect ratio | 9:16 |
| Resolução | 1080×1920 |
| Duração | 15–60s |
| Estilo | UGC TikTok/Reels |
| Legendas | Animadas (Remotion) |

---

## 7. Decisões fechadas

| Decisão | Escolha |
|---------|---------|
| Áudio | ElevenLabs 100% |
| Vídeo | Multi-provider (default `.env`, override na UI) |
| Render composição | Remotion (Node) |
| Render pós-process | FFmpeg (Go media-worker) |
| Research | Serper + Jina Reader + LLM |
| Fila | RabbitMQ |
| Estado | PostgreSQL |
| Agentes | Go + prompts versionados + JSON schema (sem LangChain) |
| Backend pattern | Lingo (handler → service → repository) |

---

## 8. Fases de entrega

### Fase 0 — Fundação
Scaffold Go API, PostgreSQL, RabbitMQ, workers vazios, frontend lista runs.

### Fase 1 — MVP criativo
Pipeline end-to-end sem AI video caro: GPT agents → ElevenLabs → imagem produto → Remotion → dashboard editável.

**Critério:** 1 produto → 1 vídeo 30–60s 9:16 no dashboard.

### Fase 2 — Escala
N ganchos → N vídeos, agendamento, thumbnail, supervisor QA, multi video provider na UI.

### Fase 3 — Feedback loop
Kafka events, métricas pós-publicação, Hook Agent aprende, publicação automática.

---

## Referências

- [backend/docs/ARCHITECTURE.md](./backend/docs/ARCHITECTURE.md)
- [frontend/docs/ARCHITECTURE.md](./frontend/docs/ARCHITECTURE.md)
- [Lingo ARCHITECTURE.md](https://github.com/woragis/lingo) (padrão Go)

---

*Última atualização: junho/2026*
