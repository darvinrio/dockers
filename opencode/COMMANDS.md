# COMMANDS.md — opencode sandbox quick reference

Dockerized dev sandbox: Alpine + git + opencode CLI + omp (oh-my-pi) CLI + bun +
uv + agent-browser (headless Chromium) + dropbear SSH (key-only, port 2222).
Container user `sudeep`, work volume at `/workspace`, GitHub PAT at `/run/secrets/github_pat`.
Chat history for both agents lives in named volumes, so it survives rebuilds.

## Build & run

```sh
docker compose build --pull --no-cache   # build image (installs latest opencode, omp, bun, uv, agent-browser)
docker compose up -d                     # start persistent container (SSH daemon host)
docker compose down                      # stop & remove container (volumes survive)
```

Components are installed by the Dockerfile at build time. A plain
`docker compose build` reuses cached layers and therefore keeps whatever was
current when those layers were first built — use `--no-cache` when you actually
want newer opencode/omp/bun/uv/agent-browser. `--pull` alone does nothing here
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
| `opencode_omp_data` | `~/.omp` | omp session JSONL + blobs, `agent/config.yml`, provider credentials, logs |

Still ephemeral (rebuilt from the image every time): `~/.config/opencode`
(including `opencode.global.json`), `~/.opencode/bin`, `~/.local/bin`,
`~/.cache/opencode`. Edit config in this repo and rebuild.

Back up each history volume:

```sh
docker run --rm -v opencode_opencode_data:/data -v "$PWD:/backup" alpine:3.24 \
  tar czf /backup/opencode-data-$(date +%F).tgz -C /data .
docker run --rm -v opencode_opencode_state:/data -v "$PWD:/backup" alpine:3.24 \
  tar czf /backup/opencode-state-$(date +%F).tgz -C /data .
docker run --rm -v opencode_omp_data:/data -v "$PWD:/backup" alpine:3.24 \
  tar czf /backup/omp-data-$(date +%F).tgz -C /data .
```

Reset one agent without touching the other:

```sh
docker compose down
docker volume rm opencode_opencode_data opencode_opencode_state   # or opencode_omp_data
docker compose up -d
```

Notes:

- opencode V2 stores credentials in `opencode.db`, so persisting history also
  persists logins. omp keeps credentials under `~/.omp`. Both volumes are
  readable by the agent — treat their contents as sensitive (transcripts can
  contain code, tokens and PII).
- Named volumes, not host bind mounts: SQLite locking/fsync against VirtioFS is
  slow and error-prone.
- Docker initializes a new volume from the image path on first use; the
  Dockerfile pre-creates the three paths owned by `sudeep` so the volume root is
  writable.
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

## omp CLI (inside container)

omp (oh-my-pi) is installed alongside opencode from its official installer
(`https://omp.sh/install`, prebuilt musl binary, sha256-verified). It is the
second agent in this sandbox and shares `/workspace` and the GitHub PAT.

```sh
omp                      # TUI in a project dir
omp --continue           # resume the most recent session for this directory
omp --resume             # pick a session
omp --version
omp update               # self-update (container layer only; lost on recreate)
```

Sessions and config live in `~/.omp/agent/` (`sessions/`, `blobs/`,
`config.yml`, provider credentials) and are persisted by the `omp_data` volume.
Log in to providers with `/login` inside the TUI. Shell completions:
`eval "$(omp completions bash)"`.

The binary is unpinned: `docker compose build --no-cache` fetches whatever
release is current. `omp update` upgrades only the running container — rebuild
the image if you want it permanent.

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
  connected — a CLI display quirk seen since 2.0.26. Use `/api/mcp` (above) or
  the log line `mcp connected server=agent-browser tools=29` to confirm.

To change the server config, edit `opencode.global.json` in this repo and
rebuild. A project can override the `agent-browser` entry by declaring the same
server name in its own `opencode.json`.

## bun & uv (inside container)

```sh
bun --version && uv --version     # both symlinked into /usr/local/bin
```

Versions are unpinned — each build installs the latest opencode, omp, bun, uv
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
uv 0.13.4) or if the opencode/omp binaries still contain a removed root, so a
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
