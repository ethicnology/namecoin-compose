# Namecoin Core 30.2 (built from source)
# Build: podman build -t namecoin-core .

FROM debian:trixie AS builder

RUN apt-get update \
    && apt-get -y --no-install-recommends install \
        build-essential cmake pkgconf python3 \
        libevent-dev libboost-dev \
        git ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src
RUN git clone --branch nc30.2 --depth 1 \
        https://github.com/namecoin/namecoin-core.git

WORKDIR /usr/src/namecoin-core
RUN cmake -B build \
        -DBUILD_GUI=OFF \
        -DENABLE_WALLET=OFF \
        -DENABLE_IPC=OFF \
        -DBUILD_TESTS=OFF
RUN cmake --build build -j$(nproc)


FROM debian:trixie-slim

RUN apt-get update \
    && apt-get -y --no-install-recommends install \
        libevent-2.1-7t64 libevent-extra-2.1-7t64 libevent-pthreads-2.1-7t64 \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd -r namecoin && useradd -r -g namecoin -m namecoin

COPY --from=builder /usr/src/namecoin-core/build/bin/namecoind /usr/local/bin/
COPY --from=builder /usr/src/namecoin-core/build/bin/namecoin-cli /usr/local/bin/
COPY namecoin.conf /etc/namecoin/namecoin.conf

RUN mkdir -p /home/namecoin/.namecoin \
    && chown -R namecoin:namecoin /home/namecoin

VOLUME /home/namecoin/.namecoin

USER namecoin

EXPOSE 8334 8336

CMD ["namecoind", "-printtoconsole", "-conf=/etc/namecoin/namecoin.conf"]
