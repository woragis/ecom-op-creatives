# Agentes de IA — ecom-op-creatives

Cada agente é código Go + prompt versionado + JSON schema. Sem LangChain como espinha dorsal.

---

## Princípios

1. **Agente = Execute(ctx, input) → output** — interface testável
2. **Prompts versionados** em repo (`internal/agent/*/prompts/`) ou DB com `version` field
3. **Structured output** via JSON Schema nativo (OpenAI / Anthropic)
4. **Validator Go** após cada resposta LLM
5. **Retry** 3x com backoff; falha → DLQ + status `failed` no step

---

## Agentes

| Agente | Step | LLM preferido | Input | Output |
|--------|------|---------------|-------|--------|
| **Research** | 1 | Claude / GPT | produto, URL, nicho | pains, consumerLanguage, objections, audience |
| **Hooks** | 2 | Claude | research JSON | hooks[] ranked |
| **Scriptwriter** | 3 | GPT / Claude | produto, hook, research | scenes[] com timing e narração |
| **Director** | 4 | GPT | script JSON | camera, transition, music por cena |
| **Prompter** | 5 | GPT | director + script | imagePrompt, videoPrompt por cena |
| **VoicePrep** | 6 prep | GPT | script | texto limpo para ElevenLabs |
| **Supervisor** | 12 | Claude | todos artefatos | qualityScore, approved, issues |

VoicePrep pode ser merged com Scriptwriter no MVP.

---

## Estrutura no backend

```text
backend/server/internal/agent/
├── research/
│   ├── agent.go
│   ├── schema.go
│   └── prompts/v1.md
├── hooks/
├── scriptwriter/
├── director/
├── prompter/
├── voiceprep/
├── supervisor/
└── shared/
    ├── llm/          # clients OpenAI + Anthropic
    ├── schema/       # JSON schemas compartilhados
    └── validate/     # validators comuns
```

Interface:

```go
type Agent interface {
    Name() string
    Version() string
    Execute(ctx context.Context, input json.RawMessage) (json.RawMessage, error)
}
```

---

## LLM providers

| Provider | Uso |
|----------|-----|
| OpenAI (GPT-4o, GPT-4.1) | Director, Prompter, Scriptwriter |
| Anthropic (Claude) | Research, Hooks, Supervisor |

Configuração por env:

```env
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
LLM_DEFAULT_PROVIDER=openai
```

Override por agente (opcional):

```env
AGENT_RESEARCH_PROVIDER=anthropic
AGENT_DIRECTOR_PROVIDER=openai
```

---

## Exemplo: Research Agent output

```json
{
  "pains": [
    "cabos emaranhados na mesa",
    "carregador cai no chão"
  ],
  "consumerLanguage": [
    "game changer",
    "não sabia que precisava"
  ],
  "objections": [
    "parece frágil"
  ],
  "targetAudience": "home office, gamers",
  "emotionalTriggers": ["organização", "produtividade"],
  "competitors": ["...", "..."]
}
```

---

## Exemplo: Director Agent output

```json
{
  "scenes": [
    {
      "sceneId": "s1",
      "camera": "close-up",
      "transition": { "type": "zoom", "durationMs": 500 },
      "music": { "track": "upbeat_01", "volume": 0.25 },
      "pace": "fast",
      "captionStyle": "tiktok-bold"
    }
  ],
  "format": { "width": 1080, "height": 1920, "fps": 30 }
}
```

---

## Supervisor Agent

Analisa o criativo completo antes de `ready_for_review`:

```json
{
  "qualityScore": 87,
  "approved": true,
  "issues": [],
  "suggestions": ["gancho poderia ser mais curto"],
  "checks": {
    "hookStrength": 90,
    "audioQuality": 95,
    "visualConsistency": 80,
    "subtitleSync": 88
  }
}
```

Threshold configurável: `SUPERVISOR_MIN_SCORE=75`

---

## Versionamento de prompts

Cada prompt tem versão no path ou metadata:

```text
prompts/research/v1.md
prompts/research/v2.md
```

CreativeRun armazena `agentVersion` por step para reprodutibilidade.

---

## Custo tracking

Cada chamada LLM registra em `api_costs`:

```text
creative_run_id, step_type, provider, model, tokens_in, tokens_out, cost_usd
```

---

*Ver também: [RESEARCH.md](./RESEARCH.md), [PIPELINE.md](./PIPELINE.md)*
