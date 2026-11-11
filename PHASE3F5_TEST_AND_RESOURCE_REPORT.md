# Phase 3F.5 — Automated Tests and Resource Cleanup

## Environment

WSL2 Ubuntu, kernel `6.18.58.1-microsoft-standard-WSL2`, Go `1.24.7 linux/amd64`, Intel Core i5-12450H (12 logical CPUs), 3,828,500 kB visible memory, and open-file limit 10,240. WSL2 is not equivalent to dedicated Linux hardware.

## Results

| Command/test | Result |
|---|---|
| `go test -count=1 ./...` | PASS |
| `go test -race -count=1 ./...` | PASS |
| `go vet ./...` | PASS |
| `go build ./cmd/...` | PASS |
| `gofmt -d xdp/loader.go xdp/linux_test.go` | PASS; no diff |
| `bash -n scripts/phase3f5_wsl_validation.sh` | PASS |
| `TestRealKernelXDPFilter` | PASS |
| `TestParseXDPAttachment` | PASS |
| `TestVirtualInterfaceXDPAttachModes` | PASS, generic and native |
| `TestSYNMapCapacityPressureAndRecovery` | PASS, 131,072 entries |
| `TestPinnedMapRecovery` | PASS with temporary bpffs |
| `TestIncompatiblePinnedMapIsRejected` | PASS with temporary bpffs |
| systemd XDP service startup | FAIL in WSL2 at verifier load |
| systemd ingress and full TCP/backend harness | NOT TESTED; blocked by XDP unit startup |

## Cleanup

The isolated harness printed `Cleanup finished.` A follow-up check confirmed that the temporary `/etc/craftsmp` configuration, installed binaries, runtime units, bpffs mount, veth, and network namespace were absent. No production host or public interface was used.

No CPU/RSS capacity or long-duration soak measurement was part of this phase's verification; those are **NOT TESTED here**.
