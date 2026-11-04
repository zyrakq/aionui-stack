# syntax=docker/dockerfile:1
# AionUi — source-built image (web/server mode).
# Upstream: https://github.com/iOfficeAI/AionUi @ 119dd2b47 (v2.2.2 line)
# Self-contained: clones upstream at the pinned commit; build with an empty context:
#   docker build -t localhost/aionui:v2.2.2 app/images/aionui

FROM oven/bun:1.2-debian AS build

ARG AIONUI_REPO=https://github.com/iOfficeAI/AionUi.git
ARG AIONUI_COMMIT=119dd2b47

RUN apt-get update \
    && apt-get install -y --no-install-recommends git ca-certificates curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
RUN git clone --filter=blob:none "${AIONUI_REPO}" . \
    && git checkout "${AIONUI_COMMIT}"

# Renderer + main bundles into out/ (electron-vite build). postinstall
# (electron-builder install-app-deps) failures are non-fatal upstream.
RUN bun install \
    && bun run package

# aioncore v0.2.2 release binary -> resources/bundled-aioncore/linux-<arch>/aioncore
# (reads "aioncoreVersion" from package.json; arch = process.arch of the builder)
RUN bun scripts/prepareAioncore.js

FROM oven/bun:1.2-debian

ARG AIONUI_COMMIT=119dd2b47
LABEL org.opencontainers.image.title="aionui" \
      org.opencontainers.image.source="https://github.com/iOfficeAI/AionUi" \
      org.opencontainers.image.revision="${AIONUI_COMMIT}" \
      org.opencontainers.image.description="AionUi web/server mode with baked aioncore v0.2.2 and gemini CLI"

RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    # agent-cli-gemini (required-for-gate): global CLI so aioncore finds it on PATH
    && bun install -g @google/gemini-cli

WORKDIR /app
COPY --from=build /app /app

ENV NODE_ENV=production \
    AIONUI_DATA_DIR=/data \
    AIONUI_STATIC_DIR=/app/out/renderer \
    AIONUI_NO_BUILD=1 \
    AIONUI_OPEN_BROWSER=0 \
    AIONUI_BACKEND_BIN=/app/resources/bundled-aioncore/linux-x64/aioncore

EXPOSE 25808

# webui CLI: web-host (backend launcher + static server + auth seeding).
# NODE_ENV=production -> default port 25808; AIONUI_NO_BUILD skips rebuild.
CMD ["bun", "scripts/webui.ts"]
