Read `AGENTS.md` and `CLAUDE.md` first.

Audit egress control for `agent-lab`.

Determine whether agent workloads can reach:

- public internet
- host machine
- Docker gateway
- LAN/RFC1918 ranges
- metadata/link-local addresses
- internal data services
- DNS resolvers outside the intended path

Classify each path as:

- blocked by structure
- allowed by explicit profile
- unclear / not proven
- unsafe

Output:

1. Network topology summary.
2. Findings with file evidence.
3. Tests that should exist.
4. Minimal patch recommendations.

Do not run destructive or network-heavy commands unless I ask.
