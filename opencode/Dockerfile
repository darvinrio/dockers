# Use a minimal base image — digest pin = multi-arch index digest for alpine:3.20
# (rotate deliberately: docker buildx imagetools inspect alpine:3.20)
FROM alpine:3.20@sha256:d9e853e87e55526f6b2917df91a2115c36dd7c696a35be12163d44e6e2a4b6bc

# Install git, openssh-client, bash, curl+tar for the opencode installer,
# and dropbear for SSH access (runs as sudeep, needs no capabilities)
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

# Install opencode CLI (latest; untrusted-within-sandbox is fine by design)
RUN curl -fsSL https://opencode.ai/install | bash -s -- --no-modify-path \
    && /home/sudeep/.opencode/bin/opencode --version
ENV PATH="/home/sudeep/.opencode/bin:${PATH}"

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

# Symlink system-wide since login shells reset PATH via /etc/profile
USER root
RUN ln -s /home/sudeep/.opencode/bin/opencode /usr/local/bin/opencode \
    && ln -s /home/sudeep/.bun/bin/bun /usr/local/bin/bun \
    && ln -s /home/sudeep/.local/bin/uv /usr/local/bin/uv \
    && ln -s /home/sudeep/.local/bin/uvx /usr/local/bin/uvx
USER sudeep

# Set the default shell
SHELL ["/bin/bash", "-c"]

# sandbox-entry starts dropbear, then execs bash (interactive) or idles (up -d)
CMD ["/usr/local/bin/sandbox-entry"]
