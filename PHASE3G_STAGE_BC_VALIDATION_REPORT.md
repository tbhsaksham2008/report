# Phase 3G — Stage B/C Implementation and Validation

## Verdict

**PARTIAL.** Stage B added bounded IPv4/IPv6 packet policy and Stage C now has a
repeatable, successful WSL2 live-path test for Minecraft protocol-47 status traffic.
Those results did not complete the Phase 3G platform: the restricted systemd unit
has not been validated successfully, and later stages were still open when this
historical report was first written.

**Status addendum:** Stages D–F now include a versioned rate-policy manager,
authenticated control-plane API, and embedded dashboard. Their exact scope and
newer validation results are recorded in
[`PHASE3G_STAGE_DF_CONTROL_PLANE_REPORT.md`](./PHASE3G_STAGE_DF_CONTROL_PLANE_REPORT.md).
The original Stage B/C test transcript below is retained as a historical record;
the newer live test also exercises the control-plane status, policy update, and
stale-version rejection paths.

No public target or production VPS was used. The live test used one isolated network
namespace, a veth pair, the mock backend, and generic XDP inside WSL2.

## Stage B implementation

### XDP packet path

- Added IPv6 address parsing and separate pinned IPv6 blocklist, reputation,
  TCP-SYN-rate, and UDP-rate maps.
- Extended the Go loader to route IPv4/IPv6 lookups and mutations to the correct
  maps, create/reuse pinned maps, and expire entries from both address families.
- Added IPv4/IPv6 UDP source-rate handling for the configured UDP game port
  (default `19132`). Its independent threshold and window use a separate pinned
  configuration map, preserving the existing TCP configuration map layout. UDP
  rate limiting defaults to disabled to avoid applying the TCP SYN threshold to
  Bedrock traffic.
- Added bounded parsing for up to two VLAN tags and IPv6 Hop-by-Hop, Routing,
  Destination Options, and AH extension headers. IPv6 extension traversal is
  limited to six headers and 256 bytes.
- Added structural checks for recognized TCP/UDP headers and packet bounds.
- Chose a fail-closed fragment policy: drop IPv4 fragments and IPv6 packets with a
  Fragment header. No fragment state, correlation, or reassembly was added.
- Added IPv6 address canonicalization/enforcement to the edge agent and wired the
  UDP game-port option through the command and service example configuration.

The separate IPv6 maps preserve the existing IPv4 pinned-map key/value layouts.
The loader still refuses to replace an already-attached XDP program.

### Known dataplane boundaries

- Fragmented legitimate traffic, including IPv6 atomic fragments, is dropped.
- IPv6 extension traversal beyond the stated bounds, Fragment headers, ESP, and
  selected unsupported/tunneled paths are dropped. This is not a general IPv6
  firewall policy.
- Unknown direct IPv6 next-header values are not all denied by the current policy.
- The TCP rate map counts SYN traffic; it is not TCP handshake state, SYN cookies,
  source validation, or a DDoS mitigation for upstream link saturation.
- UDP handling is per-source packet-rate filtering only. It does not parse Bedrock
  RakNet, perform UDP session management, or relay Bedrock traffic.
- UDP packet limiting is not Bedrock-aware, and no default threshold is claimed to
  be safe for real Bedrock clients. Enable it only with an operator-chosen,
  measured threshold. Its state maps have the same finite per-family capacity as
  the TCP source maps.

## Stage C live-path test

Added [`scripts/phase3g_wsl_e2e.sh`](./scripts/phase3g_wsl_e2e.sh). It has explicit
WSL2, root, confirmation, and pre-existing-resource guards. It builds temporary
command binaries, creates a private IPv4/IPv6 veth test network, mounts a temporary
bpffs, starts the XDP agent, ingress gateway, and mock status backend directly, then
cleans up the processes, interface, namespace, map pins, mount, and temporary
directory.

The actual test path was:

```text
namespace client -> c3ghost generic XDP -> Go ingress -> mock status backend
                       |
                       +-> authenticated edge-agent block/unblock API
```

### Observed results

| Check | Result |
|---|---|
| Generic XDP attached to isolated veth | PASS |
| IPv4 protocol-47 status handshake and backend response | PASS |
| IPv6 protocol-47 status handshake and backend response | PASS |
| Authenticated IPv4 block stops status traffic; unblock restores it | PASS |
| Authenticated IPv6 block stops status traffic; unblock restores it | PASS |
| UDP rate limit disabled by default and independent cap enforced in kernel test-run | PASS |
| Test namespace/interface, bpffs mount, and pin directory absent after cleanup | PASS |
| Login success through a live backend | NOT TESTED |
| Bedrock UDP end-to-end | NOT TESTED |
| Restricted systemd service startup in WSL2 | NOT PASS in prior Phase 3F.5 validation |
| Updated systemd unit static verification from `/mnt/c` | NOT VALIDATED; verifier reported Windows-mount mode warnings and the expected missing installed executable |
| Dedicated Linux host and production kernel | NOT TESTED |

This validates an actual packet path through the attached XDP program and TCP
Gateway for the listed protocol-47 status cases. It does not validate production
service capabilities, broad Minecraft protocol protection, or production capacity.

## Changed and generated files

Stage B changes in this work area:

- `xdp/unified_xdp.c`
- `xdp/craftsmp_bpfel.go`
- `xdp/craftsmp_bpfel.o`
- `xdp/loader.go`
- `xdp/nonlinux.go`
- `xdp/linux_test.go`
- `xdp/filter_test.go` (formatting)
- `edge/agent.go`
- `edge/agent_test.go`
- `cmd/unified-xdp/main.go`
- `systemd/craftsmp-unified-xdp.service`
- `config/unified-xdp.env.example`
- `scripts/phase3f5_wsl_validation.sh`

Stage C and report changes:

- `scripts/phase3g_wsl_e2e.sh`
- `scripts/phase3g_wsl_xdp_regression.sh`
- `PHASE3G_STAGE_A_ARCHITECTURE_AUDIT.md`
- `PHASE3G_STAGE_BC_VALIDATION_REPORT.md`

## Validation evidence

Environment: WSL2 Ubuntu 24.04, kernel
`6.18.58.1-microsoft-standard-WSL2`, Go `1.24.7 linux/amd64`, Intel Core
i5-12450H (12 logical CPUs), approximately 3.8 GB visible memory, and a 10,240
file-descriptor limit. These measurements are WSL2-only and are not equivalent to a
dedicated Linux host.

Commands run from `unified/`:

```sh
/usr/local/go/bin/go generate ./xdp
/usr/local/go/bin/go test -count=1 ./...
/usr/local/go/bin/go test -race -count=1 ./...
/usr/local/go/bin/go vet ./...
/usr/local/go/bin/go build ./cmd/gateway-bench ./cmd/unified-ingress ./cmd/unified-xdp
GOOS=windows GOARCH=amd64 /usr/local/go/bin/go build ./cmd/gateway-bench ./cmd/unified-ingress ./cmd/unified-xdp
find cmd edge ingress minecraft proxy reputation xdp -type f -name '*.go' -print0 \
  | xargs -0 /usr/local/go/bin/gofmt -l
```

Results:

- `go generate ./xdp`: PASS; BPF object compiled and bindings regenerated.
- `go test -count=1 ./...`: PASS for all nine packages.
- `go test -race -count=1 ./...`: PASS for all nine packages.
- `go vet ./...`: PASS.
- Three command builds for Linux and Windows/amd64: PASS.
- Go formatting check: PASS (no files reported).
- `bash -n` on the Phase 3F.5 and Phase 3G shell harnesses: PASS.
- The privileged XDP regression suite passed after the separate UDP configuration
  map was added: kernel test-run, disabled-by-default UDP limit, configured UDP
  limit, generic/native isolated-veth attachment, map recovery, incompatible-map
  rejection, and map-pressure cleanup. The final pressure test inserted 131,072
  source keys in 141.700 ms and pruned 131,072 expired keys in 239.396 ms; full-map
  insertion failed closed and recovered. The restricted service unit remains a
  separate failure.
- One intermediate kernel-test attempt failed because the independent-window
  assertion changed the UDP window before the IPv4 rate assertion. The test order
  was corrected, and the complete privileged XDP regression script then passed.
- `systemd-analyze verify systemd/craftsmp-unified-xdp.service` from the
  Windows-mounted source returned nonzero because the unit file inherits permissive
  Windows-mount mode bits and `/usr/local/bin/craftsmp-unified-xdp` is not installed
  in this WSL environment. This is **not** a successful unit verification; the
  service runtime remains unvalidated here.

The privileged kernel/map regression suite can be repeated in isolated WSL2 with:

```powershell
wsl.exe --user root -- bash -lc 'CRAFTSMP_PHASE3G_CONFIRM_ISOLATED=YES /mnt/c/Users/saksh/OneDrive/Desktop/ddos/CRAFTSMP-PROTECTION/unified/scripts/phase3g_wsl_xdp_regression.sh'
```

Reproduce the WSL2 live-path test from the Windows workspace:

```powershell
wsl.exe --user root -- bash -lc 'CRAFTSMP_PHASE3G_CONFIRM_ISOLATED=YES /mnt/c/Users/saksh/OneDrive/Desktop/ddos/CRAFTSMP-PROTECTION/unified/scripts/phase3g_wsl_e2e.sh'
```

Only run it in a disposable isolated WSL2 distro. It refuses to run on a non-WSL
kernel and refuses pre-existing test names, an existing bpffs mount, or the
`/sys/fs/bpf/craftsmp` pin directory.

## Remaining work

1. **Dedicated-Linux operational validation:** diagnose the WSL2 systemd XDP load
   failure on a disposable dedicated Linux host, without broadening service
   capabilities to mask it. Validate the unchanged least-privilege unit and clean
   shutdown.
2. **Stage D — independent layer manager:** define typed, bounded policies,
   dependencies, desired/effective state, health, versions, and rollback. Only
   mark a layer active when its enforcement is wired and tested.
3. **Stage E — control-plane backend:** implement versioned APIs, secure
   authentication/authorization, audit records, and bounded durable policy and
   telemetry state before exposing management actions.
4. **Stage F — frontend:** create an administration UI backed only by implemented
   APIs; do not expose kernel actions directly or show decorative controls.
5. **Stage G — integration and security tests:** cover policy authorization and
   rollback, restart/recovery, malformed and incomplete traffic, overload,
   concurrency, expiration, and real supported login behavior. Clearly test and
   document IPv6 and fragment compatibility.
6. **Bedrock path:** design and implement a bounded, authenticated/validated UDP
   path only if required; the present UDP XDP limiter is not Bedrock L7 protection.
7. **Stage H — capacity/readiness:** run reproducible capacity, CPU/RSS, map
   utilization, throughput, and soak tests on labeled dedicated Linux hardware.
   Do not derive production capacity or upstream saturation protection from WSL2
   loopback/veth results.

## Overall Phase 3G status

**PARTIAL — do not mark Phase 3G complete.** The Stage B IPv4/IPv6 dataplane slice
and Stage C protocol-47 status path are implemented and tested within the explicit
limits above. Independent layer orchestration, secure control-plane service,
website, comprehensive adversarial validation, dedicated-Linux service operation,
and capacity/readiness evidence remain incomplete.

**Recommended next phase:** Stage D, preceded by dedicated-Linux verification of
the current least-privilege attachment and map lifecycle. Keep unsupported layers
explicitly unavailable until their enforcement exists.
