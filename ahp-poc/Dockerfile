FROM ubuntu:24.04
RUN apt-get update && \
    apt-get install -y \
      openssh-server \
      git \
      curl \
      ca-certificates \
      bash \
      build-essential \
      nodejs \
      npm && \
    rm -rf /var/lib/apt/lists/*
RUN useradd -m -s /bin/bash vscode && \
    mkdir -p /run/sshd /home/vscode/.ssh && \
    chown -R vscode:vscode /home/vscode
COPY ssh/authorized_keys /home/vscode/.ssh/authorized_keys
RUN chown vscode:vscode /home/vscode/.ssh/authorized_keys && \
    chmod 700 /home/vscode/.ssh && \
    chmod 600 /home/vscode/.ssh/authorized_keys
RUN mkdir -p /workspace && \
    chown vscode:vscode /workspace
# Generate host keys explicitly. Ubuntu's openssh-server postinst normally
# does this, but it can silently be skipped in minimal/non-interactive
# Docker builds, leaving sshd unable to start.
RUN ssh-keygen -A
# Harden defaults: this account has no password set, so password auth is
# already effectively impossible, but disable it explicitly for defense in
# depth rather than relying on that implicit behavior.
RUN sed -i \
      -e 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' \
      -e 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' \
      /etc/ssh/sshd_config
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D", "-e"]
