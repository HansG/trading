FROM eclipse-temurin:26-jdk-noble

ARG USERNAME=root
ARG USERHOME=/root
ARG NODE_MAJOR=26
ARG JDK_SOURCES_URL=https://github.com/adoptium/temurin26-binaries/releases/download/jdk-26%2B35/OpenJDK26U-jdk-sources_26_35.tar.gz
ARG JDK_SOURCES_SHA256=f2efe308f37809127c96eb73ffba2d2484013aac18777d09079cea39091f0d08

# Base OS tools used by git, VS Code, NodeSource, and setup scripts.
RUN apt-get update \
    && export DEBIAN_FRONTEND=noninteractive \
    && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      git \
      gnupg \
      gzip \
      openssh-client \
    && rm -rf /var/lib/apt/lists/*

# Node.js is installed from NodeSource so Codex can be installed with npm.
RUN install -d -m 0755 /etc/apt/keyrings \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
      | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_${NODE_MAJOR}.x nodistro main" \
      > /etc/apt/sources.list.d/nodesource.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends nodejs \
    && npm install -g @openai/codex \
    && rm -rf /var/lib/apt/lists/*

# The devcontainer runs as root, matching VS Code defaults.
RUN mkdir -p "${USERHOME}/.sbt" "${USERHOME}/.cache/coursier" 

# Add JDK source files for editor navigation.
RUN set -eux; \
    curl -fsSL "$JDK_SOURCES_URL" -o /tmp/jdk-sources.tar.gz; \
    echo "$JDK_SOURCES_SHA256  /tmp/jdk-sources.tar.gz" | sha256sum -c -; \
    mkdir -p /tmp/jdk-sources /tmp/jdk-srczip; \
    tar -xzf /tmp/jdk-sources.tar.gz -C /tmp/jdk-sources --strip-components=1; \
    for classes in /tmp/jdk-sources/src/*/share/classes; do \
      module="$(basename "$(dirname "$(dirname "$classes")")")"; \
      mkdir -p "/tmp/jdk-srczip/$module"; \
      cp -R "$classes"/. "/tmp/jdk-srczip/$module"/; \
    done; \
    cd /tmp/jdk-srczip; \
    jar --create --file /opt/java/openjdk/lib/src.zip .; \
    rm -rf /tmp/jdk-sources /tmp/jdk-srczip /tmp/jdk-sources.tar.gz

# Install Coursier, then use it for Scala tools and an sbt wrapper.
#
# Option B (active): native GraalVM binary — no JVM bootstrap involved, so it is
# immune to the AbstractMethodError caused by Cats binary incompatibility when the
# Coursier bootstrap JAR runs on Java 26.
#
# Option A (reference, JVM bootstrap, pinned version):
#   ARG CS_VERSION=v2.1.25
#   curl -fsSL "https://github.com/coursier/coursier/releases/download/${CS_VERSION}/coursier" \
#     -o /usr/local/bin/cs
ARG CS_VERSION=v2.1.25
RUN set -eux; \
    curl -fsSL "https://github.com/coursier/coursier/releases/download/${CS_VERSION}/cs-x86_64-pc-linux.gz" \
      | gunzip > /usr/local/bin/cs; \
    chmod +x /usr/local/bin/cs; \
    COURSIER_CACHE="${USERHOME}/.cache/coursier" cs install scala3-compiler scala-cli metals-mcp --install-dir /usr/local/bin; \
    printf '%s\n' \
      '#!/usr/bin/env sh' \
      "export COURSIER_CACHE=\"\${COURSIER_CACHE:-${USERHOME}/.cache/coursier}\"" \
      'exec /usr/local/bin/cs launch sbt -- "$@"' > /usr/local/bin/sbt; \
    chmod +x /usr/local/bin/sbt

USER ${USERNAME}

# Fail the build early if the installed language-server bridge is broken.
RUN metals-mcp --version

# Warm the most expensive sbt downloads used by the template project.
RUN COURSIER_CACHE="${USERHOME}/.cache/coursier" \
    cs fetch org.scala-sbt:sbt-launch:1.12.11 io.get-coursier.sbt:sbt-runner:0.2.0 >/dev/null
