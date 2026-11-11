# Phase 3F.5 — systemd Validation

## Result

**PARTIAL.** Both unit files pass static systemd verification. The XDP unit did not reach healthy operation in the WSL2 test; the ingress unit and its dependencies therefore were not started.

## Evidence and remediation

- `systemd-analyze verify` accepted the XDP and ingress units.
- The ingress unit dependency output includes `craftsmp-unified-xdp.service`.
- Added `AF_NETLINK` to the XDP unit's `RestrictAddressFamilies`, required for interface lookup and rtnetlink fallback.
- The harness confirmed the systemd XDP service began and reached BPF loading, then the kernel verifier denied pointer arithmetic in the restricted systemd context (`!root`). The service has `CAP_BPF`, `CAP_NET_ADMIN`, and `CAP_SYS_RESOURCE`; no broader capability was added to force success.
- A first harness attempt exposed a startup timing race in the health check. The harness now polls `/healthz` for up to five seconds before declaring failure. On rerun, the service failure was reproducibly at BPF load, not API readiness.

## What remains unverified

The systemd XDP service has not passed startup on a dedicated Linux host. The ingress unit, authenticated reputation-to-kernel flow under systemd, and service-managed Gateway-to-backend path were not reached. WSL2 cannot establish whether the same capability policy works on the intended deployment kernel.

## Reproduction

```bash
cd unified
CRAFTSMP_PHASE3F5_CONFIRM_ISOLATED=YES bash scripts/phase3f5_wsl_validation.sh
```

The script refuses pre-existing test paths, uses a temporary veth/netns and bpffs, and cleans up created resources.
