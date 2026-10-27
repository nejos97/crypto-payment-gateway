# syntax=docker/dockerfile:1.7.1@sha256:a57df69d0ea827fb7266491f2813635de6f17269be881f696fbfdf2d83dda33e
#
# Single multi-stage Dockerfile for every deployable.
#
#   docker build -f docker/Dockerfile --target api .        → Rust gateway (apps/api)
#   docker build -f docker/Dockerfile --target dashboard .  → dashboard SPA behind nginx (apps/dashboard)
#   docker build -f docker/Dockerfile --target all .        → both in one image (Kamal roles api / dashboard)
#
# Every base image is pinned by digest (the tag only documents it): a tag can be
# re-pushed, a digest cannot. Dependabot (.github/dependabot.yml) proposes the
# bumps. The Node major must match .tool-versions / .nvmrc.

ARG PNPM_VERSION=11.25.0

# ── API: Rust release build ──────────────────────────────────────────────────
# protoc + libprotobuf-dev: the Breez SDK (Lightning) compiles its gRPC protos
# at build time and imports the google/protobuf/*.proto well-known types.
FROM rust:1.99.0-bookworm@sha256:114c7a4425406451c2866b6aafe69fe29b1b298832db1277d411ac73c82d04d6 AS api-build
RUN apt-get update \
  && apt-get install -y --no-install-recommends protobuf-compiler libprotobuf-dev \
  && rm -rf /var/lib/apt/lists/*
WORKDIR /workspace
COPY Cargo.toml Cargo.lock ./
COPY apps/api ./apps/api
COPY packages/rust ./packages/rust
# --locked: build exactly Cargo.lock, fail instead of resolving new versions.
RUN cargo build --release --locked -p gateway

# ── Dashboard: pnpm + Nx build ───────────────────────────────────────────────
FROM node:24.22.0-alpine3.24@sha256:44fce696f38ba8c987c4db89954051000bbd05ffd8d8659ba15ed4a8602e937f AS dashboard-build
ARG PNPM_VERSION
ENV NX_DAEMON=false CI=true
RUN npm install -g pnpm@${PNPM_VERSION} && pnpm config set store-dir /pnpm/store
WORKDIR /workspace
# Lockfile only: cached until the lockfile changes.
COPY pnpm-lock.yaml pnpm-workspace.yaml ./
RUN pnpm fetch
# Every workspace package.json is needed for a frozen install; only the
# dashboard sources are.
COPY package.json nx.json ./
COPY apps/api/package.json ./apps/api/package.json
COPY packages/rust/package.json ./packages/rust/package.json
COPY apps/dashboard ./apps/dashboard
RUN pnpm install --offline --frozen-lockfile
RUN pnpm nx run dashboard:build

# ── API runtime ──────────────────────────────────────────────────────────────
# Same Debian release as the rust:*-bookworm builder (glibc of the binary).
FROM debian:bookworm-20261005-slim@sha256:7c7b2c966bc9ee8cedfeef67e0e279108992c77681fa595db4a9d65c06ccc587 AS api
RUN apt-get update \
  && apt-get install -y --no-install-recommends ca-certificates \
  && rm -rf /var/lib/apt/lists/* \
  && useradd -r -u 10001 gateway
WORKDIR /app
COPY --from=api-build /workspace/target/release/gateway /app/gateway
# Breez SDK (Lightning) storage. Mount a persistent volume here; a fresh named
# volume inherits this directory's ownership.
RUN mkdir -p /app/.breez-data && chown gateway:gateway /app/.breez-data
ENV PORT=3000 BREEZ_STORAGE_DIR=/app/.breez-data
USER gateway
EXPOSE 3000
CMD ["/app/gateway"]

# ── Dashboard runtime (nginx) ────────────────────────────────────────────────
# nginx stable line; its CA bundle (/etc/ssl/certs/ca-certificates.crt, from
# ca-certificates-bundle) verifies the API's certificate when API_URL is https.
FROM nginx:1.30.5-alpine@sha256:0985e772fb9f729e6fa0980da05fca5d9c468e870eed43071545afa9d2e27d94 AS dashboard
COPY docker/dashboard/nginx.conf docker/dashboard/nginx.conf.template docker/dashboard/security-headers.conf docker/dashboard/start.sh /app/dashboard/
COPY --from=dashboard-build /workspace/apps/dashboard/dist /app/dashboard/html
ENV PORT=8080 API_URL=http://api:3000
USER nginx
EXPOSE 8080
CMD ["/app/dashboard/start.sh"]

# ── Everything (one image, Kamal roles `api` and `dashboard` with different `cmd`) ─
# Debian's nginx (security-maintained by Debian); ca-certificates provides the
# same /etc/ssl/certs/ca-certificates.crt as the alpine image.
FROM debian:bookworm-20261005-slim@sha256:7c7b2c966bc9ee8cedfeef67e0e279108992c77681fa595db4a9d65c06ccc587 AS all
RUN apt-get update \
  && apt-get install -y --no-install-recommends ca-certificates nginx gettext-base \
  && rm -rf /var/lib/apt/lists/* \
  && useradd -r -u 10001 gateway
WORKDIR /app
COPY --from=api-build /workspace/target/release/gateway /app/gateway
COPY docker/dashboard/nginx.conf docker/dashboard/nginx.conf.template docker/dashboard/security-headers.conf docker/dashboard/start.sh /app/dashboard/
COPY --from=dashboard-build /workspace/apps/dashboard/dist /app/dashboard/html
RUN mkdir -p /app/.breez-data && chown gateway:gateway /app/.breez-data
ENV PORT=3000 BREEZ_STORAGE_DIR=/app/.breez-data
USER gateway
EXPOSE 3000 8080
# Default command: the API. The dashboard role runs /app/dashboard/start.sh.
CMD ["/app/gateway"]
