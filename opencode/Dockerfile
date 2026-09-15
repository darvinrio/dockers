# Use a minimal base image
FROM alpine:3.20

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
    && rm -rf /var/cache/apk/*

COPY gh-askpass sandbox-entry /usr/local/bin/
RUN chmod 0755 /usr/local/bin/gh-askpass /usr/local/bin/sandbox-entry \
    && git config --system url."https://github.com/".insteadOf "git@github.com:"

# Create a non-root user named "sudeep"
RUN adduser -D -s /bin/bash sudeep

# Workspace mount point must be owned by sudeep so the named volume inherits it
RUN mkdir -p /workspace && chown sudeep:sudeep /workspace

# Switch to the non-root user
USER sudeep

# Set the working directory
WORKDIR /home/sudeep

# Install opencode CLI (latest); musl build is auto-detected on Alpine
RUN curl -fsSL https://opencode.ai/install | bash -s -- --no-modify-path \
    && /home/sudeep/.opencode/bin/opencode --version
ENV PATH="/home/sudeep/.opencode/bin:${PATH}"

# SSH access: ~/.ssh is a named volume — the host key is generated on first
# container start (sandbox-entry) and survives rebuilds; authorized_keys is
# bind-mounted from the "authorized_keys" compose secret
RUN mkdir -p /home/sudeep/.ssh && chmod 700 /home/sudeep/.ssh

# Symlink system-wide since login shells reset PATH via /etc/profile
USER root
RUN ln -s /home/sudeep/.opencode/bin/opencode /usr/local/bin/opencode
USER sudeep

# Set the default shell
SHELL ["/bin/bash", "-c"]

# sandbox-entry starts dropbear, then execs bash (interactive) or idles (up -d)
CMD ["/usr/local/bin/sandbox-entry"]
