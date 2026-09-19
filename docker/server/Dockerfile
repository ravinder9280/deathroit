FROM node:20-alpine

ENV PNPM_HOME="/pnpm" \
    PATH="/pnpm:$PATH"
RUN corepack enable

WORKDIR /repo

# Copy workspace manifests for dependency caching
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY apps/server/package.json apps/server/
COPY packages/ packages/

RUN pnpm install --frozen-lockfile --filter server...

# Copy server source
COPY apps/server apps/server

EXPOSE 5000

CMD ["pnpm", "--filter", "server", "dev"]