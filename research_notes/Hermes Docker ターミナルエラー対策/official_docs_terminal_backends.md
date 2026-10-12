# Hermes Agent terminal tool / terminal backends inside Docker (official docs + source)

Verification legend: [SRC] = seen in raw source on GitHub main (via WebFetch summary); [DOC] = official docs page only; [INF] = inference. Note: WebFetch returns model-summarised content, not byte-exact files; exact strings below are as reported by the fetch tool and should be re-grepped in a clone before being relied on.

## 1. Official docker run / docker-compose recommendation

### Takeaway
Official docs: mount `~/.hermes` to `/opt/data`, run `gateway run`, optionally `-e HERMES_DASHBOARD=1 -p 9119:9119`. Image runs as root at start, then drops to user `hermes` (UID 10000) via s6-overlay.

### Cited Findings
- First-time setup: `docker run -it --rm -v ~/.hermes:/opt/data nousresearch/hermes-agent setup` — [Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker) [DOC]
- Gateway: `docker run -d --name hermes --restart unless-stopped -v ~/.hermes:/opt/data -p 8642:8642 nousresearch/hermes-agent gateway run`; dashboard: add `-p 9119:9119 -e HERMES_DASHBOARD=1` — [Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker) [DOC]
- API server beyond loopback needs `-e API_SERVER_ENABLED=true -e API_SERVER_HOST=0.0.0.0 -e API_SERVER_KEY=... ` — same page [DOC]
- Compose example in docs: image `nousresearch/hermes-agent:latest`, `command: gateway run`, ports 8642 and 9119, volume `~/.hermes:/opt/data`, env `HERMES_DASHBOARD=1`, resource limits 4G/2 cpus — same page [DOC]
- Repo docker-compose.yml: services `gateway` (host networking, `~/.hermes:/opt/data`, `gateway run`) and `dashboard` (bound 127.0.0.1, `--no-open`); `HERMES_UID`/`HERMES_GID` default 10000; keep `/init` first if overriding entrypoint — [raw docker-compose.yml](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/docker-compose.yml) [SRC]
- Dockerfile: base `debian:13.4`; `HERMES_HOME=/opt/data`, `HERMES_WRITE_SAFE_ROOT=/opt/data`; `VOLUME /opt/data`; user `hermes` UID 10000 home `/opt/data`; final `USER root` so the s6 stage2 hook can fix ownership, services drop to `hermes` with `s6-setuidgid`; `ENTRYPOINT ["/opt/hermes/docker/entrypoint-dispatch.sh"]` — [Dockerfile](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/Dockerfile) [SRC]
- Docs env table: `HERMES_DASHBOARD`, `HERMES_DASHBOARD_HOST/_PORT` (0.0.0.0/9119), `HERMES_DASHBOARD_BASIC_AUTH_USERNAME/_PASSWORD`, `HERMES_UID/HERMES_GID` (or `PUID/PGID`), `HERMES_HOME_MODE`, `HERMES_SKIP_CONFIG_MIGRATION`, `HERMES_ALLOW_ROOT_GATEWAY=1`, `HERMES_GATEWAY_NO_SUPERVISE=1`, `HERMES_DOCKER_EXEC_AS_ROOT=1` — [Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker) [DOC]
- Volume warning: on virtiofs/9p (Docker Desktop macOS/Windows, OrbStack) SQLite WAL can corrupt; use named volume or `database.journal_mode: delete`. Browser tools need `--shm-size=1g` — same page [DOC]

### Gaps
- Docs page summary did not show a dedicated "terminal backend inside container" recommendation beyond a pointer to Configuration.

## 2. Which backend in Docker; does docker backend need docker CLI + socket?

### Takeaway
Default backend is `local` (commands run inside the Hermes container itself) — this needs nothing extra. `docker` backend = sibling-container sandbox; it needs the docker CLI (present in the image) AND a reachable daemon, i.e. a mounted `/var/run/docker.sock`; neither the official docs page nor compose mention mounting it.

### Cited Findings
- `terminal.backend` accepts `local | docker | ssh | modal | daytona | vercel_sandbox | singularity` — [Configuration docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) [DOC]
- Source default: `env_type = _tenv("TERMINAL_ENV", "local")`; `_ensure_terminal_env_bridged()` fills `TERMINAL_*` from config.yaml if no launcher did; no container auto-detection (`/.dockerenv` not referenced) — [tools/terminal_tool.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/tools/terminal_tool.py) [SRC]
- Image installs `docker-cli` (client only, no daemon) — Dockerfile [SRC]; docs also say image includes the Docker CLI — [Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker) [DOC]
- Docker-backend check: `find_docker()` then runs `<docker> version` with 5s timeout; nonzero exit = daemon unavailable — [tools/terminal_tool_backends.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/tools/terminal_tool_backends.py) [SRC]
- `find_docker` order: `HERMES_DOCKER_BINARY`, `docker` on PATH, `podman` on PATH, macOS Docker Desktop paths — [tools/environments/docker.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/tools/environments/docker.py) [SRC]
- No docker.sock-specific logic in the Python; connectivity left to CLI. But stage2-hook.sh: if a socket exists at `/var/run/docker.sock` or `/run/docker.sock`, adds `hermes` to a group with the socket's GID (creates `hostdocker` if none) — [docker/stage2-hook.sh](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/docker/stage2-hook.sh) [SRC]. This strongly implies mounting the socket is the intended way.
- Docs: Docker "terminal backend" is separate from running Hermes in Docker; keys `docker_image, docker_volumes, docker_forward_env, docker_env, docker_run_as_host_user, docker_extra_args, docker_persist_across_processes, docker_orphan_reaper` — [Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker) [DOC]
- Default sandbox image: `nousresearch/hermes-sandbox:desktop` (`TERMINAL_DOCKER_IMAGE`; config page comment inconsistent) — [Env var reference](https://hermes-agent.nousresearch.com/docs/reference/environment-variables) [DOC]

### Inferences
- With docker backend in a Hermes container, bind-mount host paths in `docker_volumes` are resolved by the HOST daemon, so paths must be host paths, not container paths. [INF, not verified]
- Whether the official image's default config.yaml sets `terminal.backend: docker` or `local` was not verified; code default is local. If the user's config.yaml/.env in /opt/data says `docker` (e.g. from `hermes setup`) without socket, terminal breaks.

### Gaps
- Exact seeded config.yaml / cli-config.yaml.example value for terminal.backend not retrieved.
- Official statement on socket mounting not found.

## 3. Error messages and causes

### Takeaway
Failures come from the requirements check (tool hidden/unavailable) or from env creation at call time.

### Cited Findings
- Docker not installed: `"Docker is not installed — no docker executable in PATH or the usual install locations"` — terminal_tool_backends.py [SRC]
- Daemon unreachable / nonzero `version`: `"Docker is installed but the Docker daemon is not running"` followed by `(<executable> version exited with code N)`. Typical cause in container: no socket mounted, or socket permission denied for `hermes` (CLI exits nonzero) — [SRC]
- `_ensure_docker_available()` raises `EnvironmentConnectionError` with message variants: not found in PATH/known locations; binary cannot be executed (FileNotFoundError); daemon not responding (timeout, hint: start daemon); `version` command failed (hint: start Docker or add user to `docker` group) — docker.py [SRC] (exact wording not captured)
- `"Terminal tool disabled: environment creation failed ({e})"` (ImportError during env creation) — terminal_tool.py [SRC]
- `"Terminal backend degraded: {e.reason}"` with `retry_hint`; fatal envelope if `TERMINAL_DEGRADED_MODE` = `fail` — [SRC]
- `"Terminal requirements check failed: %s"` / reason `"the requirements check failed: {e}"` — [SRC]
- SSH: `"the SSH host and user are not configured (TERMINAL_SSH_HOST / TERMINAL_SSH_USER); run `hermes setup terminal` to enter them or pick the 'local' backend"` — [SRC]
- Singularity: `"the 'singularity' backend's executable was not found"`; `"Apptainer/Singularity is installed but `--version` failed, so the install looks broken"` — [SRC]
- Modal: `"modal is required for direct modal terminal backend. Run hermes setup terminal and select Modal."`; Daytona: `"the 'daytona' backend is missing its API key or SDK"`; Vercel: `"vercel is required for the Vercel Sandbox terminal backend. ..."` — [SRC]
- Unknown backend: `"Unknown TERMINAL_ENV '<name>'. Use one of: ..."` — [SRC]
- The literal phrase "unavailable" appears not to be a user-facing message in terminal_tool.py; the "terminal unavailable" report is the model's paraphrase of `terminal_backend_unavailable_reason` output. [INF]

### Gaps
- Exact user-facing string composition around `terminal_backend_unavailable_reason` not seen.

## 4. `docker_run_as_host_user` as config key and env var

### Takeaway
Yes, it is a `terminal:` key (bool, default false) and has env form `TERMINAL_DOCKER_RUN_AS_HOST_USER`. A bare env var named `docker_run_as_host_user` is not a documented form.

### Cited Findings
- `docker_run_as_host_user` default `false`; when true adds `--user $(id -u):$(id -g)` to `docker run`; container then cannot write root-owned paths such as `/root/.npm` — [Configuration docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) [DOC]
- Source: `TERMINAL_DOCKER_RUN_AS_HOST_USER` -> `"docker_run_as_host_user"` via `_tenv_bool(..., "false")`; uid:gid via `os.getuid/getgid`; warns and falls back to image user if unavailable — terminal_tool.py, docker.py [SRC]
- Docs: "Every key under `terminal:` has an env-var override of the form `TERMINAL_<KEY_UPPERCASE>`" -> `TERMINAL_BACKEND` is inferred, though `TERMINAL_ENV` is the documented/actual name for backend — [Configuration docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration), [Env vars](https://hermes-agent.nousresearch.com/docs/reference/environment-variables) [DOC]
- List-valued env forms are JSON arrays (`TERMINAL_DOCKER_FORWARD_ENV`, `_EXTRA_ARGS`, `_VOLUMES` per config page; env reference says comma-separated for volumes — conflict) — [Config](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) vs [Env vars](https://hermes-agent.nousresearch.com/docs/reference/environment-variables)
- Bool values parsed by `_tenv_bool` (accepted truthy strings not captured).

### Gaps
- `TERMINAL_DOCKER_RUN_AS_HOST_USER` not in the visible part of the env reference page (truncated at 100k chars).

## 5. HERMES_DOCKER_RUN_AS_ROOT and /opt/data permissions

### Takeaway
I found NO evidence that `HERMES_DOCKER_RUN_AS_ROOT` exists. The documented variable is `HERMES_DOCKER_EXEC_AS_ROOT=1` (per-`docker exec` call root). Permissions are handled by `HERMES_UID/GID` (or `PUID/PGID`) remapping plus chown in stage2-hook.

### Cited Findings
- Docs page does not document `HERMES_DOCKER_RUN_AS_ROOT`; documents `HERMES_DOCKER_EXEC_AS_ROOT=1` so `docker exec` keeps root (default exec drops to `hermes`) — [Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker) [DOC]; the only `HERMES_DOCKER_*` var in docker.py is `HERMES_DOCKER_BINARY` [SRC]; stage2-hook.sh had no mention [SRC]
- stage2-hook: remaps `hermes` to `HERMES_UID/GID` (1-65534; `PUID/PGID` aliases, HERMES_* win); chowns HERMES_HOME if owner mismatches, subdirs recursively if non-hermes files, state files each boot; `.env` mode 600, `config.yaml` 640; skips symlinks; `/opt/hermes` stays root-owned — [stage2-hook.sh](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/docker/stage2-hook.sh) [SRC]
- Troubleshooting: Permission denied -> set HERMES_UID/GID to host owner; data dir resets to 0700 -> `HERMES_HOME_MODE`; `docker exec` Permission denied on old images -> `/opt/hermes` 0700, pull newer — [Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker) [DOC]
- Third-party/issue reports: UnRAID permission denied when container started as non-root so the hook never ran ([#13731](https://github.com/NousResearch/hermes-agent/issues/13731)); root-run CLI leaves root-owned files breaking gateway ([#16480](https://github.com/NousResearch/hermes-agent/issues/16480)) — only seen via search snippets.
- `HERMES_ALLOW_ROOT_GATEWAY=1` allows gateway as root (risky) — docs [DOC]

### Inferences
- `hermes` gets socket access only because the hook adds it to the socket's GID group; if the container is started with `--user` / compose `user:` (bypassing the root hook), socket group setup and chown do not run, which would break both permissions and docker backend. [INF]

### Gaps
- Could not grep full repo for HERMES_DOCKER_RUN_AS_ROOT (GitHub code search API requires auth); recommend `git clone` + `grep -rn HERMES_DOCKER_RUN_AS_ROOT`. Not verified in cli-config.yaml.example, entrypoint.sh, main-wrapper.sh, cont-init.d, or sandbox-desktop.Dockerfile.
