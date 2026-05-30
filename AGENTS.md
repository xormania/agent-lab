# AGENTS.md — agent-lab

## Project purpose

`agent-lab` is a private Docker-based containment lab for experimenting with autonomous AI agents such as OpenClaw, Hermes, OpenHands, browser agents, and tool-using model agents.

This repository is not a general AI application stack. Its primary job is to make risky agent workloads observable, reproducible, and bounded by explicit network, filesystem, credential, and egress controls.

## Current posture

Treat this project as experimental, but treat the security boundaries as real. A broken experiment is acceptable. A silent boundary bypass is not.

Default stance:

- no public ports
- no direct internet from agent containers
- no host home-directory mounts
- no Docker socket access from agents
- no privileged containers
- no real secrets in Git
- no broad persistent state shared across agents
- localhost-only operator interfaces unless explicitly approved
- explicit egress gateway for allowed outbound traffic
- every boundary relaxation must be visible in code and docs

## Source of truth

- Repository name: `agent-lab`
- Default local path: `$HOME/projects/agent-lab/`
- GitHub is source authority for tracked source.
- Local `.env`, `.env.local`, secrets, runtime data, caches, model weights, database files, and generated logs are not source authority.
- Google Drive/shared folders may hold handoffs, source packs, audits, or backups, but not live `.git` repositories or runtime dependency trees.

## Non-goals

Do not drift this repository into:

- a general “run every AI app” compose bundle
- a public SaaS control plane
- a convenience-first dev environment
- an unrestricted autonomous agent launcher
- a place to store personal secrets, API keys, browser profiles, or production credentials

Apps are allowed only when they support the containment lab mission.

## Threat model

Assume agent workloads may be:

- prompt-injected
- compromised by upstream dependencies
- configured incorrectly
- attempting unintended network access
- attempting credential discovery
- attempting LAN or host probing
- attempting tool abuse through shells, browsers, package managers, or APIs
- attempting to persist state outside assigned volumes

Assume upstream images may change unexpectedly unless pinned and reviewed.

## Security invariants

### Network

- Agent containers must not have direct internet access by default.
- Egress must pass through a designated gateway/proxy/profile.
- LAN, host, RFC1918, link-local, metadata, and Docker gateway access must be blocked unless a specific profile documents and justifies the exception.
- Published ports must bind to `127.0.0.1` by default.
- Avoid `network_mode: host`.
- Prefer explicit named networks.
- Use `internal: true` for networks that should not route externally.
- Do not attach agents to broad app/data networks unless required and documented.

### Filesystem

- Do not mount `$HOME`, project parent directories, SSH dirs, cloud-drive roots, browser profiles, or password-manager data into agent containers.
- Agent workspaces must be narrow, per-agent, and disposable where possible.
- Prefer read-only root filesystems where supported.
- Use explicit writable directories, named volumes, or tmpfs mounts.
- Do not mount `/var/run/docker.sock` into agent containers.

### Container privileges

Default hardening for agent containers should move toward:

```yaml
security_opt:
  - no-new-privileges:true
cap_drop:
  - ALL
read_only: true
user: "1000:1000"
pids_limit: 256
```

Only add capabilities, writable mounts, device mounts, or privileged mode with explicit justification.

### Secrets

- Never commit real secrets.
- Keep `.env`, `.env.local`, `.env.*.local`, `secrets/`, and key material out of Git.
- Commit only `.env.example` or env files containing obvious placeholders.
- Secrets must be scoped per service or per agent.
- Do not print secrets in logs, review notes, or final summaries.
- If a real secret is found in tracked source, stop and report it before continuing.

### Supply chain

- Prefer pinned image tags or digests for stable profiles.
- `latest`, `main`, `nightly`, and unpinned third-party images are acceptable only in clearly marked experimental profiles.
- Do not add GitHub Actions, remote install scripts, curl-pipe-shell commands, or privileged setup steps without explicit approval.
- Custom agent images must document their base image, user, writable paths, network expectations, and runtime capabilities.

## Expected repository shape

A good structure for this project is:

```text
agent-lab/
  AGENTS.md
  CLAUDE.md
  README.md
  SECURITY.md
  THREAT_MODEL.md
  compose.yaml
  profiles/
  gateway/
  images/
  env/
  policies/
  scripts/
  tests/
  .codex/
  .claude/
```

Use this shape as guidance, not as permission to create all directories prematurely.

## Compose conventions

- Use `docker compose`, not legacy `docker-compose`.
- Prefer profiles over one giant always-on stack.
- Keep base `compose.yaml` minimal.
- Put optional workloads under `profiles/*.compose.yaml` when the file count becomes useful.
- Use `127.0.0.1:${PORT}:...` for operator-facing ports by default.
- Use named volumes for durable app state.
- Do not persist secrets inside broad app volumes.
- Add healthchecks where they affect startup safety or operator clarity.
- Do not claim readiness guarantees in docs unless Compose actually enforces them.

Suggested profile names:

```text
core
egress
no-internet
allowlisted-egress
openclaw
hermes
openhands
browser
local-llm
cloud-llm
observability
danger-zone
```

## Egress-control expectations

The project should eventually prove these cases with scripts or tests:

1. An agent in no-internet mode cannot reach public internet endpoints.
2. An agent in no-internet mode cannot reach host or LAN/private addresses.
3. An agent in allowlisted-egress mode can reach only approved destinations.
4. Disallowed egress attempts are logged or otherwise observable.
5. DNS behavior is controlled and does not silently bypass the gateway.

Environment variables such as `HTTP_PROXY` and `HTTPS_PROXY` are not sufficient by themselves. Network structure must enforce the policy.

## Allowed work without extra approval

Agents may usually do the following without asking first:

- inspect tracked source
- draft docs and threat models
- add or improve `.env.example` files
- add `.gitignore` rules for secrets/runtime files
- add Compose profiles that preserve default-deny behavior
- add validation scripts that do not require secrets
- add tests that verify containment behavior
- harden containers by reducing privileges
- improve README accuracy when docs overclaim
- propose patches in small, reviewable chunks

## Ask before doing these

Ask before:

- exposing ports beyond localhost
- adding direct internet access to an agent
- adding privileged containers
- adding Docker socket access
- mounting host directories outside explicit project workspaces
- adding real third-party services with credentials
- adding GitHub Actions or other CI/CD automation
- introducing cloud resources
- adding long-running background daemons outside Compose
- changing the core threat model
- deleting large parts of the repo

## Hard stops

Stop and report immediately if you discover:

- real secrets committed to source
- `.env.local` or private env files tracked by Git
- Docker socket exposure to agent containers
- public port exposure not clearly documented
- agent containers with both credentials and unrestricted egress
- host home-directory mounts into agent containers
- commands or docs that encourage copying secrets into the repo

## Validation commands

Prefer these low-risk checks when relevant:

```bash
git status --short
git diff --stat
git diff --check
docker compose config
```

When scripts exist, prefer:

```bash
./scripts/doctor
./scripts/egress-test
```

Do not run full stacks, destructive tests, image builds, package installs, or network-heavy commands unless the task requires it or the user explicitly asks.

## Implementation style

- Make the smallest useful patch.
- Do not refactor broad surfaces while fixing one issue.
- Keep experimental and stable paths separate.
- Prefer explicit config over hidden magic.
- Preserve readable comments where they explain a security boundary.
- Do not add fake abstractions for future agents that do not exist yet.
- Do not claim a security property unless it is enforced and testable.

## Reporting format

When reporting work, include:

1. What was inspected.
2. What changed.
3. Commands run and results.
4. Security boundary impact.
5. Remaining risks or follow-up work.

For reviews, use severity labels sparingly:

- Critical: likely secret exposure, host escape, public exposure, or direct contradiction of containment goal.
- High: credible boundary bypass or dangerous default.
- Medium: misleading docs, weak validation, missing persistence, unstable image, or fragile startup order.
- Low: cleanup, naming, comments, nonblocking consistency.

## Agent-specific note

This repository is allowed to contain instructions for agents, but agents are not trusted merely because they follow instructions. The code, Compose topology, network rules, and tests must carry the real boundary.
