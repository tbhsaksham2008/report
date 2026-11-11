# Phase 3F Security Audit

## Verdict

**PARTIAL.** A read-only audit of the implementation was followed by bounded-state fixes and an authenticated reputation lookup path. Go tests, race tests, vet, builds, and XDP C syntax validation passed. IPv6 and fragmented-packet XDP coverage remain explicit gaps; privileged kernel execution, interface attachment, and production traffic were not tested. No production-readiness claim is made.

## Audit plan and scope

The audit traced XDP parsing/maps and loader lifecycle; agent API authentication and state; PROXY trust; Gateway accept/admission/stream processing/backend relay; Minecraft protocol-state inspection; resource bounds and cleanup; and existing adversarial tests. The initial review was read-only. Source changes began only after that review identified unbounded reputation state and the missing Gateway-to-agent lookup contract.

Reviewed implementation: [XDP program](./xdp/unified_xdp.c), [loader](./xdp/loader.go), [edge agent](./edge/agent.go), [reputation store](./reputation/manager.go), [Gateway](./ingress/gateway.go), [protocol inspection](./ingress/inspect.go), [Minecraft parser](./minecraft/l7.go), [PROXY parser](./proxy/proxy.go), and their tests.

## Findings

### F-01 — Unbounded reputation records and audit history

- **Severity:** MEDIUM
- **Affected files:** [manager.go](./reputation/manager.go), [agent.go](./edge/agent.go)
- **Technical explanation:** The initial snapshot had unbounded Go maps for blocks, scores, and allowlists, and an append-only audit slice. Authenticated mutations could retain state indefinitely or grow it without limit.
- **Reproduction:** In the initial implementation, repeatedly call `Block`, `Allow`, or `SetScore` with distinct valid addresses; the maps and audit slice grow. Regression commands: `go test -count=1 -run TestStoreBounds ./reputation` and `go test -count=1 -run TestBlockRejectsAtStoreCapacity ./edge`.
- **Remediation status:** **FIXED.** The store has a shared 8,192-record cap, a 1,024-event chronological ring, a 128-byte reason bound, canonical bounded-length IP keys, and a seven-day maximum positive TTL. Writes return errors. Agent handlers check capacity before kernel mutations and surface capacity exhaustion as HTTP 507. Trusted PROXY CIDRs are deduplicated and capped at 256.

### F-02 — IPv6 bypasses XDP packet and SYN filtering

- **Severity:** MEDIUM
- **Affected files:** [unified_xdp.c](./xdp/unified_xdp.c), [gateway.go](./ingress/gateway.go)
- **Technical explanation:** The XDP program returns `XDP_PASS` for non-IPv4 Ethernet frames, including IPv6 and VLAN-tagged frames that still carry a VLAN EtherType at this hook. The TCP Gateway can listen on IPv6, so an IPv6 connection may receive L7 checks but does not receive XDP blocklist, reputation-map, malformed-TCP, or per-source SYN-rate enforcement.
- **Reproduction:** Inspect the non-IPv4 ethertype branch in `unified_xdp.c`; the existing IPv6 loopback test exercises Gateway handling only, not XDP. A real IPv6 XDP test was unavailable without privileged BPF access.
- **Remediation status:** **OPEN / NOT TESTED ON KERNEL.** Do not expose the game port over IPv6 if IPv4-only XDP enforcement is a requirement. Adding IPv6 maps/parser coverage is intentionally not attempted in this phase.

### F-03 — IPv4 fragments bypass XDP TCP validation and SYN rate counting

- **Severity:** MEDIUM
- **Affected files:** [unified_xdp.c](./xdp/unified_xdp.c), [linux_test.go](./xdp/linux_test.go)
- **Technical explanation:** After IPv4 bounds and source-map checks, fragmented TCP packets are passed because a later fragment need not contain a full TCP header. Therefore malformed-TCP and SYN-rate checks are not applied to fragmented traffic. The source-IP blocklist check occurs before this branch and still applies.
- **Reproduction:** The kernel test fixture constructs a fragmented TCP packet and expects `XDP_PASS`; running the kernel test requires root and was skipped here.
- **Remediation status:** **OPEN / NOT TESTED ON KERNEL.** No unsafe attempt to parse partial transport headers was made. A future choice between rejecting fragments for the game port and adding bounded fragment-aware handling needs compatibility tests.

### F-04 — Spoofed source addresses can consume per-source SYN state

- **Severity:** MEDIUM
- **Affected files:** [unified_xdp.c](./xdp/unified_xdp.c), [loader.go](./xdp/loader.go), [main.go](./cmd/unified-xdp/main.go)
- **Technical explanation:** SYN windows are keyed only by source IPv4 address in a fixed 131,072-entry hash map. The packet program does not validate the source address or complete a TCP handshake. A source-spoofing network can distribute SYN attempts across keys; if state allocation fails, the current program drops new SYNs, which is fail-closed but may reject legitimate new sources until pruning.
- **Reproduction:** Static path: inspect SYN map lookup/update in `unified_xdp.c`; the Go expiration loop prunes state every 30 seconds. Filling the real kernel map and measuring legitimate false positives was **NOT TESTED** because the environment is unprivileged WSL2.
- **Remediation status:** **OPEN.** Keep upstream source-validation/anti-spoofing and rate controls in scope. No map redesign or performance change was made without kernel measurements.

### F-05 — Out-of-range reputation thresholds could disable score decisions

- **Severity:** MEDIUM
- **Affected files:** [gateway.go](./ingress/gateway.go), [main.go](./cmd/unified-ingress/main.go)
- **Technical explanation:** The initial configuration accepted a threshold greater than the score domain (0–100). Since every score would be below that value, the Gateway would not reject even a score-100 result.
- **Reproduction:** `NewGateway` with `ReputationThreshold: 101`; the updated regression is part of `TestGatewayRejectsInvalidAdmissionConfiguration`.
- **Remediation status:** **FIXED.** Gateway configuration and CLI now reject thresholds above 100 before starting. Zero retains its existing default-threshold meaning.

### F-06 — L7 inspection coverage ends before encrypted/compressed and modern states

- **Severity:** MEDIUM (coverage limitation; not a parser memory-safety finding)
- **Affected files:** [inspect.go](./ingress/inspect.go), [l7.go](./minecraft/l7.go), [PHASE3C_PROTOCOL_SUPPORT.md](./PHASE3C_PROTOCOL_SUPPORT.md)
- **Technical explanation:** Protocol 47 has selected handshake, status, login, encryption-negotiation, compression, and login-success structure checks. Once encryption, compression, or login success triggers opaque mode, subsequent payloads are byte-preserved rather than inspected. Other protocol versions are unsupported unless opaque forwarding is explicitly enabled. Configuration and Play packet streams are not inspected.
- **Reproduction:** Run `go test -count=1 ./minecraft ./ingress`; the protocol tests cover supported parsing and opaque transitions.
- **Remediation status:** **OPEN / DOCUMENTED LIMITATION.** No claim of full Minecraft L7 protection is made.

## Other checks

- PROXY protocol is accepted only when enabled and the immediate TCP peer is in configured trusted CIDRs. Inbound identity is parsed before the effective source is used for limits; tests cover trusted and untrusted peers. A raw PROXY-looking prefix from an untrusted peer is not trusted as identity.
- BPF maps have fixed `max_entries`; Go Gateway state, parser sessions, admission/reputation caches, API request-rate state, and trusted CIDRs have explicit limits. The reputation-store finding above was the confirmed missing bound.
- Packet-boundary arithmetic checks were reviewed: IPv4 header/total lengths and TCP data offset are checked against packet bounds before TCP fields are used; Minecraft VarInts and frame sizes are separately bounded. No integer-overflow finding was confirmed in those paths. BPF verifier behavior was not tested here.
- The Gateway uses bounded handshake/frame reads, deadlines, pending/active limits, and closes active connections during shutdown. Existing tests cover incomplete/malformed frames, backend failure, and shutdown. A universal absence-of-leaks claim is not inferred.
- XDP program attachment refuses to replace an already-attached XDP program; loader and pinned-map recovery tests requiring root were skipped.
- No production VPS, public target, or public network traffic was accessed.

## Verification

PASS in WSL2, Ubuntu 24.04.5, Go 1.24.7 linux/amd64:

```bash
/usr/local/go/bin/gofmt -l reputation/manager.go reputation/manager_test.go edge/agent.go edge/agent_test.go ingress/reputation.go ingress/adversarial_test.go ingress/admission_hardening_test.go ingress/gateway.go ingress/gateway_test.go cmd/unified-ingress/main.go
/usr/local/go/bin/go test -count=1 -timeout=90s ./...
CGO_ENABLED=1 /usr/local/go/bin/go test -race -count=1 -timeout=120s ./...
/usr/local/go/bin/go vet ./...
/usr/local/go/bin/go build ./cmd/unified-ingress ./cmd/unified-xdp ./cmd/gateway-bench
clang -target bpf -D__TARGET_ARCH_x86 -I/usr/include/x86_64-linux-gnu -O2 -g -Wall -fsyntax-only xdp/unified_xdp.c
```

`TestRealKernelXDPFilter` and `TestPinnedMapRecovery` were **SKIP** because they require root. Interface attachment and real packet tests are **NOT TESTED**.
