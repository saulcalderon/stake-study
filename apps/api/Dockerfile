# apps/api/Dockerfile
# IMPORTANT: build context must be the repo ROOT (Dokploy: Docker Context Path = ".")
# so the lockfile and workspace manifests are visible to the build.

FROM node:24-alpine AS base
RUN corepack enable
WORKDIR /app

# ---- pruner: compute the API's subset of the monorepo via Turborepo ----
# Outputs:
#   out/json/  -> package.json files + pruned yarn.lock (for cached installs)
#   out/full/  -> source of the API and any internal packages/* it depends on
FROM base AS pruner
COPY . .
RUN yarn dlx turbo prune @quiz-hackathon/api --docker

# ---- deps: install only the pruned dependency graph (cached unless manifests change) ----
FROM base AS deps
COPY --from=pruner /app/out/json/ ./
RUN yarn install --immutable

# ---- build: prisma generate + nest build ----
FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY --from=pruner /app/out/full/ ./
RUN yarn workspace @quiz-hackathon/api prisma generate
RUN yarn workspace @quiz-hackathon/api build

# ---- runner: slim production image ----
FROM node:24-alpine AS runner
RUN corepack enable
WORKDIR /app
ENV NODE_ENV=production

COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/apps/api/dist ./apps/api/dist
COPY --from=build /app/apps/api/package.json ./apps/api/package.json
COPY --from=build /app/apps/api/prisma ./apps/api/prisma

WORKDIR /app/apps/api
EXPOSE 3000

# Apply pending migrations (production-safe, non-interactive), then start the API.
CMD ["sh", "-c", "/app/node_modules/.bin/prisma migrate deploy && node dist/main"]
