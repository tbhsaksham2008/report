# Phase 3F.5 — Remediation and Limitations

## Verdict

**PARTIAL.** Kernel test-run, bounded SYN-map pressure/recovery, pinned-map recovery, direct generic/native veth attachment, automated Go tests, race tests, vet, and command builds passed. The systemd XDP service and full isolated TCP-to-backend flow did not pass on WSL2.

## Changes made

| File | Change |
|---|---|
| `xdp/loader.go` | Added fail-closed rtnetlink fallback when `BPF_PROG_QUERY` returns `EINVAL`; reject missing or malformed state. |
| `xdp/linux_test.go` | Added parser tests and verified attachment state before/after direct veth attach/detach. |
| `systemd/craftsmp-unified-xdp.service` | Allows `AF_NETLINK` for interface inspection and safe fallback. |
| `scripts/phase3f5_wsl_validation.sh` | Refuses any pre-existing `/etc/craftsmp` path and waits for XDP health readiness before proceeding. |

## Remaining findings and limits

1. **High — IPv6 policy bypass.** The current XDP program passes IPv6 (including extension-header and truncated fixtures); IPv4 rules are not IPv6 protection.
2. **High — fragment policy bypass.** IPv4 fragments pass without TCP/SYN parsing; only source blocklist lookup occurs before that decision. Fragmented game traffic can bypass the SYN policy.
3. **High — systemd dataplane not validated.** WSL2 systemd starts the service process, but BPF verifier loading fails with the unit's restricted capabilities. No capability expansion was made. Dedicated-Linux verification is required.
4. **Medium — complete live integration not established.** The backend and ingress stages were not reached after the XDP unit failure.
5. **Low — `BPF_PROG_QUERY` unavailable on tested WSL2 kernel.** The new rtnetlink fallback is directly tested, but the original query API remains unavailable there.

## Recommended next work

Repeat the guarded service and end-to-end tests on a disposable dedicated Linux host using the intended production kernel and the unchanged least-privilege unit. Resolve IPv6 and fragment policy in a separate design/test change. Do not proceed to Phase 3G based on these WSL2 results.
