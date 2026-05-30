# agent-lab — v0 Implementation Plan

> Status: planning only. No runtime code is created by this document.
> Source of truth for policy is `AGENTS.md`. Where this plan and `AGENTS.md`
> disagree, `AGENTS.md` wins and this plan must be corrected.
>
> Validation note: `docker` / `docker compose` were not available in the
> authoring environment (WSL2 distro without Docker integration), so none of
> the Compose snippets below have been run. Every snippet is illustrative and
> must be validated with `docker compose config` during implementation.

---

## 1. Executive recommendation

Build `agent-lab` v0 as a **Docker Compose** project whose containment rests on
**network structure first, proxy policy second**, in this layered order:

1. **Structural default-deny (primary enforcement).** Agent containers attach
   only to a Docker network declared `internal: true`. Such a network has no
   route off the bridge, so agents cannot reach the internet, the host, or the
   LAN regardless of what env vars, DNS, or code they run. This is the boundary
   that does not depend on agent cooperation.
2. **Mediated allowlist egress (the one sanctioned exit).** A single dual-homed
   **Squid forward proxy** is the only service attached to both the internal
   agent network and an internet-capable network. It enforces a **domain
   allowlist**, denies RFC1918/link-local/loopback destinations, denies
   raw-IP `CONNECT`, uses **peek-and-splice on TLS SNI** (allowlist by SNI
   *without* decrypting traffic or distributing a CA), and **logs every
   attempt** including denials.
3. **Controlled DNS (close the side channel).** A **CoreDNS** resolver is the
   only resolver agents are given. It refuses external recursion, so agents
   cannot resolve arbitrary names and cannot use DNS as an exfiltration or
   bypass channel. External names for allowed sites are resolved *by Squid*, on
   the internet side, not by the agent.
4. **Host firewall (documented hardening, not required for first boot).**
   `nftables` rules in the host `DOCKER-USER` chain are documented as a
   belt-and-suspenders layer for Linux hosts. They are explicitly optional in
   v0 because the internal network already fails closed, and because they are
   not cleanly portable to WSL2 / Crostini.

Egress component choice for v0: **Squid (proxy) + CoreDNS (DNS)**. Reject
mitmproxy/Envoy/OPA/Cilium/Kubernetes for the default path; keep mitmproxy as an
*optional* deep-inspection profile only.

Why this and not "just set `HTTP_PROXY`": proxy env vars are configuration, not
enforcement. In this design the env vars only tell a *cooperating* agent where
the sanctioned exit is; the `internal: true` network is what stops a
*non-cooperating* agent. The two are layered deliberately.

Do **not** start with Kubernetes. A single-operator local lab does not need a
control plane, CNI, or network policies; Compose with an internal network and a
proxy expresses the entire v0 threat model with far less surface and far more
inspectability.

---

## 2. What `agent-lab` is and is not

**It is:**

- A private, local-first Docker Compose lab for running *untrusted* autonomous
  agent workloads (OpenClaw, Hermes, OpenHands, browser agents, future runtimes)
  behind explicit network, filesystem, credential, and egress boundaries.
- A place where a broken experiment is fine but a *silent boundary bypass* is
  not (per `AGENTS.md`).
- A small default that boots safe, with heavier components behind Compose
  profiles.

**It is not:**

- A general "run every AI app" stack, a SaaS control plane, or a
  convenience-first dev environment.
- A VM-grade or kernel-grade sandbox. It is *practical Docker containment*:
  shared host kernel, Docker's namespace/cgroup isolation, plus network and
  filesystem policy. A kernel or container-runtime escape defeats it. See §18.
- A secrets store. No real secrets, keys, tokens, browser profiles, or model
  caches are ever committed.

---

## 3. Threat model

Assume each agent container is **hostile or compromised** — via prompt
injection, a poisoned dependency, a malicious tool/plugin, or operator
misconfiguration. From that assumption:

| # | Adversary capability we must contain | Primary control |
|---|---|---|
| T1 | Reach arbitrary internet hosts | `internal` network (no route) + proxy allowlist |
| T2 | Reach host services / Docker gateway / `host.docker.internal` | `internal` network + proxy RFC1918 deny; no `extra_hosts` |
| T3 | Reach LAN / other RFC1918 / link-local / cloud metadata (169.254.169.254) | `internal` network + proxy deny-private |
| T4 | Exfiltrate via direct IP (skip DNS) | proxy denies raw-IP CONNECT; agents have no direct route anyway |
| T5 | Exfiltrate / bypass via DNS | CoreDNS refuses external recursion; agent `dns:` pinned to CoreDNS |
| T6 | Reach an allowed domain but tunnel to a different SNI | Squid peek-and-splice enforces SNI = allowlist |
| T7 | Read the Docker socket → control the host | never mount `/var/run/docker.sock` |
| T8 | Read host files / `$HOME` / SSH / cloud-drive / browser profiles | no host mounts; narrow per-agent volumes only |
| T9 | Steal another service's secrets | per-service read-only secret mounts; nothing broad |
| T10 | Persist outside its workspace | `read_only` rootfs + single named volume + tmpfs |
| T11 | Escalate privileges in-container | non-root user, `cap_drop: ALL`, `no-new-privileges` |
| T12 | Exhaust host resources (fork bomb, memory) | `pids_limit`, memory/CPU limits, ulimits |
| T13 | Escape the container to the host kernel | **Not fully mitigated** — residual risk, see §18 |

Out of scope for the *trust* boundary: the allowlisted destinations themselves.
If `api.example.com` is on the allowlist, an agent can send data to it. Egress
to an *allowed* host can carry exfiltrated data — that is inherent to
allowlisting and is stated as a residual risk, not a solved problem.

Supply chain: assume upstream images change unless pinned by tag+digest and
reviewed.

---

## 4. Non-goals for v0

- No Kubernetes, no service mesh, no Cilium/eBPF, no OpenZiti overlay.
- No TLS interception (MITM) on the default path — peek-and-splice only.
- No multi-tenant / multi-operator support; single operator, single host.
- No public port exposure; no remote access; localhost only.
- No Postgres/Redis/MinIO/Qdrant unless a concrete profile needs them.
- No CI/CD, no GitHub Actions, no curl-pipe-shell installers.
- No "perfect sandbox" claims. No security property is asserted without a test
  in §17 that demonstrates it.
- No host-firewall requirement for first boot (documented hardening only).

---

## 5. Recommended v0 architecture

Two networks, three small infrastructure services, agents behind them.

**Networks**

- `agents` — `internal: true`. Agents and the test container live here. No route
  to internet, host, or LAN. IPv4 only (see IPv6 note in §15/§18).
- `egress` — normal bridge with NAT. **Only** the proxy attaches here.

**Infrastructure services (profile `core` + `egress`)**

- `dns` (CoreDNS) — on `agents` only. Pinned resolver for agents; refuses
  external recursion; logs queries.
- `egress-proxy` (Squid) — on `agents` **and** `egress`. The single sanctioned
  exit. Allowlist + private-range deny + SNI peek-and-splice + access logging to
  the `audit` volume.
- `egress-test` (busybox/curl, profile `egress`/`devtools`) — disposable
  container on `agents`, used by `scripts/egress-test` to prove containment.

**Agent services (profiles `openclaw`, later `hermes`/`openhands`/`browser`)**

- Attach to `agents` only. `dns:` pinned to the CoreDNS service. `HTTP_PROXY` /
  `HTTPS_PROXY` / `NO_PROXY` set to the Squid service for *cooperating* tools.
  Hardened per §16.

**Volumes**

- Per-agent named volume mounted at `/workspace` (the only writable persistent
  path).
- `audit` named volume for Squid/CoreDNS logs, writable by infra services only;
  agents never mount it.

Key property: **the only path from an agent to the internet is
agent → Squid (allowlist+log) → internet.** Every other path is "no route" and
therefore fails closed.

---

## 6. ASCII network diagram

```text
                          host (localhost only; 127.0.0.1 binds)
                          ┌───────────────────────────────────────────┐
   internet               │                                           │
      ▲                   │   docker network: egress  (NAT, external) │
      │ NAT               │        ▲                                  │
      │                   │        │ eth1 (only proxy is here)        │
 ┌────┴───────────────────┼────────┴──────────────────────────────┐  │
 │  egress-proxy (Squid)  │                                        │  │
 │  - domain allowlist    │  eth0 ──┐                              │  │
 │  - deny RFC1918/meta   │         │                              │  │
 │  - deny raw-IP CONNECT │         │ docker network: agents       │  │
 │  - SNI peek-and-splice │         │ (internal: true — NO route   │  │
 │  - access.log → audit  │         │  off-bridge, fail closed)    │  │
 └─────────────────────────┘        │                              │  │
                                     │                              │  │
        ┌────────────────────────────┼───────────────┬──────────────┐ │
        │                            │               │              │ │
   ┌────┴─────┐                ┌─────┴─────┐    ┌─────┴──────┐  ┌────┴┴───┐
   │ dns       │                │ openclaw  │    │ egress-test│  │ (future │
   │ (CoreDNS) │◄───agent dns──│  agent    │    │ (busybox)  │  │  agents)│
   │ refuse    │   queries     │ ro-rootfs │    │  prove     │  │         │
   │ external  │                │ non-root  │    │  blocks    │  │         │
   └───────────┘                │ /workspace│    └────────────┘  └─────────┘
                                └───────────┘
                                   (no docker.sock, no host mounts,
                                    no LAN route, cap_drop ALL)

  Legend:
    ──► allowed mediated path (agent → Squid → internet, logged)
    Everything not drawn to "egress"/internet has no route and times out.
    DNS for external names is done BY Squid on the egress side, not by agents.
```

---

## 7. Egress-control options comparison

Criteria legend: **Enf** = enforces vs merely suggests; **IP** = blocks direct-IP
egress; **LAN** = blocks RFC1918/LAN; **Log** = logs attempts; **TLS** = HTTPS
handling without breaking pinning; **DNS** = resists DNS bypass; **Compose** =
Compose-friendliness; **WSL2** = WSL2/Crostini portability.

### 7.1 Forward proxy (Squid / Tinyproxy / Privoxy / mitmproxy / Envoy)

- **Squid** — mature, boring, inspectable. Native `dstdomain` allowlist from a
  file, `CONNECT`-method control, private-range ACLs, **peek-and-splice** to
  allowlist by **SNI without MITM/CA**, rich `access.log` (incl. `TCP_DENIED`).
  Arcane config syntax is the main cost. Enf✔ IP✔ LAN✔ Log✔ TLS✔(SNI) DNS✔(proxy
  resolves) Compose✔ WSL2✔. **Best v0 fit.**
- **Tinyproxy** — tiny, simple `Allow`/`Filter` config. Weaker, regex-ish
  allowlisting; less granular deny logging; no SNI peek. Fine for trivial cases,
  weaker as a real boundary. Enf~ IP~ LAN~ Log~ TLS~ Compose✔ WSL2✔.
- **Privoxy** — content/ad filtering focus, not an egress allowlist enforcer.
  Wrong tool.
- **mitmproxy** — excellent inspection and Python-scriptable logging, but to see
  HTTPS it must intercept TLS → **install a CA in agents = MITM by default**,
  breaks cert pinning, larger trusted component. **Reject for default; keep as
  optional `inspect` profile** for deliberate deep inspection.
- **Envoy** — powerful L7 egress with RBAC/ext_authz, but xDS/listener/cluster
  config is heavy for a single-dev v0. Overkill. Defer.

### 7.2 DNS policy (CoreDNS / Unbound / dnsmasq / Pi-hole)

- **CoreDNS** — small Corefile, plugins for `forward`, `hosts`, `template`,
  `log`. Can be configured to **refuse external recursion** and log queries.
  CNCF-maintained, container-native. **Best v0 DNS fit.**
- **dnsmasq** — very small, `--server`/`--address` allowlisting; fine, but
  CoreDNS's plugin/log model is cleaner to reason about.
- **Unbound** — strong validating recursive resolver; more about DNSSEC than
  allowlisting. Heavier than needed.
- **Pi-hole** — UI/blocklist oriented; too heavy for v0.
- DNS-only is **not** sufficient alone: it controls name→IP but cannot stop
  direct-IP egress. It is a *layer*, not the boundary.

### 7.3 Linux firewall gateway (nftables/iptables container, or host `DOCKER-USER`)

- Strongest **protocol-agnostic** L3/L4 enforcement: drops by CIDR, blocks
  direct IP, blocks RFC1918, logs via `nflog`. Enf✔ IP✔ LAN✔ Log✔.
- **Compose friction:** Docker assigns a network's gateway to the *host* bridge,
  not to a container, so routing agent traffic *through* a firewall container
  requires either `network_mode: "service:gateway"` namespace-sharing (awkward
  for many agents) or host-level `DOCKER-USER` rules (root, host-specific).
- **WSL2/Crostini:** host `DOCKER-USER` rules live inside the Docker Desktop /
  Termina VM and are not cleanly portable. TLS✖ DNS✖ (it is L3/L4 only).
- **Verdict:** keep as **documented optional hardening (Layer 4)**, not the v0
  default, because the `internal` network already fails closed without host
  changes. It is the right way to *also log raw-socket bypass attempts* (see
  §17 gap note).

### 7.4 Policy engine (OPA + proxy/gateway)

- OPA via Squid ICAP or Envoy ext_authz adds Rego policy. For a *flat domain
  allowlist*, a text file is clearer, more inspectable, and easier to review.
  **Defer** until policy genuinely outgrows a list (e.g., per-agent rules,
  time-of-day, method-aware decisions).

### 7.5 Heavier (Cilium / mesh / K8s NetworkPolicy / OpenZiti)

- Cilium needs its own CNI/eBPF data plane; service mesh and K8s NetworkPolicy
  assume Kubernetes; OpenZiti is a zero-trust overlay requiring identities and
  controllers. All are disproportionate to a single-host Compose lab. **Do not
  recommend for v0.** Revisit only if the project pivots to multi-host/managed.

### 7.6 Summary

| Option | Enf | IP | LAN | Log | TLS(no MITM) | DNS | Compose | WSL2 | v0? |
|---|---|---|---|---|---|---|---|---|---|
| **Squid proxy** | ✔ | ✔ | ✔ | ✔ | ✔ (SNI peek) | ✔ | ✔ | ✔ | **Yes** |
| Tinyproxy | ~ | ~ | ~ | ~ | ~ | ~ | ✔ | ✔ | No |
| mitmproxy | ✔ | ✔ | ✔ | ✔ | ✖ (needs CA) | ✔ | ✔ | ✔ | Optional profile |
| Envoy | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ~ | ✔ | Defer |
| **CoreDNS** | (layer) | ✖ | ✖ | ✔ | n/a | ✔ | ✔ | ✔ | **Yes (DNS layer)** |
| nftables/DOCKER-USER | ✔ | ✔ | ✔ | ✔ | n/a | ✖ | ~ | ~ | Optional hardening |
| OPA+proxy | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ~ | ✔ | Defer |
| Cilium/mesh/K8s/Ziti | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ | ~ | No |
| **`internal:true` net** | ✔ | ✔ | ✔ | ✖ (silent) | n/a | ✔ | ✔ | ✔ | **Yes (substrate)** |

---

## 8. Selected egress design and rationale

**Selected stack: `internal` agent network (substrate) + Squid (allowlist proxy)
+ CoreDNS (controlled resolver), with host nftables documented as optional
Layer 4.**

Enforcement chain and *why each layer earns its place*:

1. **`internal: true` agent network — the real boundary.** Docker adds no NAT
   and no off-bridge route for an internal network, so direct internet/LAN/host
   access **fails by construction**, independent of agent behavior. This is what
   makes proxy env vars safe to rely on for *function* while not relying on them
   for *security*.

2. **Squid as the single dual-homed exit.** Only Squid touches both networks.
   - Allowlist by domain from `policies/egress.allowlist` (`dstdomain` with
     leading-dot subdomain matching), default `http_access deny all`.
   - `http_access deny CONNECT !SSL_ports` and an explicit private-range ACL
     (`10/8, 172.16/12, 192.168/16, 169.254/16, 127/8, ::1`) → blocks LAN, host,
     and **cloud metadata 169.254.169.254**.
   - Raw-IP `CONNECT` fails the domain allowlist (an IP literal matches no
     domain) → denied by default.
   - **Peek-and-splice on SNI:** Squid peeks at the TLS ClientHello SNI and
     *splices* (forwards without decryption) only if the SNI is allowlisted.
     This enforces the HTTPS allowlist **without a CA and without breaking cert
     pinning**, and closes the "CONNECT allowed-host then negotiate evil SNI"
     gap (T6).
   - `cache deny all` (don't retain agent traffic); `access.log` to the `audit`
     volume records allowed **and** denied attempts (`TCP_DENIED/403`).

3. **CoreDNS as the only resolver agents get.** Agent services set
   `dns: [<coredns-ip>]`. CoreDNS **refuses external recursion** (answers
   internal/service needs, returns `REFUSED`/`NXDOMAIN` otherwise) and logs
   queries. This matters because Docker's embedded resolver (`127.0.0.11`)
   *forwards* unknown names to the host's resolvers by default — a DNS
   exfiltration channel that exists *even on an internal network*. Pinning
   agent `dns:` to CoreDNS removes that channel (T5). External names for
   allowlisted sites are resolved by **Squid**, on the egress side, so agents
   never receive external A records and never choose the IP.

4. **Fail-closed by construction.** If Squid is down, agents have no exit (no
   route). If CoreDNS is down, external names don't resolve. Both are tested in
   §17. There is no "controls unavailable ⇒ open" state.

5. **Host nftables `DOCKER-USER` (optional Layer 4).** Documented for Linux
   operators who want to *also log* raw-socket bypass attempts and add a second
   independent L3/L4 deny. Not required for first boot; explicitly noted as
   non-portable to WSL2/Crostini.

**Honest gap (stated, not hidden):** v0 logs every *proxy-mediated* attempt.
A raw-socket attempt straight at the internet is *blocked* by the internal
network but **not logged** unless the optional nftables layer is enabled,
because "no route" drops are silent. This is the one place where "blocked" and
"observed" diverge in the default build, and it is called out again in §17/§18.

---

## 9. Minimal v0 service set

Keep v0 tiny. Default boot = safe substrate; agents/egress are additive.

| Service | Profile | Network(s) | Role | In v0? |
|---|---|---|---|
| `dns` (CoreDNS) | `core` | `agents` | controlled resolver, refuses external recursion, logs | **Yes** |
| `egress-proxy` (Squid) | `egress` | `agents`+`egress` | allowlist proxy, SNI splice, deny-private, access log | **Yes** |
| `egress-test` (busybox/curl) | `egress`/`devtools` | `agents` | disposable container that proves containment | **Yes** |
| `openclaw` (custom image) | `openclaw` | `agents` | first hardened agent profile | **Yes (target)** |

Volumes in v0: per-agent `openclaw_workspace` (→ `/workspace`), `audit`
(infra-only logs). No databases in v0.

Mode mapping (matches `AGENTS.md` profile names):
- **`no-internet` mode** = `core` only (CoreDNS, no proxy) → nothing can exit.
- **`allowlisted-egress` mode** = `core` + `egress` (proxy present) → mediated
  exit exists.

---

## 10. Optional profiles and later services

| Profile | Components | When |
|---|---|---|
| `hermes` | Hermes agent image/profile | after `openclaw` pattern proven |
| `openhands` | OpenHands runtime profile | later; verify its own sandbox assumptions don't conflict |
| `browser` | headless browser automation (Playwright/Chromium) | later; **browser escape is a distinct risk** (§18) |
| `local-llm` | Ollama (on `agents`, no egress needed; or its own internal net) | optional; keeps inference local, reduces egress need |
| `cloud-llm` | LiteLLM gateway → allowlisted provider domains via Squid | optional; centralizes provider creds, single allowlist entry |
| `ui` | Open WebUI / operator console, **`127.0.0.1` only** | optional; never public |
| `observability` | Dozzle or a log viewer for `audit` (localhost only) | optional; surfaces Squid denials to the operator |
| `inspect` | mitmproxy with explicit CA (deliberate MITM) | optional; deep inspection only, opt-in |
| `danger-zone` | intentionally relaxed configs, loudly fenced | last; for explicit experiments only |

Deferred unless a profile needs them: Postgres, Redis, MinIO, Qdrant. Add only
with a documented consumer; never as a shared "data plane" agents attach to.

---

## 11. Custom OpenClaw image plan

> Caveat: OpenClaw's exact upstream packaging/runtime must be confirmed before
> implementation (see §20, Q1). The plan below is the hardening *shape*; the
> concrete base/entry will follow once upstream is pinned. Do **not** invent
> commands for a runtime we haven't verified.

- **Base image strategy.** Prefer a slim, well-known base (`debian:bookworm-slim`
  or a `-slim`/`distroless`-style runtime if OpenClaw permits). Multi-stage:
  build deps in a builder stage, copy only artifacts into a minimal runtime
  stage.
- **Pinning.** Pin base by **tag + digest** (`@sha256:...`). Pin language/tool
  versions. No `latest`/`main` outside a clearly marked experimental tag.
- **Non-root user.** Create `app` (`10001:10001`); `USER` set; never run as root
  at runtime. Compose also sets `user:` defensively.
- **Minimal packages.** Only runtime deps. No compilers, no `curl`/`wget` in the
  final image unless required (reduces exfil tooling). Document anything kept.
- **Shells/tools.** Remove or avoid shells/package managers where the runtime
  doesn't need them. If a shell is unavoidable, document why.
- **Read-only root filesystem.** Target `read_only: true` in Compose. Identify
  every path the runtime writes and redirect to explicit mounts.
- **Writable directories.** `/workspace` (named volume, persistent), `/tmp`
  (tmpfs), and a narrow runtime/cache dir as tmpfs or a small named volume.
  Nothing else writable.
- **Entrypoint design.** Tiny `entrypoint.sh`: validate required env (e.g.,
  proxy + DNS reachable), drop to non-root if needed, `exec` the agent (PID 1
  signal handling). No secret material echoed. Fail fast and loud on missing
  config.
- **Healthcheck.** A cheap liveness check (process/port). Compose `healthcheck`
  with sane interval/retries; do not claim readiness the check can't prove.
- **Egress/proxy/DNS config.** `HTTP_PROXY`/`HTTPS_PROXY`=Squid,
  `NO_PROXY`=internal services; `dns:` pinned to CoreDNS in Compose. Image
  documents that it assumes **no direct internet**.
- **Secrets/config injection.** **File-mounted, read-only**, per-agent (Docker
  secrets or `:ro` binds from gitignored `secrets/`). **Not** via env vars
  (env leaks through `docker inspect`/`/proc`). Config (non-secret) via
  `.env.local` interpolation.
- **Volume layout.** `/workspace` (rw, persistent, per-agent), `/tmp` (tmpfs),
  `/run` (tmpfs) if needed, secrets at `/run/secrets/<name>` (ro).
- **Labels/metadata.** OCI labels: `org.opencontainers.image.source`,
  `revision`, `created`; plus `agent-lab.profile=openclaw`,
  `agent-lab.network=agents`, `agent-lab.egress=mediated`.
- **Update strategy.** Re-pin digest deliberately; review upstream diff; rebuild
  reproducibly; bump the pinned tag in one reviewable patch. No silent
  `latest`.
- **Security review checklist (per build):**
  - [ ] runs non-root; `id` shows uid ≠ 0
  - [ ] `read_only` rootfs holds; writes only to declared mounts
  - [ ] `cap_drop: ALL`, `no-new-privileges`, `pids_limit`, mem/cpu limits set
  - [ ] no `docker.sock`, no host mounts, no host networking
  - [ ] on `agents` network only; `dns:` = CoreDNS; proxy env set
  - [ ] no shell/pkg manager unless justified in README
  - [ ] secrets only as `:ro` file mounts, never baked in, never in env
  - [ ] base pinned by digest; SBOM/`docker history` reviewed
  - [ ] passes the §17 egress acceptance tests

Deliverables: `images/openclaw/Dockerfile`, optional
`images/openclaw/entrypoint.sh`, `images/openclaw/README.md` (hardening
assumptions), `profiles/openclaw.compose.yaml`, env-example updates only.

---

## 12. Repo structure

```text
agent-lab/
  AGENTS.md                      # source of truth (exists)
  CLAUDE.md                      # Claude operating notes (exists)
  README.md                      # purpose, default-deny posture, first-run warning
  SECURITY.md                    # secret handling + vuln reporting
  THREAT_MODEL.md                # untrusted-agent assumptions (expands §3)
  PLAN.md                        # this plan
  LICENSE                        # exists
  .gitignore                     # exists (covers env/secrets/runtime)
  .env.example                   # NON-secret config placeholders (committed)

  compose.yaml                   # base: networks + core (dns) + volumes
  compose.egress.yaml            # egress-proxy (Squid) + egress-test
  profiles/
    openclaw.compose.yaml
    hermes.compose.yaml          # later
    openhands.compose.yaml       # later
    browser.compose.yaml         # later
    local-llm.compose.yaml       # later
    cloud-llm.compose.yaml       # later
    observability.compose.yaml   # later
    danger-zone.compose.yaml     # last, fenced

  gateway/
    squid/squid.conf             # allowlist + deny-private + SNI splice + logging
    squid/README.md
  dns/
    coredns/Corefile             # refuse external recursion + logging
    coredns/README.md
  policies/
    egress.allowlist.example     # domains, one per line (committed example)
    lan.denylist.example         # RFC1918/link-local/meta ranges (committed)
  images/
    openclaw/Dockerfile
    openclaw/entrypoint.sh
    openclaw/README.md
  env/
    base.env.example             # if a layered env split is wanted
  scripts/
    up                           # compose up with profiles + --env-file
    down                         # compose down
    doctor                       # non-destructive preflight checks
    egress-test                  # runs tests/egress against the lab
  tests/
    egress/
      README.md                  # expected pass/fail per case
      cases.sh                   # the acceptance checks (§17)
  docs/
    architecture.md
    hardening-host-firewall.md   # optional nftables/DOCKER-USER Layer 4
  .codex/ .claude/               # agent prompts/skills (exist)
```

Create directories **as milestones reach them**, not all up front (per
`AGENTS.md`: shape is guidance, not license to scaffold prematurely).

---

## 13. Install / user workflow

```bash
git clone <repo>
cd agent-lab
cp .env.example .env.local        # fill NON-secret config; .env.local is gitignored
# put any real credentials as files under secrets/ (gitignored), never in .env.local
./scripts/doctor                  # preflight: docker present, no stray .env, perms, config valid
./scripts/up core                 # CoreDNS + networks + volumes (no internet path yet)
./scripts/up egress               # add Squid (mediated exit) — i.e. allowlisted-egress mode
./scripts/up openclaw             # start the hardened agent
./scripts/egress-test             # prove containment (see §17)
./scripts/down                    # tear down
```

`scripts/up` is additive: `up core` then `up egress` then `up openclaw`
composes profiles together (`docker compose --profile ...`). Running `core`
alone is the `no-internet` mode.

`doctor` (non-destructive) checks: `docker compose version` is v2; `.env.local`
exists; **warns if a stray `.env` exists** (it would auto-load and surprise
interpolation); `docker compose config` parses; expected ports bind only to
`127.0.0.1`; `secrets/` is gitignored and not tracked.

---

## 14. Environment and secrets policy

This section resolves the Compose env confusion explicitly.

**Two distinct kinds of values, handled differently:**

1. **Non-secret config** (ports, image tags, proxy URL, allowlist path) →
   `.env.local` (copied from committed `.env.example`, gitignored).
2. **Real secrets** (API keys, tokens) → **files under `secrets/`** (gitignored),
   mounted **read-only per service**. Never in `.env.local`, never in
   `environment:` (env leaks via `docker inspect` and `/proc/<pid>/environ`).

**How `.env.local` actually reaches Compose (the precise part):**

- Docker Compose auto-loads a file named exactly **`.env`** for `${VAR}`
  interpolation in the Compose files. It does **not** auto-load `.env.local`.
- Therefore scripts pass it **explicitly**:
  `docker compose --env-file .env.local -f compose.yaml ...`. That file then
  drives `${VAR}` interpolation.
- For values that must exist **inside** a container, reference them in the
  service's `environment:` via interpolation (`HTTP_PROXY: ${HTTP_PROXY}`), which
  pulls from the `--env-file`. Do **not** rely on the magic `.env`.
- Because a stray `.env` would auto-load and silently change interpolation,
  `doctor` warns if one exists. We deliberately do **not** ship a committed
  `.env`.

**Result:** one file (`.env.local`) for non-secret config, used by scripts via
`--env-file` and surfaced into containers via explicit interpolation; secrets
stay out of env entirely. `.gitignore` already covers `.env`, `.env.*`
(except `*.example`), `*.local`, `secrets/`, and key material.

---

## 15. Compose / network design details

Illustrative only — validate with `docker compose config`.

```yaml
# compose.yaml (base / core)
networks:
  agents:
    internal: true          # no off-bridge route → structural default-deny
    # IPv4 only; do NOT enable IPv6 here (see §18 IPv6 note)
  egress:
    driver: bridge          # NAT to internet; ONLY the proxy attaches

volumes:
  audit:                    # infra logs; agents never mount this
  openclaw_workspace:       # per-agent writable state

services:
  dns:
    image: coredns/coredns@sha256:<pinned>
    profiles: ["core"]
    networks: [agents]
    command: ["-conf", "/etc/coredns/Corefile"]
    volumes:
      - ./dns/coredns/Corefile:/etc/coredns/Corefile:ro
      - audit:/var/log/coredns
    read_only: true
    security_opt: ["no-new-privileges:true"]
    cap_drop: ["ALL"]
    pids_limit: 128
    restart: unless-stopped
```

```yaml
# compose.egress.yaml
services:
  egress-proxy:
    image: ubuntu/squid@sha256:<pinned>     # pin an audited Squid build
    profiles: ["egress"]
    networks: [agents, egress]              # the ONLY dual-homed service
    volumes:
      - ./gateway/squid/squid.conf:/etc/squid/squid.conf:ro
      - ./policies/egress.allowlist:/etc/squid/allowlist.txt:ro
      - audit:/var/log/squid
    read_only: true
    tmpfs: ["/var/spool/squid", "/run"]
    security_opt: ["no-new-privileges:true"]
    cap_drop: ["ALL"]
    cap_add: ["SETUID", "SETGID"]           # only if Squid must drop privs; justify
    pids_limit: 256
    healthcheck:
      test: ["CMD", "squid", "-k", "check"]
      interval: 30s
      retries: 3
    restart: unless-stopped

  egress-test:
    image: curlimages/curl@sha256:<pinned>
    profiles: ["egress", "devtools"]
    networks: [agents]
    dns: ["<coredns-service-ip-or-alias>"]
    environment:
      HTTP_PROXY: ${HTTP_PROXY}
      HTTPS_PROXY: ${HTTPS_PROXY}
      NO_PROXY: ${NO_PROXY}
    read_only: true
    security_opt: ["no-new-privileges:true"]
    cap_drop: ["ALL"]
    entrypoint: ["sleep", "infinity"]       # driven by scripts/egress-test
```

```yaml
# profiles/openclaw.compose.yaml
services:
  openclaw:
    image: agent-lab/openclaw@sha256:<pinned>   # built from images/openclaw
    profiles: ["openclaw"]
    networks: [agents]                          # NEVER egress
    dns: ["<coredns>"]
    environment:
      HTTP_PROXY: ${HTTP_PROXY}
      HTTPS_PROXY: ${HTTPS_PROXY}
      NO_PROXY: ${NO_PROXY}
    volumes:
      - openclaw_workspace:/workspace
      # secrets ONLY as ro file mounts, per-agent, e.g.:
      # - ./secrets/openclaw.key:/run/secrets/openclaw.key:ro
    user: "10001:10001"
    read_only: true
    tmpfs: ["/tmp", "/run"]
    security_opt: ["no-new-privileges:true"]
    cap_drop: ["ALL"]
    pids_limit: 256
    mem_limit: 2g
    cpus: 2.0
    restart: "no"
```

Design notes:
- Ports: none exposed in v0. Any operator UI later uses `127.0.0.1:${PORT}:...`.
- `network_mode: host` is forbidden.
- Squid resolves allowlisted names itself (trusted egress component); agents get
  only CoreDNS, which refuses external recursion.
- `cap_add` on Squid is the *only* capability exception, and only if the chosen
  image needs it to drop privileges; document or remove.

---

## 16. Hardening checklist

Per agent container (target defaults from `AGENTS.md`):

- [ ] `networks: [agents]` only; never on `egress`; never `network_mode: host`
- [ ] `dns:` pinned to CoreDNS; no `extra_hosts: host.docker.internal`
- [ ] `user:` non-root (e.g. `10001:10001`)
- [ ] `read_only: true`; writable only via `/workspace` volume + `/tmp` tmpfs
- [ ] `security_opt: ["no-new-privileges:true"]`
- [ ] `cap_drop: ["ALL"]`; add caps only with written justification
- [ ] `pids_limit`, `mem_limit`, `cpus`, ulimits set (anti-DoS, T12)
- [ ] no `/var/run/docker.sock`; no host bind mounts; no `$HOME`/SSH/drive mounts
- [ ] secrets only as per-service `:ro` file mounts; none in env; none committed
- [ ] images pinned by digest; non-pinned only in clearly-marked experimental
- [ ] proxy env set for cooperating tools (function), not relied on for security

Infra (Squid/CoreDNS): `read_only` + tmpfs, `cap_drop: ALL`, `no-new-privileges`,
logs to `audit` volume, pinned images, healthchecks.

Host (optional, documented in `docs/hardening-host-firewall.md`): `DOCKER-USER`
nftables drop+log for the `agents` subnet as Layer 4; note WSL2/Crostini
non-portability.

---

## 17. Egress test plan

Run from `egress-test` (and later each agent) on the `agents` network, so it has
the exact constraints of a real agent. `scripts/egress-test` executes these and
asserts pass/fail; `tests/egress/README.md` documents expected output.

| # | Requirement | Check (illustrative) | Expected |
|---|---|---|---|
| 1 | No internet without egress | `up core` only; `curl -m5 https://example.com` | fail/timeout (no route) |
| 2 | No private/LAN reach | `curl -m5 http://<a-LAN-ip>`; `curl -m5 https://169.254.169.254` | fail (no route) |
| 3 | Allowed domain works (mediated) | `up egress`; `curl -m5 --proxy $HTTPS_PROXY https://<allowed>` | 200 |
| 4 | Non-allowed domain blocked | `curl -m5 --proxy $HTTPS_PROXY https://<not-allowed>` | 403 TCP_DENIED |
| 5 | Direct-IP egress blocked | `curl -m5 --proxy $HTTPS_PROXY https://1.1.1.1` | denied (no domain match) |
| 6 | Private via proxy blocked | `curl -m5 --proxy $HTTPS_PROXY http://10.0.0.1` | denied (private ACL) |
| 7 | SNI mismatch blocked | CONNECT allowed-host, TLS SNI = other | spliced-denied (peek) |
| 8 | DNS bypass blocked | `dig @1.1.1.1 example.com`; `dig example.com` (CoreDNS) | no route; REFUSED/NXDOMAIN |
| 9 | No Docker socket | `test ! -e /var/run/docker.sock` | pass |
| 10 | No unmounted secrets | enumerate mounts; only own `:ro` secret present | pass |
| 11 | Writable state bounded | `touch /etc/x` fail; `touch /workspace/x` ok | ro rootfs holds |
| 12 | Attempts logged | after #4, grep `audit` Squid log for `TCP_DENIED` | present |
| 13 | Fails closed | `docker compose stop egress-proxy`; redo #3 | fail |

**Documented gap (re-stated):** cases 1/2/5/8 are *blocked* by the internal
network but a pure raw-socket attempt is **not logged** in the default build
(silent no-route drop). Enable the optional host nftables Layer 4 to also *log*
those attempts. Tests assert the *block*; they do not claim the *log* for raw
non-proxy attempts unless Layer 4 is on.

---

## 18. Known limitations and residual risks

This is **practical containment, not a perfect sandbox.** Explicit residuals:

- **Container/kernel escape (T13).** Agents share the host kernel. A kernel or
  Docker-runtime 0-day defeats every control here. Mitigation requires VM-grade
  isolation (gVisor, Kata, microVM, or a throwaway VM) — out of scope for v0,
  noted as the ceiling.
- **Browser escape.** A headless browser (future `browser` profile) is a large
  native attack surface (renderer/GPU sandbox escapes). Treat browser containers
  as *more* hostile; consider tighter seccomp and a dedicated network.
- **Allowlisted-destination exfil.** Any allowed domain can receive exfiltrated
  data. Keep the allowlist minimal and reviewed; allowlisting bounds *where*,
  not *what*.
- **Raw bypass attempts unlogged by default.** Blocked but silent without
  Layer 4 (see §8/§17).
- **IPv6 leak.** If Docker IPv6 is enabled, IPv6 can bypass IPv4-only rules.
  Keep the `agents` network IPv4-only/internal; if IPv6 is enabled host-wide,
  re-audit and extend deny rules. Listed as an open item.
- **DNS-over-HTTPS via an allowed host.** If a DoH endpoint is allowlisted, it
  reintroduces a resolver. Don't allowlist public DoH providers.
- **Privileged/`danger-zone` profiles.** Any relaxation (added caps, host mount,
  socket, direct egress) voids these guarantees by design and must be loudly
  fenced and never default.
- **Squid/CoreDNS bugs.** The proxy/resolver are trusted components; a flaw in
  them is in the trust base. Pin, update deliberately, keep configs minimal.
- **WSL2/Crostini divergence.** Host-firewall hardening is non-portable; the
  "host" the container sees is a VM; localhost forwarding semantics differ.
  v0 enforcement deliberately lives *inside Compose* to stay portable.
- **Supply chain.** Mitigated by digest pinning + review, not eliminated.

If any **hard stop** from `AGENTS.md` is discovered during implementation
(real secret committed, socket exposed, public port, agent with creds + open
egress, host-home mount), stop and report before continuing.

---

## 19. Milestone plan

Small, reviewable patches for Codex (one boundary concern per patch):

- **M0 — Docs & skeleton.** `README.md`, `SECURITY.md`, `THREAT_MODEL.md`,
  `.env.example`, `policies/*.example`. No runtime behavior. (Partly seeded by
  existing `.codex` bootstrap prompt.)
- **M1 — Substrate.** `compose.yaml` with `agents`(internal)+`egress` networks,
  `audit` volume, CoreDNS `dns` service + `dns/coredns/Corefile` (refuse external
  recursion), `scripts/doctor`, `scripts/up`/`down`. Validate `compose config`.
- **M2 — Mediated egress.** `compose.egress.yaml` with Squid + `squid.conf`
  (allowlist, deny-private, deny raw-IP CONNECT, SNI peek-and-splice, logging) +
  `egress-test` container + `tests/egress/cases.sh` + `scripts/egress-test`.
  Prove cases 1–6, 8–13.
- **M3 — Harden + prove SNI.** Apply §16 to infra/test; add case 7
  (SNI mismatch). Confirm fail-closed (case 13).
- **M4 — OpenClaw image/profile.** `images/openclaw/*` + `profiles/openclaw.compose.yaml`;
  run the §11 checklist and §17 tests against the real agent.
- **M5 — Optional profiles + host-firewall doc.** `local-llm`/`cloud-llm`/`ui`/
  `observability` as needed; `docs/hardening-host-firewall.md` (nftables Layer 4).
- **M6 — Polish & fences.** Residual-risk doc pass; `danger-zone` profile clearly
  fenced; README accuracy audit (no overclaiming).

Each milestone ends with: `docker compose config`, `git diff --check`, the
relevant §17 cases, and a report in the `AGENTS.md` format (inspected / changed /
commands+results / boundary impact / residual risks).

---

## 20. Open questions for the project owner

1. **OpenClaw upstream.** What is the canonical OpenClaw source/image and run
   command? Needed to finalize §11 (base, entrypoint, ports, writable paths).
2. **Host priority.** Is the primary target pure Linux, WSL2, or Crostini? This
   sets how much weight the optional host-firewall layer carries.
3. **Inspection appetite.** Is an opt-in `inspect` (mitmproxy + CA, real MITM)
   profile wanted, or is SNI-only allowlisting (no decryption) sufficient?
4. **Local vs cloud models.** Default to local `Ollama` (no egress), or expect
   cloud providers via a `cloud-llm`/LiteLLM allowlist from day one?
5. **Multi-agent scale.** One agent at a time, or several concurrently? Affects
   per-agent network/volume/secret layout and naming.
6. **Rootless Docker.** Should v0 support/recommend rootless Docker (reduces
   escape blast radius) as a documented option?
7. **IPv6.** Is Docker IPv6 enabled on the target host? If so, deny rules must be
   extended (see §18).
8. **Allowlist seed.** What initial domains belong on the egress allowlist
   (model/provider APIs, package registries)? Keep it minimal.
9. **Image registry.** Where do custom images live (local build only, or a
   private registry)? Affects pinning/update flow.
10. **Log retention.** How long should `audit` (Squid/CoreDNS) logs persist, and
    should the `observability` profile surface them by default?
