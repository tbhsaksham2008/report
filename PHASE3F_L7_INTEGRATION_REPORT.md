# Phase 3F L7 Integration Report

## Verdict

**PARTIAL.** The Go Gateway is a real TCP listener and processes stream bytes before opening a backend connection. Phase 3F wires its optional reputation query to authenticated XDP-agent state and adds a systemd dependency artifact. A loopback test verifies the Gateway-to-agent path over TCP/HTTP. Actual XDP attachment and a NIC-to-backend run were not possible in unprivileged WSL2.

## Verified packet-to-application path

1. A packet reaches the host interface. When loaded, the XDP program checks supported IPv4 fields, source block/reputation maps, malformed TCP flag combinations, and SYN rate state. It either drops the packet or returns `XDP_PASS`.
2. On `XDP_PASS`, the Linux network stack processes TCP and delivers the reassembled byte stream to the `unified-ingress` listener. XDP does not call the Go agent for each packet and does not perform TCP reassembly.
3. Gateway acceptance reserves bounded global/per-source/per-prefix capacity. For ordinary direct connections, identity is the socket peer. If inbound PROXY is configured, only trusted peer CIDRs may supply a PROXY header; its advertised client identity is then used for source policy.
4. The Gateway optionally asks the reputation provider before handshake processing. The provider sends a bounded JSON POST and, when configured, a bearer token. The new agent `/lookup` route requires that token and returns a score; active Store blocks map to score 100.
5. The Gateway reads and bounds the Minecraft handshake/frame from TCP, applies protocol/admission policy, and only then dials the backend. Invalid and policy-rejected handshakes do not reach the backend.
6. Supported status/login frames are inspected until encryption, compression, or login success transitions the connection to opaque forwarding. Opaque bytes are relayed without claiming parsing.

## Changes

- Added authenticated `POST /lookup` to [agent.go](./edge/agent.go), rate limited at 4,096 calls per source per minute with separate accounting from mutation requests.
- Added optional bearer-token support to [reputation.go](./ingress/reputation.go), preserving tokenless existing HTTPS/loopback provider use.
- Added `-reputation-token` / `CRAFTSMP_REPUTATION_TOKEN` support to [unified-ingress main](./cmd/unified-ingress/main.go). A token without an endpoint is a configuration error.
- Added [craftsmp-unified-ingress.service](./systemd/craftsmp-unified-ingress.service), ordered after and requiring the XDP service. It expects an operator-created environment file and uses `DynamicUser=yes`.
- Replaced the prior “not integrated” characterization test with `TestGatewayReputationServiceIntegrationOverLiveTCP` in [adversarial_test.go](./ingress/adversarial_test.go). It starts a real TCP Gateway listener and an HTTP agent handler, uses a valid score for source `127.0.0.2`, and confirms the connection is rejected before the mock backend is reached.

Example same-host environment values (not installed automatically):

```text
CRAFTSMP_INGRESS_LISTEN=127.0.0.1:25565
CRAFTSMP_BACKEND=127.0.0.1:25566
CRAFTSMP_REPUTATION_URL=http://127.0.0.1:8080/lookup
CRAFTSMP_REPUTATION_TOKEN=<same secret configured as CRAFTSMP_API_TOKEN>
CRAFTSMP_REPUTATION_FAIL_CLOSED=true
```

The Gateway defaults to loopback. A public binding must be an explicit operator choice. The example uses loopback only; changing it to a public address is not validated here. For a non-loopback reputation endpoint, HTTPS remains mandatory.

## Tests and results

| Test | Result | What it establishes |
|---|---|---|
| `go test -count=1 -run TestGatewayReputationServiceIntegrationOverLiveTCP ./ingress` | PASS | Real TCP Gateway client and authenticated HTTP reputation lookup; high-risk source rejected before backend |
| `go test -count=1 ./...` | PASS | All repository Go tests |
| `go test -race -count=1 ./...` | PASS | Go race detector across packages |
| `go vet ./...` | PASS | Go static checks |
| `go build ./cmd/unified-ingress ./cmd/unified-xdp ./cmd/gateway-bench` | PASS | Gateway, XDP agent, and benchmark binaries compile |
| `clang ... -fsyntax-only xdp/unified_xdp.c` | PASS | C syntax only; not BPF verifier acceptance or attach |
| `systemd-analyze verify ...` | PARTIAL | Unit parsed; exit nonzero because the expected installed executable is absent in WSL, with NTFS permission warnings |
| `TestRealKernelXDPFilter`, `TestPinnedMapRecovery` | SKIP | Need root/BPF privileges |

## Not established

- No real NIC was attached and no XDP program was loaded during Phase 3F.
- The Gateway test did not launch the XDP binary or make the kernel drop a packet before TCP.
- No systemd service was installed, enabled, or started.
- No public IPv4/IPv6 reachability, PROXY load balancer, backend, or production topology was changed or verified.
- If the agent is unavailable, the Gateway follows the configured fail-open/fail-closed behavior. In the systemd example fail-closed is recommended, but runtime failure behavior was not measured beyond existing provider tests.
- This integration does not add authentication/verification services or modern Minecraft packet handling.
