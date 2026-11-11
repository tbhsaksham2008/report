# Phase 3F.5 — Network Integration

## Result

**NOT TESTED end to end.** Direct veth XDP attach/detach was tested, but the complete client → XDP → Go Agent → ingress/Gateway → mock backend path was blocked before service health by the WSL2 systemd BPF verifier failure.

## Evidence

The guarded harness constructs an isolated network namespace and veth pair, starts the XDP reputation service, then would start the mock backend and ingress service and exercise valid status traffic, malformed/incomplete handshakes, reputation decisions, and backend failure. On the final run, the XDP unit failed during BPF program load. The harness stopped and cleaned up before starting the backend or ingress service.

Consequently, no full-path TCP status response, rejection count, or backend forwarding result is claimed by this report. Existing Go unit and integration tests passed, but they are not substitutes for the blocked live network path.

## Reproduction

```bash
cd unified
CRAFTSMP_PHASE3F5_CONFIRM_ISOLATED=YES bash scripts/phase3f5_wsl_validation.sh
```

Run only in an explicitly disposable isolated WSL2 distro. Do not point the harness at a public or production interface.
