# Codex prompt — egress gateway test harness

You are working in `agent-lab`.

Read `AGENTS.md` first and follow it.

Task: design and/or implement a non-destructive test harness for validating agent egress behavior.

Required test cases:

1. Agent in no-internet profile cannot reach public internet.
2. Agent in no-internet profile cannot reach host or LAN/private ranges.
3. Agent in allowlisted-egress profile can reach explicitly allowed destinations.
4. Agent in allowlisted-egress profile cannot reach denied destinations.
5. Disallowed attempts are observable in gateway logs or test output.
6. DNS behavior does not silently bypass the intended egress path.

Constraints:

- Do not use real secrets.
- Do not require privileged host changes for the first pass.
- Do not depend on one external service unless documented and replaceable.
- Prefer deterministic local tests where possible.
- Do not claim total egress control unless the test actually proves the claim.

Deliverables may include:

- `scripts/egress-test`
- `tests/egress/README.md`
- a tiny disposable test container profile
- documented expected pass/fail output

Final report:

- What is tested.
- What is not proven yet.
- Commands run.
- Follow-up hardening needed.
