# Phase 3F Minecraft Protocol Security Report

## Coverage verdict

**PARTIAL.** The Gateway performs bounded TCP stream framing and selected Java protocol-47 inspection. It does not provide complete Minecraft L7 protection. Compression/encryption and subsequent states are explicitly opaque; protocol versions other than 47 are not inspected by default. Bedrock UDP is outside this TCP Gateway.

## Stage behavior

| Stage / traffic | Current behavior | Tests / limitations |
|---|---|---|
| TCP accept and initial frame | Gateway applies global/source/prefix/pending limits and bounded read/handshake deadlines; it reads bytes from TCP and handles frames split across reads | Gateway tests cover fragmented stream reads, incomplete handshakes, absolute deadlines, and bounded admission |
| Minecraft handshake | Bounded VarInt/frame parsing; validates packet id, version integer, address/port and next-state structure; unsupported versions are rejected by default unless explicit opaque forwarding is enabled | `TestGatewayInspectsValidStatusAndLoginHandshakes`, malformed/oversized handshake tests; protocol acceptance does not imply the version’s later packets are inspected |
| Status request/response | Protocol-47 request shape and status response are checked; per-source/prefix/global rate policies apply before backend forwarding | Status flood and response validation tests; status JSON is bounded and shape-checked |
| Ping/pong | Eight-byte ping/pong matching and status-flow progression are checked for inspected protocol 47 | Ping flood and invalid sequence tests |
| Login Start | Protocol-47 username bounds and login/reconnect/duplicate/admission checks are applied | Synthetic protocol-47 tests; no Mojang authentication or online-mode verification is implemented by this Gateway |
| Encryption negotiation | Selected protocol-47 Encryption Request/Response packet structure is validated before transition | After negotiation, ciphertext is opaque and not parsed or inspected |
| Compression negotiation | Set Compression threshold structure is checked; subsequent traffic is opaque | No decompression or compressed/uncompressed threshold verification |
| Login Success | Selected protocol-47 fields are validated, then traffic is relayed opaquely | No further protocol validation after transition |
| Configuration and Play | Relayed only when an explicitly enabled opaque policy allows that protocol; no state-aware inspection | No packet IDs, field layouts, or transitions are implemented for these stages |
| Modern Java protocol IDs | Handshake may be structurally read; unsupported versions are rejected by default. Explicit opaque opt-in forwards without L7 packet checks | No version-specific inspection or real-client compatibility claim |
| Bedrock / RakNet UDP | Not processed by the TCP Gateway | `TestBedrockDatagramsAreOutsideTCPGatewayScope` documents that UDP mitigation is absent |

The protocol-47 packet definitions and detailed behavior are documented in [PHASE3C_PROTOCOL_SUPPORT.md](./PHASE3C_PROTOCOL_SUPPORT.md). No Java stream validation claim is based on the XDP packet helper; it only sees one packet buffer and is not used as TCP reassembly.

## Security checks and regression coverage

- VarInt parsing rejects malformed/overflow encodings; frame lengths and input chunks are bounded.
- Stream tests cover split length prefixes, split frame bodies, multiple frames in one read, incomplete and oversized frames.
- Gateway tests cover handshake trailing bytes, malformed status/login transitions, unsupported login protocol, response matching, timeouts, opaque byte preservation, and backend failure/shutdown.
- Admission and adversarial tests cover status/ping/login floods, duplicate/reconnect behavior, shared-NAT clients, distributed loopback source simulation, incomplete handshakes, and backend errors.
- PROXY identity is used only when the immediate peer is trusted and the header is valid; otherwise the connection is rejected.

Reproduce the core tests from `unified/`:

```bash
/usr/local/go/bin/go test -count=1 ./minecraft ./ingress ./proxy
/usr/local/go/bin/go test -count=1 -run TestAdversarial ./ingress
```

Both passed in WSL2 for this phase.

## Findings

### P-01 — No packet inspection after opaque transition

- **Severity:** MEDIUM (coverage limitation)
- **Affected files:** [inspect.go](./ingress/inspect.go), [l7.go](./minecraft/l7.go)
- **Technical explanation:** Encryption, compression, and Login Success trigger byte-preserving opaque relay. The Gateway cannot parse encrypted data; compression is not decompressed or validated against its negotiated threshold.
- **Reproduction:** Run `TestGatewayRelaysCompressedLoginTrafficOpaque` and `TestGatewayRelaysEncryptedLoginTrafficOpaque` in `ingress`.
- **Remediation status:** **OPEN / BY DESIGN FOR THIS IMPLEMENTATION.** No full-stage inspection is claimed.

### P-02 — Modern protocol and configuration/play layouts are not implemented

- **Severity:** MEDIUM (coverage limitation)
- **Affected files:** [l7.go](./minecraft/l7.go), [inspect.go](./ingress/inspect.go)
- **Technical explanation:** Only protocol 47 has selected inspected packet layouts. Other versions are rejected unless the operator opts into opaque forwarding; Configuration and Play streams have no parser/state transition implementation.
- **Reproduction:** `go test -count=1 -run TestGatewayRejectsUnsupportedLoginProtocolBeforeBackend ./ingress`.
- **Remediation status:** **OPEN.** Protocol-specific implementation and real-client compatibility are outside Phase 3F.

### P-03 — No authentication/verification service

- **Severity:** MEDIUM (product capability limitation)
- **Affected files:** [cmd/unified-ingress/main.go](./cmd/unified-ingress/main.go), [ingress/inspect.go](./ingress/inspect.go)
- **Technical explanation:** The Gateway’s Login Start checks are syntactic and policy-based; they do not authenticate a Minecraft account or replace backend online-mode/Velocity forwarding.
- **Reproduction:** Existing synthetic Login Start tests exercise parsing; no external authentication provider is configured or invoked.
- **Remediation status:** **NOT IMPLEMENTED.** Do not treat successful Gateway admission as Minecraft account verification.

## Not tested

Real Java clients, modern protocol interoperability, authentication, and a real Paper/Velocity backend were not used. Kernel XDP attachment and IPv6 packet filtering were not tested. The Phase 3F test environment is WSL2 and loopback only.
