# YT3D Factory — Implementační plán

> **For Hermes:** Použij skill `subagent-driven-development` pro implementaci task po tasku (fresh subagent per task, two-stage review).

**Goal:** Automatizovaná produkce YouTube videí s propracovanou 3D grafikou — od tématu po upload — s renderem výhradně na domácím PC (worker), VPS jako orchestrátor.

**Architecture:** VPS (always-on, 4 CPU, bez GPU) běží orchestrátor: FastAPI + SQLite fronta úloh, generuje scénáře (GLM/zai), objednává 3D assety (Tripo, Marble) a drží je v cache. Domácí PC s GPU běží worker daemon, který si stahuje render joby, renderuje v Blenderu headless, skládá video v FFmpeg, nahrává na YouTube a reportuje zpět. Hermes cron na VPS spouští celý řetězec a notifikuje přes Telegram.

**Tech Stack:** Python 3.11, FastAPI, SQLite (stdlib), httpx, Blender 4.x headless (bpy), FFmpeg, edge-tts, YouTube Data API v3, Tripo API, Marble (World Labs) API.

**Rozpočet:** Tripo Pro $20/měs (~200 modelů), Marble Pro $35/měs (~25 scén, komerční licence) — nebo Marble API pay-as-you-go $1/1250 kreditů. LLM ~centy. Render/TTS/střih zdarma. Maržální náklad na video: **$2–8**.

---

## Architektura

```
┌─────────────────────── VPS (vps-ae59d428, orchestrátor) ────────────────┐
│                                                                          │
│  Hermes cron (týdně) ──► make_job.py ──► jobs.db (SQLite)                │
│                                              │                           │
│  FastAPI :8787  ◄── worker polling ──────────┤                           │
│   GET  /jobs/next        (worker si vyzvedne job)                        │
│   POST /jobs/{id}/status (progress reporting)                            │
│   GET  /jobs/{id}/bundle (pipeline.json + assety .glb)                   │
│   POST /jobs/{id}/video  (upload hotového videa)                         │
│                                                                          │
│  Script generator ──► GLM (zai API)                                      │
│  Asset service ─────► Tripo API (objekty) + Marble API (prostředí)       │
│  Asset cache ───────► /home/ubuntu/projects/yt3d-factory/server/cache/   │
└──────────────────────────────────────────────────────────────────────────┘
                                    │ HTTPS + token
┌─────────────────────── Domácí PC (worker, GPU) ─────────────────────────┐
│  worker.py (daemon / Task Scheduler)                                     │
│   1. poll VPS → stáhne job bundle                                        │
│   2. blender -b -P build_scene.py  (scéna z pipeline.json)               │
│   3. render per-shot (Cycles GPU / EEVEE Next)                           │
│   4. edge-tts voiceover + ffprobe timing                                 │
│   5. ffmpeg: skládání, titulky, hudba                                    │
│   6. upload videa na VPS → YouTube API (unlisted první)                  │
│   7. status report → VPS → Telegram                                      │
└──────────────────────────────────────────────────────────────────────────┘
```

**Klíčové kontrakty:** jediný datový formát mezi všemi stádií je `pipeline.json` (schema níže). Vše ostatní jsou implementace.

---

## pipeline.json — centrální schema

```json
{
  "job_id": "20260909_tidal_energy",
  "topic": "Jak funguje přílivová elektrárna",
  "language": "cs",
  "title": "Jak funguje přílivová elektrárna | 3D vizualizace",
  "description": "...",
  "tags": ["energetika", "3d", "vzdělávání"],
  "youtube": {"privacy": "unlisted", "category_id": "27"},
  "style": {"look": "clean-educational", "resolution": [1920, 1080], "fps": 30},
  "assets": [
    {"id": "dam", "type": "object", "prompt": "concrete tidal barrage dam, industrial, cross-section", "source": "tripo", "status": "ready", "file": "assets/dam.glb"},
    {"id": "bay", "type": "world", "prompt": "coastal bay with rocky cliffs, overcast daylight", "source": "marble", "status": "pending", "file": "assets/baby.ply"}
  ],
  "shots": [
    {
      "id": "s01",
      "narration": "Přílivová elektrárna využívá rozdíl hladin moře a zálivu...",
      "duration_s": 8.0,
      "camera": {"position": [0, -30, 10], "target": [0, 0, 5], "move": "dolly-in", "move_to": [0, -20, 8]},
      "lighting": {"type": "sun", "energy": 3.0, "angle": 45},
      "objects": [{"asset": "dam", "location": [0, 0, 0], "scale": 1.0, "rotation_z": 0}],
      "world": "bay",
      "overlay": {"title": "Přílivová elektrárna", "subtitle_file": "subs/s01.srt"}
    }
  ],
  "music": {"track": "music/calm-loop.mp3", "volume": 0.15}
}
```

Stavy jobu: `pending_script → generating_assets → ready_to_render → rendering → assembling → uploading → done` (+ `failed:<stage>`).

---

## Fáze 0 — Repo a skeleton

### Task 1: Struktura repozitáře
**Objective:** Vytvořit adresářovou strukturu a konfiguraci projektu.

**Files:**
```
yt3d-factory/
├── docs/plans/                    # tento plán
├── server/                        # VPS orchestrátor
│   ├── app.py                     # FastAPI
│   ├── db.py                      # SQLite fronta
│   ├── config.py                  # načítání .env
│   ├── script_gen.py              # GLM scénář
│   ├── assets_tripo.py            # Tripo klient
│   ├── assets_marble.py           # Marble klient
│   ├── make_job.py                # CLI: vytvoř job z tématu
│   └── cache/                     # stažené assety (gitignore)
├── worker/                        # domácí PC
│   ├── worker.py                  # daemon: poll→render→upload
│   ├── blender/build_scene.py     # bpy scene builder + render
│   ├── tts.py                     # edge-tts
│   ├── assemble.py                # ffmpeg skládání
│   └── youtube_upload.py          # YT Data API
├── topics/                        # YAML témata
│   └── priklad.yaml
├── tests/
└── README.md
```

**Step 1:** Vytvořit `.gitignore`:
```
server/cache/
server/.venv/
worker/output/
worker/.venv/
__pycache__/
*.blend1
.env
```

**Step 2:** Commit a push přes GitHub MCP (`push_files`).

**Verify:** `git ls-files` ukazuje strukturu.

---

## Fáze 1 — VPS orchestrátor

### Task 2: DB vrstva s frontou úloh
**Objective:** SQLite job store s atomickým claimem pro workery.

**Files:** Create `server/db.py`, Test `tests/test_db.py`

**Step 1: Failing test**
```python
# tests/test_db.py
import tempfile, os
from server.db import JobDB

def test_claim_is_atomic():
    db = JobDB(tempfile.mktemp(suffix=".db"))
    jid = db.create_job({"topic": "test"})
    a = db.claim_next_job(worker="w1")
    b = db.claim_next_job(worker="w2")
    assert a and a["job_id"] == jid and b is None  # druhý claim dostane nic
```

**Step 2:** `pytest tests/test_db.py -v` → FAIL (modul neexistuje)

**Step 3: Implementace**
```python
# server/db.py
import sqlite3, json, time, os

SCHEMA = """
CREATE TABLE IF NOT EXISTS jobs (
  job_id TEXT PRIMARY KEY,
  state TEXT NOT NULL DEFAULT 'pending_script',
  pipeline TEXT,             -- JSON
  worker TEXT,
  claimed_at REAL,
  updated_at REAL,
  error TEXT
);
CREATE TABLE IF NOT EXISTS events (
  ts REAL, job_id TEXT, level TEXT, message TEXT
);
"""

class JobDB:
    def __init__(self, path):
        os.makedirs(os.path.dirname(path), exist_ok=True)
        self.conn = sqlite3.connect(path, check_same_thread=False)
        self.conn.row_factory = sqlite3.Row
        self.conn.executescript(SCHEMA)

    def create_job(self, pipeline: dict) -> str:
        jid = pipeline.get("job_id") or time.strftime("%Y%m%d_%H%M%S")
        self.conn.execute(
            "INSERT OR REPLACE INTO jobs (job_id, state, updated_at) VALUES (?, 'pending_script', ?)",
            (jid, time.time()))
        self.conn.commit()
        return jid

    def set_pipeline(self, jid, pipeline: dict):
        self.conn.execute("UPDATE jobs SET pipeline=?, state='ready_to_render', updated_at=? WHERE job_id=?",
                          (json.dumps(pipeline), time.time(), jid))
        self.conn.commit()

    def claim_next_job(self, worker: str):
        # atomicky: jen ready_to_render a neclaimované
        cur = self.conn.execute(
            "UPDATE jobs SET state='rendering', worker=?, claimed_at=?, updated_at=? "
            "WHERE job_id = (SELECT job_id FROM jobs WHERE state='ready_to_render' AND worker IS NULL "
            "ORDER BY updated_at LIMIT 1) RETURNING *", (worker, time.time(), time.time()))
        row = cur.fetchone()
        self.conn.commit()
        return dict(row) if row else None

    def set_state(self, jid, state, error=None):
        self.conn.execute("UPDATE jobs SET state=?, error=?, updated_at=? WHERE job_id=?",
                          (state, error, time.time(), jid))
        self.conn.commit()

    def log(self, jid, level, message):
        self.conn.execute("INSERT INTO events VALUES (?,?,?,?)", (time.time(), jid, level, message))
        self.conn.commit()
```

**Step 4:** `pytest tests/test_db.py -v` → PASS

**Step 5:** Commit `feat: job queue with atomic worker claim`

### Task 3: Config + auth token
**Objective:** Jednoduché načtení tajenek z `.env`, statický token pro worker.

**Files:** Create `server/config.py`

```python
import os
from dotenv import load_dotenv
load_dotenv(os.path.expanduser("~/projects/yt3d-factory/.env"))

WORKER_TOKEN = os.environ["YT3D_WORKER_TOKEN"]     # vygeneruj: python -c "import secrets;print(secrets.token_hex(24))"
GLM_API_KEY   = os.environ["GLM_API_KEY"]
GLM_BASE_URL  = "https://api.z.ai/api/paas/v4"
GLM_MODEL     = "glm-5.3-flash"
TRIPO_API_KEY = os.environ["TRIPO_API_KEY"]
MARBLE_API_KEY = os.getenv("MARBLE_API_KEY", "")   # volitelné do Fáze 3b
DATA_DIR      = os.path.expanduser("~/projects/yt3d-factory/server/cache")
```

**Step:** `python -c "from server import config"` po přidání klíčů do `.env` → bez chyby. Token vygenerovat a zapsat. Commit.

### Task 4: FastAPI service
**Objective:** HTTP API pro workera (claim, bundle download, status, video upload) s token auth.

**Files:** Create `server/app.py`, Test `tests/test_api.py`

**Step 1: Test** — httpx TestClient, auth 401 bez tokenu, claim vrátí job, video upload uloží soubor.

**Step 2: Implementace**
```python
# server/app.py
from fastapi import FastAPI, Depends, HTTPException, Header, UploadFile, File
from server import config
from server.db import JobDB

app = FastAPI(title="yt3d-factory")
db = JobDB(f"{config.DATA_DIR}/jobs.db")

def auth(authorization: str = Header("")):
    if authorization != f"Bearer {config.WORKER_TOKEN}":
        raise HTTPException(401, "bad token")

@app.get("/jobs/next", dependencies=[Depends(auth)])
def next_job(worker: str):
    return db.claim_next_job(worker) or {"job_id": None}

@app.get("/jobs/{jid}/bundle", dependencies=[Depends(auth)])
def bundle(jid: str):
    # zip: pipeline.json + assets/ (ze server/cache/<jid>/)
    ...

@app.post("/jobs/{jid}/status", dependencies=[Depends(auth)])
def status(jid: str, state: str, error: str = None):
    db.set_state(jid, state, error); db.log(jid, "info", f"state={state}")
    return {"ok": True}

@app.post("/jobs/{jid}/video", dependencies=[Depends(auth)])
async def video(jid: str, file: UploadFile = File(...)):
    out = f"{config.DATA_DIR}/{jid}/final.mp4"
    with open(out, "wb") as f: f.write(await file.read())
    db.set_state(jid, "done")
    return {"ok": True, "path": out}
```

**Step 3:** Ověřit volný port: `ss -tlnp | grep 8787` (musí být prázdné). Spustit `uvicorn server.app:app --host 127.0.0.1 --port 8787` a `curl -H "Authorization: Bearer <token>" http://127.0.0.1:8787/jobs/next?worker=test` → `{"job_id": null}`.

**Step 4:** systemd unit `yt3d-api.service` (vzor: existující `hermes-gateway.service`, ExecStart uvicorn, User=ubuntu, Restart=always). `sudo systemctl enable --now yt3d-api`. **Pozor:** API bindovat jen na 127.0.0.1; worker k němu přistupuje přes SSH tunel (`ssh -L 8787:127.0.0.1:8787 vps`), žádná veřejná expozice.

**Verify:** `systemctl is-active yt3d-api` → active; curl z lokálu → OK. Commit.

### Task 5: Notifikace do Telegramu
**Objective:** Stav jobů se objeví v Telegramu (worker online, hotovo, chyba).

**Files:** Create `server/notify.py`

**Step:** Použít Hermes webhook subscription (`hermes webhook subscribe yt3d`) NEBO jednodušeji: `make_job.py` a app.py volají python funkci, která spustí `hermes -z "YT3D: job {jid} → {state}"`. Overhead malý, spolehlivé.

**Verify:** Jeden test job → zpráva v Telegramu. Commit.

---

## Fáze 2 — Generování scénáře

### Task 6: Topic soubory
**Objective:** Deklarativní vstup — YAML s tématem a pokyny.

**Files:** Create `topics/priklad.yaml`

```yaml
topic: "Jak funguje přílivová elektrárna"
audience: "laici, 8-12 min delší formát NEBO 60s short"
target_duration_s: 60
extra_guidance: "důraz na vizuál řezu konstrukcí, voda by měla být animovaná"
```

### Task 7: Script generator (GLM → pipeline.json)
**Objective:** LLM vygeneruje kompletní pipeline.json: scény, narraci, kamerové pohyby, seznam assetů s prompty.

**Files:** Create `server/script_gen.py`, Test `tests/test_script_gen.py` (s mockovaným API)

**Step 1:** Test parsuje fixturovou odpověď LLM do validního pipeline.json (pydantic model `Pipeline`).

**Step 2:** Implementace — volání GLM chat completions (openai-kompatibilní, `config.GLM_BASE_URL`) se system promptem obsahujícím JSON schema; odpověď `response_format: json_object`; validace pydantic; `assets` rozdělené na `type: object` (Tripo) a `type: world` (Marble). Druhý průchod: kontrola, že každý shot referencuje existující asset id (self-heal prompt nebo drop shotu).

**Step 3:** `make_job.py --topic topics/priklad.yaml` → vytvoří job v DB, spustí script_gen, uloží pipeline.json do `server/cache/<jid>/`.

**Verify:** `sqlite3 cache/jobs.db "select state from jobs"` → `ready_to_render`; pipeline.json prochází validací. Commit.

---

## Fáze 3 — 3D assety

### Task 8: Tripo klient (objekty)
**Objective:** image→3D s texturou přes REST, polling, download GLB do cache.

**Files:** Create `server/assets_tripo.py`, Test `tests/test_tripo.py` (mockovaný httpx)

**Klíčové detaily (dle oficiální dokumentace, zkontrolovat při implementaci):**
- `POST https://api.tripo3d.ai/v2/openapi/task` body `{"type": "image_to_model", "file": {"image_data_url": "<data-uri>"}, "texture": true, "model_version": "V3.1-20250612"}` — header `Authorization: Bearer <TRIPO_API_KEY>`
- Konceptní obrázek vygenerovat také Tripem (`text_to_image`, 5 kreditů) — drží vizuální styl konzistentní
- Polling: `GET /v2/openapi/task/{task_id}` každých 5 s, `progress` 0–100, stav `success` → `output.pbr_model` URL
- Download GLB → `server/cache/<jid>/assets/<asset_id>.glb`
- Cena: ~25–35 kreditů/asset → **$0,25–0,35**
- Retry na 429 s exponential backoff; webhook nemusíme řešit (polling jednodušší pro dávkový běh)

**Verify:** Unit test s mockem; pak JEDEN živý test na malý asset (spotřebuje ~35 kreditů). Commit.

### Task 9: Marble klient (prostředí) — volitelný flag
**Objective:** World→scéna přes World API; asset `type: world`.

**Files:** Create `server/assets_marble.py`

**Poznámky:** API ve tvaru `$1.00 / 1250 kreditů`; endpointy ověřit dle docs (worldlabs.ai) při implementaci — schema klienta nechat stejný jako Tripo (create→poll→download .ply/.spz). Plán počítá s tím, že Marble až ve druhé vlně: PoC běží s jednoduchým procedural sky+HDRI světem v Blenderu místo Marble světa (`style.world_fallback: "procedural_hdri"`).

**Verify:** Mock test; živý test až s Marble přístupem. Commit.

---

## Fáze 4 — Worker (domácí PC)

### Task 10: Worker daemon
**Objective:** Cross-platform Python daemon: poll → download bundle → render → assemble → upload → status.

**Files:** Create `worker/worker.py`

```python
# worker/worker.py — jádro smyčky
import time, httpx, subprocess, zipfile, io, sys
API = "http://127.0.0.1:8787"  # přes SSH tunel
TOKEN = open("worker_token.txt").read().strip()
H = {"Authorization": f"Bearer {TOKEN}"}

def loop():
    while True:
        job = httpx.get(f"{API}/jobs/next", params={"worker": "home-pc"}, headers=H).json()
        if job.get("job_id"):
            try:
                process(job["job_id"])
            except Exception as e:
                httpx.post(f"{API}/jobs/{job['job_id']}/status", headers=H,
                           params={"state": f"failed:render", "error": str(e)[:500]})
        time.sleep(30 if not job.get("job_id") else 2)

def process(jid):
    z = httpx.get(f"{API}/jobs/{jid}/bundle", headers=H).content
    zipfile.ZipFile(io.BytesIO(z)).extractall(f"output/{jid}/")
    subprocess.run([sys.executable, "blender/build_scene.py", f"output/{jid}"], check=True)
    subprocess.run([sys.executable, "assemble.py", f"output/{jid}"], check=True)
    httpx.post(f"{API}/jobs/{jid}/video", headers=H,
               files={"file": open(f"output/{jid}/final.mp4", "rb")})
```

**Instalace na PC:**
- Linux: systemd user unit `yt3d-worker.service` (After=network-online, Restart=always) + `autossh -N -L 8787:127.0.0.1:8787 vps` jako závislost
- Windows: Task Scheduler "Spustit při přihlášení" + plink/autossh tunel, nebo `nssm` služba
- Worker token pouze na PC (nikdy v repu)

**Verify:** Se spuštěným API na VPS a jedním jobem `ready_to_render` worker stáhne bundle a zavolá build_scene. Commit.

### Task 11: Blender scene builder + render
**Objective:** `build_scene.py` (běží uvnitř Blenderu): postaví scénu per shot, renderuje klipy.

**Files:** Create `worker/blender/build_scene.py`

**Klíčové detaily:**
- Spuštění: `blender -b -P build_scene.py -- <job_dir>` (args za `--`)
- Import GLB: `bpy.ops.import_scene.gltf(filepath=...)`
- GPU: `prefs.addons['cycles'].preferences.compute_device_type = 'OPTIX'  # nebo 'CUDA'/'NONE'`; `scene.cycles.device='GPU'`; samples 128 + denoiser (OpenImageDenoise); fallback EEVEE Next (`scene.render.engine='BLENDER_EEVEE_NEXT'`) pokud GPU chybí — testovat první spuštění s `--quick` (64 samples, 50% resolution)
- Kamera per shot: location/look-at z pipeline.json, pohyb `dolly-in` = interpolace location mezi `position`→`move_to` přes shot duration (keyframes frame 1→N, ease in-out)
- World fallback: `procedural_hdri` = Nishita sky texture + slunce dle `lighting`
- Voda/animované materiály: Fáze 2 rozšíření — animated noise texture na materialu; v PoC statická
- Render: `scene.render.filepath = f"shots/{shot_id}.mp4"`, `image_settings.file_format='FFMPEG'`, codec H.264, `ffmpeg.format='MPEG4'`, constant rate factor přes `ffmpeg.constant_rate_factor='HIGH'`

**Verify:** `blender -b -P build_scene.py -- output/test --quick` → `shots/s01.mp4` existuje a má správné FPS/rozlišení (ffprobe). Commit.

### Task 12: TTS voiceover
**Objective:** Narrace per shot → mp3 + přesné délky pro timing.

**Files:** Create `worker/tts.py`

**Detaily:** `edge-tts` (zdarma, CS hlasy — `cs-CZ-AntoninNeural` muž / `cs-CZ-VlastaNeural` žena). Každý shot → `audio/s01.mp3`; délku zjistit `ffprobe -v quiet -show_entries format=duration`. Pokud narrace delší než `duration_s`, upravit shot délku v `render_plan.json` (mezivýstup pro build_scene) — narrace řídí timing, ne naopak. Rate ~1.0, případně `--rate=+5%` pro zrychlení.

**Verify:** `python tts.py output/test` → mp3 soubory + render_plan.json s upravenými délkami. Commit.

### Task 13: FFmpeg assembly
**Objective:** Slepit klipy + hlas + hudba + titulky → final.mp4.

**Files:** Create `worker/assemble.py`

**Detaily:**
1. concat klipů (identické parametry): `ffmpeg -f concat -i list.txt -c copy shots.mp4`
2. mix audio: `[narrace seq]` + hudba (amix, hudba volume 0.15, aloop pro délku): `ffmpeg -i shots.mp4 -i music.mp3 -filter_complex "[1:a]aloop=loop=-1:size=2e+09,volume=0.15,atrim=0:DUR[m];[narr...][m]amix=inputs=2:duration=first" out.mp4`
3. burn-in titulky: `subtitles=subs.srt:force_style='FontName=Inter,FontSize=20'` (srt generovat z narration per shot s timingem)
4. loudnorm na audio (`-af loudnorm=I=-16:TP=-1.5`) — YouTube standard

**Verify:** final.mp4: délka ≈ sum(shots), audio track existuje, spustitelný v přehrávači. Commit.

---

## Fáze 5 — YouTube

### Task 14: Upload na YouTube
**Objective:** Z workeru upload final.mp4 (unlisted první, ruční review před public).

**Files:** Create `worker/youtube_upload.py`

**Detaily:**
- Google Cloud projekt + OAuth consent (external, test users = vlastní účet) + API "YouTube Data v3" enabled
- OAuth Installed App flow: `google-auth-oauthlib.flow.InstalledAppFlow.from_client_secrets_file`, scopes `https://www.googleapis.com/auth/youtube.upload` + `youtube.readonly`; refresh token uložit `~/.yt3d/credentials.json` (JEDNOU ručně na PC, otevře se browser)
- Upload: resumable session, `snippet(title, description, tags, categoryId=27)`, `status(privacyStatus="unlisted", selfDeclaredMadeForKids=False)`
- Thumbnail: vybraný frame (ffmpeg `-ss 5 -frames:v 1`) — Fáze 2: LLM vybere nejlepší shot

**Verify:** Test upload 10s klipu unlisted → video viditelné v Studio. Commit.

---

## Fáze 6 — Orkestrace a provoz

### Task 15: make_job end-to-end na VPS + Hermes cron
**Objective:** Jeden příkaz vytvoří celý job; Hermes cron spouští týdně a notifikuje.

**Files:** Modify `server/make_job.py`

**Flow:** `make_job.py --topic topics/x.yaml` → DB job → script_gen → assety (Tripo [+Marble]) → zip bundle do cache → state `ready_to_render` → Telegram: "Job hotový ve frontě, zapni PC". Hermes cron: `0 5 * * 1` (`hermes cron create`) s promptem: "Vyber nediskutované téma z ~/projects/yt3d-factory/topics/ a spusť make_job.py; výsledek nahlas." Worker si to pak vyzvedne, jakmile je PC online.

### Task 16: Úklid a monitoring
**Objective:** VPS disk (20G volných) nezaplavit; viditelnost stavů.

**Detaily:**
- `make_job.py --cleanup` maže `server/cache/<jid>/` pro joby ve stavu `done` starší 7 dní (final.mp4 1 kopií nechat, pokud <2G, jinak jen YouTube)
- Hermes cron denní kontrola: `sqlite3 cache/jobs.db "select state,count(*) from jobs group by state"` → Telegram výpis jen když něco ve `failed:*`
- Cost tracking: `server/cache/costs.csv` — append řádek per asset (jid, asset, credits, USD), měsíční součet do týdenní notifikace

---

## Fáze 7 — Proof of Concept

### Task 17: E2E PoC video
**Objective:** Jedno reálné 60s video end-to-end, ručně dotlačené, odhalí všechny mezery.

**Kroky:**
1. `make_job.py --topic topics/priklad.yaml` (Tripo: 3–5 assetů, ~$1–2)
2. Worker: render `--quick` (50% res) → kontrola klipů → plný render
3. Assemble + TTS → final.mp4 → YouTube unlisted
4. **Retrospektiva:** sepsat `docs/lessons.md` — co selhalo, co upravit v promptech LLM (nejčastější problém: kamera protne geometrii, špatné proporce assetů — řešit scale pinningem v pipeline.json)

**Accept criteria:** Video od tématu po YouTube bez ručního zásahu do renderu; celkový náklad <$5; čas <2 h (z toho většina render na PC).

---

## Časový plán a pořadí

| Fáze | Tasky | Kde běží | Odhad |
|---|---|---|---|
| 0 skeleton | 1 | VPS/MCP | 15 min |
| 1 orchestrátor | 2–5 | VPS | 3–4 h |
| 2 scénář | 6–7 | VPS | 2–3 h |
| 3 assety | 8–9 | VPS | 3–4 h (včetně živého testu) |
| 4 worker | 10–13 | domácí PC | 4–6 h |
| 5 YouTube | 14 | domácí PC | 1–2 h (setup GCP navíc) |
| 6 provoz | 15–16 | VPS | 1–2 h |
| 7 PoC | 17 | všude | 2–4 h |

Celkem ~2–3 pracovní dny. Marble fáze (9) lze posunout za PoC — PoC běží s procedural HDRI worldem.

## Rizika a mitigace
- **Marble API schema se může lišit od očekávání** → klient izolovaný v jednom souboru, PoC bez něj
- **GLM generuje shoty odkazující na neexistující assety** → validace pydantic + self-heal druhý průchod
- **Blender GPU konfigurace na PC** → `--quick` režim + EEVEE fallback; specs PC ověřit před Fází 4 (VRAM ≥ 8 GB pro Cycles GPU pohodlně)
- **Edge TTS délky nesedí se scénou** → narrace řídí délku shotů (render_plan.json)
- **VPS disk** → cleanup task 16 je povinný, ne volitelný
- **Tripo/LLM výpadky** → retry s backoff, job zůstává ve stavu failed, cron to hlásí na Telegram
