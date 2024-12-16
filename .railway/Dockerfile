ARG NODE_VERSION=22.11.0-bullseye

# Build stage
FROM node:${NODE_VERSION} AS builder

RUN apt-get update && apt-get install -yqq --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

RUN corepack enable

WORKDIR /misskey

COPY ["pnpm-lock.yaml", "pnpm-workspace.yaml", "package.json", "./"]
COPY ["scripts", "./scripts"]
COPY ["packages/backend/package.json", "./packages/backend/"]
COPY ["packages/frontend-shared/package.json", "./packages/frontend-shared/"]
COPY ["packages/frontend/package.json", "./packages/frontend/"]
COPY ["packages/frontend-embed/package.json", "./packages/frontend-embed/"]
COPY ["packages/sw/package.json", "./packages/sw/"]
COPY ["packages/misskey-js/package.json", "./packages/misskey-js/"]
COPY ["packages/misskey-reversi/package.json", "./packages/misskey-reversi/"]
COPY ["packages/misskey-bubble-game/package.json", "./packages/misskey-bubble-game/"]

ARG NODE_ENV=production

RUN pnpm i --frozen-lockfile --aggregate-output

COPY . ./

RUN pnpm build

# Runner stage
FROM node:${NODE_VERSION}-slim AS runner

ARG UID="991"
ARG GID="991"

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    ffmpeg tini curl libjemalloc-dev libjemalloc2 \
    && ln -s /usr/lib/$(uname -m)-linux-gnu/libjemalloc.so.2 /usr/local/lib/libjemalloc.so \
    && corepack enable \
    && groupadd -g "${GID}" misskey \
    && useradd -l -u "${UID}" -g "${GID}" -m -d /misskey misskey \
    && find / -type d -path /sys -prune -o -type d -path /proc -prune -o -type f -perm /u+s -ignore_readdir_race -exec chmod u-s {} \; \
    && find / -type d -path /sys -prune -o -type d -path /proc -prune -o -type f -perm /g+s -ignore_readdir_race -exec chmod g-s {} \; \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists

USER misskey
WORKDIR /misskey

COPY --chown=misskey:misskey ./package.json ./package.json
RUN corepack install

COPY --chown=misskey:misskey --from=builder /misskey/node_modules ./node_modules
COPY --chown=misskey:misskey --from=builder /misskey/packages/backend/node_modules ./packages/backend/node_modules
COPY --chown=misskey:misskey --from=builder /misskey/packages/misskey-js/node_modules ./packages/misskey-js/node_modules
COPY --chown=misskey:misskey --from=builder /misskey/packages/misskey-reversi/node_modules ./packages/misskey-reversi/node_modules
COPY --chown=misskey:misskey --from=builder /misskey/packages/misskey-bubble-game/node_modules ./packages/misskey-bubble-game/node_modules
COPY --chown=misskey:misskey --from=builder /misskey/built ./built
COPY --chown=misskey:misskey --from=builder /misskey/packages/misskey-js/built ./packages/misskey-js/built
COPY --chown=misskey:misskey --from=builder /misskey/packages/misskey-reversi/built ./packages/misskey-reversi/built
COPY --chown=misskey:misskey --from=builder /misskey/packages/misskey-bubble-game/built ./packages/misskey-bubble-game/built
COPY --chown=misskey:misskey --from=builder /misskey/packages/backend/built ./packages/backend/built
COPY --chown=misskey:misskey --from=builder /misskey/fluent-emojis /misskey/fluent-emojis

ENV NODE_ENV=production
ENV LD_PRELOAD=/usr/local/lib/libjemalloc.so

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["pnpm", "run", "start"]

HEALTHCHECK --start-period=5s --interval=5s --timeout=5s --retries=20 \
    CMD curl -f http://localhost:3000/api/ping || exit 1
