# UPSTREAM — aionui

Provenance of the deployment model (see stack-conventions §17; rows append-only).

## app — https://github.com/iOfficeAI/AionUi

Upstream product: AionUi — a chat UI wrapping command-line AI agents (Gemini CLI,
Claude Code, Codex, Qwen, …) over ACP, electron-vite monorepo
(`packages/{desktop,web-host,web-cli,shared-scripts}`), partner backend binary
`aioncore` (Rust, repo `iOfficeAI/AionCore`, release-pinned via the
`aioncoreVersion` field of upstream `package.json`).

App's web/server mode (`scripts/webui.ts` → `@aionui/web-host`): serves the
`out/renderer` static bundle, spawns `aioncore` as a subprocess, admin auth
(bcrypt + JWT) and all records in SQLite. Official `.deb`+Xvfb headless deployment
is documented in `docs/guides/deploy-server.md` (port 25808).

NOTE: upstream `Dockerfile` is BROKEN at the recorded head — it references
`scripts/build-server.mjs` and the `build:renderer:web` npm script, both removed
upstream in commit `b157719a5` ("chore: remove standalone server build script and
npm scripts", 2026-04-22). The stack ships its own source-built image per §19; the
upstream Dockerfile recipe (bun runtime, `libicu-dev` for the officecli .NET
component, SQLite `/data` volume) is followed as a spec, not copied.

| date | upstream commit | image tag / digest | note |
|------|-----------------|--------------------|------|
| 2026-11-04 | 119dd2b47 | localhost/aionui:v2.2.2 (source-built, planned) | initial dockerization |
