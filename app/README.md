# 🤖 AionUi (app)

Docker Compose deployment of [AionUi](https://github.com/iOfficeAI/AionUi) —
an open-source chat UI for command-line AI agents (Gemini CLI, Claude Code,
Codex, Qwen Code, …) built around the ACP (Agent Client Protocol). AionUi
pairs a web renderer with a Rust agent host (`aioncore`) that spawns agent
CLIs, manages conversations in SQLite, and exposes a `/health` readiness
endpoint. This stack runs the **web/server mode** (no Electron desktop app),
pinned to upstream commit `119dd2b47` (tag `v2.2.2`), with the image built
from source per stack-conventions §19 (see [`images/aionui/`](images/aionui/)).

## 🚀 Quick Start

### 1. Build the Image

The image is source-built (upstream's published Dockerfile is broken at head):

```bash
docker build -t localhost/aionui:v2.2.2 images/aionui
```

### 2. Build Configurations

```bash
cd app/
sb build
```

This will create all combinations in the `build/` directory based on
[`stackbuilder.toml`](stackbuilder.toml).

### 3. Choose Your Configuration

```bash
# For development with port forwarding
cd build/forwarding/base/

# For production with Let's Encrypt SSL
cd build/letsencrypt/base/

# For production with Step CA SSL
cd build/step-ca/base/
```

### 4. Configure Environment

```bash
cp .env.example .env
# Edit .env with your values (at minimum GEMINI_API_KEY or GOOGLE_API_KEY)
```

### 5. Deploy

```bash
docker compose up -d
```

Access (forwarding mode): `http://localhost:25808`

On a **fresh database** the first admin login credentials are printed to the
container logs — retrieve them with:

```bash
docker logs aionui-app
```

## 🧩 Product Parts

The product's functional base per [`DEPLOYMENT.md`](../DEPLOYMENT.md)
(stack-conventions §20) — one compose service per part:

| part | service | purpose |
|------|---------|---------|
| aionui-app | `aionui-app` | WebUI server (web-host + renderer bundle, admin auth over SQLite, port 25808) hosting the `aioncore` v0.2.2 subprocess (Rust ACP agent host, agent-CLI spawning, `/health`) as an in-container process; SQLite state lives in the `aionui-data` volume at `/data`. Without it there is no deployment — it is the single product part. |

Notes:

- `aioncore` is **not** a separate container — it is baked into the image
  (pinned release binary) and spawned by the WebUI server.
- The Electron desktop app is intentionally excluded; web/server mode is the
  deployment.

## 🧩 Extensions

Status per the extension catalog in [`DEPLOYMENT.md`](../DEPLOYMENT.md)
(stack-conventions §20):

| extension | status | notes |
|-----------|--------|-------|
| agent-cli-gemini | **required-for-gate** | Gemini CLI is baked into the image; credentials come via `GEMINI_API_KEY` / `GOOGLE_API_KEY` env or the UI settings. The functional gate cannot complete an agent run without it. |
| agent-cli-claude | available (not implemented) | needs Anthropic credentials + CLI binary in image |
| agent-cli-codex | available (not implemented) | needs OpenAI credentials + CLI binary in image |
| agent-cli-qwen | available (not implemented) | needs credentials + CLI binary in image |
| mcp-servers | available (not implemented) | MCP tool config, set via UI |
| im-bot-telegram / im-bot-lark / im-bot-dingtalk / im-bot-wecom | available (not implemented) | bot/sdk credentials; reachability from the webui path unverified |
| database/postgresql + pgrust | **refused** | upstream data path is SQLite-only — see [DEPLOYMENT.md refusals](../DEPLOYMENT.md#functional-base) |
| oidc (kanidm) | **refused** | upstream auth is a local admin user + JWT, no OIDC surface — see [DEPLOYMENT.md refusals](../DEPLOYMENT.md#functional-base) |
| s3-storage | **refused** | persistence is SQLite + work-dir filesystem, no S3 client surface — see [DEPLOYMENT.md refusals](../DEPLOYMENT.md#functional-base) |

Everything marked `available` needs explicit user confirmation before
implementation, one at a time, each gated by a stack-test stage.

## 📁 Project Structure

- [`components/`](components/) — Source compose components
  - [`base/`](components/base/) — Base aionui-app configuration
  - [`environments/`](components/environments/) — Environment-specific configurations
- [`images/aionui/`](images/aionui/) — Source-built image recipe (§19)
- [`build/`](build/) — Generated configurations (auto-generated, ready to deploy)
- [`stackbuilder.toml`](stackbuilder.toml) — Build configuration for stackbuilder

## 🔧 Available Configurations

### Environments

- **forwarding**: host port forwarding via `AIONUI_PORT` (default 25808)
- **letsencrypt**: joins `letsencrypt-network` behind the SSL stack (nginx-proxy + Let's Encrypt)
- **step-ca**: joins `step-ca-network` behind the SSL stack (step-ca manager)

### Extensions

- none implemented yet — see the Extensions table above for the catalog.

## 🔧 Environment Variables

### Base Configuration

- `IMAGE`: Docker image (default: `localhost/aionui:v2.2.2`)
- `LOG_MAX_SIZE`: json-file log max size (default: `10m`)
- `LOG_MAX_FILE`: json-file log file count (default: `3`)
- `LOG_COMPRESS`: compress rotated logs (default: `true`)
- `AIONUI_PORT`: host-side published port used by the forwarding overlay; the container always listens on 25808 (default: `25808`)
- `AIONUI_ALLOW_REMOTE`: bind the WebUI server to `0.0.0.0` so the published port works (default: `1`)
- `AIONUI_LOG_DIR`: log directory inside the container (default: `/data/logs`)
- `TZ`: container timezone (default: `UTC`)
- `GEMINI_API_KEY`: Gemini CLI API key (agent-cli-gemini, required-for-gate; also settable in UI) (default: empty)
- `GOOGLE_API_KEY`: alternative Google API key for the Gemini CLI (default: empty)

### Environment Overlay Only (letsencrypt / step-ca)

These variables are declared only in the environment overlays — never in base:

- `DOMAIN`: public domain of the deployment
- `VIRTUAL_HOST`: domain for nginx-proxy
- `VIRTUAL_PORT`: container port nginx-proxy forwards to (default: `25808`)
- `LETSENCRYPT_HOST`: domain for the SSL certificate (letsencrypt overlay)
- `LETSENCRYPT_EMAIL`: email for certificate registration (letsencrypt overlay)

## 🌐 Networks

- **forwarding**: `aionui-network` (internal)
- **letsencrypt**: `letsencrypt-network` (external)
- **step-ca**: `step-ca-network` (external)

## 🔒 Security

⚠️ **Production Checklist:**

- Set a strong admin password after first login (fresh-DB credentials from
  container logs are temporary).
- Keep `AIONUI_ALLOW_REMOTE` enabled only when the port is actually published;
  behind the SSL stack the container port is not published.
- Store `GEMINI_API_KEY` / `GOOGLE_API_KEY` in `.env` (gitignored), never commit them.
- SQLite is single-writer — never share the `aionui-data` volume across replicas.
- In TLS environments, join the appropriate SSL stack (letsencrypt or step-ca).

## 🆘 Troubleshooting

**Build Issues:**

- Image build fails on the renderer step: the recipe pins upstream commit
  `119dd2b47`; check [`images/aionui/`](images/aionui/) and
  [`UPSTREAM.md`](../UPSTREAM.md) for known build notes.
- `aioncore` missing at startup (`resolve_binary` / `which aioncore` error):
  the binary is baked into the image — do not swap `IMAGE` for a foreign build.

**Runtime Issues:**

- Container unhealthy: check `docker logs aionui-app` for the WebUI ready and
  `AIONCORE_READY` markers; `/health` must return 200.
- Agent run fails: verify the Gemini credentials (`GEMINI_API_KEY` /
  `GOOGLE_API_KEY` or UI settings) and that the CLI is on PATH in the image.
- Forgot admin password: reset via the `/api/webui/reset-password` endpoint or
  `bun run resetpass` inside the container.

**SSL Issues:**

- **Let's Encrypt**: Verify domain DNS and letsencrypt-manager
- **Step CA**: Check step-ca-manager and virtual network config

## ✅ Stage A Functional Gate

The Stage A gate (per [`DEPLOYMENT.md`](../DEPLOYMENT.md)) validates the base
deployment in 5 steps:

1. Container healthy; `/health` returns 200 (WebUI ready + `AIONCORE_READY` in logs).
2. Browser loads the renderer at `http://localhost:25808`.
3. Login with the fresh-DB admin credentials printed in the container logs.
4. One Gemini agent run completes end-to-end in a new conversation
   (browser → WebUI → aioncore → agent CLI → ACP round-trip).
5. Restart the container — login state and the conversation persist
   (SQLite in `aionui-data`).

## 📝 Notes

- The `build/` directory is automatically generated and should not be edited manually
- Environment variables in generated files use `$VARIABLE_NAME` format for proper interpolation
- Each generated configuration includes a complete `docker-compose.yml` and `.env.example`
- Use `sb build` to regenerate all configurations after modifying components
- Upstream provenance and pinning details live in [`UPSTREAM.md`](../UPSTREAM.md);
  the deployment contract in [`DEPLOYMENT.md`](../DEPLOYMENT.md)
