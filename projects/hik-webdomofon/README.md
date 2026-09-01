# Web Doorphone for Hikvision

Modern web‑doorphone for Hikvision cameras offering video streaming (WebRTC/HLS), push‑to‑talk audio via HCNetSDK, and Telegram Mini App integration.

## Quick start
```bash
git clone https://github.com/…/hik-webdomofon.git && cd hik-webdomofon
cp example.env .env   # set CAMERA_* variables
docker compose up -d
```

## Documentation
- Project overview: `docs/00_Project.md`
- Architecture diagram: `docs/03_ARCHITECTURE.md`
- API reference: `docs/06_API.md`
- Front‑end (Mini App) details: `docs/FRONTEND_OVERVIEW.md`
- Runtime flow: `docs/RUNTIME_FLOW.md`
- Planned AI integration (Ollama): `docs/ai/README_AI.md`

*For a full list of documentation see `docs/log.md`.*

## License
See the LICENSE file.
