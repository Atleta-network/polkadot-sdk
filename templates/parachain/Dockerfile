# syntax=docker/dockerfile:1.7
FROM docker.io/paritytech/ci-unified:latest as builder

WORKDIR /polkadot
COPY . /polkadot

RUN cargo fetch
COPY tools/ci/install_sccache.sh tools/ci/with_cargo_cache.sh /tmp/cargo-cache/
RUN sh /tmp/cargo-cache/install_sccache.sh

RUN --mount=type=secret,id=cargo_cache_env \
    sh /tmp/cargo-cache/with_cargo_cache.sh cargo build --workspace --locked --release

FROM docker.io/parity/base-bin:latest

COPY --from=builder /polkadot/target/release/parachain-template-node /usr/local/bin

USER root
RUN useradd -m -u 1001 -U -s /bin/sh -d /polkadot polkadot && \
	mkdir -p /data /polkadot/.local/share && \
	chown -R polkadot:polkadot /data && \
	ln -s /data /polkadot/.local/share/polkadot && \
# unclutter and minimize the attack surface
	rm -rf /usr/bin /usr/sbin && \
# check if executable works in this container
	/usr/local/bin/parachain-template-node --version

USER polkadot

EXPOSE 30333 9933 9944 9615
VOLUME ["/data"]

ENTRYPOINT ["/usr/local/bin/parachain-template-node"]
