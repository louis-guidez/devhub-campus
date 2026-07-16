# Squelette — étape 3.
# Cible : image FINALE ≤ 50 Mo (Go produit un binaire statique, profitez-en).

# ---- Stage 1 : build du binaire Go ----
FROM golang:1.22-alpine AS build

WORKDIR /src

COPY go.mod ./
RUN go mod download

COPY cmd ./cmd
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/notif ./cmd

# ---- Stage 2 : runtime minimal ----
FROM gcr.io/distroless/static:nonroot AS runtime

ARG SOURCE_REPOSITORY="https://github.com/louis-guidez/devhub-campus"
LABEL org.opencontainers.image.source="${SOURCE_REPOSITORY}"

COPY --from=build --chown=65532:65532 /out/notif /notif

ENV PORT=8080 \
    LOG_LEVEL=info

EXPOSE 8080

USER 65532:65532

ENTRYPOINT ["/notif"]
