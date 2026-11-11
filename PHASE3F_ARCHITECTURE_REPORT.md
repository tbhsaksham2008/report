# Phase 3F Architecture Report

## Current runtime flow

```text
Client TCP
  -> XDP program on the ingress NIC (only if separately loaded and attached)
  -> XDP_PASS to the Linux network stack
  -> craftsmp-unified-ingress TCP listener
  -> trusted PROXY parsing when explicitly configured
  -> source identity and capacity/admission checks
  -> bounded Minecraft handshake/frame inspection
  -> backend TCP dial after handshake/admission
  -> inspected protocol relay, then opaque relay after negotiated transitions
```

The XDP hook is a packet filter, not a TCP proxy. It sees packet buffers and IPv4/TCP headers only; it does not deliver data to the Gateway or reconstruct a byte stream. The kernel TCP stack performs reassembly and delivers stream bytes to `craftsmp-unified-ingress`. The Gateway then parses the Minecraft stream.

## Components and trust boundaries

| Component | Responsibility | Bounds / important behavior |
|---|---|---|
| [XDP program](./xdp/unified_xdp.c) | IPv4 source blocks/reputation flags, conservative TCP field checks, per-source SYN window | Fixed block/reputation maps (262,144 entries each), SYN map (131,072); IPv6 and fragmented TCP pass the XDP packet policy |
| [XDP loader](./xdp/loader.go) | Loads embedded program, reuses compatible pinned maps, refuses to replace an existing XDP attachment | Linux-only; native/generic/offload selection is explicit; privileged tests not run here |
| [XDP agent](./edge/agent.go) | Authenticated map updates, health, trusted PROXY helper, reputation lookup | Mutating APIs use bearer auth and per-client rate limits; trusted CIDRs capped at 256; lookup capped separately and response contains only score |
| [Reputation Store](./reputation/manager.go) | In-process block/score/allow state | Shared maximum of 8,192 records, 1,024 audit entries, bounded reason/IP lengths and TTL |
| [Ingress Gateway](./ingress/gateway.go) | TCP admission, source/prefix accounting, handshake processing, backend relay | Active/pending connections and parser/resource limits; socket peer is the client identity unless a trusted PROXY peer is configured |
| [Minecraft parser](./minecraft/l7.go) | Bounded VarInt/frame parsing and selected protocol-47 state transitions | TCP-stream-driven in Gateway; standalone packet helper is not TCP reassembly |
| [PROXY parser](./proxy/proxy.go) | Validates supported PROXY header forms and trust | Gateway requires configured trusted peer CIDRs when inbound PROXY is enabled |

## Integration gap and remediation

Before Phase 3F, `unified-xdp` and `unified-ingress` were separate executables, and no ingress systemd service tied the public TCP listener to the XDP service. The XDP agent also had no read-only lookup contract usable by the Gateway; the Gateway’s optional HTTP provider was not connected to agent state.

This phase adds:

1. Authenticated `POST /lookup` on the agent, returning the stored score; an active block is represented as score 100 and an allowlisted address returns the Store’s cleared/neutral score.
2. Optional bearer-token support in the Gateway HTTP reputation provider and the `CRAFTSMP_REPUTATION_TOKEN` setting.
3. [craftsmp-unified-ingress.service](./systemd/craftsmp-unified-ingress.service), which requires and starts after the XDP service and runs the Gateway as a dynamic unprivileged user.
4. A loopback integration test that sends a real TCP handshake through the Gateway, queries the agent’s HTTP lookup endpoint, and verifies rejection before a backend connection.

The systemd unit is a wiring artifact, not an installed service. Its environment file and binaries are operator-provided; no installer or deployment automation was run. `systemd-analyze verify` parsed the unit but returned failure because `/usr/local/bin/craftsmp-unified-ingress` is not installed in WSL and warned about NTFS mount file permissions. Service startup is **NOT TESTED**.

## PROXY and source identity

Without inbound PROXY mode, the Gateway uses the socket peer address. With inbound PROXY mode, it checks that the actual peer is in `TrustedProxies`, parses a bounded PROXY header, applies per-client limits to the advertised source, and requires outbound PROXY forwarding to preserve that identity to the backend. Do not configure `ExpectProxyProtocol` behind a listener reachable directly by arbitrary clients.

The new agent `/lookup` API does not parse client-supplied forwarding headers. Its caller is the Gateway, which submits a validated IP address obtained from the TCP peer or a trusted PROXY header.

## Findings

### A-01 — L7 runtime wiring was absent

- **Severity:** MEDIUM (integration gap)
- **Affected files:** [cmd/unified-ingress/main.go](./cmd/unified-ingress/main.go), [systemd/](./systemd)
- **Technical explanation:** A real TCP Gateway executable existed, but the prior service definitions did not start it alongside XDP and agent state was process-local.
- **Reproduction:** Inspect existing systemd unit list and run `go test -run TestGatewayReputationServiceIntegrationOverLiveTCP ./ingress`; the former had only the XDP unit, while the latter is the added in-process network test.
- **Remediation status:** **PARTIALLY FIXED.** Added a dependent ingress unit, authenticated lookup endpoint/client, and loopback live-TCP test. Actual service startup, NIC attachment, and production port routing remain **NOT TESTED**.

### A-02 — Separate processes have separate in-memory stores

- **Severity:** LOW (consistency limitation)
- **Affected files:** [cmd/unified-xdp/main.go](./cmd/unified-xdp/main.go), [cmd/unified-ingress/main.go](./cmd/unified-ingress/main.go)
- **Technical explanation:** The processes still create independent Stores. The explicit local HTTP lookup lets the Gateway query current agent state; it does not make the maps transactionally shared or eliminate lookup latency/outage behavior.
- **Reproduction:** Start the agent and a Gateway with the documented URL/token and observe lookup requests; stop the agent to exercise the configured fail-open/fail-closed behavior.
- **Remediation status:** **PARTIALLY ADDRESSED.** Configurable authenticated lookup is implemented; fail-closed policy is operator-configured. A single shared process/store is not introduced.

## Runtime limitations

- XDP only filters IPv4; IPv6 traffic is passed to the OS.
- Fragmented IPv4 TCP is passed to avoid parsing an incomplete transport header.
- XDP does not validate TCP sequence numbers, complete handshakes, or Minecraft bytes.
- A deployment with an upstream TCP proxy must configure the trusted PROXY chain correctly.
- The Gateway’s L7 inspection is limited to tested protocol 47 transitions; opaque stages and versions are not packet-inspected.
- No production service, VPS, public interface, or physical NIC was changed or tested.

## Verification commands

From `unified/` in WSL:

```bash
/usr/local/go/bin/go test -count=1 -run TestGatewayReputationServiceIntegrationOverLiveTCP ./ingress
/usr/local/go/bin/go test -count=1 ./...
/usr/local/go/bin/go test -race -count=1 ./...
/usr/local/go/bin/go vet ./...
/usr/local/go/bin/go build ./cmd/unified-ingress ./cmd/unified-xdp ./cmd/gateway-bench
```
