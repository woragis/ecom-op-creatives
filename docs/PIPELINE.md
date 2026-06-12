# Pipeline — ecom-op-creatives

Fluxo completo de geração de um criativo, do produto ao vídeo final.

---

## Visão geral

```text
Product + Campaign
       │
       ▼
CreativeRun (1 vídeo)
       │
       ├── Step 1:  research
       ├── Step 2:  hooks
       ├── Step 3:  script
       ├── Step 4:  director
       ├── Step 5:  prompter
       ├── Step 6:  voice
       ├── Step 7:  image (opcional)
       ├── Step 8:  video
       ├── Step 9:  subtitles
       ├── Step 10: render
       ├── Step 11: postprocess
       └── Step 12: supervisor
```

Cada step produz `output_json` persistido no PostgreSQL. O usuário pode editar qualquer step no dashboard; a edição invalida steps downstream e re-enfileira a partir do próximo.

---

## Steps

### 1. Research

**Agente:** Research Agent  
**Fila:** `pipeline.research`  
**Input:** nome do produto, URL (opcional), nicho  
**Output:** dores, linguagem do consumidor, objeções, público, triggers emocionais

Ver [RESEARCH.md](./RESEARCH.md).

---

### 2. Hooks

**Agente:** Hook Agent  
**Fila:** `pipeline.llm`  
**Input:** research JSON  
**Output:** lista de ganchos rankeados (20–50), top 5 selecionados

Exemplo de gancho:

```text
"Eu não acreditava que isso funcionava — até testar por 3 dias"
```

---

### 3. Script

**Agente:** Scriptwriter  
**Fila:** `pipeline.llm`  
**Input:** produto, gancho escolhido, research  
**Output:** cenas com timing, narração, objetivo emocional

```json
{
  "scenes": [
    {
      "id": "s1",
      "startMs": 0,
      "endMs": 3000,
      "narration": "Eu não acreditava que isso funcionava.",
      "emotion": "curiosity",
      "goal": "hook"
    }
  ],
  "totalDurationMs": 45000
}
```

---

### 4. Director

**Agente:** Director  
**Fila:** `pipeline.llm`  
**Input:** script JSON  
**Output:** instruções de câmera, transição, música, ritmo por cena

```json
{
  "scenes": [
    {
      "sceneId": "s1",
      "camera": "close-up",
      "transition": { "type": "zoom", "durationMs": 500 },
      "music": { "track": "upbeat_01", "volume": 0.25 },
      "pace": "fast"
    }
  ]
}
```

Não gera mídia — apenas instruções para render e video providers.

---

### 5. Prompter

**Agente:** Prompt Engineer  
**Fila:** `pipeline.llm`  
**Input:** director + script  
**Output:** prompts finais para imagem e vídeo por cena

```json
{
  "scenes": [
    {
      "sceneId": "s1",
      "imagePrompt": "Ultra realistic UGC style, 25yo woman surprised...",
      "videoPrompt": "Woman holding product, natural lighting, TikTok style..."
    }
  ]
}
```

---

### 6. Voice

**Provider:** ElevenLabs (fixo)  
**Fila:** `pipeline.audio`  
**Input:** texto de narração limpo (Voice Script Agent ou extraído do script)  
**Output:** `narration.mp3` no S3

---

### 7. Image (opcional)

**Fila:** `pipeline.image`  
**Input:** image prompts + foto do produto (scraped)  
**Output:** `scene_N.png` no S3

Usado quando o clip de vídeo não cobre a cena ou para overlay de produto.

---

### 8. Video

**Fila:** `pipeline.video`  
**Provider:** escolhido na UI ou `VIDEO_PROVIDER_DEFAULT` no `.env`  
**Input:** video prompts por cena  
**Output:** `scene_N.mp4` no S3

Providers suportados: Kling, Runway Gen-4, Luma Dream Machine, Google Veo.

Polling assíncrono até clip ready (pode levar 5–15 min).

---

### 9. Subtitles

**Fila:** `pipeline.llm` ou worker dedicado  
**Input:** áudio gerado  
**Output:** word-level timing JSON + SRT

APIs: Whisper, Deepgram, AssemblyAI.

---

### 10. Render

**Worker:** worker-render (Node/Remotion)  
**Fila:** `pipeline.render`  
**Input:** Render Manifest JSON (director + assets + captions)  
**Output:** `draft.mp4` 9:16 no S3

Ver [RENDER.md](./RENDER.md).

---

### 11. Postprocess

**Worker:** worker-media (Go + FFmpeg)  
**Fila:** `pipeline.media` ou inline após render  
**Ações:**
- Loudness normalization (EBU R128)
- Thumbnail extraction
- Validação (duração, resolução, codec)
- Watermark (opcional)

**Output:** `final.mp4` + `thumbnail.jpg`

---

### 12. Supervisor

**Agente:** Supervisor  
**Fila:** `pipeline.supervisor`  
**Input:** todos os artefatos + metadata  
**Output:**

```json
{
  "qualityScore": 87,
  "approved": true,
  "issues": [],
  "suggestions": []
}
```

Se `approved: false`, run fica em `needs_review` no dashboard.

---

## Status de um step

```text
pending → running → done
                  → failed → (retry) → done | DLQ
                  → invalidated (edit upstream)
```

## Status de um CreativeRun

```text
draft → running → needs_review → approved → published
                → failed
```

---

## Reprocessamento

```text
User edita output_json do step 4 (director)
  → API marca steps 5–12 como invalidated
  → API enfileira step 5 (prompter)
  → pipeline continua normalmente
```

Steps 1–3 permanecem intactos.

---

## Agendamento

Campanhas podem ter `scheduledAt`. Um scheduler (cron no Go ou job RabbitMQ delayed) enfileira o CreativeRun na data/hora definida.

---

*Ver também: [AGENTS.md](./AGENTS.md), [WORKERS.md](./WORKERS.md)*
