# Threat Model

## Assumption

Agent workloads may be prompt-injected, compromised by dependencies, configured incorrectly, or actively trying to reach credentials, the host, the LAN, Docker, or arbitrary internet endpoints.

## Controlled Assets

- Host files and home-directory data.
- Docker socket and Docker daemon control.
- Local network and host services.
- Cloud metadata endpoints.
- Secrets and future service credentials.
- Audit logs.
- Agent workspaces and runtime state.

## Trusted Components

- Docker Compose and Docker's bridge-network enforcement.
- The `agents` network declared `internal: true`.
- CoreDNS configuration for agent/test DNS.
- Squid configuration for mediated outbound HTTP/HTTPS.
- Helper scripts in `scripts/`.

These are trusted components, not perfect components. Bugs or misconfiguration in any of them can weaken the lab.

## Primary Controls

- Agent/test containers attach only to `agents`.
- `agents` is `internal: true`, so it has no route to the internet, host, or LAN.
- `egress` is the only internet-capable bridge.
- Only `egress-proxy` is dual-homed to `agents` and `egress`.
- CoreDNS is pinned as DNS for agent/test containers and refuses arbitrary external recursion.
- Squid enforces a minimal domain allowlist, denies private/link-local/loopback/metadata ranges, denies unsafe ports, and defaults to deny.
- No Docker socket, host home mounts, privileged containers, host networking, or public ports are used in v0.

## Residual Risks

- This is practical Docker containment, not VM isolation. A host kernel or container-runtime escape defeats this design.
- Allowlisted destinations can receive exfiltrated data.
- Raw direct egress attempts are blocked by lack of route but are not logged unless a future host firewall layer is added.
- TLS SNI mismatch protection is not implemented in v0. Squid enforces CONNECT host/domain allowlisting only.
- If Docker IPv6 is enabled host-wide, IPv6 must be re-audited. The v0 `agents` network is intended to be IPv4 only.
- DNS-over-HTTPS can reintroduce external DNS if a DoH endpoint is allowlisted.
- Squid and CoreDNS are part of the trust base.
- Upstream images are pinned by non-`latest` tags, not digests, in this v0 patch because no digest was verified during implementation.

## Out of Scope

- Kernel-grade sandboxing.
- Kubernetes, service mesh, Cilium, OpenZiti, or host firewall automation.
- Browser automation hardening. Future browser profiles must treat browser profiles as secrets and browser escape as a separate high-risk attack surface.
- Cloud resources and CI/CD.
- General application stacks, databases, local LLM services, and agent runtimes.
