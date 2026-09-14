# Use a minimal base image
FROM alpine:3.20

# Install git, openssh-client, and bash for the terminal
RUN apk add --no-cache \
    git \
    openssh-client \
    bash \
    && rm -rf /var/cache/apk/*

# Create a non-root user named "sandbox"
RUN adduser -D -s /bin/bash sudeep

# Switch to the non-root user
USER sudeep

# Set the working directory
WORKDIR /home/sudeep

# Set the default shell
SHELL ["/bin/bash", "-c"]

# Start an interactive bash shell
CMD ["bash"]
