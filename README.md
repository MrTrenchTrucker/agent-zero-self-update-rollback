# Agent Zero Self-Update vs Container Recreation -- Silent Code Rollback

**Report Author:** MrTrench (CONTAK Production Environment)
**Date:** 2026-04-02
**Agent Zero Versions Tested:** v1.6 (latest as of 2026-04-01)
**Severity:** Medium -- Silent data loss, no user warning
**Repository:** [agent0ai/agent-zero](https://github.com/agent0ai/agent-zero)**Previous Reports:** [Issue #1360](https://github.com/agent0ai/agent-zero/issues/1360), [PR #1412](https://github.com/agent0ai/agent-zero/pull/1412)

---

## 1. Executive Summary

Agent Zero v1.x includes a self-update mechanism that updates application code inside a running container via git operations. This update works correctly within the lifecycle of a single container instance.

However, when the container is recreated (via `docker compose down && docker compose up -d`, image pull + recreate, or any operation that removes and replaces the container), all self-update changes are silently lost. The container reverts to whatever code was baked into the Docker image at the time it was last pulled.

There is no warning, no error message, and no indication that the rollback occurred. The operator believes they are running the updated version when they are actually running the older image version.

This affects every Agent Zero installation that uses the web UI self-updater and subsequently recreates their container for any reason.

This issue was discovered during routine operations on the CONTAK hybrid intelligence platform, where human operator instinct ("that doesn't look right") combined with AI-driven forensic analysis confirmed the behavior through controlled testing on quarantined fresh images.

---

## 2. How the Self-Updater Works

Agent Zero's self-update mechanism (`/exe/self_update_manager.py`) operates as follows:

1. The web UI writes a YAML trigger file to `/exe/a0-self-update.yaml` specifying the desired version
2. Agent Zero restarts within the container
3. The updater performs `git fetch` and `git checkout` of the requested version tag from the official repository
4. Application code in `/a0/` is updated while preserving gitignored paths (including `/a0/usr/`)
5. The web UI restarts and a health check confirms the update succeeded
6. Status is recorded in `/exe/a0-self-update-status.yaml`

The update guide (`docs/guides/self-update.md`) correctly states that user data in `/a0/usr/` is preserved because it is gitignored. The mechanism itself works as designed.

---

## 3. The Problem -- Container Recreation Destroys Updates

### What Happens

The updated application code lives in `/a0/` inside the container. This directory is part of the Docker image layer -- it is NOT a volume mount.

When an operator recreates the container:

```
docker compose down      # Removes the container (writable layer destroyed)
docker compose up -d     # Creates a NEW container from the cached image
```

The new container starts with the original `/a0/` code from the image. The self-update changes existed only in the previous container's writable layer, which was destroyed by `docker compose down`.

### What the Operator Sees

Nothing. No error, no warning, no rollback notification. The container starts normally. The web UI loads. Everything appears functional. The operator has no way to know that their self-update was silently undone unless they manually check version numbers or git hashes.

### Verification Methodology

We verified this behavior by pulling a fresh `agent0ai/agent-zero:latest` image and creating an isolated test container with no volume mounts:

```bash
docker pull agent0ai/agent-zero:latest
docker create --name a0-test --network none agent0ai/agent-zero:latest bash -c "sleep infinity"
docker start a0-test
```

Key findings from the stock image:

- `/a0/` is NOT a git repository in the stock image (no `.git` directory)
- Application source lives at `/git/agent-zero/` (this IS a git repository)
- `/a0/usr/` does not exist in the stock image (created at runtime by volume mounts)
- The Dockerfile declares no `VOLUME` directives (`null`)
- `/per/` (persistent overlay, copied to `/` on startup) contains only `/root/` configuration -- no `/a0/` code
- The `initialize.sh` startup script copies `/per/*` to `/` but does not check for or restore self-update state

### What Survives vs What is Lost

| Path               | Mounted?              | Survives Recreation? | Contains                                   |
|--------------------|-----------------------|----------------------|--------------------------------------------|
| `/a0/`             | No (image layer)      | **No -- LOST**       | Application code (updated by self-updater) |
| `/a0/usr/`         | Yes (operator mounts) | Yes                  | User data, chats, memory, plugins          |
| `/exe/`            | Depends on setup      | Depends              | Self-update trigger, status, logs          |
| `/git/agent-zero/` | No (image layer)      | **No -- LOST**       | Source repository                          |
| `/per/`            | No (image layer)      | **No -- LOST**       | Persistent overlay (root config only)      |

---

## 4. Scope of Impact

This affects ALL Agent Zero installations where:

1. The operator uses the web UI self-updater (Settings -> Update -> Self Update)
2. The container is subsequently recreated for any reason

Common scenarios that trigger container recreation:

- `docker compose down && docker compose up -d` (routine maintenance)
- `docker compose up -d --force-recreate` (compose file changes)
- Host system reboot with `restart: no` policy (container not restarted, manually recreated)
- Docker engine upgrade (may recreate containers)
- Automated tools like Watchtower that pull new images and recreate containers
- Operator follows the "Manual Docker Update" path without realizing the self-update existed

The Agent Zero documentation (`docs/guides/self-update.md`) does not warn about this behavior. The troubleshooting guide mentions that mounting the entire `/a0/` directory "will overwrite application code and break upgrades" but does not address the inverse problem -- that NOT mounting `/a0/` means self-updates are ephemeral.

---

## 5. Existing Partial Mitigation

Agent Zero already has a durable update system that partially addresses this. The files at `/exe/` (trigger YAML, status YAML, log file) are designed to survive restarts because `/exe/` is outside `/a0/`.

However, `/exe/` is also part of the image layer unless the operator explicitly mounts it as a volume. If `/exe/` is not mounted, the trigger file is also lost on recreation, and the system cannot detect that a re-application is needed.

If `/exe/` IS mounted (as some deployment guides suggest), the trigger file survives -- but the `initialize.sh` startup script does not check for it. It simply copies `/per/*` to `/` and starts supervisord. The self-update manager only runs when explicitly triggered, not on every startup.

---

## 6. Proposed Fixes

### Fix A: Breadcrumb in /a0/usr/ (Recommended -- Minimal Change)

After a successful self-update, write the target git hash and tag to `/a0/usr/.a0_self_update_state`:

```json
{
  "updated_at": "2026-04-01T12:00:00Z",
  "target_tag": "v1.6.1",
  "target_hash": "abc123def456",
  "branch": "main"
}
```

On container startup, `initialize.sh` checks for this file. If it exists and the current `/a0/` code does not match the recorded hash, the startup script automatically re-applies the self-update by running the update manager.

Since `/a0/usr/` is the directory operators are instructed to mount as a persistent volume, this breadcrumb survives container recreation. This pattern is widely used -- PostgreSQL, MySQL, and other official Docker images use sentinel files on mounted volumes to track initialization state.

**Pros:** Minimal code change (one write after update, one check on startup), backward compatible, self-healing

**Cons:** Adds startup time for git checkout on container recreation (typically 10-30 seconds)

### Fix B: Warning Banner in Web UI (Complementary)

After a successful self-update, display a persistent notice in the web UI:

> "Your Agent Zero code was updated inside this container. If you recreate the container (docker compose down/up), the update will be lost unless you first run `docker pull agent0ai/agent-zero:latest` to get the latest image."

This is a defense-in-depth measure that relies on the operator seeing and understanding the warning.

**Pros:** Zero risk, no startup behavior change

**Cons:** Relies on humans reading warnings

### Fix C: Startup Reconciliation (Most Robust)

Modify `initialize.sh` to always check `/exe/a0-self-update-status.yaml` (if it exists on a mounted volume) and compare the recorded version against the current `/a0/` state. If a mismatch is detected:

1. Log a warning: "Detected container recreation after self-update. Re-applying update..."
2. Run the self-update manager in automatic mode to restore the desired version
3. Continue with normal startup

This leverages the existing durable update infrastructure at `/exe/` and requires only that `/exe/` be on a persistent mount.

**Pros:** Fully automatic, leverages existing code

**Cons:** Requires `/exe/` to be mounted (not all deployments do this)

### Recommendation

We recommend **Fix A + Fix B** together:

- Fix A provides automatic self-healing using the volume mount that operators already configure (`/a0/usr/`)
- Fix B provides a visible reminder for operators who may not have Fix A deployed yet

Fix C is an excellent alternative if the team prefers to keep the breadcrumb in `/exe/` rather than `/a0/usr/`.

---

## 7. References

- Docker Documentation -- Container Storage Drivers: "When the container is deleted, the writable layer is also deleted. The underlying image remains unchanged."
  - https://docs.docker.com/engine/storage/drivers/
- Docker Documentation -- `docker compose down`: "Stops containers and removes containers, networks, volumes, and images created by up."
  - https://docs.docker.com/reference/cli/docker/compose/down/
- Docker Documentation -- Bind Mounts: "When you use a bind mount, a file or directory on the host machine is mounted into a container."
  - https://docs.docker.com/engine/storage/bind-mounts/
- Docker Documentation -- Immutability Best Practices: "DHI users are encouraged to rebuild and redeploy images rather than patch running containers."
  - https://docs.docker.com/dhi/core-concepts/immutability/
- Google Cloud Architecture -- Best Practices for Operating Containers: Treat containers as immutable.
  - https://cloud.google.com/architecture/best-practices-for-operating-containers
- Kubernetes git-sync -- Official SIG project for syncing git repositories to shared volumes.
  - https://github.com/kubernetes/git-sync
- Docker Compose Issue #2912 -- Data loss when recreating containers (volume detachment).
  - https://github.com/docker/compose/issues/2912
- Agent Zero Discussion #542 -- Users reporting data loss on container restart.
  - https://github.com/agent0ai/agent-zero/discussions/542
- Agent Zero Issue #1200 -- Data loss after improper update methodology.
  - https://github.com/agent0ai/agent-zero/issues/1200
- Agent Zero Self-Update Guide: `/docs/guides/self-update.md`
  - https://github.com/agent0ai/agent-zero

---

## 8. Disclosure

This issue was discovered during routine production operations on the CONTAK-01 hybrid intelligence platform, where a human operator's instinct that "something wasn't right" after a container recreation triggered an AI-assisted forensic investigation. The combination of human skepticism and AI verification capability confirmed the behavior through controlled testing on quarantined fresh Docker images.

All findings were verified against stock `agent0ai/agent-zero:latest` images with no volume mounts, ensuring no contamination from production customizations.

---

## License

MIT License

Copyright (c) 2026 MrTrench

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.