Title: fix: map permissions for agent-proxied registries, system health and image update check routes

## Checklist

- [ ] This PR is **not** opened from my fork’s `main` branch
- [ ] All new user-facing strings are translated via Paraglide (`m.*()`)

## What This PR Implements

Fixes: #4821

Through `/api/agents/{id}/...`, a key that already holds the right permission still got 403 on `GET /registries`, `GET /system/health`, and `POST /images/updates/check`, while sibling routes guarded by the same permission (`/system/docker/info`, `/system/df`) returned 200 and the manager-local routes worked. The environment proxy denies any proxied method+path the permission matcher does not know.

## Changes Made

- Register `GET /agents/{id}/system/health` with `system:read` (health was HEAD-only, so GET had no mapping) and allowlist it on the agent tunnel.
- Register `GET /agents/{id}/registries` with `registries:list` next to the existing manager-local `GET /registries`, and allowlist the proxied path. Registries stay global; the new route exists so the matcher and the agent can serve the same list the proxy already intercepts.
- Register `POST /agents/{id}/images/updates/check` with `image-updates:check` instead of action-query permissions. Missing `action` now defaults to `all`; an unknown action is still 400. The matcher also maps a missing action query when every listed action uses the same permission, and still fails closed for mixed-permission or unknown actions and for unknown paths.
- Walk registered agent-proxied Huma routes and assert each has a matcher entry. Table tests cover the three issue routes: allowed with the permission, 403 without it, 403 when the grant is scoped to another environment.

## Testing Done

Baseline on untouched upstream `693a5f39a` (`/tmp/arcane-baseline`):

```
cd backend && go test -tags=exclude_frontend,buildables -ldflags="-X getarcane.dev/arcane/v3/buildables.EnabledFeatures=autologin" ./... -count=1
```

130 packages `ok`, 35 `[no test files]`, 0 `FAIL`.

This branch (`9cbac5520`):

```
just format go
just lint backend
cd backend && go test -tags=exclude_frontend,buildables -ldflags="-X getarcane.dev/arcane/v3/buildables.EnabledFeatures=autologin" ./... -count=1
cd backend && go build -tags=exclude_frontend,buildables ./...
cd backend && go build -tags=exclude_frontend,buildables -o /tmp/arcane-live/arcane-feature ./cmd
```

- format: golangci-lint 2.15.0 fmt, no extra files
- lint backend: 0 issues
- tests: 130 `ok`, 35 `[no test files]`, 0 `FAIL`
- `go build ./...` without `exclude_frontend` fails here (`frontend/frontend.go` embed `all:dist` missing); tagged build succeeds

Live, SQLite manager + agent on Docker 29.1.3, environment `726c2ee0-fb65-45dd-8e0d-e3efe6377cd5` (`live-agent`). Admin static key, a scoped key with `registries:list` / `registries:read` / `system:read` / `image-updates:check`, and a scoped key with only `environments:list`:

Upstream main manager (agent not connected yet):

- `GET /api/agents/{id}/registries` admin+privileged → 403
- `GET /api/agents/{id}/system/health` admin+privileged → 403
- `POST /api/agents/{id}/images/updates/check` admin+privileged → 403
- `GET /api/agents/{id}/system/docker/info` and `/system/df` admin+privileged → 503 Agent is not connected
- under-privileged → 403 on all of the above
- manager-local `GET /api/registries` → 200

This branch with the agent connected:

- privileged + admin → 200 on registries, health, image-update check, docker info, and df
- under-privileged → 403 on those routes

Playwright (Chromium, autologin): `/images/registries` loads the empty registries table; dashboard shows local engine and `live-agent` both online.

Manual testing by Matt: <!-- MATT: fill in after testing by hand -->

## AI Tool Used (if applicable)

<!-- If you used AI coding assistance, disclose it here. See AI_POLICY.md for details. -->

## Additional Context

`GET /api/registries` stays manager-local. The UI still uses that path. The proxied `GET /api/agents/{id}/registries` is what 403'd in the issue and is what this maps.

`just test backend` was run as the tagged `go test ./...` above without `-race` / coverage in this environment.

## Demo

![demo](https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1116/agent-proxy-permission-map/preview.gif)

Full recording (MP4): https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1116/agent-proxy-permission-map/out.mp4
