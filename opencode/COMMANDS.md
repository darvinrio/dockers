# COMMANDS.md — opencode sandbox quick reference

Dockerized dev sandbox: Alpine + git + opencode CLI + pi CLI + Node.js + bun +
uv + agent-browser (headless Chromium) + dropbear SSH (key-only, port 2222).
Container user `sudeep`, work volume at `/workspace`, GitHub PAT at `/run/secrets/github_pat`.
Chat history for both agents lives in named volumes, so it survives rebuilds.

## Build & run

```sh
docker compose build --pull --no-cache   # build image (installs latest opencode, pi, bun, uv, agent-browser)
docker compose up -d                     # start persistent container (SSH daemon host)
docker compose down                      # stop & remove container (volumes survive)
```

Components are installed by the Dockerfile at build time. A plain
`docker compose build` reuses cached layers and therefore keeps whatever was
current when those layers were first built — use `--no-cache` when you actually
want newer opencode/pi/bun/uv/agent-browser. `--pull` alone does nothing here
because the base image is pinned by digest.

## Shells inside the container

```sh
docker compose run --rm sudeep-agent               # ephemeral interactive bash (opencode TUI works)
docker compose exec sudeep-agent bash              # extra shell in the running (up -d) container
docker compose run --rm sudeep-agent opencode --version
```

## Persistent chat history

Both agents keep their state in named volumes, so sessions, snapshots and saved
logins survive `docker compose down`, `up --force-recreate` and image rebuilds
(but not `down -v` / `docker volume rm`).

| Volume | Container path | Holds |
|--------|----------------|-------|
| `opencode_opencode_data` | `~/.local/share/opencode` | `opencode.db` (sessions/messages/credentials), `storage/`, `snapshot/`, `tool-output/`, `shell/`, `log/` |
| `opencode_opencode_state` | `~/.local/state/opencode` | prompt/up-arrow history, last model, TUI tabs, frecency, lock/heartbeat dirs |
| `opencode_pi_data` | `~/.pi` | pi agent dir: session JSONL, `settings.json`, `mcp.json`, `auth.json` (provider credentials), package installs |

Still ephemeral (rebuilt from the image every time): `~/.config/opencode`
(including `opencode.global.json`), `~/.opencode/bin`, `~/.local/bin`,
`~/.cache/opencode`. Edit config in this repo and rebuild.

Back up each history volume:

```sh
docker run --rm -v opencode_opencode_data:/data -v "$PWD:/backup" alpine:3.24 \
  tar czf /backup/opencode-data-$(date +%F).tgz -C /data .
docker run --rm -v opencode_opencode_state:/data -v "$PWD:/backup" alpine:3.24 \
  tar czf /backup/opencode-state-$(date +%F).tgz -C /data .
docker run --rm -v opencode_pi_data:/data -v "$PWD:/backup" alpine:3.24 \
  tar czf /backup/pi-data-$(date +%F).tgz -C /data .
```

Reset one agent without touching the other:

```sh
docker compose down
docker volume rm opencode_opencode_data opencode_opencode_state   # or opencode_pi_data
docker compose up -d
```

Notes:

- opencode V2 stores credentials in `opencode.db`, so persisting history also
  persists logins. pi keeps credentials in `~/.pi/agent/auth.json`. Both volumes
  are readable by the agent — treat their contents as sensitive (transcripts can
  contain code, tokens and PII).
- Named volumes, not host bind mounts: SQLite locking/fsync against VirtioFS is
  slow and error-prone.
- Docker initializes a new volume from the image path on first use; the
  Dockerfile pre-creates the three paths owned by `sudeep` so the volume root is
  writable.
- pi itself is **not** on a volume: the release lives in `/opt/pi/agent` in the
  image layer, so `docker compose build --no-cache` actually upgrades it. The
  volume only holds pi's config/auth/sessions.
- Upgrading from the old omp image: the `opencode_omp_data` volume is left
  orphaned. omp sessions are not readable by pi, so just
  `docker volume rm opencode_omp_data`.
- `docker compose down -v` wipes the `work` volume too — do not use it casually.

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
opencode serve           # foreground HTTP server (alternative to the shared service)
```

The sandbox also starts the shared background service on port `4096` for the
web UI and API — see below.

### Two install channels — don't mix them

| URL | Line | Notes |
|-----|------|-------|
| `https://opencode.ai/v2/install` | V2 | What this image uses |
| `https://opencode.ai/install` | V1 | **Downgrade trap** — resolves to 1.x |

The bare `opencode.ai/install` URL is the *V1* channel. Point the Dockerfile at
it by accident and the next `docker compose build` silently replaces V2 with
V1. If `opencode --version` ever reports `1.x`, check the install URL in
`Dockerfile`.

## pi CLI (inside container)

pi is installed alongside opencode from its official installer
(`https://pi.dev/install.sh`). It is a Node.js program (this image ships Alpine's
`nodejs` + `npm`) and shares `/workspace` and the GitHub PAT.

```sh
pi                       # TUI in a project dir
pi -c                    # continue the most recent session for this directory
pi -r                    # pick a session to resume
pi --version
pi -p "summarize this repo"   # print mode, one-off, no TUI
pi mcp list              # check MCP servers (agent-browser)
pi update                # WILL FAIL: the release is root-owned in /opt/pi — rebuild instead
```

Sessions, `settings.json`, `mcp.json` and `auth.json` live in `~/.pi/agent/` and
are persisted by the `pi_data` volume. Log in to providers with `/login` inside
the TUI; `/mcp` inspects or reloads MCP servers; `/reload` re-reads config.

The **release** itself is under `/opt/pi/agent` (image layer, not the volume), on
purpose: a volume at `~/.pi` would mask a freshly baked version and pin pi to
whatever it was the first day. Consequence: upgrades are image rebuilds, and
`pi update` cannot write there.

The binary is unpinned: `docker compose build --no-cache` fetches whatever
release is current.

## agent-browser (headless browsing, inside container)

Native Rust CLI driving Alpine's `chromium` over CDP. Available two ways:

- **as MCP tools** — `agent-browser` is registered in the global opencode config
  (`opencode.global.json` → `~/.config/opencode/opencode.json`) and in pi's
  user-level MCP config (`pi.mcp.json` → `~/.pi/agent/mcp.json`), so every
  project gets browser tools (snapshot / click / fill / screenshot / …)
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
| Binary | `agent-browser-linux-musl-{arm64,x64}` from GitHub releases | The npm package is a `#!/usr/bin/env node` shim; the native binary has no runtime deps and is what the MCP config points at (node now exists for pi, but the native binary is still the right choice) |
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
pi mcp list
# agent-browser: connected, N tools

opencode api get /api/mcp
# {"location":{"directory":"..."},"data":[{"name":"agent-browser","status":{"status":"connected"}}]}
```

For pi, `pi mcp list` exits 1 if an enabled server fails to connect, so it works
as a script check. Two opencode gotchas:

- The **first** call after a fresh `up -d` races the MCP handshake and can
  return an empty list while the server is still spawning. Re-run it.
- `opencode mcp list` reports "No MCP servers configured" even when the server is
  connected — a CLI display quirk seen since 2.0.26. Use `/api/mcp` (above) or
  the log line `mcp connected server=agent-browser tools=29` to confirm.

To change the server config, edit `opencode.global.json` / `pi.mcp.json` in this
repo and rebuild. A project can override the `agent-browser` entry by declaring
the same server name in its own `opencode.json` (opencode) or `.pi/mcp.json`
(pi, only read in trusted projects). Inside pi, `/mcp` can also add, disable or
reload servers without a rebuild; `pi mcp add agent-browser -- agent-browser mcp`
does the same from the shell.

## bun & uv (inside container)

```sh
bun --version && uv --version     # both symlinked into /usr/local/bin
```

Versions are unpinned — each build installs the latest opencode, pi, bun, uv
and agent-browser. Those installer layers are cached, so a plain
`docker compose build` keeps the versions from the first build; use
`docker compose build --no-cache` to actually pick up new upstream releases.

## Updating the agents and components

```sh
docker compose build --no-cache && docker compose up -d --force-recreate
```

The base image pin does not move by itself: the digest in `Dockerfile` needs a
deliberate bump when Alpine cuts a new image (check with
`docker buildx imagetools inspect alpine:3.24`). The build also fails if a
bundled CA store is older than the eMudhra fix (opencode 2.2.0 / Bun 1.4.4 /
uv 0.13.4 / Node's own `tls.rootCertificates`), if the opencode binary still
contains a removed root, or if pi is older than 1.0 (no built-in MCP), so a
failed build means the upstream channel rolled back, not that your machine is
broken.

Verify the trust store from inside the container. A nonzero exit alone is not
a pass — DNS and network failures also exit nonzero — so check the actual
error: `curl` must exit 60 (`curl: (60) ... SSL certificate problem`) and
`bun` must reject with a certificate error, not a connection error. The host
is the community incident tracker, which serves the compromised chains:

```sh
docker compose run --rm sudeep-agent \
  sh -c 'curl -sS https://dv-g3b-emudhra.we-have-a-problem.com/; echo "curl exit=$?"'
docker compose run --rm sudeep-agent \
  bun -e 'await fetch("https://dv-g3b-emudhra.we-have-a-problem.com/")'
```

`curl` uses the Alpine bundle; `bun` exercises the embedded store that opencode
is also compiled with. A 200 from either one means the fix did not land.

## SSH keys

```sh
# rotate which key can log in: replace the pub key, then recreate (secrets bind at create)
cp ~/.ssh/new_key.pub secrets/authorized_keys/opencode_container.pub
docker compose up -d --force-recreate
```

- Private key lives on the Mac at `~/.ssh/opencode_container` (never in the repo/image)
- dropbear host key persists in the `ssh_keys` volume — rebuilds do NOT rotate it
- Key-only auth (`-s`); port published on `127.0.0.1` only

## Web UI & HTTP API (from the Mac)

The container starts opencode's shared background service on `0.0.0.0:4096` and
compose publishes it on `127.0.0.1:4096` only, so it is reachable from the Mac
and not from the LAN. It is the same server the TUI uses, so web and terminal
sessions share the database.

```sh
open http://127.0.0.1:4096                    # web UI (asks for a pairing link/password)
docker compose exec sudeep-agent opencode pair   # one-time login link (5 min TTL)
docker compose exec sudeep-agent opencode service status
```

For scripts, the API takes Basic auth with username `opencode` and the server
password:

```sh
curl -u "opencode:$PASSWORD" http://127.0.0.1:4096/api/info
```

No password is configured in the repo, so opencode generates one per container
and stores it in `~/.config/opencode/service.json` (0600). That path is not on a
volume, so a recreate — `up -d --force-recreate`, a rebuild, even `down`/`up` —
mints a new password, which signs out existing browser sessions:

```sh
docker compose exec sudeep-agent sed -n 's/.*"password": *"\([^"]*\)".*/\1/p' \
  ~/.config/opencode/service.json
```

For a password you choose, export `OPENCODE_PASSWORD` before creating the
container (`docker-compose.yaml` passes it through; `sandbox-entry` writes it
into the service config):

```sh
OPENCODE_PASSWORD='a-long-secret' docker compose -f opencode/docker-compose.yaml up -d
```

`docker compose exec ... opencode service set password '...' && opencode service restart`
works too, but only until the next recreate.

To turn the listener off, set `OPENCODE_SERVE=0` in `docker-compose.yaml` and
recreate. If you only need occasional access, drop the port mapping and forward
over the existing SSH channel instead, which needs no container config at all:

```sh
ssh -N -L 4096:127.0.0.1:4096 -p 2222 sudeep@localhost
```

Requiring `hostname 0.0.0.0` is not optional for the published port: Docker
forwards to the container's `eth0`, so a service left on `127.0.0.1` inside the
container is unreachable from the host no matter what `ports:` says.

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
| 4096 | opencode web UI / HTTP API |
