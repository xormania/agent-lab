Read `AGENTS.md` and `CLAUDE.md` first.

Review this repository as `agent-lab`: a private Docker containment lab for autonomous AI agents.

Focus on whether the repo actually enforces its stated boundaries.

Check:

- tracked secrets or unsafe env files
- public port bindings
- direct agent internet access
- host/LAN reachability risk
- Docker socket exposure
- privileged containers
- broad host mounts
- missing profile separation
- misleading docs
- unstable image tags
- missing validation scripts

Output:

1. What you inspected.
2. Severity-ranked findings.
3. File-specific evidence.
4. Minimal patch plan.
5. Things not inspected or not provable from source.

Do not implement changes unless I explicitly ask.
