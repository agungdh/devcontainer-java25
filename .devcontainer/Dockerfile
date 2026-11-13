# Debian 13 (trixie) - Java 25 devcontainer
FROM debian:13

ARG DEBIAN_FRONTEND=noninteractive
ARG USERNAME=agungdh
ARG USER_UID=1000
ARG USER_GID=1000

# ---------------------------------------------------------------------------
# Base packages (JDK tidak dari apt; dipasang lewat SDKMAN di bawah)
# ---------------------------------------------------------------------------
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        wget \
        git \
        openssh-client \
        gnupg \
        sudo \
        make \
        jq \
        unzip \
        zip \
        xz-utils \
        tar \
        less \
        vim \
        nano \
        procps \
        htop \
        tree \
        ripgrep \
        file \
        locales \
        sqlite3 \
    && rm -rf /var/lib/apt/lists/*

# Locale (UTF-8)
RUN sed -i -e 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen \
    && locale-gen
ENV LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8

# ---------------------------------------------------------------------------
# Non-root user with passwordless sudo
# ---------------------------------------------------------------------------
RUN groupadd --gid ${USER_GID} ${USERNAME} \
    && useradd --uid ${USER_UID} --gid ${USER_GID} --create-home --shell /bin/bash ${USERNAME} \
    && echo "${USERNAME} ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME} \
    && chmod 0440 /etc/sudoers.d/${USERNAME} \
    && mkdir -p /workspaces \
    && chown ${USER_UID}:${USER_GID} /workspaces

# ---------------------------------------------------------------------------
# Java: Liberica JDK 25 (LTS) terbaru via SDKMAN
# Cek versi baru: sdk list java | grep librca
# ---------------------------------------------------------------------------
USER ${USERNAME}
ENV SDKMAN_DIR=/home/${USERNAME}/.sdkman
RUN curl -s "https://get.sdkman.io?rcupdate=false" | bash \
    && bash -c 'set -e; export sdkman_auto_answer=true; . "${SDKMAN_DIR}/bin/sdkman-init.sh" \
        && sdk install java 25.0.5.0+11-librca \
        && sdk default java 25.0.5.0+11-librca'
ENV JAVA_HOME=/home/${USERNAME}/.sdkman/candidates/java/current \
    PATH=/home/${USERNAME}/.sdkman/candidates/java/current/bin:$PATH

# ---------------------------------------------------------------------------
# Docker CLI only (daemon di host, container pakai /var/run/docker.sock)
# Sumber repo resmi: https://docs.docker.com/engine/install/debian/
# ---------------------------------------------------------------------------
USER root
RUN install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
    && chmod a+r /etc/apt/keyrings/docker.asc \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
        | tee /etc/apt/sources.list.d/docker.list > /dev/null \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        docker-ce-cli \
        docker-buildx-plugin \
        docker-compose-plugin \
    && rm -rf /var/lib/apt/lists/*

# Samakan GID grup docker dengan socket host saat container start
RUN printf '%s\n' \
        '#!/bin/sh' \
        'set -e' \
        'if [ -S /var/run/docker.sock ]; then' \
        '    gid=$(stat -c "%g" /var/run/docker.sock)' \
        '    if getent group docker >/dev/null 2>&1; then' \
        '        current=$(getent group docker | cut -d: -f3)' \
        '        [ "$current" = "$gid" ] || sudo groupmod --gid "$gid" docker' \
        '    else' \
        '        sudo groupadd --gid "$gid" docker' \
        '    fi' \
        '    sudo usermod -aG docker '"${USERNAME}" \
        'fi' \
        'exec "$@"' \
        > /usr/local/share/docker-init.sh \
    && chmod +x /usr/local/share/docker-init.sh

# ---------------------------------------------------------------------------
# Node (nvm + Node.js 24)
# ---------------------------------------------------------------------------
USER ${USERNAME}
ENV NVM_DIR=/home/${USERNAME}/.nvm
RUN curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.41.0/install.sh | bash \
    && bash -c '. "${NVM_DIR}/nvm.sh" && nvm install 24 && nvm alias default 24'

# ---------------------------------------------------------------------------
# opencode
# ---------------------------------------------------------------------------
RUN curl -fsSL https://opencode.ai/v2/install | bash
ENV PATH=/home/${USERNAME}/.opencode/bin:$PATH

# ---------------------------------------------------------------------------
# Global npm packages
# ---------------------------------------------------------------------------
RUN bash -c '. "${NVM_DIR}/nvm.sh" \
    && npm install -g npm@latest \
    && npm i -g command-code@latest serve concurrently \
    && npm update -g'

# ---------------------------------------------------------------------------
# Claude Code
# ---------------------------------------------------------------------------
RUN curl -fsSL https://claude.ai/install.sh | bash
ENV PATH=/home/${USERNAME}/.local/bin:$PATH

# Login shell (/etc/profile) me-reset PATH, jadi daftarkan ulang via profile.d
USER root
RUN PROFILE_PATH="/home/${USERNAME}/.local/bin:/home/${USERNAME}/.opencode/bin:/home/${USERNAME}/.sdkman/candidates/java/current/bin" \
    && echo "export PATH=\"${PROFILE_PATH}:\$PATH\"" \
        | tee /etc/profile.d/devcontainer-path.sh > /dev/null \
    && echo "export JAVA_HOME=\"/home/${USERNAME}/.sdkman/candidates/java/current\"" \
        | tee /etc/profile.d/java.sh > /dev/null \
    && echo "export NVM_DIR=\"${NVM_DIR}\"; [ -s \"\$NVM_DIR/nvm.sh\" ] && . \"\$NVM_DIR/nvm.sh\"" \
        | tee /etc/profile.d/nvm.sh > /dev/null \
    && echo "export SDKMAN_DIR=\"${SDKMAN_DIR}\"; [ -s \"\$SDKMAN_DIR/bin/sdkman-init.sh\" ] && . \"\$SDKMAN_DIR/bin/sdkman-init.sh\"" \
        | tee /etc/profile.d/sdkman.sh > /dev/null

WORKDIR /workspaces

ENTRYPOINT ["/usr/local/share/docker-init.sh"]
CMD ["sleep", "infinity"]
