# Squelette à compléter — étape 3.
# Mêmes contraintes que les autres services (multi-stage, non-root, ≤ 200 Mo).

# ---- Stage 1 : build / install des dépendances ----
FROM python:3.12.11-alpine3.22 AS build

WORKDIR /app

COPY requirements.txt ./
RUN python -m venv /opt/venv && \
    /opt/venv/bin/pip install --no-cache-dir --requirement requirements.txt

COPY app ./app

# ---- Stage 2 : runtime ----
FROM python:3.12.11-alpine3.22 AS runtime

ARG SOURCE_REPOSITORY="https://github.com/louis-guidez/devhub-campus"
LABEL org.opencontainers.image.source="${SOURCE_REPOSITORY}"

WORKDIR /app

RUN addgroup -S -g 1001 app && adduser -S -D -H -u 1001 -G app app

COPY --from=build --chown=1001:1001 /opt/venv /opt/venv
COPY --from=build --chown=1001:1001 /app/app ./app

ENV PORT=8080 \
    LOG_LEVEL=info \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:${PATH}"

EXPOSE 8080

USER 1001:1001

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
