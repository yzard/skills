# Build and runtime paths

Use when changing playground, artifact, or Docker locations; repository contracts override
these defaults.

- `playground/config.{toml,yaml,json}` uses the service's real parser format. Keep one clear
  config per service at the playground root; `data/`, `upload/`, and `output/` hold runtime
  data, uploaded inputs, and generated output. Track sample configuration; ignore generated data.
- Source code receives paths through typed configuration, not hardcoded playground locations.
- Root `run_playground.sh` resolves its own location and takes no subcommands. For a Docker
  playground, it calls `build_docker.sh`, prepares runtime directories, then runs `docker/docker-compose.yaml` directly
  in the foreground with continuous logs; Ctrl+C stops the containers and exits without
  deleting volumes. See `docker-build` for the full startup contract.
- Compilation/intermediate state belongs in root `build/<component>/`; final runnable or
  served products belong in `dist/<component>/`. Built asset settings point to deliverables.
  Ignore root build/dist trees. Clean only owned outputs needed to avoid stale artifacts;
  do not erase shared caches or another process's workspace on every run.
- Host fixtures/tools follow the repository's temporary-path and cleanup rules. Runtime
  data is not disposable merely because its directory is called playground.
- Docker's build context stays the git root. With a Dockerfile in `docker/`, use the matching
  `Dockerfile.dockerignore` beside it; a plain `docker/.dockerignore` does not cover root context.
- Compose in `docker/` uses `context: ..`, `dockerfile: docker/Dockerfile`, and relative bind
  paths starting from `docker/`. A root Compose file has different relative paths.
- Update consumers and CI/docs when changing paths. Preserve mandatory platform build scripts;
  do not replace them with ad hoc compiler or container calls.
