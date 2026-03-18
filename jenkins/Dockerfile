FROM jenkins/jenkins:lts-jdk17

USER root

# Install Node.js (npm/npx), Python, uv, and socketcli
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        python3 python3-venv git curl ca-certificates && \
    curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \
    apt-get install -y --no-install-recommends nodejs && \
    python3 -m venv /opt/socket-env && \
    /opt/socket-env/bin/pip install socketsecurity && \
    ln -s /opt/socket-env/bin/socketcli /usr/local/bin/socketcli && \
    curl -LsSf https://astral.sh/uv/install.sh | UV_INSTALL_DIR=/usr/local/bin sh && \
    rm -rf /var/lib/apt/lists/*

USER jenkins

ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli -f /usr/share/jenkins/ref/plugins.txt
