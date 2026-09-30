# frpc - frp client
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
    && mv /tmp/frp_${FRP_VERSION}_linux_${TARGETARCH}/frpc /usr/local/bin/frpc \
    && rm -rf /tmp/frp*

COPY frpc.toml /etc/frp/frpc.toml

# frpc has no fixed listening port: publish whatever your configuration uses,
# and note that a proxy's remotePort is opened on the frps side.
EXPOSE 7400

# Started without arguments the bundled configuration is used:
#   docker run -d -v $PWD/frpc.toml:/etc/frp/frpc.toml ghcr.io/jiangood/frpc
#
# Pass your own file, or any other flag, as the container command:
#   docker run -d -v $PWD/frpc.toml:/data/frpc.toml ghcr.io/jiangood/frpc -c /data/frpc.toml
#   docker run --rm ghcr.io/jiangood/frpc --version
ENTRYPOINT ["frpc"]
CMD ["-c", "/etc/frp/frpc.toml"]
