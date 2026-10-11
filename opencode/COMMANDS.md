# COMMANDS.md — opencode sandbox quick reference

Dockerized dev sandbox: Alpine + git + opencode CLI + bun + uv + agent-browser
(headless Chromium) + dropbear SSH (key-only, port 2222).
Container user `sudeep`, work volume at `/workspace`, GitHub PAT at `/run/secrets/github_pat`.

## Build & run

```sh
docker compose build                     # build image (installs latest opencode, bun, uv, agent-browser)
docker compose up -d                     # start persistent container (SSH daemon host)
docker compose down                      # stop & remove container (volumes survive)
```

## Shells inside the container

```sh
docker compose run --rm sudeep-agent               # ephemeral interactive bash (opencode TUI works)
docker compose exec sudeep-agent bash              # extra shell in the running (up -d) container
docker compose run --rm sudeep-agent opencode --version
```

## SSH access (from the Mac)

```sh
ssh -p 2222 -i ~/.ssh/opencode_container sudeep@localhost
```

With a `~/.ssh/config` entry (see below) just:

```sh
ssh opencode
```

`~/.ssh/config`:

```
Host opencode
    HostName localhost
    Port 2222
    User sudeep
    IdentityFile ~/.ssh/opencode_container
    IdentitiesOnly yes
```

Passphrase: typed on each connect (or `ssh-add --apple-use-keychain ~/.ssh/opencode_container` once).

## Zed remote

1. `ctrl-cmd-shift-O` (Remote Projects) → Connect New Server
2. Enter: `ssh opencode`  (or `ssh sudeep@localhost -p 2222`)
3. Open `/workspace/<project>` — Zed installs its server into `~/.zed_server` in the container
4. Zed's terminal runs inside the container → `opencode` there

CLI shortcut: `zed ssh://opencode/workspace/<project>`

## opencode CLI (inside container)

```sh
opencode                 # TUI in a project dir
opencode --version
opencode serve           # HTTP server mode (API/SDK)
```

### Two install channels — don't mix them

| URL | Line | Notes |
|-----|------|-------|
| `https://opencode.ai/v2/install` | V2 | What this image uses |
| `https://opencode.ai/install` | V1 | **Downgrade trap** — resolves to 1.x |

The bare `opencode.ai/install` URL is the *V1* channel. Point the Dockerfile at
it by accident and the next `docker compose build` silently replaces V2 with
V1. If `opencode --version` ever reports `1.x`, check the install URL in
`Dockerfile`.

## agent-browser (headless browsing, inside container)

Native Rust CLI driving Alpine's `chromium` over CDP. Available two ways:

- **as MCP tools** — `agent-browser` is registered in the global opencode config
  (`opencode.global.json` → `~/.config/opencode/opencode.json`), so every project
  gets `agent_browser_*` tools (snapshot / click / fill / screenshot / …)
- **as a CLI** — for when you want it by hand

```sh
agent-browser --version
agent-browser open https://example.com      # navigate
agent-browser snapshot -i                   # accessibility tree + @refs (best for agents)
agent-browser click @e2
agent-browser screenshot /workspace/shot.png
agent-browser close
```

Typical loop: `open` → `snapshot -i` → act on `@refs` → re-snapshot.

### How it is wired

| Piece | Value | Why |
|-------|-------|-----|
| Binary | `agent-browser-linux-musl-{arm64,x64}` from GitHub releases | The npm package is a `#!/usr/bin/env node` shim; this image ships no node |
| Browser | Alpine's `chromium` package | `agent-browser install` fetches Chrome for Testing, which is **glibc-only** and cannot run on musl |
| `AGENT_BROWSER_EXECUTABLE_PATH` | `/usr/bin/chromium-browser` | Points agent-browser at the distro chromium |
| `AGENT_BROWSER_ARGS` | `--no-sandbox,--disable-dev-shm-usage,--disable-gpu` | See below |

`--no-sandbox` is required, not a shortcut: chromium's own sandbox needs
namespace syscalls that `cap_drop: ALL` + `no-new-privileges` block, so without
it chromium dies at startup with `Failed to move to new namespace`. The
container boundary is already the isolation boundary that matters.

Override per-call with `agent-browser --args "..." <cmd>`, or drop
`AGENT_BROWSER_ARGS` entirely if you ever loosen the compose hardening.

### Fonts

Only DejaVu + Liberation are installed. Latin/Cyrillic/Greek/Arabic render
correctly; **CJK renders as tofu boxes** in screenshots. Text extraction
(`snapshot`, `get text`) is unaffected — it reads the DOM, not pixels. If you
need CJK screenshots, add a font package to the `apk add` list and rebuild.

### Check the MCP wiring

```sh
opencode api get /api/mcp
# {"location":{"directory":"..."},"data":[{"name":"agent-browser","status":{"status":"connected"}}]}
```

Two gotchas:

- The **first** call after a fresh `up -d` races the MCP handshake and can
  return an empty list while the server is still spawning. Re-run it.
- `opencode mcp list` reports "No MCP servers configured" even when the server is
  connected — a CLI display quirk on 2.0.26. Use `/api/mcp` (above) or the log
  line `mcp connected server=agent-browser tools=29` to confirm.

To change the server config, edit `opencode.global.json` in this repo and
rebuild. A project can override the `agent-browser` entry by declaring the same
server name in its own `opencode.json`.

## bun & uv (inside container)

```sh
bun --version && uv --version     # both symlinked into /usr/local/bin
```

Versions are unpinned — each build installs the latest opencode, bun, uv and
agent-browser. `docker compose build` pulls whatever is current at build time.

## Updating opencode

```sh
docker compose build && docker compose up -d --force-recreate
```

## SSH keys

```sh
# rotate which key can log in: replace the pub key, then recreate (secrets bind at create)
cp ~/.ssh/new_key.pub secrets/authorized_keys/opencode_container.pub
docker compose up -d --force-recreate
```

- Private key lives on the Mac at `~/.ssh/opencode_container` (never in the repo/image)
- dropbear host key persists in the `ssh_keys` volume — rebuilds do NOT rotate it
- Key-only auth (`-s`); port published on `127.0.0.1` only

## Troubleshooting

```sh
docker compose ps                                    # is it up? ports bound?
docker compose logs --tail=50 sudeep-agent           # sandbox-entry / sleep logs
docker compose restart sudeep-agent                  # restart dropbear keeper

# "REMOTE HOST IDENTIFICATION HAS CHANGED" → stale known_hosts entry
ssh-keygen -R "[localhost]:2222"

# auth debugging: restart dropbear with stderr logging, then reconnect and read log
docker compose exec sudeep-agent sh -c 'kill $(cat ~/.ssh/dropbear.pid); \
  dropbear -E -r ~/.ssh/dropbear_host_ed25519 -p 2222 -s -P ~/.ssh/dropbear.pid 2>/tmp/dropbear.log'
docker compose exec sudeep-agent cat /tmp/dropbear.log

# fingerprint comparison (server vs client key)
docker compose exec sudeep-agent ssh-keygen -lf ~/.ssh/authorized_keys
ssh-keygen -lf ~/.ssh/opencode_container.pub
```

## Ports (all localhost-only)

| Port | Use |
|------|-----|
| 2222 | dropbear SSH |
| 4321 | astro |
| 8080 | litestar |
