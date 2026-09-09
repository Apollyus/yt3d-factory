# YT3D Factory

Automatizovaná produkce YouTube videí s 3D grafikou.

**Architektura:** VPS = orchestrátor (scénáře přes GLM, 3D assety přes Tripo/Marble, fronta úloh). Domácí PC s GPU = worker (Blender render, TTS, FFmpeg, YouTube upload).

- 📋 [Implementační plán](docs/plans/2026-09-09-yt3d-factory-plan.md)
- Pipeline: téma → scénář (LLM) → 3D assety (AI) → Blender render → hlas + střih → YouTube
- Maržální náklad na video: $2–8 · Worker i renderer zdarma

## Struktura
```
server/   # VPS: FastAPI fronta, script generator, asset clients, cache
worker/   # domácí PC: worker daemon, Blender build_scene, TTS, assembly, YT upload
topics/   # YAML témata videí
```

## Stavy jobu
`pending_script → generating_assets → ready_to_render → rendering → assembling → uploading → done`

Komunikace worker↔VPS přes SSH tunel + Bearer token (žádná veřejná expozice).
