# KEV feed fallback demo

Recorded against `feat/kev-feed-fallback` with a local mock as `ARCANE_KEV_FEED_URLS`:

- `http://127.0.0.1:8765/cisa` always returns HTTP 403
- `http://127.0.0.1:8765/mirror` returns HTTP 200 (valid catalog) or HTTP 503

Docker was not available (`/var/run/docker.sock` missing). The manager ran with `go run -tags=exclude_frontend ./cmd` on `127.0.0.1:3552` and the UI with `vp run dev` on `127.0.0.1:3000`.

## Files

- `out.mp4` — real xfce4-terminal capture of the httptest suite and backend log grep, then a Playwright `recordVideo` of Activity Center (failed Partial with both source statuses, then Succeeded after the mirror answered 200). 1920x1080, 30 fps, H.264, yuv420p.
- `preview.gif` — palette-reduced 960x540 / 8 fps copy of `out.mp4`.
- `screenshots/` — dashboard and Activity Center stills from that Playwright run.
- `commands.log` — commands, exit codes, and live HTTP/job results.
- `logs/ffprobe.txt` — ffprobe for both media files.
- `logs/live-kev-sync.log` — KEV lines from the live backend log.
- `logs/baseline-backend-tests.log` — `just test backend` on untouched `upstream/main`.
- `PR_kev-feed-fallback.md` — PR title and body (not opened).
