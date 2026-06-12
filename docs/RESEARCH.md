# Research — pesquisa de produto e nicho

Pipeline de research usando Serper + extração de páginas + LLM synthesis.

---

## Objetivo

Antes de escrever ganchos e roteiro, entender:

- Dores reais do consumidor
- Linguagem que o público usa
- Objeções comuns
- Concorrentes e posicionamento
- Triggers emocionais do nicho

---

## Fluxo

```text
Input: productName, productUrl (opcional), niche
       │
       ▼
1. LLM gera 6–10 queries de busca
       │
       ▼
2. Serper API → resultados Google (snippets + URLs)
       │
       ▼
3. Fetch top URLs (Jina Reader ou Firecrawl)
       │
       ▼
4. LLM sintetiza → Research JSON
       │
       ▼
Output: pains, consumerLanguage, objections, audience, competitors
```

---

## Serper

API de resultados Google. ~$0.001/query.

```env
SERPER_API_KEY=
```

### Queries geradas (exemplo)

Produto: "Organizador de cabos magnético"

```text
organizador cabos magnético review
organizador cabos problemas reddit
cable organizer tiktok
organizador cabos vale a pena
cable management desk pain points
organizador cabos amazon reviews 1 star
```

LLM gera queries adaptadas ao produto e nicho.

---

## Extração de conteúdo

| Ferramenta | Uso |
|------------|-----|
| **Jina Reader** | `https://r.jina.ai/{url}` — texto limpo |
| **Firecrawl** | scrape estruturado (alternativa) |
| Snippets Serper | suficiente para muitos casos |

Cache: mesma query + produto → não refaz Serper (PostgreSQL ou Redis TTL 7d).

---

## Research Agent output

```json
{
  "pains": [
    "cabos emaranhados na mesa",
    "carregador cai no chão",
    "mesa bagunçada em videochamada"
  ],
  "consumerLanguage": [
    "game changer",
    "não sabia que precisava",
    "minha mesa nunca mais foi a mesma"
  ],
  "objections": [
    "parece frágil",
    "não gruda em superfície texturizada"
  ],
  "competitors": ["...", "..."],
  "targetAudience": "home office, gamers, estudantes",
  "emotionalTriggers": ["organização", "produtividade", "estética"],
  "sources": [
    { "url": "...", "snippet": "..." }
  ]
}
```

Hook Agent usa `consumerLanguage` e `pains` para copy autêntico.

---

## Custo estimado

| Item | Custo por run |
|------|---------------|
| Serper 8 queries | ~$0.008 |
| Jina Reader 5 URLs | ~$0.005 |
| LLM synthesis | ~$0.02 |
| **Total** | **~$0.03** |

---

## Alternativas

| API | Quando usar |
|-----|-------------|
| Tavily | otimizado para LLM, menos controle |
| Exa | busca semântica |
| Brave Search API | alternativa ao Serper |

Serper permanece default — experiência existente, resultados previsíveis.

---

## Implementação (worker-llm)

```text
internal/agent/research/
├── agent.go
├── serper.go
├── fetch.go        # Jina Reader
├── synthesize.go   # LLM call
└── prompts/v1.md
```

Fila: `pipeline.research` (pode ser mais lenta que LLM puro por causa de HTTP externo).

---

*Ver também: [AGENTS.md](./AGENTS.md), [PIPELINE.md](./PIPELINE.md)*
