# --------------------------
# Builder
# --------------------------
FROM crystallang/crystal:1.16.3 AS builder

# Install build dependencies
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

# Copy source, scripts, assets
COPY ./src/ ./src/
COPY ./scripts/ ./scripts/
COPY ./assets/ ./assets/
COPY ./videojs-dependencies.yml ./videojs-dependencies.yml

# --------------------------
# Set default Git info
# --------------------------
ENV CURRENT_COMMIT=0000000
ENV CURRENT_BRANCH=main
ENV CURRENT_VERSION=0.0.1
ENV RELEASE=1

# Build Invidious
RUN crystal build ./src/invidious.cr \
    --release --warnings all \
    --define CURRENT_COMMIT=$CURRENT_COMMIT \
    --define CURRENT_BRANCH=$CURRENT_BRANCH \
    --define CURRENT_VERSION=$CURRENT_VERSION \
    --link-flags "-lxml2 -llzma"

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
    postgresql-client \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /invidious

# Create non-root user
RUN addgroup --system invidious && adduser --system --ingroup invidious invidious

# Copy config and locales
COPY --chown=invidious ./config/config.* ./config/
RUN mv -n config/config.example.yml config/config.yml
COPY ./config/sql/ ./config/sql/
COPY ./locales/ ./locales/

# Copy built assets and binary
COPY --from=builder /invidious/assets ./assets/
COPY --from=builder /invidious/invidious .

# Set permissions
RUN chmod o+rX -R ./assets ./config ./locales

# --------------------------
# Configure database via environment variables
# --------------------------
ENV DB_HOST=invidious-db
ENV DB_PORT=5432
ENV DB_USER=postgres
ENV DB_PASSWORD=postgres
ENV DB_NAME=invidious

# Patch config.yml at runtime to use env variables
RUN sed -i "s/host:.*/host: ${DB_HOST}/" config/config.yml && \
    sed -i "s/dbname:.*/dbname: ${DB_NAME}/" config/config.yml && \
    sed -i "s/user:.*/user: ${DB_USER}/" config/config.yml && \
    sed -i "s/password:.*/password: ${DB_PASSWORD}/" config/config.yml && \
    sed -i "s/port:.*/port: ${DB_PORT}/" config/config.yml

# Expose port
EXPOSE 3000

# Run as non-root user
USER invidious

# Tini as init process
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD [ "/invidious/invidious" ]
