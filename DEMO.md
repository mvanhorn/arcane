# Demo: agent-proxy permission mappings

Recorded against a local manager on `127.0.0.1:3552` plus a connected agent (`live-agent`, id `726c2ee0-fb65-45dd-8e0d-e3efe6377cd5`) on the same Docker 29.1.3 daemon.

## What `out.mp4` shows

1. Replay of the real baseline curl matrix (upstream `main` / `693a5f39a`): a global-admin key and a privileged key both get **403** on `GET /agents/{id}/registries`, `GET /agents/{id}/system/health`, and `POST /agents/{id}/images/updates/check`, while sibling mapped routes (`/system/docker/info`, `/system/df`) return **503 Agent is not connected** (mapping exists; agent was not up yet).
2. Live curls on this branch with the agent connected:
   - privileged key (`registries:list`, `registries:read`, `system:read`, `image-updates:check`) → **200** on the three issue routes
   - under-privileged key (`environments:list` only) → **403** on those same routes
3. Playwright recording of `/images/registries` after autologin: empty registries table loads; dashboard shows both the local engine and `live-agent` online.

## Media

- `out.mp4`: 1920x1080, 30 fps, H.264 yuv420p, 38.67 s, 1160 frames
- `preview.gif`: 1280x720, 8 fps, 1.4 MB

## Not in the recording

Frontend hot-reload was not exercised. The registries UI still calls manager-local `GET /api/registries` (200); the proxied `GET /api/agents/{id}/registries` path is what the curl matrix covers.
