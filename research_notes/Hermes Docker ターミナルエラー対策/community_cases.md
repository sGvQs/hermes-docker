# Hermes Agent in Docker: community cases of terminal tool failures

Note: all findings come from WebFetch summaries of GitHub issue pages (a small model summarized them). "Fixed version" is often not stated on the pages. Issue dates are 2026. Reddit/Qiita/Zenn searches returned nothing specific to this problem.

## Which GitHub issues relate to terminal/docker failures when Hermes runs in Docker?

### Takeaway
The dominant real-world failure is the "Docker-out-of-Docker" setup: Hermes in a container with `terminal.backend: docker` and the host `/var/run/docker.sock` mounted. It fails in three distinct ways: socket permission/group, bind-mount path translation, and health-check false negatives.

### Cited Findings
- #16703 (opened Apr 27, closed May 29, 2026; v0.11.0): Tool error "Tool execution failed: RuntimeError: Docker command is available but 'docker version' failed. Check your Docker installation." Inside container as `hermes`: "permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock". Cause: `group_add` applies only to PID 1; entrypoint's `exec gosu hermes` rebuilds groups from /etc/group, which lacks the host docker GID. On messaging platforms the file toolset was also dropped from the schema. — [#16703](https://github.com/NousResearch/hermes-agent/issues/16703)
- #34648 (opened May 29, closed Jun 1, 2026): v0.15.0/v0.15.1 regression, non-root `user: "10000:10000"` + `group_add` + docker.sock gives s6 boot loop: "s6-applyuidgid: fatal: unable to set supplementary group list: Operation not permitted". Introduced by PR #34407 (merged 2026-05-29); v0.14.0 works; closed via PR #34837. — [#34648](https://github.com/NousResearch/hermes-agent/issues/34648)
- #50798 (open, Jun 22, 2026; v0.17.0, docker-compose self-host): sandbox skill/cache bind mounts are empty; skill scripts fail "No such file or directory"; adding to `docker_volumes` gives "Duplicate mount point". Cause: bind sources built from in-container `/opt/data` but resolved by the host daemon. Closed PR #50874 (HERMES_HOST_DATA_DIR env) was rejected; no merged fix as of Oct 10, 2026. — [#50798](https://github.com/NousResearch/hermes-agent/issues/50798)
- #40368 (open, Jun 6, 2026; image v2026.6.5, Unraid): same path-translation problem; sandbox mounts point to host `/opt/data`. Partial PR #40404 open. — [#40368](https://github.com/NousResearch/hermes-agent/issues/40368)
- #72927 (Jul 27, closed Sep 5, 2026; v0.18.2, compose + socket-proxy `DOCKER_HOST=tcp://socket-proxy:2375`): `hermes doctor` shows "✗ docker daemon not running" falsely; fixed in PR #103671 (probe uses `docker version`, not `docker info`). — [#72927](https://github.com/NousResearch/hermes-agent/issues/72927)
- #63356 (open, Jul 12, 2026; WSL2): "docker version failed" message but real failure is `docker run -d` timing out after 120s. Windows/WSL2, not Mac. — [#63356](https://github.com/NousResearch/hermes-agent/issues/63356)
- #45792 (open, Jun 13, 2026; v0.16.0): in the container the agent looks for config at `/opt/hermes/.hermes/config.yaml` instead of `/opt/data`, writes fail; PR #45811 open. — [#45792](https://github.com/NousResearch/hermes-agent/issues/45792)
- Other Docker-backend issues listed by search (titles only, not read in depth): #39615 `docker_extra_args` unreadable (closed Jul 14), #132313 `--rm --network=none` falsely triggers recursive-delete approval (open Oct 3), #135899 `skill_view` returns host paths in the Docker backend (open Oct 10), #135732 image lacks agent-browser (open Oct 9), #4084 Podman request (closed Apr 15). — [search listing](https://github.com/NousResearch/hermes-agent/issues?q=docker+terminal)

### Inferences
- A compose file with docker.sock mounted + `terminal.backend: docker` will hit one of: permission denied (fixed), or host-path mismatch (unfixed).

### Gaps
- No Reddit/HN/Qiita/Zenn posts describing the exact "terminal not available" message were found.

## What resolutions/config/env/compose changes were given?

### Takeaway
Use the right UID/GID env vars, avoid overriding the entrypoint, use named volumes on Docker Desktop, and for DooD make the host and container data paths identical.

### Cited Findings
- Official Docker guide: runtime user `hermes` (UID 10000); set `HERMES_UID`/`HERMES_GID` (or PUID/PGID) to match bind-mount owner; `docker exec` drops to hermes (use `HERMES_DOCKER_EXEC_AS_ROOT=1` for root); old-image "Permission denied" on `docker exec` fix: `docker exec -u root hermes chmod 0755 /opt/hermes` or recreate with newer image. — [Docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- Same docs: overriding compose `entrypoint:` makes hermes PID 1 and orphans are not reaped; restore the default entrypoint or add `init: true`. Browser tools need `--shm-size=1g`. — [Docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- #16703 workarounds: add a group matching socket GID, entrypoint shim using `stat` on the socket (compose-only), upstream fix commit ec7736f "auto-join Docker socket group" (PR #34407). — [#16703](https://github.com/NousResearch/hermes-agent/issues/16703)
- #50798 workaround: `sudo mount --bind` of host dirs onto `/opt/data/...` paths (symlinks fail; `docker_volumes` gives duplicate mount). — [#50798](https://github.com/NousResearch/hermes-agent/issues/50798)
- #40368 workarounds: host symlink `/opt/data` -> real appdata (fragile), DinD, manual `/workspace` in `docker_volumes` (incomplete). — [#40368](https://github.com/NousResearch/hermes-agent/issues/40368)
- #72927 verification: `docker compose exec -T hermes docker version`, and `cat /etc/hostname` via the agent's terminal differing from container ID. — [#72927](https://github.com/NousResearch/hermes-agent/issues/72927)
- Official FAQ: Docker backend connection failures are due to a stopped daemon or missing user permissions; check `docker info`, `docker run hello-world`. — [FAQ](https://hermes-agent.nousresearch.com/docs/reference/faq)

### Inferences
- A simple way to avoid DooD problems is not to use `terminal.backend: docker` inside the container; use `local` (container is the sandbox) — inference, not stated by a source.

### Gaps
- No source states how a Mac user should obtain the docker.sock GID (Docker Desktop's socket semantics differ).

## Are there bugs fixed in specific versions (latest vs pinned tag)?

### Takeaway
Yes. Version matters: v0.15.0/v0.15.1 had a boot-loop regression; socket-group fix landed May 29, 2026 (PR #34407) but caused that regression; `hermes doctor` false negative fixed Sep 5, 2026.

### Cited Findings
- Broken v2026.5.28 (v0.15.0) and v2026.5.29 (v0.15.1); working v2026.5.16 (v0.14.0); closed via #34837 Jun 1, 2026. — [#34648](https://github.com/NousResearch/hermes-agent/issues/34648)
- Image tags: `latest`/`stable` (release-gated), `main` (dev), `X.Y.Z` (pin; digest for exact). A main push doesn't advance latest. — [Docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- Docs note the tini -> s6 entrypoint switch is breaking for wrappers relying on tini signal behavior; #35394 (v0.15.0-0.15.2 gateway restart loop, closed as duplicate) – root cause not established. — [Docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker); [search summary](https://github.com/NousResearch/hermes-agent/issues/35394)

### Inferences
- Pin to a known-good X.Y.Z tag and read release notes before upgrading.

### Gaps
- Exact release versions containing fixes for #16703, #34648, #72927 were not stated on the pages.

## macOS / Docker Desktop / Mac mini + Ollama + Telegram cases?

### Takeaway
No directly matching case found. Closest: Docker Desktop virtiofs WAL corruption warning, plus unrelated mac/Ollama items.

### Cited Findings
- Docs warn state.db (WAL) can silently corrupt on Docker Desktop (macOS/Windows) and OrbStack bind mounts; use a named volume or `database.journal_mode: delete`. — [Docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- A reported Ollama hang with `stream=true` plus many tool definitions (issue #25629, via search snippet; not fetched). — [#25629](https://github.com/NousResearch/hermes-agent/issues/25629)
- Hermes v0.16.0 on macOS 15: `hermes gateway start` may fail to register launchd (native install, not Docker; snippet only). — [search result](https://hermes-agent.nousresearch.com/docs/reference/faq)

### Inferences
- On Docker Desktop for Mac, `/var/run/docker.sock` is provided via VM proxy, so DooD GID issues likely differ (speculative). Ollama on host from a container requires `host.docker.internal` rather than localhost (speculative; not confirmed by a Hermes source).

### Gaps
- No Mac mini/Ollama/Telegram Docker terminal report found; no Japanese (Qiita/Zenn) or Reddit case with exact terminal error found.
