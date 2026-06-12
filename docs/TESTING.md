# Testing — primeiro run pago (Runway)

Checklist para evitar runs desorganizados e gasto desnecessário.

## Pré-requisitos

- [ ] `docker compose up -d --build`
- [ ] `.env` com keys reais (não commitar)
- [ ] Postgres em `./data/postgres` (bind mount)
- [ ] `VIDEO_PROVIDER_DEFAULT=runway`
- [ ] `VIDEO_MOCK=0`
- [ ] `IMAGE_MAX_SCENES=1` e `VIDEO_MAX_SCENES=1` no primeiro teste
- [ ] `PAUSE_BEFORE_VIDEO=1` — pausa após `image` para revisar antes do vídeo pago

## Convenção de nome

Produto na UI:

```text
[TEST-001] Nome do Produto — runway-1scene
```

## Fluxo recomendado

1. Criar produto com nome `[TEST-001] ...`
2. Upload `persona` + `product` (opcional `intro`)
3. Criar run → provider **runway** + **dalle**
4. **Start**
5. Acompanhar: `docker compose logs -f worker-pipeline`
6. Após step `image` → run em `needs_review` → revisar `storage/runs/{id}/artifacts/`
7. **Continue** na UI (ou `POST /v1/creative-runs/{id}/continue`) → dispara vídeo Runway
8. Se falhar em `video` → **Retry step** (não criar run novo)

## Monitoramento

| O quê | Onde |
|-------|------|
| Logs estruturados | `docker compose logs -f worker-pipeline` |
| JSON por step | `storage/runs/{id}/artifacts/` |
| Mídia | `storage/runs/{id}/` |
| DB | `docker compose exec postgres psql -U creatives -c "SELECT step_type, status, error_message FROM pipeline_steps WHERE creative_run_id='...';"` |

## Custos por provider

| Step | Provider | Conta |
|------|----------|-------|
| LLM (1–5, 12) | OpenAI | OpenAI |
| Research web | Serper | Serper |
| Voice | OpenAI TTS | OpenAI |
| Image | DALL·E | OpenAI |
| Subtitles | Whisper | OpenAI |
| **Video** | **Runway** | **Runway** |
| Render/FFmpeg | local | grátis |

## image2video

Se o director escolher `image2video`, `API_PUBLIC_URL` precisa ser URL pública (ngrok). Para o 1º teste, prefira **text2video** (1 cena).
