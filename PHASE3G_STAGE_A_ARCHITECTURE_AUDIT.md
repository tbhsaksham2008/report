# Phase 3G — Stage A Architecture and Remediation Audit

> Historical baseline: this audit inventory was captured before Stage B. See the
> Stage B/C addendum at the end for the current implemented state and validation.

## Status

**Stage A complete. No implementation code was changed during this audit.**
The only new file is this audit record. Stage B work must not interpret a planned or
documented capability as implemented enforcement.

## Scope and evidence

Read all nine local Phase 3F.5 reports:

- `PHASE3F5_ATTACHMENT_SAFETY_REPORT.md`
- `PHASE3F5_KERNEL_VALIDATION_REPORT.md`
- `PHASE3F5_MAP_RECOVERY_REPORT.md`
- `PHASE3F5_NETWORK_INTEGRATION_REPORT.md`
- `PHASE3F5_PACKET_BEHAVIOR_REPORT.md`
- `PHASE3F5_REMEDIATION_AND_LIMITATIONS_REPORT.md`
- `PHASE3F5_SYN_PRESSURE_REPORT.md`
- `PHASE3F5_SYSTEMD_REPORT.md`
- `PHASE3F5_TEST_AND_RESOURCE_REPORT.md`

Inspected the unified Go module, XDP C program and loader, Gateway, Minecraft parser,
PROXY parser, reputation manager, edge agent, command entry points, systemd units,
build file, tests, root `SOURCE/` scaffold, project architecture/readme, and existing
Phase 3F architecture/security reports.

The root project and `unified/` are not Git repositories. `Australis/` is a Git
repository with pre-existing modified and untracked files; it was not modified.
`Minecraft-XDP-eBPF/` is a separate clean Git repository. This phase should limit
implementation to `unified/` unless a specific cross-repository change is justified
and separately reviewed.

## Current architecture

```text
Client
  -> XDP hook (separate privileged Go process; current packet policy is IPv4-centric)
  -> Linux TCP stack and stream reassembly
  -> unified-ingress TCP Gateway
       -> optional trusted-peer PROXY protocol parsing
       -> bounded connection/admission and event limits
       -> Minecraft handshake/status/login parser for its implemented version set
       -> configured single backend destination
  -> backend

Gateway --optional authenticated HTTP reputation lookup--> XDP/edge agent
Agent --authenticated mutations--> in-memory reputation store and IPv4 kernel maps
```

### Implemented modules

| Area | Existing implementation | Important boundary |
|---|---|---|
| XDP | `xdp/unified_xdp.c`, embedded BPF object, `xdp/loader.go` | IPv4 address-keyed block/reputation/SYN maps; no IPv6 policy; IPv4 fragments pass; XDP does not reassemble TCP or inspect Minecraft streams. |
| Attachment and map lifecycle | Linux loader with explicit native/generic/offload mode, pinned-map compatibility checks, occupied-interface refusal, `BPF_PROG_QUERY` plus fail-closed rtnetlink fallback | Direct isolated-veth attachment passed on WSL2. Systemd-restricted BPF load failed on WSL2; dedicated Linux service operation remains unverified. |
| Edge agent | `edge/agent.go` | Single bearer token for mutation and lookup endpoints; IPv4 kernel enforcement; no user accounts, roles, versioned control API or durable administrative policy service. |
| Gateway | `ingress/gateway.go`, `ingress/policy.go`, `ingress/inspect.go` | Bounded TCP connection/session/admission state, per-IP/prefix and global event rate limits, protocol forwarding; backend is a startup-configured address. |
| Minecraft | `minecraft/l7.go` | Bounded stream/frame parser and tested transitions for configured supported protocol versions; unsupported/negotiated stages may be rejected or passed opaquely according to configuration. Not full modern protocol inspection. |
| PROXY protocol | `proxy/` and Gateway configuration | Trusted peer CIDRs are checked before using the advertised identity; forwarding is explicit. |
| Reputation | `reputation/manager.go` | Bounded in-memory store and audit ring; state is not a durable, versioned policy database. Kernel pins persist only the maps the loader manages. |
| Services | `systemd/craftsmp-unified-xdp.service`, `systemd/craftsmp-unified-ingress.service` | Ingress requires XDP. Static unit validation passed; the XDP unit did not reach healthy operation in the WSL2 test. |
| Metrics | Gateway and XDP counters/health endpoints | Runtime counters exist, but there is no historical metrics store, incident service, or web dashboard. |
| Website/control plane | No application frontend/backend found | `SOURCE/` contains scaffold READMEs only. No `package.json`, app source, login/session implementation, role model, or control-plane database was found outside generated test/runtime content. |

## Fifteen-layer implementation map

“Partial” means some adjacent behavior exists, not that the layer meets the full Phase 3G
requirements. Nothing in this inventory is a claim of production readiness.

| # | Layer | Current evidence | Stage A assessment |
|---:|---|---|---|
| 01 | Network ACL and source admission | Gateway source/prefix admission; reputation allow/block state; XDP IPv4 source blocks | **PARTIAL** — no unified IPv4/IPv6 ACL policy manager or independently managed layer. |
| 02 | XDP ingress filtering | Explicit-mode XDP loader and packet program | **PARTIAL** — tested direct attach; WSL2 systemd startup failed; no layer toggle/policy control. |
| 03 | Packet structure validation | IPv4/TCP XDP bounds checks; bounded Minecraft stream parser | **PARTIAL** — IPv6 and UDP structure validation are absent in XDP; not every protocol path is validated. |
| 04 | IPv4 anti-spoof/source validation | Per-source accounting keyed by IPv4 address | **MISSING** — no upstream route/source-validation mechanism; spoofed source keys can consume rate-map capacity. |
| 05 | IPv6 validation/protection | Gateway supports IPv6 sockets/prefix grouping | **MISSING in XDP** — IPv6 packets bypass XDP filtering and address-keyed reputation. |
| 06 | Fragmentation protection | IPv4 fragment checks avoid unsafe TCP parsing | **MISSING** — fragments are passed; there is no bounded fragment correlation/reassembly policy. |
| 07 | SYN/handshake protection | Bounded IPv4 per-source SYN map; Gateway handshake limits | **PARTIAL** — no TCP handshake state in XDP, no IPv6 equivalent, and spoofed-source/map-pressure limitations remain. |
| 08 | Connection/resource exhaustion | Gateway connection/pending limits, cleanup, bounded application state | **PARTIAL** — tested in prior phases; operational XDP service and production-resource behavior not validated here. |
| 09 | TCP anomaly detection | Selected IPv4 TCP flag/data-offset checks | **PARTIAL** — no sequence-state anomaly detection; IPv6/fragment bypass remains. |
| 10 | Minecraft protocol validation | Stream/frame inspection for implemented protocol(s) | **PARTIAL** — narrow supported versions/stages; encrypted/compressed and later stages are opaque or unsupported. |
| 11 | Connection/handshake rate limits | Gateway per-IP/prefix/global status, ping, login, reconnect controls | **PARTIAL** — functional Gateway policy, but not independently versioned or orchestrated as a layer. |
| 12 | Reputation and temporary enforcement | Agent API, bounded store, expiry, kernel IPv4 block/reputation maps | **PARTIAL** — no IPv6 kernel enforcement, durable policy history, RBAC or complete control-plane audit. |
| 13 | IP/subnet policy | Gateway per-IP and prefix admission controls | **PARTIAL** — no common versioned IPv4/IPv6 prefix policy across dataplane and Gateway. |
| 14 | Adaptive anomaly detection | Related attack/reconnect logic exists in the separate Australis plugin | **MISSING as unified layer** — no central bounded explainable policy module, monitor-only control, or verified dataplane wiring. |
| 15 | Backend isolation/forwarding | Gateway forwards to configured backend; service runs unprivileged | **PARTIAL** — no backend destination allowlist/health policy in a control plane, no verified network isolation or live integrated path. |

## Confirmed critical gaps and dependencies

1. **IPv6 bypass:** current XDP program returns pass for non-IPv4 frames; reputation and
   block maps are keyed only by IPv4. Supporting IPv6 requires coordinated C parsing,
   new bounded maps, loader/API changes, expiry/recovery, and kernel tests.
2. **IPv4 fragmentation bypass:** fragmented packets are passed. A safe policy must
   decide how noninitial fragments are correlated without unbounded state, define
   exhaustion behavior, and document compatibility for fragmented traffic.
3. **Service startup:** WSL2 direct attach and `BPF_PROG_TEST_RUN` passed, but the
   restricted systemd context failed BPF loading with a verifier permission error.
   Do not add `CAP_SYS_ADMIN`; the intended dedicated Linux environment is not
   available in this workspace.
4. **End-to-end path:** the guarded WSL harness did not reach ingress or backend because
   the XDP service failed first. Existing unit/in-process tests are not live service
   evidence.
5. **No control plane:** there is no website/API for account-authenticated server,
   policy, gateway, incident, metrics, and audit management. The existing edge agent
   API is not an administrative control plane.
6. **No independent layer manager:** command-line startup configuration and code-level
   policy structs do not provide 15 separately versioned, health-aware modules,
   activation/rollback or truthful dependency state.
7. **Metrics/operations gap:** counters are not a durable telemetry or incident
   history; no production CPU/RSS or extended Phase 3G soak evidence exists.

## Baseline verification

Environment recorded by the Phase 3F.5 evidence: WSL2 Ubuntu, kernel
`6.18.58.1-microsoft-standard-WSL2`, Go `1.24.7 linux/amd64`, Intel Core i5-12450H
(12 logical CPUs), 3,828,500 kB visible RAM, and file-descriptor limit 10,240.
This is not equivalent to a dedicated Linux host.

Executed before implementation changes in this phase:

```text
cd unified
/usr/local/go/bin/go test -count=1 ./...
PASS: cmd/gateway-bench, cmd/unified-ingress, cmd/unified-xdp, edge, ingress,
      minecraft, proxy, reputation, xdp
```

Previously documented Phase 3F.5 results include passing race tests, vet, builds,
kernel test-run, isolated direct attachment, pinned-map recovery, and map-pressure
testing. The systemd service and end-to-end network path were not passing/verified.

## Ordered remediation plan

1. **Stage B — dataplane criticals:** establish explicit IPv6 and fragmentation
   semantics, implement bounded maps/state and expiry/recovery, add unit and kernel
   regressions, preserve fail-closed attachment safety. Separately investigate service
   capability behavior; do not broaden privilege to mask WSL2.
2. **Stage C — live integration:** extend the isolated harness and verify actual
   client-to-XDP-to-agent-to-Gateway-to-backend behavior on an environment that can
   run the service. In this WSL2 environment, mark unavailable paths NOT TESTED.
3. **Stage D — layer manager:** introduce typed, bounded per-layer policy interfaces,
   validation, effective/desired state, dependencies, health, versions and rollback.
   A layer is active only when enforcement is actually connected.
4. **Stage E — control-plane backend:** define versioned APIs, authentication,
   authorization, audit and durable bounded policy/telemetry state before exposing
   privileged management operations.
5. **Stage F — frontend:** build a responsive UI against implemented backend APIs;
   never add decorative toggles or direct browser-to-kernel access.
6. **Stage G — integration/security/reliability:** test authorization, rollback,
   restarts, bounded resources, packet and protocol behavior, and only the isolated
   network targets explicitly owned by the harness.
7. **Stage H — benchmark/readiness:** collect reproducible CPU/RSS, capacity,
   latency, map utilization and soak data on labeled hardware/kernel. Do not claim
   production readiness or comparison with commercial providers from WSL2 tests.

## Stage A verdict

**PASS — audit complete; implementation gaps and ordering are identified.**
The platform is not Phase 3G-ready. Proceeding requires foundational dataplane work;
the control-plane website cannot be truthfully built as an enforcement UI until the
policy manager and APIs exist.

## Stage B/C follow-up

The Stage A tables above remain the pre-implementation baseline. The following
changes supersede the corresponding IPv6, UDP, fragment, and live-integration
statements; they do not imply that all 15 layers or a production deployment are
complete.

### Implemented and verified

- XDP now parses IPv4 and IPv6, including up to two VLAN tags and bounded IPv6
  extension chains. It applies source block/reputation checks, per-source TCP SYN
  limits, and optional per-source UDP limits for the configured game port. UDP
  rate limiting has its own setting and defaults to disabled.
- IPv6 block, reputation, SYN-rate, and UDP-rate maps are separate from the
  established IPv4 maps so the existing pinned-map layouts remain compatible.
  A separate pinned UDP-configuration map preserves the existing TCP configuration
  map layout. Loader map selection, recovery, enforcement, lookup, unblocking, and
  expiry cover both address families.
- IPv4 fragments and IPv6 packets containing a Fragment header are explicitly
  dropped. No fragment reassembly or correlation is implemented. IPv6 extension
  chains are bounded to six headers and 256 bytes; unsupported/overlong chains are
  rejected. ICMPv6 receives minimum-length validation and is passed.
- `scripts/phase3g_wsl_e2e.sh` now exercises a real isolated generic-XDP veth path:
  client → XDP → Go ingress → mock status backend. IPv4 and IPv6 status requests
  succeeded; authenticated IPv4 and IPv6 block requests stopped the corresponding
  source, and unblock restored service. The harness removed its namespace, veth,
  processes, BPF pins, and temporary bpffs mount after completion.
- Full Go tests, race tests, vet, formatting check, generated BPF compilation, and
  all three Go command builds passed after these changes.

### Current boundaries

- Live integration was run in WSL2 by starting the XDP, ingress, and test backend
  processes directly as root with generic attachment. It does **not** validate the
  restricted systemd unit; prior Phase 3F.5 evidence records that unit's WSL2 BPF
  load failure. Dedicated-Linux service validation remains necessary.
- The live harness covers Minecraft protocol-47 status only. It does not demonstrate
  login success, modern protocol phases, or full stream/protocol security.
- UDP filtering is not Bedrock protocol inspection or a UDP relay. Bedrock/RakNet
  end-to-end handling remains unimplemented and untested. The optional UDP packet
  rate limit is disabled by default and has no Bedrock-aware traffic model.
- Fail-closed fragment behavior can drop legitimate fragmented traffic. IPv6 ESP
  and unsupported or over-limit extension paths do not receive Minecraft
  inspection. These compatibility effects require explicit operational testing.
- A versioned rate-policy manager, authenticated control-plane API, durable policy
  storage, and embedded dashboard now exist. Only TCP SYN and optional UDP rate
  settings are mutable there; the 15-layer inventory is descriptive, not a set of
  independent toggles. The service uses one shared bearer token and has no RBAC.
- Dedicated-Linux service validation, the full adversarial matrix, and dedicated
  Linux capacity/soak validation remain outstanding. No production-readiness claim
  is justified.

For exact commands, outputs, file inventory, and next actions, see
`PHASE3G_STAGE_BC_VALIDATION_REPORT.md` and
`PHASE3G_STAGE_DF_CONTROL_PLANE_REPORT.md`.
