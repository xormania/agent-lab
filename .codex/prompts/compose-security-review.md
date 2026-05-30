# Codex prompt — Compose security review

You are reviewing the `agent-lab` Docker Compose configuration.

Read `AGENTS.md` first and follow it.

Inspect all Compose-related files, env examples, gateway configs, and scripts. Do not read real `.env` or secret files.

Review for:

- public port exposure instead of `127.0.0.1`
- direct agent internet access
- agents attached to broad networks
- missing `internal: true` where isolation is expected
- `privileged: true`
- Docker socket mounts
- host home-directory mounts
- unsafe bind mounts
- excessive capabilities
- missing `no-new-privileges`
- missing read-only/rootless posture where practical
- unpinned or unstable image tags
- misleading README/security claims
- secrets in tracked files
- Compose profiles that accidentally start too much

Output:

1. Severity-ranked findings.
2. Exact file paths and relevant snippets/lines.
3. Minimal patch recommendations.
4. Any claims that cannot be verified from source.

Do not implement unless explicitly asked.
