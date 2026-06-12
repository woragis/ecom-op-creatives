# E2E fixtures

Optional sample assets for manual or automated end-to-end runs. **Do not commit large files or secrets.**

## Suggested layout

```text
e2e/fixtures/
├── product.json          # sample product metadata (committed)
├── persona.jpg           # UGC creator face — you add (~512×512+)
├── product.jpg           # product photo — you add (~768×768+)
└── intro.mp4             # optional 9:16 hook clip (~2–3s)
```

## product.json

Use this when creating a product via API or UI:

```json
{
  "name": "Portable Neck Fan",
  "url": "https://example.com/neck-fan",
  "niche": "summer gadgets"
}
```

## Images

| File | Purpose | Notes |
|------|---------|--------|
| `persona.jpg` | Upload as **persona** asset | Clear face, good lighting, vertical-friendly |
| `product.jpg` | Upload as **product** asset | Product on plain background works best |
| `intro.mp4` | Upload as **intro** clip | 9:16, 2–3 seconds; drives `INTRO_DURATION_MS` offset |

You can use any royalty-free stock photos for testing. The pipeline reads uploaded files from the run detail page — nothing is auto-loaded from this folder unless you upload or wire a seed script.

## Real AI tips

1. Start with mocks off in `.env` (see root `.env.example`).
2. Use **text2video** first (no public image URL required).
3. For **image2video**, set `API_PUBLIC_URL` to a tunnel URL (ngrok) so Kling/Runway can fetch your images.
4. Keep `VIDEO_MAX_SCENES=1` and `IMAGE_MAX_SCENES=1` for cheaper first tests.
