Read `AGENTS.md` and `CLAUDE.md` first.

Review the OpenClaw-related files in this repository.

Focus on custom image hardening and runtime containment:

- base image pinning
- non-root user
- writable paths
- capabilities
- read-only filesystem feasibility
- secrets handling
- network access
- proxy assumptions
- health/readiness
- persistent state
- update risk
- docs accuracy

Output:

1. What you inspected.
2. Findings by severity.
3. Minimal patch recommendations.
4. Any behavior that cannot be verified without running the image.

Do not implement unless I ask.
