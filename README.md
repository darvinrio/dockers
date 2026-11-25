# Repo of Dockers

- `hermes.yaml` — Hermes agent
- `opencode/docker-compose.yaml` — opencode + pi sandbox

## Commands

```sh
docker compose -f hermes.yaml up -d
docker compose -f opencode/docker-compose.yaml up -d
```

The opencode sandbox keeps both agents' chat history in named volumes. See
`opencode/COMMANDS.md` for shells, SSH, persistence, backup and update commands,
and `opencode/SANDBOXING.md` for the threat model.
