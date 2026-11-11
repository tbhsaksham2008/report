# Phase 3F.5 — XDP Packet Behavior

## Result

**PARTIAL.** IPv4 filtering and malformed-packet checks passed in kernel test-run. IPv6 and fragmented IPv4 are explicitly not inspected by the current XDP policy.

## Verified behavior

`TestRealKernelXDPFilter` passed with these observed decisions:

- Malformed IPv4 version, undersized/oversized IPv4 length, truncated TCP header, invalid TCP data offset, and malformed TCP flag combinations: **DROP**.
- Normal game-port SYN and ACK: **PASS** when within configured policy.
- Null scan and selected invalid flag combinations: **DROP**.
- Non-game-port traffic: **PASS**.
- First, subsequent, and empty subsequent IPv4 fragments: **PASS**; fragment parsing and SYN limiting are bypassed because fragments do not all contain a complete TCP header.
- IPv6 TCP, UDP, TCP behind an extension header, and truncated IPv6 fixture: **PASS**; the current XDP program does not enforce its IPv4 policy on IPv6.
- A source blocklist match is applied before the IPv4 fragment pass decision and drops the packet.

The test also exercised an authenticated Agent block/unblock request against an in-memory kernel test object; the kernel decision changed from DROP to PASS after unblock.

## Limitations

Passing IPv6 and fragmented packets is not equivalent to IPv6 or fragment protection. If those paths can reach the game backend, they bypass IPv4 SYN-rate and protocol-shape checks. No change was made to packet policy during this validation because changing fragment or IPv6 behavior requires a separate compatibility and security decision.

These are `BPF_PROG_TEST_RUN` results, not proof of end-to-end packet behavior through an attached production interface.

## Reproduction

```bash
cd unified
go test ./xdp -run '^TestRealKernelXDPFilter$' -count=1 -v
```
