# Layer 7 Adversarial Validation Report

**Status: NOT PRODUCTION-READY.** Local tests demonstrate bounded admission and safe handshake rejection, but the mixed-traffic availability criterion failed, and multiple requested protections are not implemented or integrated.

## Scope and predeclared acceptance criteria

The criteria were recorded before the adversarial runs. All generated traffic was confined to TCP/UDP loopback test endpoints. No public target, production server, DNS, firewall, or production XDP interface was used.

1. **Isolation and bounds:** active connections never exceed configured global/per-IP limits; malformed handshakes do not reach the backend; incomplete handshakes time out.
2. **Availability under mixed traffic:** at least 95% of valid legitimate clients complete the mock-backend exchange during simulated attacks. If attack traffic saturates admission and rejects valid clients, this criterion fails.
3. **Protocol inspection:** malformed/unsupported initial handshakes are rejected. Do not claim inspection of status, ping, login progression, encryption, compression, configuration, or play unless the ingress actually validates those packets.
4. **Recovery and session preservation:** extra rejected connections do not close established sessions; capacity returns after clients leave and a new legitimate connection can proceed.
5. **Proxy trust:** inbound PROXY data is accepted only from configured peer CIDRs; forwarded identity must match the validated header.
6. **Measurement integrity:** report only directly observed counts/timings/resource counters. Per-scenario CPU/RAM not captured is marked not measured.
7. **Production gate:** critical tests must also pass on supported real Linux infrastructure and with representative Java clients/backend configuration before production readiness.

## Test method and environment

Go integration tests started a local TCP gateway and mock backends on loopback. Distributed-source simulation bound clients to `127.0.0.2` through `127.0.0.5`; this represents multiple local source addresses, **not multiple hosts or Internet paths**. A separate 1,000-client test used 1,000 local TCP dials. The constrained-resource case set `GOMAXPROCS=1`; it did not impose a hard OS memory limit. CPU is process CPU time divided by wall time (one-core equivalent). Memory reports process high-water RSS delta and Go total allocations, not container/VM memory.

There is no application queue: the gateway admits/rejects at the accept boundary. Queue utilization is therefore reported as 0; active slots are shown separately.

Ingress integration tests also log process CPU-time delta, Go heap/allocation delta, and process high-water RSS delta per scenario. The process is shared with the Go test harness and mock backend, so short-scenario CPU percentages are noisy and RSS high-water deltas can remain zero after an earlier peak. These values are observations, not gateway-only profiling or capacity guarantees.

## Results

| Scenario | Traffic received | Rejected by gateway | Legit accepted / false reject | Latency / resources | Parser errors / queue / recovery |
|---|---:|---:|---:|---|---|
| 1,000 connection attempts, 1 source | 1,000 successful TCP dials | 968 at configured 32-slot cap | 32 admitted; no separate legitimate client in this case | Repeated runs: 88.0–171.4 ms; latest measured interval 129.9 ms; process CPU 360.8% one-core equivalent; RSS HWM delta 0 KiB; Go total allocation delta 3,552 KiB | Peak 32/32; queue 0; 32 backend connections |
| Distributed connection flood, 4 loopback source IPs | 32 attack connections admitted + 1 legitimate attempt | 0 attacks; legitimate client rejected | 0/1 accepted during saturation; 1 false reject | 95.8 ms scenario wall; process CPU 92.9%; RSS HWM delta 0 KiB; Go total allocation delta 1,423 KiB | Peak 32/32; queue 0; established sessions remained open |
| Recovery after distributed traffic | 16 attack connections closed | N/A | New legitimate connection admitted | 6.6 ms from closing half the attack sockets to observing an additional active connection | Active count moved from 32 to 17 |
| Status request flood | 1 attack connection, 100 status packets | 0 packets | No separate legitimate client in this case | 0.66 ms to mock response; process CPU 43.3%; RSS HWM delta 0 KiB; Go allocation delta 44 KiB | 0 parser errors; queue 0; all 100 packets reached backend |
| Ping flood | 1 attack connection, 100 ping packets | 0 packets | No separate legitimate client in this case | 0.59 ms to mock response; process CPU 46.3%; RSS HWM delta 0 KiB; Go allocation delta 49 KiB | 0 parser errors; queue 0; all 100 packets reached backend |
| Login-start duplicates | 1 attack connection, 20 login-start packets | 0 packets | No separate legitimate client in this case | 1.76 ms to mock response; process CPU 73.2%; RSS HWM delta 0 KiB; Go allocation delta 41 KiB | 0 parser errors; queue 0; all 20 packets reached backend |
| Random-timing reconnects / duplicate login | 24 connections; 48 duplicate login packets | 0 | No separate legitimate client in this case; all 24 forwarded | 77.1 ms scenario wall; process CPU 65.7%; RSS HWM delta 0 KiB; Go allocation delta 717 KiB | Peak 2/64 active; 0 parser errors; policy absent |
| Shared-NAT legitimate clients | 6 concurrent local clients | 0 | 6 accepted; 0 false rejects | 7.4 ms scenario wall; process CPU 127.7% one-core equivalent (multiple test/runtime threads); RSS HWM delta 0 KiB; Go allocation delta 186 KiB | Peak 6/8 active; queue 0 |
| Single-core resource test | 256 dials | 224 at 32-slot cap | 32 admitted | 60.3 ms ingress-load interval; 76.3% one-core-equivalent CPU; RSS HWM delta 0 KiB; sampled Go heap delta +1,016 KiB; Go total allocations +1,163 KiB | Peak 32/32; queue 0 |
| Malformed VarInt, oversized frame, invalid next-state | 3 connections (one per case) | 3 | 0 reached backend | 6.3–7.0 ms wall per case; process CPU 29.5–41.8%; RSS HWM delta 0 KiB; Go allocations 24–25 KiB/case | 1 parser error per case; queue 0 |
| Slow incomplete handshake | 1 connection | 1 timeout | 0 forwarded | 100 ms configured timeout; 107.2 ms test wall; process CPU 2.4%; RSS HWM delta 0 KiB; Go allocation delta 24 KiB | 1 handshake timeout; 0 parser errors |
| Backend unavailable during traffic | 8 connections | 8 closed after dial failure | 0 reached backend | 10.1 ms scenario wall; process CPU 182.3%; RSS HWM delta 0 KiB; Go allocation delta 143 KiB | 8 backend failures; active returned to 0 |
| Backend disconnect during relay | 1 established connection | Session closed when backend closed | Not preserved after backend failure | Scenario wall 19.5 ms; process CPU 35.9%; RSS HWM delta 0 KiB; Go allocation delta 39 KiB | Active returned to 0 |
| IPv6 TCP loopback | 1 connection | 0 | 1 passed handshake preflight | Scenario wall 7.8 ms; process CPU 53.9%; RSS HWM delta 0 KiB; Go allocation delta 39 KiB | Does not validate IPv6 XDP enforcement |
| Bedrock/RakNet UDP | 1 datagram sent to an unserved local UDP port | Not handled or filtered | N/A | Test wall 107.6 ms; process CPU 2.9%; RSS HWM delta 0 KiB; Go allocation delta 11 KiB | No UDP handler; not a mitigation test |
| Reputation / verification outage | Not applicable | Not testable as remote services | Local in-memory block check only | Scenario wall 8.4 ms; process CPU 51.3%; RSS HWM delta 0 KiB; Go allocation delta 15 KiB | Neither remote service is wired into ingress |

### Observed output highlights

- 1,000-dial test: `dial_succeeded=1000 accepted=32 rejected=968 active=32 peak_active=32 backend=32`; latest test wall time 129.9 ms.
- Distributed mixed test: 32/32 attack connections occupied the cap; the next valid client was rejected. Existing sessions remained active. After closing 16 attack sockets, a new valid client was admitted in 6.6 ms.
- Status/ping/login packet floods were each forwarded in full (100/100, 100/100, 20/20).
- Single-core test: 256 attempts, 32 peak active, 224 rejected; CPU 76.3% one-core-equivalent, 1,163 KiB Go allocation delta. RSS HWM delta was 0 KiB and should not be read as total gateway memory use.
- IPv6 loopback passed the TCP handshake path in this environment. Bedrock UDP was not served by the gateway.

## Criteria assessment

- **Isolation and configured connection bounds:** Pass in the local tests. The 1,000-attempt scenario never exceeded 32 active connections.
- **At least 95% legitimate acceptance during attacks:** **Fail.** With the global cap full, the one additional legitimate client was rejected (100% false rejection for that one attempt).
- **Initial handshake validation:** Pass for the tested malformed VarInt, oversized frame, invalid state, unsupported parser input, fragmentation, and slow-handshake cases.
- **Higher-layer flood controls:** **Fail / not implemented in the network ingress.** Status, ping, and login packets were passed to the backend without inspection or rate limiting.
- **Recovery and session preservation:** Pass in this limited local scenario; existing connections survived admission rejection and a later client was admitted after capacity freed.
- **PROXY trust:** Local trust/deny and IPv6 outbound-header tests pass; no deployed Velocity/backend compatibility test was performed.
- **Resource evidence:** Partial. One local single-core test recorded CPU and Go allocations. It did not enforce a RAM ceiling or establish production capacity.
- **Production validation:** **Not performed.**

## Security assessment

### Successfully mitigated in tested scope

- Invalid, oversized, unsupported, or malformed initial Minecraft handshakes are rejected before reaching the backend.
- Slow/incomplete initial handshakes are bounded by deadlines.
- Active connection count is bounded and tested capacity is released after close/failure.
- Untrusted inbound PROXY peers are rejected; validated client identity can be forwarded in PROXY v2.
- Backend dial failures clean up the client connection and active accounting.

### Partially mitigated

- Connection floods are capped, but first-come global capacity is not fair or reserved for legitimate clients.
- Per-IP limits constrain a single source, but are bypassable by distributed sources and may affect shared-NAT users when configured too low.
- IPv6 works in the tested TCP loopback path only. XDP enforcement remains IPv4-only and was not validated here.
- Established relays are opaque by design; this preserves protocol bytes but offers no post-handshake packet policy.

### Unmitigated / known bypasses

- Status and ping floods after a valid handshake are forwarded without gateway limits.
- Login-start floods, duplicate usernames, reconnect storms, and login progression are forwarded without gateway-level detection.
- The standalone parser's status/login tests do not mean the ingress invokes those checks.
- Distributed clients can consume all global slots and reject legitimate clients; source-IP blocking alone does not solve this.
- No remote reputation-service client or verification/limbo service integration exists in the gateway; their outage behavior cannot be validated.
- Bedrock UDP/RakNet has no ingress implementation; Java TCP protection does not apply.
- XDP is outside this suite. These tests prove no kernel packet enforcement, IPv6 XDP support, or protection against upstream link saturation.

### Performance bottlenecks and production risks

- The global fixed cap is the primary overload control and causes legitimate false positives under saturation.
- No queue, fair scheduler, or reserved legitimate capacity exists.
- The test used loopback, one host, mock backends, and brief bursts. It is not representative of Internet latency, distributed source behavior, real Minecraft client versions, Velocity forwarding, or real-host CPU/memory constraints.
- Backend failures terminate affected sessions; preserving a player session across an origin outage is not implemented.
- No performance guarantee should be inferred from local millisecond timings.

## Commands and validation status

Run from `unified/`:

```text
go test -count=1 ./...
go test -race -count=1 ./...
go vet ./...
go build ./cmd/unified-ingress ./cmd/unified-xdp
```

Final results in WSL Ubuntu: all four commands passed. The adversarial scenarios are local automated integration tests; no real Java client, live production backend, real Linux XDP interface, or public network was used.

## Final decision

**Do not deploy as production-ready.** A critical availability criterion failed under a simple distributed local saturation test, and application-layer floods, remote reputation/verification failures, and Bedrock UDP remain unmitigated or untestable. These must be implemented and validated on an isolated supported Linux host with representative clients and a separate controlled test plan before reconsidering production readiness.
