# Phase 3F.5 — Pinned Map Recovery

## Result

**PASS** for recovery and incompatible-map rejection in the isolated WSL2 test.

## Evidence

With a temporary bpffs mount:

- `TestPinnedMapRecovery`: **PASS**. Compatible pinned map state was reopened and retained.
- `TestIncompatiblePinnedMapIsRejected`: **PASS**. A pinned map with an incompatible definition was rejected rather than silently reused.
- The test used its own temporary pin directory and removed it on completion.

The global `/sys/fs/bpf/craftsmp` path was not used. The temporary bpffs mount was unmounted after the test.

## Reproduction

Run as root only inside an isolated WSL2 environment with bpffs not already mounted:

```bash
cd unified
mount -t bpf bpf /sys/fs/bpf
trap 'umount /sys/fs/bpf' EXIT
go test ./xdp -run 'TestPinnedMapRecovery|TestIncompatiblePinnedMapIsRejected' -count=1 -v
```

The Phase 3F.5 harness refuses a pre-existing bpffs mount or pin directory. These tests do not establish reboot recovery on a dedicated Linux host.
