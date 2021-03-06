# This file was generated using a Jinja2 template.
# Please make your changes in `Dockerfile.j2` and then `make` the individual Dockerfile's.

# Using multistage build:
# 	https://docs.docker.com/develop/develop-images/multistage-build/
# 	https://whitfin.io/speeding-up-rust-docker-builds/
####################### VAULT BUILD IMAGE  #######################

#  This hash is extracted from the docker web-vault builds and it's prefered over a simple tag because it's immutable.
#  It can be viewed in multiple ways:
#  - From the https://hub.docker.com/repository/docker/bitwardenrs/web-vault/tags page, click the tag name and the digest should be there.
#  - From the console, with the following commands:
#      docker pull bitwardenrs/web-vault:v2.15.1
#      docker image inspect --format "{{.RepoDigests}}" bitwardenrs/web-vault:v2.15.1
#
#  - To do the opposite, and get the tag from the hash, you can do:
#      docker image inspect --format "{{.RepoTags}}" bitwardenrs/web-vault@sha256:afba1e3bded09dc0a6a0dbacb3363ac33b6f122b4b26d3682cafb9115bdf785c
FROM bitwardenrs/web-vault@sha256:afba1e3bded09dc0a6a0dbacb3363ac33b6f122b4b26d3682cafb9115bdf785c as vault

########################## BUILD IMAGE  ##########################
# We need to use the Rust build image, because
# we need the Rust compiler and Cargo tooling
FROM messense/rust-musl-cross:armv7-musleabihf as build

# ARM only supports SQLite for now
ARG DB=sqlite

# Build time options to avoid dpkg warnings and help with reproducible builds.
ENV DEBIAN_FRONTEND=noninteractive LANG=C.UTF-8 TZ=UTC TERM=xterm-256color

# Don't download rust docs
RUN rustup set profile minimal

ENV CARGO_HOME "/root/.cargo"
ENV USER "root"

# Creates a dummy project used to grab dependencies
RUN USER=root cargo new --bin /app
WORKDIR /app

# Copies over *only* your manifests and build files
COPY ./Cargo.* ./
COPY ./rust-toolchain ./rust-toolchain
COPY ./build.rs ./build.rs

ENV CROSS_COMPILE="1"
ENV OPENSSL_INCLUDE_DIR="/usr/include/arm-linux-gnueabihf"
ENV OPENSSL_LIB_DIR="/usr/lib/arm-linux-gnueabihf"
RUN rustup target add armv7-unknown-linux-musleabihf

# Builds your dependencies and removes the
# dummy project, except the target folder
# This folder contains the compiled dependencies
RUN cargo build --features ${DB} --release
RUN find . -not -path "./target*" -delete

# Copies the complete project
# To avoid copying unneeded files, use .dockerignore
COPY . .

# Make sure that we actually build the project
RUN touch src/main.rs

# Builds again, this time it'll just be
# your actual source files being built
RUN cargo build --features ${DB} --release --target=armv7-unknown-linux-musleabihf
RUN musl-strip target/armv7-unknown-linux-musleabihf/release/bitwarden_rs

######################## RUNTIME IMAGE  ########################
# Create a new stage with a minimal image
# because we already have a binary built
FROM cmosh/alpine-arm

ENV ROCKET_ENV "staging"
ENV ROCKET_PORT=80
ENV ROCKET_WORKERS=10

RUN [ "cross-build-start" ]

# Install needed libraries
RUN apk update && apk add openssl curl sqlite
RUN apk add --no-cache -X http://dl-cdn.alpinelinux.org/alpine/edge/testing catatonit

RUN mkdir /data

RUN [ "cross-build-end" ]

VOLUME /data
EXPOSE 80
EXPOSE 3012

# Copies the files from the context (Rocket.toml file and web-vault)
# and the binary from the "build" stage to the current stage
COPY Rocket.toml .
COPY --from=vault /web-vault ./web-vault
COPY --from=build /app/target/armv7-unknown-linux-musleabihf/release/bitwarden_rs .

COPY docker/healthcheck.sh /healthcheck.sh
COPY docker/start.sh /start.sh

HEALTHCHECK --interval=60s --timeout=10s CMD ["/healthcheck.sh"]

# Configures the startup!
WORKDIR /
CMD ["/start.sh"]
