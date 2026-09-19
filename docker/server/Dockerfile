# syntax=docker/dockerfile:1.7
#
# Production image for apps/server (Express 5 + Socket.IO + Prisma 7).
#
# This is a pnpm + Turborepo workspace, so the build context MUST be the repo
# root — the lockfile and the linked packages/* live there:
#
#   docker build -f docker/server/Dockerfile -t dethroits-server .
#   docker run --rm -p 3001:3001 --env-file apps/server/.env dethroits-server
#
# Migrations are deliberately NOT run by this image. Apply them as a separate
# step so concurrent replicas can't race each other:
#   pnpm --filter server db:deploy

# Node 22: Prisma 7 requires ^20.19 || ^22.12 || >=24, and Node 20 is now EOL.
# Alpine is safe here because the schema enables the queryCompiler +
# driverAdapters features, so Prisma ships no native query-engine binary, and
# bcrypt 6 publishes musl prebuilds for linux/amd64 and linux/arm64 — no
# compiler toolchain is needed. Pin by digest in CI for reproducible rebuilds.
ARG NODE_IMAGE=node:22-alpine

# Keep in sync with the root package.json's turbo dependency.
ARG TURBO_VERSION=2.5.8


# ---------------------------------------------------------------------------
# base — shared pnpm setup, inherited by every other stage
# ---------------------------------------------------------------------------
FROM ${NODE_IMAGE} AS base

# Corepack pins pnpm to the "packageManager" field in the root package.json, so
# the image never silently builds with a different pnpm than developers use.
ENV PNPM_HOME="/pnpm" \
    PATH="/pnpm:$PATH"
RUN corepack enable

WORKDIR /repo


# ---------------------------------------------------------------------------
# pruner — cut the monorepo down to just what apps/server needs
# ---------------------------------------------------------------------------
# Without this, `pnpm install` resolves the whole workspace lockfile and drags
# in apps/web's toolchain (the two @next/swc binaries alone are ~225MB).
# `turbo prune --docker` emits a sub-monorepo with a rewritten lockfile:
#   out/json  → package.json files only, for a cache-friendly install layer
#   out/full  → the source of server + the workspace packages it depends on
FROM base AS pruner
ARG TURBO_VERSION

COPY . .
RUN pnpm dlx turbo@${TURBO_VERSION} prune server --docker


# ---------------------------------------------------------------------------
# deps — install the pruned dependency graph (dev deps included; tsc needs them)
# ---------------------------------------------------------------------------
FROM base AS deps

# Manifests first: this layer is rebuilt when a dependency changes, not when
# application source changes.
COPY --from=pruner /repo/out/json/ ./
COPY --from=pruner /repo/out/pnpm-lock.yaml ./pnpm-lock.yaml

# The cache mount reuses pnpm's content-addressable store across builds without
# baking it into a layer. --frozen-lockfile makes the build fail loudly rather
# than silently resolving different versions than developers have.
RUN --mount=type=cache,id=pnpm-store,target=/pnpm/store \
    pnpm install --frozen-lockfile


# ---------------------------------------------------------------------------
# build — generate the Prisma client, compile to dist/, emit a prod-only bundle
# ---------------------------------------------------------------------------
FROM deps AS build

# Source comes in after the install so editing code doesn't invalidate deps.
COPY --from=pruner /repo/out/full/ ./

# prisma generate emits src/db/prisma/generated, which db/client.ts imports, so
# it must run before tsc or the compile fails on a missing module. It reads
# prisma.config.ts, which expects DIRECT_URL; generate never opens a connection,
# so a placeholder satisfies it and no real credentials enter the build.
ENV DATABASE_URL="postgresql://placeholder:placeholder@localhost:5432/placeholder" \
    DIRECT_URL="postgresql://placeholder:placeholder@localhost:5432/placeholder"
RUN pnpm --filter server db:generate

RUN pnpm --filter server build



# ---------------------------------------------------------------------------
# prod-deps — resolve the SAME lockfile again, production dependencies only
# ---------------------------------------------------------------------------
# A separate stage rather than pruning the build stage's node_modules in place:
# it reuses the cached manifest layers and runs in parallel with `build`.
# (`pnpm deploy` is deliberately not used here — in pnpm 9 it re-resolves
# against the registry instead of the lockfile, which both defeats
# --frozen-lockfile and drags dev-only graphs such as next into the image.)
FROM base AS prod-deps

COPY --from=pruner /repo/out/json/ ./
COPY --from=pruner /repo/out/pnpm-lock.yaml ./pnpm-lock.yaml

# --ignore-scripts is required, not cosmetic: the workspace root's `prepare`
# script runs husky, which is a devDependency and therefore absent from a --prod
# install, so the install would abort with "husky: not found". Skipping scripts
# is safe for this dependency set — bcrypt resolves its prebuilt musl binding at
# require() time through node-gyp-build, not during install.
RUN --mount=type=cache,id=pnpm-store,target=/pnpm/store \
    pnpm install --frozen-lockfile --prod --ignore-scripts


# ---------------------------------------------------------------------------
# runner — minimal runtime: no source tree, no dev deps, no package manager
# ---------------------------------------------------------------------------
FROM ${NODE_IMAGE} AS runner

# dumb-init reaps zombies and forwards SIGTERM/SIGINT. Without an init process,
# Node as PID 1 has no default signal handlers and would ignore `docker stop`,
# so every deploy would drop in-flight Socket.IO connections at the kill timeout.
RUN apk add --no-cache dumb-init

ENV NODE_ENV=production \
    PORT=3001

WORKDIR /app

# node:*-alpine already provides an unprivileged `node` user (uid 1000).
# --chown sets ownership during the copy instead of adding a second layer.
#
# The workspace layout is reproduced exactly, because pnpm's isolated
# node_modules are relative symlinks into node_modules/.pnpm — flattening the
# directories here would leave every one of them dangling.
COPY --from=prod-deps --chown=node:node /repo/node_modules ./node_modules
COPY --from=prod-deps --chown=node:node /repo/packages ./packages
COPY --from=prod-deps --chown=node:node /repo/apps/server/node_modules ./apps/server/node_modules
COPY --from=prod-deps --chown=node:node /repo/apps/server/package.json ./apps/server/package.json

# Compiled output only: no .ts sources, no tsconfig, no prisma CLI.
COPY --from=build --chown=node:node /repo/apps/server/dist ./apps/server/dist

# Drop root before any application code runs.
USER node

# Documentation only; main.ts reads $PORT and binds :: (dual-stack).
EXPOSE 3001

# Marks the container unhealthy if the event loop stops serving requests.
# wget comes from Alpine's busybox, so no extra package is required.
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
    CMD wget -q --spider "http://127.0.0.1:${PORT}/keep-alive" || exit 1

# exec form under dumb-init so node receives signals directly. Calling node
# rather than `pnpm start` keeps a package manager out of the process tree.
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "apps/server/dist/main.js"]
