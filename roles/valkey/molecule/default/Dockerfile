ARG BASE_IMAGE=ubuntu:24.04
FROM ${BASE_IMAGE}
ARG SSH_PORT=2222
ENV container=docker
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    --no-install-recommends systemd systemd-sysv dbus python3 python3-apt sudo iproute2 \
    openssh-server openssh-client nftables ca-certificates && rm -rf /var/lib/apt/lists/* \
    && mkdir -p /etc/ssh/sshd_config.d \
    && printf 'Port %s\n' "$SSH_PORT" > /etc/ssh/sshd_config.d/valkey-test.conf \
    && systemctl disable ssh.socket && systemctl enable ssh.service
STOPSIGNAL SIGRTMIN+3
CMD ["/sbin/init"]
