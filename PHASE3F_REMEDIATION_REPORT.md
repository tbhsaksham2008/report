# Phase 3F Remediation Report

## Summary

**PARTIAL.** Confirmed in-memory state growth was bounded and surfaced as an explicit error. The disconnected Gateway reputation lookup contract is now wired through authenticated HTTP and a service dependency artifact. High-impact XDP coverage limitations remain open because IPv6, fragments, and real kernel behavior require separate compatibility and privileged testing.

## Implemented remediations

### R-01 — Bound reputation state

- **Finding:** Unbounded Store maps and append-only audit trail.
- **Affected files:** [manager.go](./reputation/manager.go), [agent.go](./edge/agent.go)
- **Change:** Shared cap of 8,192 records across block, score, allowlist, and blacklist maps; ring-buffer audit history capped at 1,024 events; IP keys canonicalized and length-bounded; reasons capped at 128 bytes; positive TTL capped at seven days. Store mutations now return errors. The agent serializes its map/store mutations, preflights capacity, and returns HTTP 507 on capacity exhaustion rather than silently accepting state loss.
- **Regression tests:** `TestStoreBoundsRecordsAndAuditTrail`, `TestStoreRejectsInvalidIPAndAllowsReplacementAtCapacity`, `TestBlockRejectsAtStoreCapacityBeforeKernelMutation`.
- **Status:** **PASS** in targeted, full, and race tests.

### R-02 — Bound trusted proxy configuration

- **Finding:** Programmatic `AddTrustedCIDR` calls could append duplicate/unbounded entries.
- **Affected files:** [agent.go](./edge/agent.go), [agent_test.go](./edge/agent_test.go)
- **Change:** Duplicate CIDRs are idempotent and the list is capped at 256 entries.
- **Regression test:** `TestTrustedProxyCIDRListIsBoundedAndDeduplicated`.
- **Status:** **PASS**.

### R-03 — Wire reputation state into L7 lookup

- **Finding:** Gateway’s HTTP provider did not send authentication and the XDP agent had no query endpoint; the two processes held separate in-memory Stores.
- **Affected files:** [agent.go](./edge/agent.go), [reputation.go](./ingress/reputation.go), [unified-ingress main](./cmd/unified-ingress/main.go), [systemd service](./systemd/craftsmp-unified-ingress.service), [integration test](./ingress/adversarial_test.go).
- **Change:** Added authenticated, bounded `POST /lookup`; agent returns the source score and represents an active manual block as 100. Gateway provider sends an optional bearer token; CLI accepts `CRAFTSMP_REPUTATION_TOKEN`. Added an ingress systemd unit requiring the XDP service. Unit activation is not part of this source change.
- **Regression test:** `TestGatewayReputationServiceIntegrationOverLiveTCP` verifies a real loopback TCP client is rejected by the Gateway before backend connection after an authenticated HTTP reputation lookup.
- **Status:** **PASS** for the test path; **NOT TESTED** for service startup, kernel attachment, public route, or actual host deployment.

### R-04 — Reject thresholds outside the score domain

- **Finding:** A configured reputation threshold above 100 could make every valid score compare below threshold and disable score-based rejection.
- **Affected files:** [gateway.go](./ingress/gateway.go), [unified-ingress main](./cmd/unified-ingress/main.go), [gateway_test.go](./ingress/gateway_test.go).
- **Change:** Gateway construction and CLI reject values above 100; zero continues to select the existing default.
- **Regression test:** `TestGatewayRejectsInvalidAdmissionConfiguration` includes threshold 101.
- **Status:** **PASS** in full tests.

## Open findings and required follow-up

| Finding | Severity | Affected files | Reproduction / evidence | Status |
|---|---|---|---|---|
| IPv6 bypass of XDP packet policy | MEDIUM | [unified_xdp.c](./xdp/unified_xdp.c), [gateway.go](./ingress/gateway.go) | XDP returns `XDP_PASS` for non-IPv4 EtherTypes; privileged IPv6 XDP execution unavailable | OPEN; avoid exposing IPv6 when XDP L4 coverage is required |
| IPv4 fragment bypass of TCP/SYN parsing | MEDIUM | [unified_xdp.c](./xdp/unified_xdp.c), [linux_test.go](./xdp/linux_test.go) | Fragment branch deliberately passes; privileged kernel test skipped | OPEN; define compatibility policy and implement bounded behavior with kernel regression tests |
| Source spoofing/map saturation | MEDIUM | [unified_xdp.c](./xdp/unified_xdp.c), [loader.go](./xdp/loader.go) | Source-keyed 131,072-entry SYN map, fixed bounds, periodic prune; cannot fill map in unprivileged WSL | OPEN; validate upstream anti-spoofing and real map pressure on isolated Linux |
| Protocol coverage ends at opaque transitions / versions beyond 47 | MEDIUM | [inspect.go](./ingress/inspect.go), [l7.go](./minecraft/l7.go) | Existing protocol tests document encryption/compression opaque forwarding and unsupported versions | OPEN; no full L7 claim |
| No authentication/verification integration | MEDIUM | [unified-ingress main](./cmd/unified-ingress/main.go), [inspect.go](./ingress/inspect.go) | No verification provider invoked by login code | NOT IMPLEMENTED |

No changes were made to BPF map layout or packet semantics. No speculative performance optimization was performed. No BAT installer, public deployment automation, production VPS, or Phase 3G work was started.

## Validation record

Environment: WSL2 Ubuntu 24.04.5, kernel `6.18.58.1-microsoft-standard-WSL2`, Go 1.24.7 linux/amd64, 12 logical CPUs, ~3.65 GiB guest RAM (3.83 GB), FD soft limit 10,240, user uid 1000 (not root).

| Command | Result |
|---|---|
| `gofmt -l` on changed Go files | PASS; no files listed |
| `go test -count=1 -timeout=90s ./...` | PASS |
| `CGO_ENABLED=1 go test -race -count=1 -timeout=120s ./...` | PASS |
| `go vet ./...` | PASS |
| `go build ./cmd/unified-ingress ./cmd/unified-xdp ./cmd/gateway-bench` | PASS |
| `clang -target bpf -D__TARGET_ARCH_x86 -I/usr/include/x86_64-linux-gnu -O2 -g -Wall -fsyntax-only xdp/unified_xdp.c` | PASS (syntax only) |
| `go test -v -run TestRealKernelXDPFilter ./xdp` | SKIP; needs root for `BPF_PROG_TEST_RUN` |
| `go test -v -run TestPinnedMapRecovery ./xdp` | SKIP; needs root for BPF map creation |
| `systemd-analyze verify .../craftsmp-unified-ingress.service` | PARTIAL; unit parsed, but expected installed executable is absent and NTFS permission warnings were emitted |

The exact full test/build commands and protocol/adversarial results are in [PHASE3F_SECURITY_AUDIT.md](./PHASE3F_SECURITY_AUDIT.md), [PHASE3F_L7_INTEGRATION_REPORT.md](./PHASE3F_L7_INTEGRATION_REPORT.md), and [PHASE3F_ADVERSARIAL_TEST_REPORT.md](./PHASE3F_ADVERSARIAL_TEST_REPORT.md).

## Recommended next phase: Phase 3F.5 — Privileged Linux End-to-End Validation

Do **not** begin Phase 3G. The next step should use a privileged, isolated Linux host to validate: (1) real XDP verifier/load/attach and pinned-map recovery; (2) IPv6 and fragmented IPv4 policy decisions; (3) spoofed-source/SYN-map saturation with legitimate-client false-positive measurement; and (4) service startup and an actual client → XDP → Gateway → mock backend run. Keep public/VPS deployment out of scope until those tests pass and protocol coverage is explicitly accepted.
