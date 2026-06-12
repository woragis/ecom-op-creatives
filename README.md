# ecom-op-creatives

Plataforma de geração automatizada de criativos para e-commerce (dropshipping, afiliados, conteúdo viral).

Pipeline completo: pesquisa de produto → ganchos → roteiro → direção → mídia (voz, imagem, vídeo) → render UGC 9:16 → dashboard editável.

## Repositórios

Este repositório é o **monorepo raiz** com submodules:

| Path | Repositório |
|------|-------------|
| `backend/` | [ecom-op-creatives-backend](https://github.com/woragis/ecom-op-creatives-backend) |
| `frontend/` | [ecom-op-creatives-frontend](https://github.com/woragis/ecom-op-creatives-frontend) |

## Clone

```bash
git clone --recurse-submodules git@github.com:woragis/ecom-op-creatives.git
cd ecom-op-creatives
```

Se já clonou sem submodules:

```bash
git submodule update --init --recursive
```

## Stack

| Camada | Tecnologia |
|--------|------------|
| API | Go (stdlib HTTP, GORM, PostgreSQL) |
| Filas | RabbitMQ |
| Workers | Go (LLM, media) + Node (Remotion render) |
| Áudio | ElevenLabs |
| Vídeo | Kling, Runway, Luma, Veo (plugável) |
| Render | Remotion + FFmpeg |
| Research | Serper + LLM |
| Frontend | Next.js |
| Storage | S3/MinIO |

## Documentação

- [ARCHITECTURE.md](./ARCHITECTURE.md) — visão geral do sistema
- [docs/PIPELINE.md](./docs/PIPELINE.md) — etapas do pipeline
- [docs/AGENTS.md](./docs/AGENTS.md) — agentes de IA
- [docs/WORKERS.md](./docs/WORKERS.md) — arquitetura de workers e filas
- [docs/RENDER.md](./docs/RENDER.md) — Remotion + FFmpeg
- [docs/RESEARCH.md](./docs/RESEARCH.md) — pesquisa de produto e nicho

## Estrutura

```text
creatives/
├── ARCHITECTURE.md
├── docs/
├── backend/          # submodule
├── frontend/         # submodule
└── docker-compose.yml
```

## Docker (stack completo)

```bash
cp .env.example .env
# Edite .env com suas API keys (veja comentários no arquivo)

docker compose up -d --build
```

| Serviço | URL |
|---------|-----|
| Dashboard | http://localhost:3000 |
| API | http://localhost:8080 |
| RabbitMQ UI | http://localhost:15672 (guest/guest) |

Media gerada fica em `./storage/` (volume compartilhado entre API e worker).

## Desenvolvimento local (sem Docker nos apps)

```bash
# Só infra
docker compose up -d postgres rabbitmq

# Backend (ver backend/README.md)
cd backend && make dev

# Frontend (ver frontend/README.md)
cd frontend && npm run dev
```

## Licença

Proprietary — woragis
