# Builder
FROM crystallang/crystal:1.16.3 AS builder

# Install dependencies
RUN apt-get update && apt-get install -y \
    libxml2-dev \
    liblzma-dev \
    libyaml-dev \
    libsqlite3-dev \
    build-essential \
    git \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /invidious

# Shards
COPY ./shard.yml ./shard.lock ./
RUN shards install --production

# Copy source
COPY ./src/ ./src/
COPY ./scripts/ ./scripts/
COPY ./assets/ ./assets/
COPY ./videojs-dependencies.yml ./videojs-dependencies.yml

# Optional: set version info
ARG CURRENT_COMMIT=0000000
ARG CURRENT_BRANCH=main
ARG CURRENT_VERSION=0.0.1

# Build
ARG release=1
RUN if [ "$release" -eq 1 ]; then \
        crystal build ./src/invidious.cr --release --warnings all --link-flags "-lxml2 -llzma"; \
    else \
        crystal build ./src/invidious.cr --warnings all --link-flags "-lxml2 -llzma"; \
    fi

# Runtime image
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    librsvg2-bin \
    fonts-dejavu \
    tini \
    tzdata \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /invidious
RUN addgroup --system invidious && adduser --system --ingroup invidious invidious

# Copy runtime files
COPY --chown=invidious ./config/config.* ./config/
RUN mv -n config/config.example.yml config/config.yml
COPY ./config/sql/ ./config/sql/
COPY ./locales/ ./locales/
COPY --from=builder /invidious/assets ./assets/
COPY --from=builder /invidious/invidious .

RUN chmod o+rX -R ./assets ./config ./locales

EXPOSE 3000
USER invidious
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD [ "/invidious/invidious" ]
