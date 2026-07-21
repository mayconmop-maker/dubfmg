# Production Dockerfile for self-hosting apps/web.
# Build context must be the monorepo root, e.g.:
#   docker build -f apps/web/Dockerfile -t dub-web .

FROM node:20-slim AS base
RUN apt-get update -y && apt-get install -y openssl ca-certificates && rm -rf /var/lib/apt/lists/*
RUN corepack enable
WORKDIR /app

FROM base AS build
COPY . .
RUN pnpm install --frozen-lockfile

# prisma generate only reads the schema, it doesn't need a reachable database,
# but DATABASE_URL must be set to a syntactically valid MySQL URL.
ARG DATABASE_URL="mysql://user:pass@localhost:3306/db"
ENV DATABASE_URL=$DATABASE_URL
ENV NEXT_TELEMETRY_DISABLED=1
RUN pnpm --filter web prisma:generate
RUN pnpm turbo build --filter=web...

FROM base AS runner
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
COPY --from=build /app /app
WORKDIR /app/apps/web
EXPOSE 3000
CMD ["pnpm", "start"]
