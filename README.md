# 💬 AionUi Stack

Self-hosted [AionUi](https://github.com/iOfficeAI/AionUi) — a web chat UI for
command-line AI agents (Gemini CLI, Claude Code, Codex, …) speaking the ACP
protocol. Single-service stack: a Bun-based WebUI server hosts the renderer,
spawns the pinned Rust `aioncore` backend as a subprocess, and persists
everything (chats, files, admin auth) in embedded SQLite.

## 🧩 Components

### 🤖 [AionUi App](app/)

The complete functional base: WebUI server + `aioncore` backend + SQLite in a
named volume, served on port **25808**. See [`app/README.md`](app/README.md)
for component docs, quick start, and environment variables.

## 🚀 Quick Start

```bash
cd app/
sb build
cd build/forwarding/            # or the environment combo you need
cp .env.example .env        # edit values
docker compose up -d
```

Access: `http://localhost:25808` — fresh-database admin credentials are
printed to the container log on first start.

## 🏗️ Source-Built Image

Upstream's published Dockerfile is broken at head, so the image is built from
source per stack-conventions §19: [`app/images/aionui/Dockerfile`](app/images/aionui/Dockerfile)
clones upstream at pinned commit `119dd2b47` (tag v2.2.2), builds the renderer
with Bun, and bakes the pinned `aioncore` v0.2.2 release binary plus the
Gemini CLI (required for the functional gate) into the image
`localhost/aionui:v2.2.2`. No upstream source is vendored into this repo.

## 📋 Deployment Model

The deployment model contract lives in [`DEPLOYMENT.md`](DEPLOYMENT.md)
(stack-conventions §20): product parts, interaction map, functional gate
scenario, and the extension catalog with statuses. Upstream provenance and
build decisions are logged in [`UPSTREAM.md`](UPSTREAM.md).

Mandatory infra decisions (all **refused**, reasons in DEPLOYMENT.md):

- **database/postgresql + pgrust**: refused — upstream data path is SQLite-only, no Postgres support exists.
- **oidc (kanidm)**: refused — auth is a local admin user in SQLite + JWT; no OIDC integration surface.
- **s3-storage**: refused — chat/file persistence is SQLite + work-dir filesystem; no S3 client surface.

## 📋 Requirements

- Docker & Docker Compose
- [stackbuilder](https://github.com/zyrakq/docker-stack-collection) (`sb`) for builds
- An agent provider API key (e.g. Gemini) to drive agent runs

## 📄 License

This project is dual-licensed under:

- [Apache License 2.0](LICENSE-APACHE)
- [MIT License](LICENSE-MIT)
