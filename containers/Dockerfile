FROM alpine:edge AS git
ARG TAG
ENV TAG=${TAG:-main}
RUN apk update && apk add --no-cache \
	git \
	wget \
	curl
WORKDIR /tmp/
RUN arch=$(uname -m) \
  && s6_version=$(curl -s https://api.github.com/repos/just-containers/s6-overlay/releases/latest | grep '"tag_name":' | awk -F '"' '{print $4}') \
  && wget https://github.com/just-containers/s6-overlay/releases/download/${s6_version}/s6-overlay-noarch.tar.xz -O /tmp/s6-overlay-noarch.tar.xz \
  && wget https://github.com/just-containers/s6-overlay/releases/download/${s6_version}/s6-overlay-${arch}.tar.xz -O /tmp/s6-overlay-arch.tar.xz
RUN git -c advice.detachedHead=false clone --branch $TAG --recurse-submodules --shallow-submodules https://github.com/pfm-powerforme/rustdesk-server.git rustdesk-server
WORKDIR /tmp/rustdesk-server/


FROM rust:1-alpine AS builder
ARG ARCH="x86-64"
RUN rustup component add rustfmt
RUN apk update && apk add --no-cache \
	musl-dev \
	build-base
COPY --from=git /tmp/rustdesk-server/ /tmp/rustdesk-server/
WORKDIR /tmp/rustdesk-server/
RUN cargo build --release --all-features


FROM busybox:stable AS runtime
COPY --from=git /tmp/s6-overlay-noarch.tar.xz /tmp/s6-overlay-noarch.tar.xz
COPY --from=git /tmp/s6-overlay-arch.tar.xz /tmp/s6-overlay-arch.tar.xz
RUN \
  tar -C / -Jxpf /tmp/s6-overlay-noarch.tar.xz && \
  tar -C / -Jxpf /tmp/s6-overlay-arch.tar.xz && \
  rm /tmp/s6-overlay*.tar.xz && \
  ln -s /run /var/run
COPY rootfs /
COPY --from=builder /tmp/rustdesk-server/target/release/hbbs /usr/bin/hbbs
COPY --from=builder /tmp/rustdesk-server/target/release/hbbr /usr/bin/hbbr
COPY --from=builder /tmp/rustdesk-server/target/release/rustdesk-utils /usr/bin/rustdesk-utils
ENV RELAY=relay.example.com
ENV ENCRYPTED_ONLY=0
EXPOSE 21115 21116 21116/udp 21117 21118 21119
HEALTHCHECK --interval=10s --timeout=5s CMD /usr/bin/healthcheck.sh
WORKDIR /data
VOLUME /data
ENTRYPOINT ["/init"]
