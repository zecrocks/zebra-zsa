# syntax=docker/dockerfile:1

ARG FEATURES="default-release-binaries"
ARG APP_HOME="/opt/zebrad"
ARG RUST_VERSION=1.82.0

FROM rust:${RUST_VERSION}-bookworm AS builder

WORKDIR /build

RUN apt-get -qq update && \
    apt-get -qq install -y --no-install-recommends \
    llvm \
    libclang-dev \
    clang \
    protobuf-compiler \
    && rm -rf /var/lib/apt/lists/* /tmp/*

ARG FEATURES
ENV RUST_LOG=info
ENV RUST_BACKTRACE=1

COPY . .

RUN cargo build --locked --release --features "${FEATURES}" --package zebrad --bin zebrad && \
    cp target/release/zebrad /usr/local/bin/

COPY ./docker/entrypoint.sh /etc/zebrad/entrypoint.sh

FROM debian:bookworm-slim AS runtime

# Set the default path for the zebrad binary
ARG APP_HOME
ENV APP_HOME=${APP_HOME}
WORKDIR ${APP_HOME}

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    rocksdb-tools \
    gosu \
    && rm -rf /var/lib/apt/lists/* /tmp/*

ARG USER=zebra
ENV USER=${USER}
ARG UID=10001
ENV UID=${UID}
ARG GID=10001
ENV GID=${GID}

RUN addgroup --system --gid ${GID} ${USER} \
    && adduser \
    --system \
    --disabled-login \
    --shell /bin/bash \
    --home ${APP_HOME} \
    --uid "${UID}" \
    --gid "${GID}" \
    ${USER}

# Config settings for zebrad
ARG FEATURES
ENV FEATURES=${FEATURES}

# Path and name of the config file
# These are set to a default value when not defined in the environment
ENV ZEBRA_CONF_DIR=${ZEBRA_CONF_DIR:-/etc/zebrad}
ENV ZEBRA_CONF_FILE=${ZEBRA_CONF_FILE:-zebrad.toml}

RUN mkdir -p ${ZEBRA_CONF_DIR} && chown ${UID}:${UID} ${ZEBRA_CONF_DIR} \
    && chown ${UID}:${UID} ${APP_HOME}

COPY --from=builder /usr/local/bin/zebrad /usr/local/bin
COPY --from=builder /etc/zebrad/entrypoint.sh /etc/zebrad/entrypoint.sh

# Expose configured ports
EXPOSE 8233 18233

# Update the config file based on the Docker run variables,
# and launch zebrad with it
ENTRYPOINT [ "/etc/zebrad/entrypoint.sh" ]
CMD ["zebrad"]
