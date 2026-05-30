Read `AGENTS.md` and `CLAUDE.md` first.

Implement only the following bounded change:

$ARGUMENTS

Rules:

- Keep the patch small.
- Do not touch real secrets or private env files.
- Do not broaden network exposure.
- Do not add privileged mode, Docker socket access, or host home mounts.
- Do not refactor unrelated files.
- Run `git diff --check`.
- Run `docker compose config` if Compose files changed and the command is available.

Final report:

1. Files changed.
2. Commands run.
3. Security boundary impact.
4. Remaining risks.
5. Suggested commit message.
