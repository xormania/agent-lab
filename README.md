# agent-lab

`agent-lab` is a private Docker Compose containment lab for experimenting with autonomous agent workloads behind explicit network, filesystem, credential, and egress controls.

It is not a general AI application stack, a public SaaS control plane, or an unrestricted agent launcher. The v0 slice only implements the containment substrate: an internal agent network, controlled DNS, a Squid egress proxy, and acceptance tests.

## Default Posture

- No public ports.
- No direct internet from agent/test containers.
- No Docker socket mounts.
- No host home-directory mounts.
- No privileged containers.
- No real secrets in Git.
- Agent/test containers attach only to the `agents` network.
- The `agents` network is `internal: true`; this is the primary default-deny boundary.
- Squid is the only sanctioned outbound path.
- Proxy environment variables help cooperating tools, but they are not the security boundary.

This is practical Docker containment, not VM-grade isolation. Containers still share the host kernel. A container-runtime or kernel escape is outside the guarantees of this v0 lab.

## Quick Start

```bash
cp .env.example .env.local
./scripts/doctor
./scripts/up core
./scripts/up egress
./scripts/egress-test
```

Stop the stack:

```bash
./scripts/down
```

`./scripts/down --volumes` also removes named volumes, including the `audit` log volume.

## Profiles

- `core`: creates the internal network substrate and starts CoreDNS.
- `egress`: starts Squid as the only dual-homed service on `agents` and `egress`.
- `devtools`: enables the disposable `egress-test` container used by acceptance tests.

Nothing starts by default. The helper scripts activate profile combinations intentionally.

`./scripts/up egress` automatically includes `core`. `./scripts/up devtools` also includes `core`, so it can be used for no-internet tests without starting Squid.

## Egress Modes

No-internet mode is `core` plus `devtools`, without `egress-proxy`. The test container is attached only to the internal `agents` network, so raw direct internet attempts fail because there is no off-bridge route.

Allowlisted-egress mode is `core` plus `egress` plus `devtools`. The test container still has no direct internet route. Cooperating tools can use `HTTP_PROXY` or `HTTPS_PROXY` to reach Squid at `172.30.0.20:3128`. Squid allows only domains in `policies/egress.allowlist.example` by default.

Raw direct egress attempts are blocked by the internal Docker network, but they are not logged in v0 unless optional host firewall hardening is added later. Proxy-mediated allowed and denied requests are logged by Squid.

## Static Network Defaults

```text
agents subnet: 172.30.0.0/24
CoreDNS:       172.30.0.10
Squid:         172.30.0.20:3128
```

These values are duplicated in `.env.example`, `compose.yaml`, `compose.egress.yaml`, `dns/coredns/Corefile`, and `gateway/squid/squid.conf` where needed so the generated configuration stays readable.

## What v0 Does Not Prove

- TLS SNI peek/splice enforcement is not enabled yet. Squid currently enforces the CONNECT host/domain allowlist, private destination denies, unsafe port denies, and default deny. SNI mismatch protection is an explicit M3 TODO.
- Raw blocked attempts are not logged without a future host firewall layer.
- IPv6 is not enabled on the `agents` network. If Docker IPv6 is enabled host-wide, re-audit before relying on these rules.
- Allowlisted domains can still receive exfiltrated data. The allowlist bounds where data may go, not what data is sent.
