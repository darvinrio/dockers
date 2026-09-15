# COMMANDS.md — opencode sandbox quick reference

Dockerized dev sandbox: Alpine + git + opencode CLI + dropbear SSH (key-only, port 2222).
Container user `sudeep`, work volume at `/workspace`, GitHub PAT at `/run/secrets/github_pat`.

## Build & run

```sh
docker compose build                     # build image (installs latest opencode)
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

## Updating opencode

```sh
docker compose build && docker compose up -d --force-recreate
```

Pin a version instead: replace the `curl -fsSL https://opencode.ai/install | bash -s -- --no-modify-path`
line in the Dockerfile with `bash -s -- --no-modify-path -v <x.y.z>` and rebuild.

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
