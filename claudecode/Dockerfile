# syntax=docker/dockerfile:1
ARG BUILD_FROM
FROM ${BUILD_FROM}

# Labels
LABEL maintainer="Robson Felix"
LABEL org.opencontainers.image.title="Claude Code for Home Assistant"
LABEL org.opencontainers.image.description="Claude Code CLI with web terminal and HA integration"
LABEL org.opencontainers.image.source="https://github.com/robsonfelix/robsonfelix-hass-addons"
LABEL org.opencontainers.image.licenses="MIT"

# Environment setup
ENV \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    TERM=xterm-256color

# Install system dependencies (ttyd installed separately as static binary)
RUN apk add --no-cache \
    bash \
    curl \
    git \
    github-cli \
    jq \
    nodejs \
    npm \
    openssh-client \
    libstdc++ \
    ncurses \
    vim \
    nano \
    tmux \
    coreutils \
    findutils \
    grep \
    sed \
    gawk \
    ca-certificates \
    openssl \
    ripgrep \
    p7zip \
    socat \
    docker-cli \
    && update-ca-certificates

# Install ttyd static binary (avoids libwebsockets issues)
ARG BUILD_ARCH
RUN case "${BUILD_ARCH}" in \
        amd64) TTYD_ARCH="x86_64" ;; \
        aarch64) TTYD_ARCH="aarch64" ;; \
        armv7|armhf) TTYD_ARCH="armhf" ;; \
        i386) TTYD_ARCH="i686" ;; \
        *) TTYD_ARCH="x86_64" ;; \
    esac && \
    curl -fsSL "https://github.com/tsl0922/ttyd/releases/download/1.7.7/ttyd.${TTYD_ARCH}" -o /usr/bin/ttyd && \
    chmod +x /usr/bin/ttyd

# Claude Code is intentionally NOT baked into the image. It is installed at
# runtime into persistent storage (/homeassistant/.claudecode/npm-global) so its
# version is decoupled from the add-on and can always be updated to the latest.

# Install hass-mcp for Home Assistant integration and pymodbus for Modbus operations
RUN pip3 install --no-cache-dir --break-system-packages hass-mcp pymodbus pyserial

# Install mbpoll (Modbus command line tool) from source
RUN apk add --no-cache --virtual .build-deps \
    build-base \
    cmake \
    libmodbus-dev \
    && git clone https://github.com/epsilonrt/mbpoll.git /tmp/mbpoll \
    && cd /tmp/mbpoll \
    && mkdir build && cd build \
    && cmake .. \
    && make \
    && make install \
    && cd / \
    && rm -rf /tmp/mbpoll \
    && apk del .build-deps \
    && apk add --no-cache libmodbus

# Pre-cache Playwright MCP server (npx cache avoids /usr/local/bin/mcp conflict with hass-mcp)
ENV NPM_CONFIG_CACHE=/root/.npm
RUN npx @playwright/mcp --version || true

# Install Home Assistant CLI (ha command)
RUN case "${BUILD_ARCH}" in \
        amd64) HA_ARCH="amd64" ;; \
        aarch64) HA_ARCH="aarch64" ;; \
        armv7|armhf) HA_ARCH="armhf" ;; \
        i386) HA_ARCH="i386" ;; \
        *) HA_ARCH="amd64" ;; \
    esac && \
    curl -fsSL "https://github.com/home-assistant/cli/releases/latest/download/ha_${HA_ARCH}" -o /usr/local/bin/ha && \
    chmod +x /usr/local/bin/ha

# Create necessary directories (NOT /root/.claude - will be symlinked at runtime)
RUN mkdir -p \
    /homeassistant \
    /root/.config \
    /root/.local/bin

# Configure tmux: large scrollback + touch/mobile-friendly defaults.
# Mouse capture is OFF so native browser scrolling, long-press selection and
# copy/paste work on phones/tablets; the status bar is hidden to save space.
RUN cat > /root/.tmux.conf << 'EOF'
set -g history-limit 50000
# Disable alternate screen buffer so the terminal's own scrollback works
set -ga terminal-overrides ',xterm*:smcup@:rmcup@'
# Mouse OFF: let the browser/touchscreen handle scrolling and copy/paste
set -g mouse off
# Snappy ESC handling for the Claude Code TUI (avoids arrow/ESC lag)
set -sg escape-time 10
# Hide the status bar to save screen space on small/mobile displays
set -g status off
EOF

# Configure bash with aliases, prompt, and the persistent Claude Code on PATH
RUN cat > /root/.bashrc << 'EOF'
export TERM=xterm-256color
export LANG=C.UTF-8

# Claude Code lives in persistent storage (decoupled from the add-on image so it
# always runs the latest version). Keep it on PATH and make npm -g install/update
# land in the same persistent prefix.
export NPM_CONFIG_PREFIX=/homeassistant/.claudecode/npm-global
export PATH="/homeassistant/.claudecode/npm-global/bin:/root/.local/bin:$PATH"

PS1='\[\033[1;36m\]claude-code\[\033[0m\]:\[\033[1;34m\]\w\[\033[0m\]\$ '

# Function to update MCP token before starting Claude
update_mcp_token() {
  local SETTINGS_FILE=/root/.claude/settings.json
  if [ -f "$SETTINGS_FILE" ] && [ -n "$SUPERVISOR_TOKEN" ]; then
    jq ".mcpServers.homeassistant.env.HASS_TOKEN = \"$SUPERVISOR_TOKEN\"" "$SETTINGS_FILE" > /tmp/settings.tmp 2>/dev/null && mv /tmp/settings.tmp "$SETTINGS_FILE"
  fi
}

# Aliases
alias ll='ls -la'
alias c='update_mcp_token && claude'
alias cc='update_mcp_token && claude --continue'
alias claude-yolo='update_mcp_token && IS_SANDBOX=1 claude --dangerously-skip-permissions'
alias claude-update='npm install -g @anthropic-ai/claude-code@latest && hash -r && claude --version'
alias ha-config='cd /homeassistant'
alias ha-logs='cat /homeassistant/home-assistant.log 2>/dev/null || echo "Log not found"'
EOF

# Ensure .bashrc is sourced for login shells too
RUN echo 'source ~/.bashrc' > /root/.profile

# Point npm's global prefix at persistent storage so Claude Code (and any other
# global npm package) survives add-on rebuilds and is updatable to the latest
# version at any time. This is what decouples the Claude Code version from the
# add-on image. Created after all build-time npm/npx usage so the build is
# unaffected (the prefix dir only exists at runtime).
RUN printf 'prefix=/homeassistant/.claudecode/npm-global\n' > /root/.npmrc

# Auto-launch wrapper: starts Claude Code automatically inside the persistent
# session and drops to a shell when Claude exits, so the session stays usable.
RUN cat > /usr/local/bin/claude-launch << 'EOF'
#!/bin/bash
export TERM=xterm-256color
export LANG=C.UTF-8
export NPM_CONFIG_PREFIX=/homeassistant/.claudecode/npm-global
export PATH="/homeassistant/.claudecode/npm-global/bin:/root/.local/bin:$PATH"

OPTIONS=/data/options.json
SKIP_PERMS=$(jq -r '.claude_skip_permissions // true' "$OPTIONS" 2>/dev/null)
EXTRA_ARGS=$(jq -r '.claude_extra_args // ""' "$OPTIONS" 2>/dev/null)
WORKDIR=$(jq -r '.working_directory // "/homeassistant"' "$OPTIONS" 2>/dev/null)

# Keep the Home Assistant Supervisor token fresh for the hass-mcp server
if [ -f /root/.claude/settings.json ] && [ -n "$SUPERVISOR_TOKEN" ]; then
  jq ".mcpServers.homeassistant.env.HASS_TOKEN = \"$SUPERVISOR_TOKEN\"" /root/.claude/settings.json > /tmp/settings.tmp 2>/dev/null && mv /tmp/settings.tmp /root/.claude/settings.json
fi

# Make sure Claude Code is installed (first boot / fresh persistent storage)
if ! command -v claude >/dev/null 2>&1; then
  echo '[claude-launch] Claude Code not found - installing the latest version...'
  npm install -g @anthropic-ai/claude-code@latest && hash -r
fi

cd "${WORKDIR:-/homeassistant}" 2>/dev/null || cd /homeassistant

ARGS=""
if [ "$SKIP_PERMS" = "true" ]; then
  export IS_SANDBOX=1
  ARGS="--dangerously-skip-permissions"
fi
if [ -n "$EXTRA_ARGS" ] && [ "$EXTRA_ARGS" != "null" ]; then
  ARGS="$ARGS $EXTRA_ARGS"
fi

echo "[claude-launch] Launching: claude $ARGS"
echo "[claude-launch] Exit Claude to drop to a shell. This session persists across browser reconnects."
echo
claude $ARGS

# When Claude exits, hand over to an interactive login shell so the tmux pane
# (and thus the persistent session) stays alive and usable.
echo
echo "[claude-launch] Claude exited - you are now in a shell. Run 'claude-launch' to relaunch Claude."
exec bash --login
EOF
RUN chmod +x /usr/local/bin/claude-launch

# Set up PATH (persistent npm prefix first so the user-updatable Claude wins)
# and make npm's global prefix persistent for every context (CMD + shells).
ENV PATH="/homeassistant/.claudecode/npm-global/bin:/root/.local/bin:${PATH}" \
    NPM_CONFIG_PREFIX="/homeassistant/.claudecode/npm-global"

# Set working directory
WORKDIR /homeassistant

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -sf http://localhost:7681/ || exit 1

# Startup script that reads config and persists Claude auth
# Use /homeassistant/.claudecode which is always persistent
ENTRYPOINT []
CMD ["/bin/bash", "-c", "\
  export HA_TOKEN=\"$SUPERVISOR_TOKEN\" && \
  export HA_URL=\"http://supervisor/core\" && \
  PERSIST_DIR=/homeassistant/.claudecode && \
  mkdir -p $PERSIST_DIR/config /root/.config && \
  printf '%s\n' \
    '# Claude Code - Home Assistant Add-on' \
    '' \
    '## Path Mapping' \
    '' \
    'In this add-on container, paths are mapped differently than HA Core:' \
    '- \\`/homeassistant\\` = HA config directory (equivalent to \\`/config\\` in HA Core)' \
    '- \\`/config\\` does NOT exist - always use \\`/homeassistant\\`' \
    '' \
    'When users mention \\`/config/...\\`, translate to \\`/homeassistant/...\\`' \
    '' \
    '## Available Paths' \
    '' \
    '| Path | Description | Access |' \
    '|------|-------------|--------|' \
    '| \\`/homeassistant\\` | HA configuration | read-write |' \
    '| \\`/share\\` | Shared folder | read-write |' \
    '| \\`/media\\` | Media files | read-write |' \
    '| \\`/ssl\\` | SSL certificates | read-only |' \
    '| \\`/backup\\` | Backups | read-only |' \
    '' \
    '## Home Assistant Integration' \
    '' \
    'Use the \\`homeassistant\\` MCP server to query entities and call services.' \
    '' \
    '## Reading Home Assistant Logs' \
    '' \
    '**Log levels (from most to least verbose):**' \
    '- \\`debug\\` - Only shown if explicitly enabled in configuration.yaml' \
    '- \\`info\\` - General information, shown by default' \
    '- \\`warning\\` - Warnings, always shown' \
    '- \\`error\\` - Errors, always shown' \
    '' \
    '**Commands to read logs:**' \
    '\\`\\`\\`bash' \
    '# View recent logs (ha CLI)' \
    'ha core logs 2>&1 | tail -100' \
    '' \
    '# Filter by keyword' \
    'ha core logs 2>&1 | grep -i keyword' \
    '' \
    '# Filter errors only' \
    'ha core logs 2>&1 | grep -iE \"(error|exception)\"' \
    '' \
    '# Alternative: read log file directly' \
    'tail -100 /homeassistant/home-assistant.log' \
    '\\`\\`\\`' \
    '' \
    '**To enable debug logging for an integration**, add to \\`configuration.yaml\\`:' \
    '\\`\\`\\`yaml' \
    'logger:' \
    '  default: info' \
    '  logs:' \
    '    custom_components.YOUR_INTEGRATION: debug' \
    '\\`\\`\\`' \
    '' \
    '**Key insight:** \\`_LOGGER.debug()\\` calls are invisible unless the logger level is set to debug. Use \\`_LOGGER.info()\\` or \\`_LOGGER.warning()\\` for logs that should always appear.' \
    > $PERSIST_DIR/CLAUDE.md && \
  if [ ! -L /root/.claude ]; then rm -rf /root/.claude; ln -s $PERSIST_DIR /root/.claude; fi && \
  if [ ! -L /root/.config/claude-code ]; then rm -rf /root/.config/claude-code; ln -s $PERSIST_DIR/config /root/.config/claude-code; fi && \
  if [ ! -L /root/.claude.json ]; then touch $PERSIST_DIR/.claude.json; rm -f /root/.claude.json; ln -s $PERSIST_DIR/.claude.json /root/.claude.json; fi && \
  FONT_SIZE=$(jq -r '.terminal_font_size // 14' /data/options.json) && \
  THEME=$(jq -r '.terminal_theme // \"dark\"' /data/options.json) && \
  SESSION_PERSIST=$(jq -r '.session_persistence // true' /data/options.json) && \
  ENABLE_MCP=$(jq -r '.enable_mcp // true' /data/options.json) && \
  ENABLE_PLAYWRIGHT=$(jq -r '.enable_playwright_mcp // false' /data/options.json) && \
  PLAYWRIGHT_HOST=$(jq -r '.playwright_cdp_host // \"\"' /data/options.json) && \
  if [ -z \"$PLAYWRIGHT_HOST\" ] && [ \"$ENABLE_PLAYWRIGHT\" = \"true\" ]; then \
    echo '[INFO] Auto-detecting Playwright Browser hostname...' && \
    PLAYWRIGHT_HOST=$(curl -s -H \"Authorization: Bearer $SUPERVISOR_TOKEN\" http://supervisor/addons | jq -r '.data.addons[] | select(.slug | endswith(\"playwright-browser\") or endswith(\"_playwright-browser\")) | .hostname' | head -1) && \
    if [ -n \"$PLAYWRIGHT_HOST\" ] && [ \"$PLAYWRIGHT_HOST\" != \"null\" ]; then \
      echo \"[INFO] Found Playwright Browser: $PLAYWRIGHT_HOST\"; \
    else \
      echo '[WARN] Playwright Browser add-on not found, using default hostname'; \
      PLAYWRIGHT_HOST=\"playwright-browser\"; \
    fi; \
  fi && \
  AUTO_UPDATE=$(jq -r '.auto_update_claude // true' /data/options.json) && \
  mkdir -p /homeassistant/.claudecode/npm-global && \
  hash -r && \
  if ! claude --version >/dev/null 2>&1; then \
    echo '[INFO] Installing Claude Code (latest) into persistent storage...' && \
    npm install -g @anthropic-ai/claude-code@latest || echo '[WARN] Claude Code install failed - check network, then run claude-update in the terminal'; \
  elif [ \"$AUTO_UPDATE\" = \"true\" ]; then \
    echo '[INFO] Updating Claude Code to the latest version...' && \
    npm install -g @anthropic-ai/claude-code@latest >/dev/null 2>&1 || echo '[WARN] Update failed - using the installed version'; \
  fi && \
  hash -r && \
  echo \"[INFO] Claude Code version: $(claude --version 2>/dev/null || echo 'not installed')\" && \
  claude mcp remove homeassistant -s user 2>/dev/null || true && \
  claude mcp remove playwright -s user 2>/dev/null || true && \
  if [ \"$ENABLE_MCP\" = \"true\" ] && command -v claude >/dev/null 2>&1; then \
    claude mcp add-json homeassistant '{\"command\":\"hass-mcp\"}' -s user && \
    SETTINGS_FILE=/root/.claude/settings.json && \
    ALLOWED_TOOLS='[\"mcp__homeassistant__get_version\",\"mcp__homeassistant__get_entity\",\"mcp__homeassistant__list_entities\",\"mcp__homeassistant__search_entities_tool\",\"mcp__homeassistant__domain_summary_tool\",\"mcp__homeassistant__list_automations\",\"mcp__homeassistant__get_history\",\"mcp__homeassistant__get_error_log\",\"Read(/homeassistant/**)\",\"Read(/config/**)\",\"Read(/share/**)\",\"Read(/media/**)\",\"Glob(/homeassistant/**)\",\"Glob(/config/**)\",\"Grep(/homeassistant/**)\",\"Grep(/config/**)\"]' && \
    jq --argjson tools \"$ALLOWED_TOOLS\" '.permissions.allow = ($tools + (.permissions.allow // []) | unique)' $SETTINGS_FILE > /tmp/settings.tmp && mv /tmp/settings.tmp $SETTINGS_FILE && \
    echo '[INFO] MCP configured with Home Assistant integration'; \
    echo '[INFO] Pre-authorized read-only MCP tools'; \
  else \
    echo '[INFO] MCP disabled'; \
  fi && \
  if [ \"$ENABLE_PLAYWRIGHT\" = \"true\" ] && command -v claude >/dev/null 2>&1; then \
    claude mcp add-json playwright \"{\\\"command\\\":\\\"npx\\\",\\\"args\\\":[\\\"--no-install\\\",\\\"@playwright/mcp\\\",\\\"--cdp-endpoint\\\",\\\"http://${PLAYWRIGHT_HOST}:9222\\\"]}\" -s user && \
    echo \"[INFO] Playwright MCP enabled (CDP: http://${PLAYWRIGHT_HOST}:9222)\"; \
    echo '[INFO] Make sure the Playwright Browser add-on is installed and running'; \
  else \
    echo '[INFO] Playwright MCP disabled'; \
  fi && \
  if [ \"$THEME\" = \"dark\" ]; then \
    COLORS='background=#1e1e2e,foreground=#cdd6f4,cursor=#f5e0dc'; \
  else \
    COLORS='background=#eff1f5,foreground=#4c4f69,cursor=#dc8a78'; \
  fi && \
  AUTO_START=$(jq -r '.auto_start_claude // true' /data/options.json) && \
  if [ \"$AUTO_START\" = \"true\" ]; then \
    LAUNCH_CMD='/usr/local/bin/claude-launch'; \
  else \
    LAUNCH_CMD='bash --login'; \
  fi && \
  if [ \"$SESSION_PERSIST\" = \"true\" ]; then \
    SHELL_CMD=\"tmux new-session -A -s claude $LAUNCH_CMD\"; \
  else \
    SHELL_CMD=\"$LAUNCH_CMD\"; \
  fi && \
  cd /homeassistant && \
  exec ttyd --port 7681 --writable --ping-interval 30 --max-clients 5 \
    -t fontSize=$FONT_SIZE \
    -t fontFamily=Monaco,Consolas,monospace \
    -t scrollback=20000 \
    -t \"theme=$COLORS\" \
    $SHELL_CMD \
"]
