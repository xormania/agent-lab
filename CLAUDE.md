# CLAUDE.md — agent-lab

Read `./AGENTS.md` first and treat it as the source of truth for this repository.

This file intentionally delegates project policy to `AGENTS.md` so Codex, Claude Code, and other agent tools share one consistent rule set.

Claude-specific operating notes:

- Prefer planning, review, and small patches over broad rewrites.
- Treat this as a security-sensitive Docker containment project.
- Do not touch real secrets or private env files.
- Do not open ports beyond localhost unless explicitly requested.
- Do not add Docker socket access, privileged containers, broad host mounts, or direct agent internet access without explicit approval.
- When editing, keep diffs small and report validation commands run.
- When reviewing, cite file paths and line-relevant evidence where possible.
- If `AGENTS.md` and this file conflict, follow `AGENTS.md`.
