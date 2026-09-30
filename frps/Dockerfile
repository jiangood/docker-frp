# frps - frp server
# Upstream project: https://github.com/fatedier/frp
#
# The official release archive is downloaded with ADD and unpacked, so the
# binary is the same one upstream publishes.
#
# Platform matrix (which TARGETARCH maps to which upstream archive, and which
# platforms are built by default) is documented in the README.
FROM alpine:3.24

ARG FRP_VERSION=0.71.0
ARG TARGETARCH=amd64

ADD https://github.com/fatedier/frp/releases/download/v${FRP_VERSION}/frp_${FRP_VERSION}_linux_${TARGETARCH}.tar.gz /tmp/frp.tar.gz

RUN tar -xzf /tmp/frp.tar.gz -C /tmp \
    && mv /tmp/frp_${FRP_VERSION}_linux_${TARGETARCH}/frps /usr/local/bin/frps \
    && rm -rf /tmp/frp*

COPY frps.toml /etc/frp/frps.toml

# 7000 = bindPort, 7500 = dashboard. Publish whatever your configuration uses.
EXPOSE 7000 7500

# Started without arguments the bundled configuration is used:
#   docker run -d -p 7000:7000 ghcr.io/jiangood/frps
#
# Pass your own file, or any other flag, as the container command:
#   docker run -d -v $PWD/frps.toml:/data/frps.toml ghcr.io/jiangood/frps -c /data/frps.toml
#   docker run --rm ghcr.io/jiangood/frps --version
#
# frps usually needs to bind privileged ports such as 80/443 for vhost
# forwarding, so the image intentionally stays on root. Override with
# --user 65534:65534 when your configuration only uses ports >= 1024.
ENTRYPOINT ["frps"]
CMD ["-c", "/etc/frp/frps.toml"]
