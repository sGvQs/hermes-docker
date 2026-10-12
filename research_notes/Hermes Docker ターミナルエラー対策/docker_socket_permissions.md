# Docker-level causes and fixes when Hermes Agent (nousresearch/hermes-agent) cannot run terminal commands / use Docker from inside a container on macOS Docker Desktop

## 1. Is the docker CLI in the image? What user does it run as?

### Takeaway
The current image installs the Debian `docker-cli` package and starts as root, remapping to user `hermes` (UID 10000) via an s6-overlay hook. So "docker: command not found" is unlikely on a current image. Failures are far more likely to be socket access or path mapping.

### Cited Findings
- Dockerfile base is debian:13.4; the `docker-cli` apt package (not docker-ce-cli) is installed in the `runtime_base` layer — [Dockerfile](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/Dockerfile)
- No `USER` directive: the container starts as root so the s6 stage2 hook can remap UIDs and fix volume ownership; a `hermes` user (UID 10000, home `/opt/data`) is created and services drop to it with `s6-setuidgid hermes` — [Dockerfile](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/Dockerfile)
- ENTRYPOINT is `/opt/hermes/docker/entrypoint-dispatch.sh`, `CMD []`, `VOLUME /opt/data` (= HERMES_HOME) — [Dockerfile](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/Dockerfile)
- `docker/entrypoint.sh` is a deprecated shim; do not override ENTRYPOINT with it, use the image default — [entrypoint.sh](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/docker/entrypoint.sh)
- `docker exec hermes <cmd>` starts as root but a shim re-execs as `hermes` (UID 10000); `HERMES_DOCKER_EXEC_AS_ROOT=1` for root; if `s6-setuidgid` is missing the shim exits 126 — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- Images built before late Aug 2026 could set `/opt/hermes` to 0700 causing "Permission denied" on every `docker exec`; fix by newer image or `docker exec -u root hermes chmod 0755 /opt/hermes` — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- `hermes gateway run` as root is refused unless `HERMES_ALLOW_ROOT_GATEWAY=1` — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- Official docs do not mention docker.sock, docker-out-of-docker, or tty/stdin_open; the CLI is listed but nothing says how it reaches a daemon — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- Docs describe `terminal.backend: docker` where Hermes runs on the host and executes commands in one persistent sandbox container; keys: `terminal.docker_image`, `docker_volumes`, `docker_forward_env`, `docker_env`, `docker_run_as_host_user`, `docker_extra_args` — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)

### Inferences
- If Hermes itself runs in a container and `terminal.backend: docker`, it needs a reachable daemon (socket mount or DOCKER_HOST). The official docs only describe the backend with Hermes on the host. Simplest alternative: set `terminal.backend: local` so commands run inside the Hermes container itself.
- Older image tags may lack the CLI; verify with `which docker`. I did not verify historical Dockerfiles.

### Gaps
- Did not retrieve the exact Configuration > Docker Backend section; exact config semantics when Hermes is itself containerized are unconfirmed.
- Did not confirm which image tag first included `docker-cli`.

## 2. docker.sock permissions on Docker Desktop for Mac

### Takeaway
On Docker Desktop for Mac, the mounted `/var/run/docker.sock` is the socket inside the Linux VM, not a macOS file. Container root normally can use it; non-root (e.g. `hermes` UID 10000) usually gets "permission denied" unless the socket's group is added. Behaviour has shifted across Desktop versions (4.19 regression reports).

### Cited Findings
- Docker forum reply: on Desktop for Mac the mounted socket is the VM's socket; `-u` doesn't alter access as on native Linux — [Docker forums: Permission for -v /var/run/docker.sock](https://forums.docker.com/t/permission-for-v-var-run-docker-sock/132976)
- Default on Linux is `root:docker` 660; `root root srwxr-xr-x` indicates modification — same thread above
- Users reported mounting `/var/run/docker.sock.raw` as the fix pre-4.19, which stopped working on 4.19 with "permission denied while trying to connect to the Docker daemon socket"; no official resolution on the page — [Docker forums 4.19](https://forums.docker.com/t/docker-desktop-version-4-19-broke-var-run-docker-sock-raw-to-resolve-permission-denied/135864)
- Community workarounds: socat forwarder, or create a matching group in the container from the socket's GID — [DEV: Docker-in-Docker access for non-root](https://dev.to/jonfriesen/docker-in-docker-access-for-non-root-users-3ijh) (blog, not official)
- Docker Desktop's `/var/run/docker.sock` symlink on the Mac host is optional (Advanced setting "default Docker socket"), cleared on reboot since /var/run is tmpfs; otherwise set `DOCKER_HOST` — [Docker docs: Mac permission requirements](https://docs.docker.com/desktop/setup/install/mac-permission-requirements/)
- Containers run as root inside a Linux VM, which isn't root on the Mac host; bind-mounted host dirs keep original permissions — same Docker docs page
- Community Hermes setup: for non-root, mount socket and `group_add` the socket's GID (default 970 on author's host, check with `stat`) — [hermes-agent-docker](https://code.mehalter.com/mehalter/hermes-agent-docker)

### Inferences
- Fix order for "permission denied": (a) check `id` in container; (b) `stat -c '%u:%g %a' /var/run/docker.sock`; (c) `group_add: ["<that gid>"]` in compose, or run service as root; (d) as last resort root-user only. Use the GID reported inside the container, not from macOS (macOS has no docker group; see [forum](https://forums.docker.com/t/do-we-have-docker-group-on-mac/35740)).
- Because Hermes starts as root and drops to `hermes`, a `group_add` on the compose service should propagate to s6-setuidgid children only if the drop preserves supplementary groups; unverified, test with `docker exec hermes id`.

### Gaps
- No official Docker doc found stating the exact owner/mode of the socket inside containers on current (2026) Desktop for Mac.
- Current Desktop behaviour post-4.19 for `docker.sock.raw` unconfirmed.

## 3. Sibling containers: host path vs container path

### Takeaway
The daemon resolves `-v` source paths on the daemon host (the Desktop VM / Mac file shares), not inside the Hermes container. So a path like `/opt/data/...` passed to a child container does not point to `~/.hermes`.

### Cited Findings
- "Docker resolves bind mounts on the daemon host, not inside the Hermes container"; `HERMES_HOME` must be the same absolute path on host and container, and remapping (e.g. `/home/x/.hermes:/opt/data`) is marked incorrect because Hermes builds sandbox bind mounts from that path — [hermes-agent-docker](https://code.mehalter.com/mehalter/hermes-agent-docker)
- Standard official mapping is `~/.hermes:/opt/data`; sandbox mounts configured separately via `terminal.docker_volumes` — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- Skills and declared credential files are bind-mounted read-only into the sandbox — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- `HERMES_DOCKER_PROPAGATE_RUNTIME_USER=1` injects `--user` into docker run/create — [hermes-agent-docker](https://code.mehalter.com/mehalter/hermes-agent-docker)

### Inferences
- Fix pattern: mount the same path at both sides, e.g. `${HOME}/.hermes:${HOME}/.hermes` with `HERMES_HOME=${HOME}/.hermes`, instead of `:/opt/data`. Caveat: this conflicts with the image's `VOLUME /opt/data`/ HERMES_HOME default and home=/opt/data for user hermes; test. Alternative: keep `/opt/data` and give `terminal.docker_volumes` host-absolute paths.
- On Mac, host path must be inside a Docker Desktop file-sharing location (default includes /Users) to be visible to the VM daemon.

### Gaps
- Behaviour of the official image when HERMES_HOME differs from /opt/data not verified.

## 4. Duplicate/overlapping mounts, SQLite, and Mac bind-mount permission errors

### Takeaway
Never point two containers/paths at the same data dir; on Mac bind mounts SQLite WAL can corrupt, and permission errors are fixed with HERMES_UID/GID, not chmod.

### Cited Findings
- Never run two gateway containers against the same data directory — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- On virtiofs/9p mounts (Docker Desktop macOS), SQLite WAL may corrupt `state.db`; use a named volume `hermes-data:/opt/data` — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- "Permission denied": set `HERMES_UID`/`HERMES_GID` (or `PUID`/`PGID`) to host owner; hook remaps and chowns on first boot; don't make data world-readable; `HERMES_HOME_MODE` forces mode — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- `/opt/hermes` install tree is read-only to the runtime user; mutable state must live in `/opt/data` — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)

### Inferences
- Mounting the same host dir at two container paths (e.g. `/opt/data` and `${HOME}/.hermes`) gives two views of one state dir; it works for path mapping but doubles the WAL/locking risk only if two processes write; keep one writer.
- Mac host uid is typically 501; set `HERMES_UID=501 HERMES_GID=20` (staff) if bind-mounting.

### Gaps
- Exact behaviour of osxfs/virtiofs chown from container root not sourced.

## 5. TTY / interactive terminal

### Takeaway
Official docs do not require `tty`/`stdin_open` in compose for the gateway; they appear only as `-it` in `docker run` examples. Interactive use is via `docker exec -it`.

### Cited Findings
- Compose examples don't set `tty` or `stdin_open`; `-it` used in docker run examples; interactive CLI via `/opt/hermes/.venv/bin/hermes` in running container — [Hermes Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)

### Inferences
- A missing PTY affects only interactive CLI (`docker exec -it`, `docker run -it`); it is not a cause of docker.sock permission errors. Add `tty: true`/`stdin_open: true` only for `docker attach`-style use.

### Gaps
- No source found on PTY allocation problems specific to Hermes's terminal tool.

## 6. Security implications of mounting docker.sock

### Takeaway
Socket access is root-equivalent on the daemon host (the Desktop VM on Mac); read-only mount doesn't help.

### Cited Findings
- Socket access is effectively root on the daemon host regardless of read-only mount; Hermes can request privileged sibling containers — [hermes-agent-docker](https://code.mehalter.com/mehalter/hermes-agent-docker)
- Prefer `DOCKER_HOST=ssh://...` over a local socket mount — same source
- Widening socket permissions lets anyone create containers usable for privilege escalation — [Docker forums](https://forums.docker.com/t/permission-for-v-var-run-docker-sock/132976)
- On Mac, container root is root only in the Linux VM, not on macOS; bind mounts remain subject to host permissions — [Docker docs](https://docs.docker.com/desktop/setup/install/mac-permission-requirements/)

### Inferences
- On Mac the blast radius is the VM plus any shared host directories (e.g. /Users), still large for an LLM agent.

### Gaps
- None significant.

## 7. Diagnostic checklist (commands; not sourced, standard Docker usage)

```
docker compose ps; docker inspect hermes --format '{{json .Mounts}}'
docker exec hermes id                      # uid/gid/groups of exec user
docker exec -u root hermes id
docker exec -u root hermes ls -l /var/run/docker.sock
docker exec -u root hermes stat -c '%u:%g %a' /var/run/docker.sock
docker exec hermes which docker; docker exec hermes docker version
docker exec hermes sh -c 'echo $DOCKER_HOST; env | grep -i -E "hermes|docker|puid|pgid"'
docker exec hermes ls -ld /opt/hermes /opt/data; docker exec hermes touch /opt/data/.t
docker exec hermes mount | grep -E "opt/data|docker.sock"
docker exec -it hermes /opt/hermes/.venv/bin/hermes
docker logs hermes --tail 100
grep -n "terminal" ~/.hermes/config.yaml    # backend: local|docker, docker_volumes
```
Interpretation: socket `root:root 660` + user hermes -> permission denied (add group_add or run root). `docker: command not found` -> old image/custom image. `Cannot connect to the Docker daemon` -> socket not mounted / DOCKER_HOST wrong. Child container sees empty dir -> host/container path mismatch. `exit 126` on exec -> s6-setuidgid missing / /opt/hermes 0700.
