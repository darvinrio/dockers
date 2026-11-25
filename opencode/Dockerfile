# Use a minimal base image — digest pin = multi-arch index digest for alpine:3.24
# (rotate deliberately: docker buildx imagetools inspect alpine:3.24)
# 2026-11-11: re-pinned to 3.24.4. ca-certificates is still 20261020-r0, so the
# eMudhra trust-store checks below are unchanged from the 2026-10-27 pin.
FROM alpine:3.24@sha256:f59abe656b692d901e184be3445e084786602108599d2a51d229c0f5a48c71bf

# Install git, openssh-client, bash, curl+tar for the opencode/pi installers,
# and dropbear for SSH access (runs as sudeep, needs no capabilities)
# chromium + a font set back agent-browser's headless browsing — fontconfig
# alone leaves screenshots full of tofu boxes for anything outside opensans
# (libstdc++/libgcc are runtime deps of the opencode binary)
#
# nodejs + npm are Pi's runtime (it needs Node >= 22.19 and npm). Alpine's musl
# build is the only usable Node here — nodejs.org's official tarballs are
# glibc-linked. npm lives in the "community" repo and has to be >= 11.6 for the
# pi installer's --min-release-age flag.
#
# Upgrade the pinned base first: a stale digest also means a stale
# ca-certificates bundle, and TLS clients here do not consult revocation lists,
# so the only fix for a distrusted CA is a newer bundle. ca-certificates(-bundle)
# is explicit so a future base without it cannot silently regress.
#
# eMudhra: eight emSign intermediate CAs had recoverable private keys (Oct
# 2026). NSS 3.131.1 / alpine ca-certificates 20261020-r0 drop the affected
# roots (ECC G3, C1, ECC C3) and keep emSign Root CA - G1. Fail the build if
# one comes back — the container pulls code and model responses over TLS.
RUN apk upgrade --no-cache \
    && apk add --no-cache \
    ca-certificates \
    ca-certificates-bundle \
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
    nodejs \
    npm \
    && rm -rf /var/cache/apk/* \
    && [ ! -e /usr/share/ca-certificates/mozilla/emSign_ECC_Root_CA_-_G3.crt ] \
    && [ ! -e /usr/share/ca-certificates/mozilla/emSign_Root_CA_-_C1.crt ] \
    && [ ! -e /usr/share/ca-certificates/mozilla/emSign_ECC_Root_CA_-_C3.crt ]

COPY gh-askpass sandbox-entry /usr/local/bin/
RUN chmod 0755 /usr/local/bin/gh-askpass /usr/local/bin/sandbox-entry \
    && git config --system url."https://github.com/".insteadOf "git@github.com:" \
    && git config --system core.askPass /usr/local/bin/gh-askpass \
    && git config --system user.name "sudeep" \
    && git config --system user.email "yohermesbot@gmail.com"

# Create a non-root user named "sudeep"
RUN adduser -D -s /bin/bash sudeep

# Pi's release is installed here, outside $HOME: ~/.pi becomes a named volume at
# runtime, and a volume would mask a release baked into the image, pinning pi to
# whatever version the volume was first created with.
RUN mkdir -p /opt/pi && chown sudeep:sudeep /opt/pi

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

# Install pi (latest; untrusted-within-sandbox is fine by design). The managed
# installer pins dependencies with pi's npm-shrinkwrap and verifies the release
# metadata. Retried because pi.dev and the npm registry are external deps.
# PI_CODING_AGENT_DIR retargets the build-time install to /opt/pi/agent so the
# release itself never sits under ~/.pi (see /opt/pi above). At runtime the
# variable is unset, so pi reads settings/auth/sessions from ~/.pi/agent.
RUN attempt=0; until curl -fsSL -o /tmp/pi-install.sh https://pi.dev/install.sh \
        && PI_CODING_AGENT_DIR=/opt/pi/agent sh /tmp/pi-install.sh; do \
        attempt=$((attempt + 1)); \
        [ "$attempt" -ge 3 ] && exit 1; \
        echo "pi install failed (attempt $attempt), retrying in 5s..." >&2; \
        sleep 5; \
    done \
    && rm -f /tmp/pi-install.sh \
    && /opt/pi/agent/bin/pi --version

# Mount points for the compose named volumes. They must exist in the image and
# be owned by sudeep: Docker copies the directory's ownership into a new named
# volume on first use, and a missing path would leave the volume root owned by
# root, so opencode/pi could not write their session data.
RUN mkdir -p /home/sudeep/.local/share/opencode \
             /home/sudeep/.local/state/opencode \
             /home/sudeep/.pi/agent

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

# The system ca-certificates bundle above does not reach tools that ship their
# own trust store: opencode is Bun-compiled, Node embeds its own root list, and
# both Bun and uv embed CA lists.
#   - Bun < 1.4.4 trusts the compromised eMudhra roots (GHSA-26q8-h657-pf84);
#     opencode 2.2.0 is the first release built with Bun 1.4.4.
#   - uv < 0.13.4 has the same problem (GHSA-hq85-5cpj-mr62).
# Enforce the patched floors so a channel rollback fails the build instead of
# silently shipping a toolchain that accepts forged certificates. The binary
# checks are the real assertion: they look for the removed root name in the
# compiled-in bundle, so they also catch an opencode/pi rebuild on an old
# Node/Bun. Node's check asks the running binary for its roots instead of
# grepping it, because Alpine splits those out into libnode.so.
RUN set -eu; \
    ge() { \
        awk -v a="$1" -v b="$2" 'BEGIN { \
            n = split(a, x, "."); m = split(b, y, "."); \
            for (i = 1; i <= 3; i++) { \
                xi = (i <= n) ? x[i] + 0 : 0; yi = (i <= m) ? y[i] + 0 : 0; \
                if (xi > yi) exit 0; if (xi < yi) exit 1; \
            } exit 0; \
        }'; \
    }; \
    ver=$(/home/sudeep/.opencode/bin/opencode --version | awk '{print $NF}'); ver=${ver#v}; \
    ge "$ver" 2.2.0 || { echo "opencode $ver predates the eMudhra trust-store fix" >&2; exit 1; }; \
    ! grep -q "emSign ECC Root CA - G3" /home/sudeep/.opencode/bin/opencode; \
    node -e 'var roots = require("node:tls").rootCertificates; if (roots.some(function (c) { return c.indexOf("emSign ECC Root CA - G3") !== -1; })) process.exit(1)' \
        || { echo "node's embedded root store still contains emSign ECC Root CA - G3" >&2; exit 1; }; \
    ver=$(/opt/pi/agent/bin/pi --version | awk '{print $NF}'); ver=${ver#v}; \
    ge "$ver" 1.0.0 || { echo "pi $ver predates built-in MCP support" >&2; exit 1; }; \
    ver=$(/home/sudeep/.bun/bin/bun --version); \
    ge "$ver" 1.4.4 || { echo "bun $ver predates the eMudhra trust-store fix" >&2; exit 1; }; \
    ver=$(/home/sudeep/.local/bin/uv --version | awk '{print $2}'); \
    ge "$ver" 0.13.4 || { echo "uv $ver predates the eMudhra trust-store fix" >&2; exit 1; }

# SSH access: ~/.ssh is a named volume — the host key is generated on first
# container start (sandbox-entry) and survives rebuilds; authorized_keys is
# bind-mounted from the "authorized_keys" compose secret
RUN mkdir -p /home/sudeep/.ssh && chmod 700 /home/sudeep/.ssh

# Global opencode config for the sandbox: registers agent-browser as an MCP
# server, so headless browsing is available as tools in every project rather
# than something the agent has to shell out to. Projects may still override the
# "agent-browser" server by defining it in their own opencode.json.
COPY opencode.global.json /home/sudeep/.config/opencode/opencode.json

# Same server for pi, from pi's user-level MCP file (~/.pi/agent/mcp.json).
# Projects override it with a .pi/mcp.json entry of the same name, which pi only
# reads after project trust is granted.
COPY pi.mcp.json /home/sudeep/.pi/agent/mcp.json

# Symlink system-wide since login shells reset PATH via /etc/profile
USER root
RUN ln -s /home/sudeep/.opencode/bin/opencode /usr/local/bin/opencode \
    && ln -s /opt/pi/agent/bin/pi /usr/local/bin/pi \
    && ln -s /home/sudeep/.bun/bin/bun /usr/local/bin/bun \
    && ln -s /home/sudeep/.local/bin/uv /usr/local/bin/uv \
    && ln -s /home/sudeep/.local/bin/uvx /usr/local/bin/uvx \
    && ln -s /home/sudeep/.local/bin/agent-browser /usr/local/bin/agent-browser \
    && chown -R sudeep:sudeep /home/sudeep/.config /home/sudeep/.pi
USER sudeep

# Set the default shell
SHELL ["/bin/bash", "-c"]

# sandbox-entry starts dropbear, then execs bash (interactive) or idles (up -d)
CMD ["/usr/local/bin/sandbox-entry"]
