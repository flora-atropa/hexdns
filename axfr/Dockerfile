FROM ubuntu:22.04 AS builder
ARG TOOLCHAIN=stable

ARG CARGO_ABOUT_VERSION=0.4.4
ARG CARGO_AUDIT_VERSION=0.16.0
ARG CARGO_DENY_VERSION=0.11.0
ARG ZLIB_VERSION=1.3.1

RUN apt-get update && \
    export DEBIAN_FRONTEND=noninteractive && \
    apt-get install -yq build-essential cmake curl file git graphviz musl-dev musl-tools libpq-dev libsqlite-dev \
    libssl-dev linux-libc-dev pkgconf sudo unzip xutils-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    useradd rust --user-group --create-home --shell /bin/bash --groups sudo

RUN curl -fLO https://github.com/EmbarkStudios/cargo-about/releases/download/$CARGO_ABOUT_VERSION/cargo-about-$CARGO_ABOUT_VERSION-x86_64-unknown-linux-musl.tar.gz && \
    tar xf cargo-about-$CARGO_ABOUT_VERSION-x86_64-unknown-linux-musl.tar.gz && \
    mv cargo-about-$CARGO_ABOUT_VERSION-x86_64-unknown-linux-musl/cargo-about /usr/local/bin/ && \
    rm -rf cargo-about-$CARGO_ABOUT_VERSION-x86_64-unknown-linux-musl.tar.gz cargo-about-$CARGO_ABOUT_VERSION-x86_64-unknown-linux-musl && \
    curl -fLO https://github.com/rustsec/rustsec/releases/download/cargo-audit%2Fv${CARGO_AUDIT_VERSION}/cargo-audit-x86_64-unknown-linux-gnu-v${CARGO_AUDIT_VERSION}.tgz && \
    tar xf cargo-audit-x86_64-unknown-linux-gnu-v${CARGO_AUDIT_VERSION}.tgz && \
    cp cargo-audit-x86_64-unknown-linux-gnu-v${CARGO_AUDIT_VERSION}/cargo-audit /usr/local/bin/ && \
    rm -rf cargo-audit-x86_64-unknown-linux-gnu-v${CARGO_AUDIT_VERSION}.tgz cargo-audit-x86_64-unknown-linux-gnu-v${CARGO_AUDIT_VERSION} && \
    curl -fLO https://github.com/EmbarkStudios/cargo-deny/releases/download/$CARGO_DENY_VERSION/cargo-deny-$CARGO_DENY_VERSION-x86_64-unknown-linux-musl.tar.gz && \
    tar xf cargo-deny-$CARGO_DENY_VERSION-x86_64-unknown-linux-musl.tar.gz && \
    mv cargo-deny-$CARGO_DENY_VERSION-x86_64-unknown-linux-musl/cargo-deny /usr/local/bin/ && \
    rm -rf cargo-deny-$CARGO_DENY_VERSION-x86_64-unknown-linux-musl cargo-deny-$CARGO_DENY_VERSION-x86_64-unknown-linux-musl.tar.gz

RUN ln -s "/usr/bin/g++" "/usr/bin/musl-g++"

RUN echo "Building zlib" && \
    cd /tmp && \
    curl -fLO "http://zlib.net/zlib-$ZLIB_VERSION.tar.gz" && \
    tar xzf "zlib-$ZLIB_VERSION.tar.gz" && cd "zlib-$ZLIB_VERSION" && \
    CC=musl-gcc ./configure --static --prefix=/usr/local/musl && \
    make && make install && \
    rm -r /tmp/*

ENV RUSTUP_HOME=/opt/rust/rustup \
    PATH=/home/rust/.cargo/bin:/opt/rust/cargo/bin:/usr/local/musl/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

RUN curl https://sh.rustup.rs -sSf | \
    env CARGO_HOME=/opt/rust/cargo \
        sh -s -- -y --default-toolchain $TOOLCHAIN --profile minimal --no-modify-path && \
    env CARGO_HOME=/opt/rust/cargo \
        rustup component add rustfmt && \
    env CARGO_HOME=/opt/rust/cargo \
        rustup component add clippy && \
    env CARGO_HOME=/opt/rust/cargo \
        rustup target add x86_64-unknown-linux-musl
ADD cargo-config.toml /opt/rust/cargo/config

ENV X86_64_UNKNOWN_LINUX_MUSL_OPENSSL_DIR=/usr/local/musl/ \
    X86_64_UNKNOWN_LINUX_MUSL_OPENSSL_STATIC=1 \
    PQ_LIB_STATIC_X86_64_UNKNOWN_LINUX_MUSL=1 \
    PG_CONFIG_X86_64_UNKNOWN_LINUX_GNU=/usr/bin/pg_config \
    PKG_CONFIG_ALLOW_CROSS=true \
    PKG_CONFIG_ALL_STATIC=true \
    LIBZ_SYS_STATIC=1 \
    TARGET=musl \
    AR=ar

USER rust
RUN mkdir -p /home/rust/libs /home/rust/src /home/rust/.cargo && \
    ln -s /opt/rust/cargo/config /home/rust/.cargo/config

WORKDIR /home/rust/src

RUN curl -LO https://github.com/protocolbuffers/protobuf/releases/download/v23.3/protoc-23.3-linux-x86_64.zip && \
  mkdir -p ~/.local && \
  unzip protoc-23.3-linux-x86_64.zip -d ~/.local && \
  rm protoc-23.3-linux-x86_64.zip && \
  chmod +x ~/.local/bin/protoc

ENV PROTOC=/home/rust/.local/bin/protoc

ADD --chown=rust:rust . ./
RUN USER=rust cargo build --release

FROM scratch

COPY --from=builder --chown=0:0 /etc/ssl/certs /etc/ssl/certs
COPY --from=builder --chown=0:0 /home/rust/src/target/x86_64-unknown-linux-musl/release/axfr /

ENTRYPOINT ["/axfr"]
