# Top-30 Films & Series — Backend PoC

FastAPI-backend die per genre een top 30 genereert (films of tv-series) via de TMDB API, met filters op jaar, acteur, regisseur en min/max rating. Optioneel worden IMDb-, Rotten Tomatoes- en Metacritic-scores toegevoegd via OMDb.

## Snel testen (zonder API-key)

```bash
pip install -r requirements.txt
MOCK_MODE=1 uvicorn main:app --host 0.0.0.0 --port 8000
```

Open daarna `http://<host>:8000/docs` voor de interactieve Swagger UI.

## Deployment op Proxmox

**Optie A — LXC-container (lichtgewicht, aanbevolen voor PoC)**

1. Maak een Debian 12 / Ubuntu 24.04 LXC aan (1 core, 512 MB RAM is genoeg).
2. In de container:
   ```bash
   apt update && apt install -y python3-pip git
   pip3 install -r requirements.txt --break-system-packages
   cp .env.example .env   # vul je TMDB_API_KEY in
   set -a; source .env; set +a
   uvicorn main:app --host 0.0.0.0 --port 8000
   ```
3. Voor automatisch starten: maak een systemd-service aan (voorbeeld hieronder).

**Optie B — Docker (bijv. in een Docker-LXC of VM)**

```bash
cp .env.example .env   # keys invullen
docker compose up -d --build
```

## Systemd-service (optie A)

`/etc/systemd/system/top30-api.service`:

```ini
[Unit]
Description=Top30 Films API
After=network.target

[Service]
WorkingDirectory=/opt/fdo-poc
EnvironmentFile=/opt/fdo-poc/.env
ExecStart=/usr/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always

[Install]
WantedBy=multi-user.target
```

Activeren: `systemctl enable --now top30-api`

## API-endpoints

| Endpoint | Beschrijving |
|---|---|
| `GET /api/health` | Status en configuratie |
| `GET /api/genres?media_type=movie\|tv` | Beschikbare genres (Nederlandstalig) |
| `GET /api/top30` | De top 30 met filters (zie hieronder) |

### Parameters van `/api/top30`

- `media_type` — `movie` of `tv`
- `genre_id` — ID uit `/api/genres`
- `actor` — naam, bijv. `Tom Hanks`
- `director` — naam, alleen bij films
- `year_min` / `year_max` — releasejaar
- `rating_min` / `rating_max` — 0–10
- `sort_by` — `rating`, `year_asc` of `year_desc`
- `min_votes` — minimum aantal stemmen (default 200, filtert obscure titels)
- `enrich=true` — voegt IMDb/RT/Metacritic toe (OMDb-key vereist, langzamer)

### Voorbeeld

```bash
curl "http://<host>:8000/api/top30?media_type=movie&genre_id=18&year_min=1990&year_max=1999&rating_min=7.5&sort_by=rating&enrich=true"
```

## Architectuurnotities

- **Caching**: genres 24 u, lijstresultaten 1 u, persoon-lookups 24 u (in-memory TTL). Bij herstart is de cache leeg; voor productie vervangen door Redis of SQLite.
- **Rating-aggregatie**: de PoC sorteert op TMDB-score; met `enrich=true` komen IMDb/RT-cijfers mee als extra velden. Een gewogen gecombineerde score is een logische volgende stap.
- **Volgende stappen**: Redis-cache, gewogen score-berekening, reverse proxy (bijv. Nginx Proxy Manager op Proxmox) met HTTPS, en daarna de Flutter/React Native-app die deze API consumeert.
