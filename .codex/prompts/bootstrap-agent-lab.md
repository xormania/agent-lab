# Codex prompt — bootstrap agent-lab skeleton

You are working in the `agent-lab` repository.

Read `AGENTS.md` first and follow it.

Task: create the first minimal repository skeleton for a private Docker-based AI agent containment lab.

Requirements:

1. Do not add real secrets.
2. Do not add broad public port exposure.
3. Do not add Docker socket mounts.
4. Do not add privileged containers.
5. Do not create a giant app stack.
6. Keep the first patch small.

Create or update:

- `README.md` with the project purpose, default-deny posture, and first-run warning.
- `SECURITY.md` with secret handling and vulnerability reporting notes.
- `THREAT_MODEL.md` with assumptions for untrusted agents, egress, filesystem, and secrets.
- `.gitignore` for env files, secrets, runtime data, caches, logs, and Compose overrides.
- `compose.yaml` with only minimal networks and placeholder-safe structure.
- `env/base.env.example` with placeholders only.
- `policies/egress.allowlist.example` and `policies/lan.denylist.example`.
- `scripts/doctor` that performs non-destructive checks.
- `scripts/egress-test` as a stub or safe test harness with clear TODOs.

Compose expectations:

- Use an internal agent network.
- Use a separate egress/gateway network concept.
- Do not claim total egress control until it is actually enforced.
- Bind any operator UI examples to `127.0.0.1` only.

Validation:

- Run `git diff --check`.
- Run `docker compose config` if Docker Compose is available.
- Report any command that could not be run.

Final report:

- What changed.
- Commands run.
- Security boundary impact.
- Remaining risks.
