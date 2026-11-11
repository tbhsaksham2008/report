# Phase 3F.5 — SYN Map Pressure

## Result

**PASS** for bounded-map saturation and expiration recovery under the WSL2 kernel.

## Measurement

The opt-in test filled all **131,072** source-key entries in the SYN-rate map:

| Operation | Measured time |
|---|---:|
| Insert 131,072 distinct source keys | 188.625 ms |
| Prune 131,072 expired keys | 282.209 ms |

When full, insertion for a new key failed closed and the packet was dropped; after expiration and pruning, the map accepted new state again. This confirms bounded map capacity and recovery behavior in this kernel test. It does not characterize production packet rate or capacity.

## Reproduction

```bash
cd unified
mount -t bpf bpf /sys/fs/bpf
trap 'umount /sys/fs/bpf' EXIT
CRAFTSMP_RUN_MAP_PRESSURE=1 go test ./xdp -run '^TestSYNMapCapacityPressureAndRecovery$' -count=1 -v
```

The test is opt-in because it creates 131,072 kernel map entries and requires root/BPF privileges. Do not run it against a shared or production system.
