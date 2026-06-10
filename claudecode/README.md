# Claude Code for Home Assistant

Run [Claude Code](https://docs.anthropic.com/en/docs/claude-code), Anthropic's AI-powered coding assistant, directly in your Home Assistant sidebar with full access to your configuration.

## Quick Start

```bash
claude "List all my automations"
claude "Turn off all lights in the living room"
claude "Create an automation to turn on lights at sunset"
claude "Why isn't my motion sensor automation working?"
```

## Requirements

- Home Assistant OS or Supervised installation
- [Anthropic account](https://console.anthropic.com/) (authentication handled in terminal)

## Features

- **Web Terminal**: Access Claude Code through a browser-based terminal
- **Always Latest**: Claude Code is decoupled from the add-on and updated to the newest version on every boot
- **Auto-Start**: Claude launches automatically when you open the terminal — nothing to type after a reboot
- **Config Access**: Read and write Home Assistant configuration files
- **hass-mcp Integration**: Direct control of HA entities and services
- **Background Sessions**: tmux keeps Claude running across browser disconnects, tuned to stay out of the way on mobile
- **Customizable Theme**: Choose between dark and light terminal themes
- **Multi-Architecture**: Supports amd64, aarch64, armv7, armhf, and i386
- **Secure Authentication**: Claude Code handles its own authentication securely

## Setup

### 1. Install the Add-on

1. Add the repository to Home Assistant
2. Install the "Claude Code" add-on
3. Start the add-on
4. Open the Web UI from the sidebar

### 2. Authenticate with Claude Code

On first launch, Claude Code will prompt you to authenticate:

1. Open the terminal from the HA sidebar
2. Type `claude` to start
3. Follow the authentication prompts
4. Your credentials are stored securely by Claude Code

**Note**: The add-on does NOT require you to enter API keys in the configuration. Claude Code handles authentication itself, storing credentials securely in its own configuration directory. This is more secure than storing keys in Home Assistant's add-on config.

## Using Claude Code

### Basic Usage

Once authenticated, Claude Code is ready to help with:

- Editing Home Assistant YAML configurations
- Creating automations and scripts
- Debugging configuration issues
- Writing custom integrations

### Home Assistant Integration

With hass-mcp enabled, Claude can:

- Query entity states: "What's the temperature in the living room?"
- Control devices: "Turn off all lights in the bedroom"
- List services: "What services are available for climate control?"
- Debug automations: "Why didn't my morning routine trigger?"

### Example Commands

```bash
# Start interactive session
claude

# One-off commands
claude "Add a new automation that turns on the porch light at sunset"
claude "Check my configuration.yaml for errors"
claude "List all unavailable entities"

# Continue previous conversation
claude --continue
```

### Keyboard Shortcuts

| Shortcut | Command |
|----------|---------|
| `c` | `claude` |
| `cc` | `claude --continue` |
| `claude-yolo` | `claude --dangerously-skip-permissions` (with `IS_SANDBOX=1`) |
| `claude-update` | Update Claude Code to the latest version |
| `ha-config` | Navigate to config directory |
| `ha-logs` | View Home Assistant logs |

## Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `enable_mcp` | Enable HA integration | true |
| `terminal_font_size` | Font size (10-24) | 14 |
| `terminal_theme` | dark or light | dark |
| `working_directory` | Start directory | /homeassistant |
| `session_persistence` | Use tmux to keep Claude running across reconnects | true |
| `auto_update_claude` | Update Claude Code to the latest version on every boot | true |
| `auto_start_claude` | Launch Claude automatically when the terminal opens | true |
| `claude_skip_permissions` | Run Claude with `--dangerously-skip-permissions` (no per-action prompts) | true |
| `claude_extra_args` | Extra CLI flags appended to the auto-started Claude (e.g. `--model opus`) | "" |
| `enable_chat_ui` | Also run the CloudCLI chat web UI on port 3001 (mobile-friendly; see below) | false |

## Chat Web UI (CloudCLI) — mobile-friendly, experimental

Prefer a chat window over the terminal (especially on a phone)? Enable `enable_chat_ui`
to also run **[CloudCLI](https://github.com/siteboon/claudecodeui)** — a responsive web
chat front-end for Claude Code. It's an alternative to the terminal TUI: a normal web
page that scrolls natively, with no tmux and no pull-to-refresh fighting.

**Enable it:** set `enable_chat_ui: true` and restart the add-on. On first start it
installs CloudCLI into persistent storage (takes a minute), then serves it on port
**3001**. It reuses your existing Claude login, HA file access and MCP servers — nothing
extra to set up on the Claude side.

**Open it:** `http://<your-ha-ip>:3001` (e.g. `http://homeassistant.local:3001`).
CloudCLI has its **own login**, which you create on first visit; it persists in
`/homeassistant/.claudecode/cloudcli`.

> **Why not the HA sidebar?** CloudCLI's frontend uses absolute asset/websocket paths,
> which don't survive HA ingress's dynamic sub-path — so it runs on a direct port rather
> than a sidebar panel. The terminal add-on keeps its sidebar panel.

**Optional sidebar shortcut** — add a "Claude Chat" entry to the HA sidebar by putting
this in `configuration.yaml` and restarting HA:

```yaml
panel_iframe:
  claude_chat:
    title: "Claude Chat"
    icon: mdi:message-processing
    url: "http://<your-ha-ip>:3001"
    require_admin: true
```

Caveat: the iframe only loads if your browser allows it — if you reach HA over **https**
(e.g. Nabu Casa), it may block the **http** iframe (mixed content), and CloudCLI may
refuse to be framed. If so, just open `http://<your-ha-ip>:3001` directly (add it to your
phone's home screen for a one-tap app).

**Notes**
- Requires **Node 22+** (the add-on base provides it); otherwise the chat UI is skipped
  with a warning and the terminal is unaffected.
- Logs: `/homeassistant/.claudecode/cloudcli/cloudcli.log`.
- Opt-in and additive — with `enable_chat_ui: false` (default) nothing changes.

## Always-Latest Claude Code

Claude Code is **not** baked into the add-on image. On first boot it is installed into
persistent storage at `/homeassistant/.claudecode/npm-global`, and on every start it is
refreshed to the latest published version (while `auto_update_claude` is on). Because it
lives on your HA config volume, the install survives add-on restarts, rebuilds, and
reinstalls — and its version is fully decoupled from the add-on version.

Update manually at any time from the terminal:

```bash
claude-update          # installs @anthropic-ai/claude-code@latest
```

To pin a specific version, install it the same way:

```bash
npm install -g @anthropic-ai/claude-code@<version>
```

## Automatic Startup

With `auto_start_claude` enabled (default), opening the terminal launches Claude
automatically — there is nothing to type after a reboot. When `claude_skip_permissions`
is enabled (default) it starts as `IS_SANDBOX=1 claude --dangerously-skip-permissions`,
so Claude can read/change files and call services without asking each time.

- Exit Claude (`/exit`, or Ctrl+C twice) to drop to a normal shell — the session stays
  alive; type `claude-launch` to relaunch Claude.
- Set `auto_start_claude: false` to get a plain shell on start instead.
- Set `claude_skip_permissions: false` to have Claude prompt before acting.

## File Locations

| Path | Description | Access |
|------|-------------|--------|
| `/homeassistant` | HA configuration directory | read-write |
| `/share` | Shared folder | read-write |
| `/media` | Media folder | read-write |
| `/ssl` | SSL certificates | read-only |
| `/backup` | Backups | read-only |

## Session Persistence & Mobile Use

When `session_persistence` is enabled (default), the add-on runs your terminal inside
tmux. **This is what lets Claude keep running in the background:** you can close the
browser (or your phone can drop the connection) and, when you reopen the terminal,
Claude is still there exactly where it left off. The web terminal (ttyd) starts a fresh
process for each connection and kills it when the tab closes, so **without tmux a running
Claude session would end** the moment you disconnect — that's why tmux is required for
background persistence.

tmux is configured to stay **out of your way**, especially on mobile:

- Claude launches automatically and fills the screen, so you almost never touch tmux key combos.
- Mouse capture is **off** and the status bar is hidden, so **native scrolling and
  long-press copy/paste work normally** on phones and tablets.
- The session **auto-attaches** on reconnect — there's nothing to type to "get back in".

If you do drop to the shell and want tmux controls: `Ctrl+b d` detaches, `Ctrl+b [`
enters scroll mode (`q` to exit).

### Authenticating Claude Code (first launch)

On first launch Claude prints a login URL:

1. **Tap or click the link** to open it in a new tab (zoom out with `Ctrl/Cmd -` if a long URL wraps and is hard to tap).
2. Complete authentication in the browser and copy the auth code.
3. Return to the terminal and paste it (long-press → **Paste** on mobile, or `Ctrl+Shift+V` / `Shift+Insert` on desktop), then press Enter.

### tmux vs. plain terminal

**`session_persistence: true` (default)**
- ✅ Claude keeps running in the background across disconnects/refreshes
- ✅ Reopen the terminal and resume exactly where you left off
- ✅ Native mobile scrolling and copy/paste (mouse capture off)
- ✅ 50,000-line scrollback

**`session_persistence: false`**
- ✅ Plain single-process terminal, simplest possible behavior
- ❌ Closing the tab or an add-on restart ends a running Claude session

## Security

> ⚠️ **This add-on runs with maximum privilege by default.** It is configured for full,
> unconfined root control of the host, and Claude is launched with permission checks
> bypassed. Only run it on a system you trust, and read this section first.

### Elevated access (enabled by default)
- `full_access`, `privileged` capabilities, `host_pid`, `host_dbus`, and the Docker
  socket (`docker_api`) are all enabled — the container can see and control the host.
- **AppArmor confinement is disabled** (`apparmor: false`), so root inside the container
  is not restricted to the mapped directories.
- `claude_skip_permissions: true` runs `claude --dangerously-skip-permissions`, so Claude
  reads/modifies files and calls services **without asking**.

### Dialing it back
If you don't need all of that, edit the add-on configuration (or `config.yaml`):
- Set `claude_skip_permissions: false` so Claude prompts before acting.
- Remove `privileged`, `host_pid`, `host_dbus`, and set `apparmor: true` to restore
  confinement (the bundled `apparmor.txt` profile is applied again).
- Narrow the `map:` list to only the directories you actually need.

### Authentication
- **No API keys in add-on config**: Claude Code handles authentication itself
- Credentials are stored securely in Claude Code's own directory (`~/.claude/`)
- This is more secure than storing keys in Home Assistant's configuration
- The Supervisor token is automatically managed and not exposed in the config

## Troubleshooting

### Authentication issues

Claude Code manages its own authentication. If you have issues:
1. Type `claude` to start the authentication flow
2. Follow the prompts to log in or enter your API key
3. Credentials are saved automatically for future sessions

**Can't copy the URL or paste the auth code?** The terminal uses tmux, which changes how copy/paste works. See [Copy and Paste in tmux](#copy-and-paste-in-tmux) for instructions.

### hass-mcp not working

1. Verify `enable_mcp` is true in configuration
2. Check add-on logs for connection errors
3. Restart the add-on after configuration changes

### Terminal not loading

1. Check that the add-on is running (green indicator)
2. Try refreshing the page
3. Check browser console for errors
4. Review add-on logs for ttyd errors

### Session not persisting

1. Ensure `session_persistence` is set to true
2. The session is named "claude" - it will auto-attach on reconnect

### Configuration changes not applying

After changing configuration:
1. Save the configuration
2. Restart the add-on completely

## Support

- [GitHub Issues](https://github.com/robsonfelix/robsonfelix-hass-addons/issues)
- [Home Assistant Community](https://community.home-assistant.io/)
