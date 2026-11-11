# Phase 3F Adversarial Test Report

## Scope and safety

All Phase 3F automated traffic was limited to WSL2 loopback and in-process test listeners. No public host, production VPS, or third-party service was targeted. Existing Phase 3E.1 soak/load measurements are cited as prior evidence and were not repeated as Phase 3F results.

## Results matrix

| Scenario | Test evidence | Phase 3F result | Qualification |
|---|---|---|---|
| Connection floods / global capacity | `TestGatewayThousandsOfAttemptsRemainBounded`, `TestGatewayGlobalConnectionLimit` | PASS | In-process/loopback; not public SYN capacity |
| Distributed source simulation and mixed traffic | `TestAdversarialDistributedFloodAndMixedLegitimateTraffic`, `TestAdversarialSharedNATLegitimateClients` | PASS | Loopback source addresses; not a true distributed network |
| Handshake and malformed traffic | `TestGatewayRejectsMalformedAndOversizedHandshakes`, incomplete/deadline tests | PASS | TCP streams through Gateway |
| Status-query flood | `TestAdversarialStatusPingAndLoginFloodsAreOpaque` | PASS | Verifies configured request cap and backend request count |
| Ping flood | Same adversarial suite plus status-flow tests | PASS | Synthetic protocol-47 traffic |
| Login flood / duplicates / reconnect | `TestAdversarialRandomizedReconnectAndDuplicateLogins`; Phase 3E.1 reconnect stress evidence | PASS | Reconnect stress report is historical Phase 3E.1, not rerun here |
| Slow incomplete connections | `TestPendingHandshakeCapacityPromotesAndExpiresReservations`, `TestGatewayHandshakeTimeout`, absolute deadline test | PASS | Unit/integration-level TCP clients |
| Oversized and malformed VarInts/frames | `TestStreamParserRejectsOversizedFrameBeforeBodyAllocation`, `TestVarIntOverflowRejected`, Gateway malformed-frame tests | PASS | Parser and Gateway regressions |
| Backend failure/disconnect | `TestAdversarialBackendFailureUnderTraffic`, `TestAdversarialBackendDisconnectClosesEstablishedSession` | PASS | Mock backend only |
| Backend latency | Phase 3E.1 250 ms mock-backend delay profile | PASS (prior phase) | Not rerun as Phase 3F |
| Authenticated reputation service through live Gateway TCP | `TestGatewayReputationServiceIntegrationOverLiveTCP` | PASS | Gateway listener and HTTP test server; no real XDP/NIC |
| Store exhaustion | `TestStoreBoundsRecordsAndAuditTrail`, `TestBlockRejectsAtStoreCapacityBeforeKernelMutation` | PASS | Fills the 8,192-record application bound; kernel map exhaustion not tested |
| Abrupt disconnect and shutdown | Existing Gateway shutdown/active connection tests | PARTIAL | Shutdown with active sessions is covered; no separate broad client-abort storm was run in Phase 3F |
| IPv6 XDP traffic | No privileged kernel test available | NOT TESTED | XDP currently passes non-IPv4 frames |
| Fragmented IPv4 SYN traffic | Kernel test fixture expects pass; kernel execution skipped | NOT TESTED | Fragmented packets bypass the XDP TCP/SYN parsing path |
| Real SYN flood / source spoofing / kernel map saturation | No privileged isolated NIC or BPF test environment | NOT TESTED | Must not infer from Go packet-model tests |
| Statistical false-positive / false-negative rates | No labeled production-like population | NOT TESTED | Loopback regressions only verify individual expected outcomes |

## Focused commands

All passed:

```bash
/usr/local/go/bin/go test -count=1 -run TestAdversarial ./ingress
/usr/local/go/bin/go test -count=1 -run TestGatewayReputationServiceIntegrationOverLiveTCP ./ingress
/usr/local/go/bin/go test -count=1 -run TestStoreBounds ./reputation
/usr/local/go/bin/go test -count=1 -run TestBlockRejectsAtStoreCapacity ./edge
/usr/local/go/bin/go test -count=1 -timeout=90s ./...
CGO_ENABLED=1 /usr/local/go/bin/go test -race -count=1 -timeout=120s ./...
```

`TestRealKernelXDPFilter` and `TestPinnedMapRecovery` were invoked explicitly and skipped with their root-required messages. `clang -target bpf ... -fsyntax-only xdp/unified_xdp.c` passed but is only a C syntax check, not verifier or runtime evidence.

## False-positive / false-negative observations

- Per-source, per-prefix, global admission limits and reserved capacity are exercised by existing tests, including shared NAT behavior and distributed loopback source simulation.
- At global saturation, the mixed-traffic test observes rejection of a later legitimate connection until capacity recovers; it does not claim zero false positives under attack.
- The XDP SYN map’s fail-closed allocation behavior may reject new sources while the fixed map is full. This possible false-positive condition was not induced in a kernel test.
- The Gateway uses a score threshold and configured failure mode for remote reputation. The new live-path test verifies a high score is rejected before backend dial, not a population-wide classification rate.

## Limitations

These tests establish parser, policy, cleanup, and mock-service behavior only. They do not prove physical-network throughput, upstream bandwidth defense, production connection capacity, resilience to spoofed packets, or compatibility with real player populations.
