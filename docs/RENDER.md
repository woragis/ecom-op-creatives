# Render — Remotion + FFmpeg

Composição UGC 9:16 e pós-processamento do vídeo final.

---

## Decisão

| Camada | Ferramenta | Worker |
|--------|------------|--------|
| Composição (legendas animadas, transições, layout) | **Remotion** | worker-render (Node) |
| Pós-process (loudness, thumbnail, validação) | **FFmpeg** | worker-media (Go) |

Remotion usa FFmpeg internamente para encode. FFmpeg no Go worker faz o polish final.

---

## Fluxo

```text
Assets (clips, áudio, captions JSON)
       │
       ▼
Render Manifest JSON (Director + assets)
       │
       ▼
worker-render (Remotion) → draft.mp4
       │
       ▼
worker-media (FFmpeg) → final.mp4 + thumbnail.jpg
```

---

## Render Manifest JSON

Contrato entre Director Agent e template Remotion:

```json
{
  "format": {
    "width": 1080,
    "height": 1920,
    "fps": 30,
    "durationMs": 45000
  },
  "scenes": [
    {
      "id": "s1",
      "startMs": 0,
      "durationMs": 3000,
      "videoUrl": "s3://.../scene_1.mp4",
      "transition": { "type": "zoom", "durationMs": 500 }
    }
  ],
  "captions": {
    "style": "tiktok-bold",
    "words": [
      { "text": "Eu", "startMs": 0, "endMs": 200 },
      { "text": "não", "startMs": 200, "endMs": 400 }
    ]
  },
  "audio": {
    "narrationUrl": "s3://.../narration.mp3",
    "music": { "track": "upbeat_01", "volume": 0.25 }
  },
  "brand": {
    "productImageUrl": "s3://.../product.png"
  }
}
```

---

## Remotion (worker-render)

### Stack

- Node.js 20+
- Remotion 4.x
- React 18
- `@remotion/lambda` (opcional, fase 2 — escala AWS)

### Template UGC

```text
worker-render/remotion/
├── src/
│   ├── Root.tsx
│   ├── compositions/
│   │   └── UGCVertical.tsx    # 1080x1920
│   ├── components/
│   │   ├── AnimatedCaptions.tsx
│   │   ├── SceneClip.tsx
│   │   └── ProductOverlay.tsx
│   └── manifest.ts            # parse Render Manifest
└── package.json
```

### Job flow

```text
1. Consume pipeline.render
2. Download assets from S3 to /tmp
3. bundle Remotion + render frames
4. Upload draft.mp4 to S3
5. Publish postprocess job OR update step done
```

### Performance

- ~1–5 min para 60s de vídeo (depende de CPU)
- ~1–2 GB RAM por job
- Escala: múltiplos worker-render ou Remotion Lambda

---

## FFmpeg (worker-media)

Executado após Remotion:

```bash
# Loudness normalization
ffmpeg -i draft.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11 final.mp4

# Thumbnail
ffmpeg -ss 00:00:03 -i final.mp4 -vframes 1 thumbnail.jpg

# Validação (ffprobe)
ffprobe -v error -show_entries format=duration -of json final.mp4
```

Go invoca via `os/exec` ou wrapper `ffmpeg-go`.

---

## Fallback FFmpeg-only (MVP opcional)

Se Remotion indisponível, pipeline degradado:

- Concat clips com xfade
- Legendas SRT/ASS estáticas (sem animação palavra a palavra)
- Menos "UGC TikTok", mais rápido e barato

Útil para MVP ou disaster recovery.

---

## Por que Node para render (não Go/Rust)

| Opção | UGC animado | Maturidade |
|-------|-------------|------------|
| Remotion (Node) | Excelente | Maduro |
| Go + FFmpeg | Fraco (ASS manual) | Ótimo para concat |
| Rust + ffmpeg-next | Fraco | Curva alta |

Node fica **isolado** em worker-render. Backend Go não depende de Node no API server.

---

## Variáveis

```env
# worker-render
REMOTION_CONCURRENCY=2
S3_BUCKET=creatives
AWS_REGION=us-east-1

# worker-media (FFmpeg)
FFMPEG_PATH=/usr/bin/ffmpeg
FFPROBE_PATH=/usr/bin/ffprobe
```

---

*Ver também: [PIPELINE.md](./PIPELINE.md), [WORKERS.md](./WORKERS.md)*
