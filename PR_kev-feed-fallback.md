Title: fix: fall back to CISA's GitHub mirror when the KEV feed download is refused

## Checklist

- [ ] This PR is **not** opened from my fork’s `main` branch
- [ ] All new user-facing strings are translated via Paraglide (`m.*()`)

## What This PR Implements

Threat intelligence sync tries more than one KEV catalog URL. The CISA feed is first; if that request is not 200 or 304 (including HTTP 403 from the CDN) or the transport fails, the CISA GitHub mirror is tried next. Each source keeps its own ETag. The body is rejected unless it is JSON with `catalogVersion` and a `vulnerabilities` array, so an HTML 200 cannot wipe cached KEV rows. The failure message lists every source and why it failed.

`ARCANE_KEV_FEED_URLS` overrides the ordered list.

Fixes: #4823

## Changes Made

- `syncKEV` walks CISA, then `https://raw.githubusercontent.com/cisagov/kev-data/refs/heads/develop/known_exploited_vulnerabilities.json`, unless `ARCANE_KEV_FEED_URLS` is set.
- `If-None-Match` is per URL. A 304 on the primary is a successful sync and does not hit the mirror. An ETag from one source is never sent to another.
- HTTP errors, transport errors, invalid JSON, missing catalog shape, and incomplete catalogs fail that source and continue. `ReplaceKnownExploited` errors still abort.
- The aggregate error looks like `KEV catalog failed: <url> (HTTP 403); <url> (HTTP 503)`.
- `ARCANE_KEV_FEED_URLS` is documented in `.env.example` and the config schema comment. Empty is the default pair. Production requires https; http is allowed for loopback always, and for any host when `ARCANE_ENVIRONMENT` is development or test.
- KV state stores per-source ETags and the source that last succeeded. The legacy single `etag` field still maps onto the CISA URL.

## Testing Done

`httptest` only (no live CISA/GitHub in unit tests): primary 403 then mirror 200 succeeds and does not forward the primary ETag; primary 304 skips the mirror and keeps the cache; both failing lists both URLs and HTTP statuses; HTML 200 is `invalid JSON` and the cache is kept; `ARCANE_KEV_FEED_URLS` is the only list requested; invalid overrides are refused (`http` in production, empty list, `ftp`).

Baseline on untouched `upstream/main` (`693a5f39a`), `just test backend`: 2195 PASS, 0 FAIL, 1 SKIP, 130 ok packages, exit 0.

Same suite on this branch: 2201 PASS, 0 FAIL, 1 SKIP, 131 ok packages, exit 0. The extra package is `children/store` now that it has tests; the extra cases are the KEV fallback and `ARCANE_KEV_FEED_URLS` validation tests.

`just format all` (gofumpt/golangci-lint fmt + `vp fmt`), `just lint all` (frontend svelte-check 0 errors 0 warnings, frontend eslint, tests `tsc --noEmit`, golangci-lint 0 issues on backend/cli/types, `buf lint`), `go build -tags=exclude_frontend ./...` in `backend/`, `go build ./...` in `cli/`.

Live, SQLite, `ARCANE_ENVIRONMENT=development`, `go run -tags=exclude_frontend ./cmd` on `127.0.0.1:3552` and `vp run dev` on `:3000` (Docker socket missing, so not `just dev docker`). Override `ARCANE_KEV_FEED_URLS=http://127.0.0.1:8765/cisa,http://127.0.0.1:8765/mirror`. Admin session, `POST /api/agents/0/jobs/vulnerability-risk?action=run`:

- mirror 200: job 202 then run `succeeded`. Logs: `KEV catalog source failed source=http://127.0.0.1:8765/cisa error="HTTP 403"` then `KEV catalog synced source=http://127.0.0.1:8765/mirror`. Activity Center shows Vulnerability Risk succeeded.
- mirror 503: job 202 then run `partial`, message `Threat intelligence sync failed: KEV catalog failed: http://127.0.0.1:8765/cisa (HTTP 403); http://127.0.0.1:8765/mirror (HTTP 503)`. Activity Center shows Partial with that error.

Manual testing by Matt: <!-- MATT: fill in after testing by hand -->

## AI Tool Used (if applicable)

<!-- MATT: disclosure required by AI_POLICY.md; text pending your decision -->

## Additional Context

EPSS fetching is unchanged; it does not share the KEV source list.

The job still returns Partial (not Failed) when intel sync fails, so snapshots keep recording on offline hosts.

## Demo

![demo](https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1116/kev-feed-fallback/preview.gif)

Full recording (MP4): https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1116/kev-feed-fallback/out.mp4
