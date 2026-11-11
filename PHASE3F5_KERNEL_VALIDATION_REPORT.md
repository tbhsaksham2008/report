# Phase 3F.5 — Kernel Validation

## Result

**PARTIAL.** The unified XDP object loads and runs through the WSL2 kernel test-run API, but the production XDP systemd unit cannot load it under the unit's restricted capability set on this WSL2 kernel.

## Environment

| Item | Value |
|---|---|
| Environment | WSL2 Ubuntu |
| Kernel | `6.18.58.1-microsoft-standard-WSL2` |
| Architecture | `linux/amd64` |
| Go | `go1.24.7 linux/amd64` |
| CPU | Intel Core i5-12450H, 12 logical CPUs |
| Memory visible to WSL | 3,828,500 kB |
| Open-file limit | 10,240 |
| `kernel.unprivileged_bpf_disabled` | `2` |

These are WSL2 results, not dedicated-Linux or production results.

## Evidence

- `TestRealKernelXDPFilter`: **PASS**. The kernel accepted the program and the tests exercised packet decisions through `BPF_PROG_TEST_RUN`.
- Direct isolated-veth attachment in generic and native modes: **PASS**; see the attachment report.
- `systemd-analyze verify` for both XDP and ingress unit files: **PASS**.
- Starting the XDP unit: **FAIL on this WSL2 environment**. The loader reached BPF program loading, then the verifier returned `permission denied` and `pointer arithmetic with it prohibited for !root`. The unit has `CAP_BPF`, `CAP_NET_ADMIN`, and `CAP_SYS_RESOURCE`; the matching transient unit showed those capabilities effective. The failure is specific to the restricted systemd context/kernel combination and is not evidence that the object itself fails verification in the kernel test-run context.

No broader capability was added to the service to work around this result. In particular, `CAP_SYS_ADMIN` was not added.

## Reproduction

```bash
cd unified
go test ./xdp -run '^TestRealKernelXDPFilter$' -count=1 -v
```

The guarded service harness is `scripts/phase3f5_wsl_validation.sh`. It requires an isolated WSL2 distro and `CRAFTSMP_PHASE3F5_CONFIRM_ISOLATED=YES`.
