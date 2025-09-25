FROM mcr.microsoft.com/devcontainers/base:debian

# Set non-interactive frontend for apt
ENV DEBIAN_FRONTEND=noninteractive

# Switch to root for installing packages
USER root

# Install additional dependencies
RUN apt update && apt install -y \
    curl \
    jq \
    python3 \
    && apt clean \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js 20.x using NodeSource
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \
    apt update && \
    apt install -y nodejs && \
    apt clean && \
    rm -rf /var/lib/apt/lists/*

# Install Claude Code
RUN npm install -g @anthropic-ai/claude-code

# Ensure all users have access to the tools
RUN chmod 755 /usr/local/bin/* && \
    # Create a directory for vscode user's binaries
    mkdir -p /home/vscode/.local/bin && \
    chown -R vscode:vscode /home/vscode/.local/bin

# Switch back to vscode user
USER vscode

# Set environment variables
ENV PATH="/home/vscode/.local/bin:/root/.local/bin:$PATH"

# Create .bashrc additions for PATH
RUN echo 'export PATH="/usr/local/bin:$HOME/.local/bin:$PATH"' >> $HOME/.bashrc

# Set the default command
CMD ["sleep", "infinity"]
