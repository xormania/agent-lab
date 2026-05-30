# Codex prompt — OpenClaw image hardening plan/patch

You are working on the custom OpenClaw image/profile for `agent-lab`.

Read `AGENTS.md` first and follow it.

Goal: create or improve a custom OpenClaw runtime image that is safer for a private agent containment lab.

Constraints:

- Do not add real credentials.
- Do not mount host home directories.
- Do not mount Docker socket.
- Do not grant privileged mode.
- Do not assume direct internet access from the agent container.
- Keep the image reproducible and documented.

Desired hardening direction:

- pinned base image or clearly marked experimental base
- non-root runtime user
- minimal writable directories
- no-new-privileges in Compose
- dropped capabilities where practical
- healthcheck or documented readiness check
- explicit workdir
- explicit cache/state paths
- clear env example with placeholders only
- comments explaining any required writable path or capability

Deliverables:

- `images/openclaw/Dockerfile`
- `images/openclaw/entrypoint.sh` if useful
- `images/openclaw/README.md` documenting hardening assumptions
- optional `profiles/openclaw.compose.yaml`
- env example updates only, no real secrets

Validation:

- Run `git diff --check`.
- Run `docker compose config` for any modified Compose path if possible.
- Do not build unless explicitly asked.
