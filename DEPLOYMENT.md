# DEPLOYMENT — aionui

Deployment model of upstream AionUi (analyzed at upstream commit `119dd2b47`,
tag v2.2.2). Living document per stack-conventions §20; produced by stack-analysis.

## Product parts

| part | purpose | image source |
|------|---------|--------------|
| WebUI server (`scripts/webui.ts` → `@aionui/web-host`) | serves the `out/renderer` web bundle, hosts admin auth (bcrypt+JWT over SQLite), proxies browser calls to the backend; prod port 25808 (`AIONUI_PORT`/`PORT`), remote mode via `AIONUI_HOST=0.0.0.0`/`AIONUI_ALLOW_REMOTE` | source-built (§19) — upstream published Dockerfile is broken at head (references `scripts/build-server.mjs` + `build:renderer:web`, deleted in upstream commit `b157719a5`); build recipe: `bun install` → `bun run package` (renderer bundle) → bun runtime + `libicu` |
| aioncore backend | Rust ACP agent host: SQLite store (`--data-dir`), agent-CLI spawning, `/health` readiness, system info | external release binary, pinned `aioncoreVersion` v0.2.2 from `iOfficeAI/AionCore` GitHub releases — downloaded during image build (upstream `scripts/prepareAioncore.js` idiom) and spawned by the WebUI server, NOT a separate container |
| Electron desktop app (`packages/desktop`) | native desktop wrapper of the same UI | NOT a container part — excluded from dockerization (upstream's headless story for it is .deb + Xvfb, unsuitable here); the web/server mode is the deployment |

Single-part product: one compose service, `<stack>-app` (§3 naming).

## Interaction map

- browser → WebUI server (HTTP static + API + WebSocket) — works.
- WebUI server → aioncore subprocess (localhost backend port allocated at runtime
  by `findAvailablePort`, excluded from fetch-blocked ports; readiness via
  `AIONCORE_LISTENING`/`AIONCORE_READY` stdout markers + `/health` 200).
- aioncore → agent CLIs (gemini / claude / codex / qwen / cursor …, spawned per
  ACP handshake; needs the CLI on PATH inside the container and its credentials).
- aioncore → SQLite in `/data` (single-writer; in-process data-dir instance guard
  `BOOTSTRAP_PEER_ALREADY_RUNNING` — do not share one volume across 2 replicas).
- officecli (.NET Office preview component) is auto-installed at runtime by the
  backend and aborts without ICU → image ships `libicu` (upstream Dockerfile
  comment at upstream commit `632bcebdb`).

What breaks when a part is absent: no aioncore binary → startup fails with
`resolve_binary`/`which aioncore` error, WebUI never starts (hard dependency of
`startWebHost`).

## Functional base

Tier groups (core outward):

1. **core** — `aionui-app`: WebUI server + aioncore subprocess + SQLite in a named
   volume + officecli/ICU. Core value ("command-line AI agent chat UI") is fully
   served by this one service. Justification: upstream's own pure-Bun webui path
   and its source Dockerfile treat this as the complete server deployment; no
   external service exists to chart.
2. No ring groups: agent CLIs run inside the same container (hop = none).

Mandatory infra:

- **Database**: product-mandated embedded SQLite (better-sqlite3 in the web layer,
  Rust SQLite in aioncore), single file in `/data` → named volume `aionui-data`.
  The database postgresql/pgrust extension (§8) is **REFUSED**: upstream has no
  Postgres support anywhere in the web/server code path — inventing one would be
  a re-implementation, not a deployment.
- **OIDC**: **REFUSED** — upstream auth is a local admin user in SQLite (seeded on
  first start, password reset via `/api/webui/reset-password` / `bun run resetpass`).
  No OIDC/SAML provider support found in the web/server code path.
- **S3**: **REFUSED** — upstream persistence for chats/files is SQLite + work-dir
  filesystem only; no S3 client configuration surface found.
- **step-ca-trust**: applies per §14 only in TLS environments (group runs behind
  the SSL stack); no inter-group https links (single group) to force it in base.

Environment surface (all `${VAR:-default}` in Phase 1 compose): `IMAGE`,
`PORT`/`AIONUI_PORT` (25808), `AIONUI_HOST=0.0.0.0`, `AIONUI_ALLOW_REMOTE=true`,
`AIONUI_DATA_DIR=/data`, `AIONUI_LOG_DIR`, `TZ`.

## Base stage plan

- **Stage A (core gate)** — deploy `aionui-app` alone; run the functional gate:
  1. container up; container log shows WebUI ready and aioncore ready
     (`AIONCORE_READY` / `/health` ok) on its runtime port;
  2. browser opens `http://localhost:${PORT}` → renderer loads;
  3. fresh-DB admin credentials are printed in the log; log in (or, on an
     existing DB, log in with the user's credentials);
  4. create a conversation and drive one agent run to completion with the
     gate-enabling agent extension configured (this exercises
     browser → WebUI → aioncore → agent CLI → ACP round-trip);
  5. restart the container → login state and the conversation persist
     (SQLite in `aionui-data` survives).
- **Stage A2 (DB pair swap)** — N/A: no postgres/pgrust swap possible (refused §3).
- **Stage A3** — pgrust: N/A (_refused_).
- Outer stages: none (no external groups). TLS environments (`letsencrypt` /
  `step-ca`) add the SSL-stack join + `step-ca-trust` in the environment overlay,
  then verify the same gate over the https domain.

Build order note for Phase 1: the source-built image must bake both the WebUI
build artifacts (`out/renderer` + web-host/runtime deps) and the pinned
`aioncore` binary (release asset per platform arch, per upstream
`prepareAioncore.js`); runtime PATH lookup of `aioncore` cannot be relied on.

## Extension catalog

| extension | type | status | dependencies |
|-----------|------|--------|--------------|
| agent-cli-gemini (Gemini CLI + API key) | product (executor) | **required-for-gate** — the functional gate can't complete an agent run without one executor agent + credentials | baked into image or injected; needs provider API key via env/UI settings |
| agent-cli-claude (Claude Code) | product | available | Anthropic credentials; CLI binary in image |
| agent-cli-codex | product | available | OpenAI credentials; CLI binary in image |
| agent-cli-qwen (Qwen Code) | product | available | credentials; CLI binary in image |
| mcp-servers (MCP tool config) | product | available | none; configured via UI |
| im-bot-telegram (grammy) | product | available | bot token; verify it is reachable from the webui code path before confirming |
| im-bot-lark / im-bot-dingtalk / im-bot-wecom | product | available | sdk credentials; same caveat as telegram |
| office-preview deep-dive (pptx/xlsx/docx render via officecli) | product | base-bundled (ICU in image); status `available` beyond the base sanity check | officecli runtime install needs outbound network at first run |
| database/postgresql + pgrust (§8) | infra | **refused** — no Postgres support in the upstream web/server data path (SQLite only) | — |
| oidc (kanidm, §10) | infra | **refused** — upstream auth is local admin + JWT; no OIDC provider integration surface | — |
| s3-storage (§11) | infra | **refused** — no S3 client surface for chat/file storage in the web/server path | — |

Confirmation rule: everything marked `available` needs explicit user confirmation
before implementation (§20), one at a time, each gated by a stack-test stage.

## Open hypotheses

| hypothesis | confidence | what the Stage A gate must observe |
|------------|-----------|-------------------------------------|
| `bun run package` (electron-vite build) produces a usable `out/renderer` without Electron binaries present in the build stage | medium | image builds in Phase 1; if electron-vite demands the electron runtime, fall back to the removed upstream renderer recipe restored from commit `b157719a5^` (reference-only, never vendored) |
| aioncore v0.2.2 release assets exist for linux ({x64,arm64}) and run in an OCI container without extra glibc deps | medium | image build downloads the pinned asset; gate step 1 shows AIONCORE_READY; if asset missing → record in UPSTREAM.md and pin the nearest working tag |
| officecli runtime auto-install works inside the container at first Office-preview use (network + ICU present) | low–medium | out-of-gate smoke: open an office document attachment in chat preview |
| IM bot features (telegram/lark/dingtalk/wecom) run from the pure-Bun webui path rather than only inside Electron | low | out-of-gate smoke if the extension is confirmed: configured bot answers a webhook message |
| all WebUI functionality served by the bundled server equals the Electron path (no Electron-IPC-only endpoints) | medium | gate steps 2–4 fail would falsify; survey `/api/*` handlers during Phase 1 implementation |
