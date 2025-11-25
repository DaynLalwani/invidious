# --------------------------
# Builder
# --------------------------
FROM crystallang/crystal:1.16.3 AS builder

# Install required system dependencies
RUN apt-get update && apt-get install -y \
    libxml2-dev \
    liblzma-dev \
    libyaml-dev \
    libsqlite3-dev \
    libevent-dev \
    build-essential \
    git \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /invidious

# Copy shards and install
COPY ./shard.yml ./shard.lock ./
RUN shards install --production

# Copy source and scripts
COPY ./src/ ./src/
COPY ./scripts/ ./scripts/
COPY ./assets/ ./assets/
COPY ./videojs-dependencies.yml ./videojs-dependencies.yml

# --------------------------
# Default Git info (no .git required)
# --------------------------
ARG release=1
ENV CURRENT_COMMIT=0000000
ENV CURRENT_BRANCH=main
ENV CURRENT_VERSION=0.0.1

# Build Invidious binary
RUN if [ "$release" -eq 1 ]; then \
        crystal build ./src/invidious.cr \
        --release --warnings all \
        --define CURRENT_COMMIT=\"$CURRENT_COMMIT\" \
        --define CURRENT_BRANCH=\"$CURRENT_BRANCH\" \
        --define CURRENT_VERSION=\"$CURRENT_VERSION\" \
        --link-flags "-lxml2 -llzma"; \
    else \
        crystal build ./src/invidious.cr \
        --warnings all \
        --define CURRENT_COMMIT=\"$CURRENT_COMMIT\" \
        --define CURRENT_BRANCH=\"$CURRENT_BRANCH\" \
        --define CURRENT_VERSION=\"$CURRENT_VERSION\" \
        --link-flags "-lxml2 -llzma"; \
    fi

# --------------------------
# Runtime image
# --------------------------
FROM debian:bookworm-slim

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    librsvg2-bin \
    fonts-dejavu \
    tini \
    tzdata \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /invidious

# Add non-root user
RUN addgroup --system invidious && adduser --system --ingroup invidious invidious

# Copy config and runtime files
COPY --chown=invidious ./config/config.* ./config/
RUN mv -n config/config.example.yml config/config.yml
COPY ./config/sql/ ./config/sql/
COPY ./locales/ ./locales/
COPY --from=builder /invidious/assets ./assets/
COPY --from=builder /invidious/invidious .

# Set permissions
RUN chmod o+rX -R ./assets ./config ./locales

# Expose port
EXPOSE 3000

# Run as non-root user
USER invidious

# Tini as init process
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD [ "/invidious/invidious" ]
