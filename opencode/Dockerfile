# Use a minimal base image
FROM alpine:3.20

# Install git, openssh-client, and bash for the terminal
RUN apk add --no-cache \
    git \
    openssh-client \
    bash \
    && rm -rf /var/cache/apk/*

COPY gh-askpass /usr/local/bin/gh-askpass
RUN chmod 0755 /usr/local/bin/gh-askpass \
    && git config --system url."https://github.com/".insteadOf "git@github.com:"

# Create a non-root user named "sudeep"
RUN adduser -D -s /bin/bash sudeep

# Workspace mount point must be owned by sudeep so the named volume inherits it
RUN mkdir -p /workspace && chown sudeep:sudeep /workspace

# Switch to the non-root user
USER sudeep

# Set the working directory
WORKDIR /home/sudeep

# Set the default shell
SHELL ["/bin/bash", "-c"]

# Start an interactive bash shell
CMD ["bash"]
