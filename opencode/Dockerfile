# Use a minimal base image — digest pin = multi-arch index digest for alpine:3.24
# (rotate deliberately: docker buildx imagetools inspect alpine:3.24)
FROM alpine:3.24@sha256:294b683cb724975bec92580e1e685676bd4b50bda910ddb8c51d4cabeaec77e6

# Install git, openssh-client, bash, curl+tar for the opencode installer,
# and dropbear for SSH access (runs as sudeep, needs no capabilities)
# chromium + a font set back agent-browser's headless browsing — fontconfig
# alone leaves screenshots full of tofu boxes for anything outside opensans
# (libstdc++/libgcc are runtime deps of the opencode binary)
RUN apk add --no-cache \
    git \
    openssh-client \
    bash \
    curl \
    tar \
    dropbear \
    libstdc++ \
    libgcc \
    git-lfs \
    chromium \
    font-dejavu \
    font-liberation \
    && rm -rf /var/cache/apk/*

COPY gh-askpass sandbox-entry /usr/local/bin/
RUN chmod 0755 /usr/local/bin/gh-askpass /usr/local/bin/sandbox-entry \
    && git config --system url."https://github.com/".insteadOf "git@github.com:" \
    && git config --system core.askPass /usr/local/bin/gh-askpass \
    && git config --system user.name "sudeep" \
    && git config --system user.email "yohermesbot@gmail.com"

# Create a non-root user named "sudeep"
RUN adduser -D -s /bin/bash sudeep

# Workspace mount point must be owned by sudeep so the named volume inherits it
RUN mkdir -p /workspace && chown sudeep:sudeep /workspace

# Switch to the non-root user
USER sudeep

# Set the working directory
WORKDIR /home/sudeep

# Install opencode CLI V2 (latest; untrusted-within-sandbox is fine by design).
# Must be /v2/install — the bare opencode.ai/install URL is the V1 channel and
# would silently downgrade the sandbox to V1 on the next build.
# Retried because the installer's version-metadata endpoint intermittently 500s,
# and the installer treats that as fatal.
RUN attempt=0; until curl -fsSL https://opencode.ai/v2/install | bash -s -- --no-modify-path; do \
        attempt=$((attempt + 1)); \
        [ "$attempt" -ge 3 ] && exit 1; \
        echo "opencode install failed (attempt $attempt), retrying in 5s..." >&2; \
        sleep 5; \
    done \
    && /home/sudeep/.opencode/bin/opencode --version
ENV PATH="/home/sudeep/.opencode/bin:${PATH}"

# Install agent-browser (native Rust CLI) from its release binary: the npm
# package is a `#!/usr/bin/env node` shim and this image ships no node.
# musl builds exist, so Alpine is a supported target. Installed under $HOME
# because /usr/local/bin is root-owned at this point; symlinked below.
RUN mkdir -p /home/sudeep/.local/bin \
    && case "$(uname -m)" in \
        aarch64) ab_arch=arm64 ;; \
        x86_64)  ab_arch=x64 ;; \
        *) echo "unsupported arch: $(uname -m)" >&2; exit 1 ;; \
    esac \
    && curl -fsSL -o /home/sudeep/.local/bin/agent-browser \
        "https://github.com/vercel-labs/agent-browser/releases/latest/download/agent-browser-linux-musl-$ab_arch" \
    && chmod 0755 /home/sudeep/.local/bin/agent-browser \
    && /home/sudeep/.local/bin/agent-browser --version

# `agent-browser install` downloads Chrome for Testing, which is glibc-only
# and cannot run on Alpine — point agent-browser at the distro chromium instead.
ENV AGENT_BROWSER_EXECUTABLE_PATH=/usr/bin/chromium-browser
# --no-sandbox: chromium's own sandbox needs namespace syscalls that
# cap_drop:ALL + no-new-privileges block, so it aborts at startup without this.
# The container boundary is already the isolation boundary that matters here.
# --disable-dev-shm-usage: /dev/shm is Docker's 64 MB default, too small for
# chromium, which then spams errors and can hang on render-heavy pages.
ENV AGENT_BROWSER_ARGS="--no-sandbox,--disable-dev-shm-usage,--disable-gpu"

# Install bun (latest)
RUN curl -fsSL https://bun.sh/install | bash -s -- \
    && /home/sudeep/.bun/bin/bun --version

# Install uv (latest)
RUN UV_NO_MODIFY_PATH=1 curl -LsSf https://astral.sh/uv/install.sh | sh \
    && /home/sudeep/.local/bin/uv --version

# SSH access: ~/.ssh is a named volume — the host key is generated on first
# container start (sandbox-entry) and survives rebuilds; authorized_keys is
# bind-mounted from the "authorized_keys" compose secret
RUN mkdir -p /home/sudeep/.ssh && chmod 700 /home/sudeep/.ssh

# Global opencode config for the sandbox: registers agent-browser as an MCP
# server, so headless browsing is available as tools in every project rather
# than something the agent has to shell out to. Projects may still override the
# "agent-browser" server by defining it in their own opencode.json.
COPY opencode.global.json /home/sudeep/.config/opencode/opencode.json

# Symlink system-wide since login shells reset PATH via /etc/profile
USER root
RUN ln -s /home/sudeep/.opencode/bin/opencode /usr/local/bin/opencode \
    && ln -s /home/sudeep/.bun/bin/bun /usr/local/bin/bun \
    && ln -s /home/sudeep/.local/bin/uv /usr/local/bin/uv \
    && ln -s /home/sudeep/.local/bin/uvx /usr/local/bin/uvx \
    && ln -s /home/sudeep/.local/bin/agent-browser /usr/local/bin/agent-browser \
    && chown -R sudeep:sudeep /home/sudeep/.config
USER sudeep

# Set the default shell
SHELL ["/bin/bash", "-c"]

# sandbox-entry starts dropbear, then execs bash (interactive) or idles (up -d)
CMD ["/usr/local/bin/sandbox-entry"]
