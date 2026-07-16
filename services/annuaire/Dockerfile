# ---- Stage 1 : build / install des dépendances ----
FROM node:20.19.4-alpine3.22 AS build

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY src ./src

# ---- Stage 2 : image runtime ----
FROM node:20.19.4-alpine3.22 AS runtime

ARG SOURCE_REPOSITORY="https://github.com/louis-guidez/devhub-campus"
LABEL org.opencontainers.image.source="${SOURCE_REPOSITORY}"

WORKDIR /app

RUN addgroup -S -g 1001 app && adduser -S -D -H -u 1001 -G app app

COPY --from=build --chown=1001:1001 /app/node_modules ./node_modules
COPY --from=build --chown=1001:1001 /app/src ./src
COPY --from=build --chown=1001:1001 /app/package.json ./package.json

USER 1001:1001

ENV PORT=8080 \
    LOG_LEVEL=info \
    NODE_ENV=production

EXPOSE 8080

CMD ["node", "src/index.js"]
