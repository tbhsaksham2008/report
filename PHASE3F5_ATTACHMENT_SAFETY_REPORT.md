# Phase 3F.5 — XDP Attachment Safety

## Result

**PASS for direct isolated-veth attachment checks; PARTIAL for service startup.**

## Finding and change

The loader's existing `BPF_PROG_QUERY` safety check returned `EINVAL` on the WSL2 kernel even when direct attachment and detach worked. Treating that error as “no program” would make attachment unsafe. The loader now falls back only on `EINVAL` to an rtnetlink interface dump. It proceeds only when the target interface is present and includes explicit XDP attachment state. Missing or malformed state remains an error; any reported attachment or nonzero program ID is treated as occupied.

Changed code:

- `xdp/loader.go`: fail-closed rtnetlink fallback and bounded nested-attribute parsing.
- `xdp/linux_test.go`: parser regressions and assertions before/after attach and detach.

## Evidence

`CRAFTSMP_RUN_INTERFACE_TEST=1 go test ./xdp -run '^TestVirtualInterfaceXDPAttachModes$' -count=1 -v` passed for both generic and native modes. The test confirmed the fallback reported the program while attached and no program after detach, then successfully reattached and detached.

`TestParseXDPAttachment` passed for explicit unattached state, generic attachment, nonzero program ID, missing state, and truncated attributes.

The loader still refuses to start the XDP service on this WSL2 kernel because BPF program loading is denied by the systemd capability context; this fallback does not bypass that separate restriction.

## Reproduction

```bash
cd unified
CRAFTSMP_RUN_INTERFACE_TEST=1 go test ./xdp -run '^TestVirtualInterfaceXDPAttachModes$' -count=1 -v
go test ./xdp -run '^TestParseXDPAttachment$' -count=1 -v
```
