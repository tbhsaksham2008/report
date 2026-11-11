# Phase 3G — Stages D–F Policy Control Plane

## Verdict

**PARTIAL.** The unified XDP process now has a durable, versioned rate-policy
manager, an authenticated HTTP API, and an embedded browser dashboard. The manager
can change the implemented TCP SYN and optional UDP source-rate settings. It does
not provide independent toggles for all 15 catalog layers, user accounts/RBAC, or a
general policy engine. WSL2 tests do not establish dedicated-Linux or production
readiness.

No production host or public service was used.

## Implementation

```text
Administrator browser
  -> embedded dashboard (same-origin, no persistent token storage)
  -> authenticated control-plane API
  -> optimistic-version policy manager
       -> XDP loader rate-config map updates
       -> atomic JSON state file and bounded audit history
```

### Policy manager

- Validates the mandatory TCP SYN threshold/window and independent optional UDP
  threshold/window before applying a change.
- Rejects stale expected versions and records successful changes in a 512-entry
  bounded audit trail.
- Applies updates through the loader, persists state using a same-directory
  temporary file, restrictive file mode, file sync, atomic rename, and directory
  sync, then publishes the new in-memory version.
- Rolls back enforcement when persistence fails before the state-file rename.
  If the rename succeeds but the directory sync fails, the new policy remains
  active and visible in memory; the API returns the committed snapshot with an
  explicit durability warning. This avoids reporting a failed update while leaving
  a different policy active in the kernel.
- Rejects non-regular or group/world-writable state files, unknown JSON fields,
  oversized files, invalid policy values, and invalid stored versions.
- An empty `-policy-file` disables durable state and leaves policy/audit history
  process-local.

The policy-state file is separate from the XDP pinned maps. Persisting a rate
policy does not make reputation blocks durable.

### API and dashboard

| Route | Behavior |
|---|---|
| `GET /api/v1/status` | Bearer-authenticated health, policy, audit, and 15-layer inventory. Returns HTTP 503 with the truthful layer state when XDP is unhealthy. |
| `GET /api/v1/rates` | Authenticated current policy; returns unavailable if XDP health fails. |
| `PUT /api/v1/rates` | Authenticated, same-origin, bounded JSON update with optimistic version; rejects conflicts and invalid policy. |
| `GET /api/v1/audit` | Authenticated bounded policy audit history. |
| `/`, `/app.js`, `/app.css` | Embedded same-origin dashboard assets with restrictive response headers. |

Remote non-loopback HTTP listeners are rejected by the command; a certificate
and key are required for non-loopback listeners. The UI keeps its bearer token in
memory only and clears it on page reload. Browser mutations require same-origin,
and API mutation requests are rate limited. The service uses a shared bearer
credential with the edge agent; there are no individual identities, roles, or
per-user audit identities.

The dashboard labels unavailable/partial layers and exposes controls only for the
rate policy this process can actually update. The 15-layer catalog is descriptive,
not a set of independently configurable switches.

## Files created or modified

- `policy/manager.go` — versioned policy state, validation, persistence, audit,
  rollback, and durability-uncertain handling.
- `policy/catalog.go` — 15-layer inventory and desired/effective XDP state.
- `policy/xdp_adapter.go` — policy-to-loader integration.
- `policy/manager_test.go` — persistence, recovery, validation, conflict, rollback,
  health-state, and catalog tests.
- `controlplane/server.go` — authenticated API, origin checks, bounded request
  decoding, rate limiting, headers, and embedded assets.
- `controlplane/server_test.go` — API authorization, policy update, conflict,
  health, malformed request, mutation-limit, and UI tests.
- `controlplane/web/index.html`, `controlplane/web/app.js`,
  `controlplane/web/app.css` — the embedded operational dashboard.
- `xdp/loader.go`, `xdp/nonlinux.go`, `xdp/linux_test.go` — runtime TCP/UDP
  rate-map configuration and platform stub/regression coverage.
- `cmd/unified-xdp/main.go` — policy manager, API, and dashboard startup wiring.
- `systemd/craftsmp-unified-xdp.service`,
  `config/unified-xdp.env.example` — policy-state and UDP-window configuration.
- `scripts/phase3g_wsl_e2e.sh` — authenticated control-plane/UI-asset and policy
  update/conflict checks added to the isolated dataplane integration harness.
- `PHASE3G_STAGE_A_ARCHITECTURE_AUDIT.md`,
  `PHASE3G_STAGE_BC_VALIDATION_REPORT.md` — status addenda.

## Validation results

Environment: WSL2 Ubuntu 24.04, kernel
`6.18.58.1-microsoft-standard-WSL2`, Go `1.24.7 linux/amd64`. This is not a
dedicated Linux host.

Commands run from `unified/`:

```sh
/usr/local/go/bin/go test -count=1 ./...
/usr/local/go/bin/go test -race -count=1 ./...
/usr/local/go/bin/go vet ./...
/usr/local/go/bin/go build ./cmd/...
GOOS=windows GOARCH=amd64 /usr/local/go/bin/go build ./cmd/...
bash -n scripts/phase3g_wsl_e2e.sh scripts/phase3g_wsl_xdp_regression.sh
```

All commands passed. The full Go suite and race suite passed for all 11 packages.
The browser JavaScript passed `node --check`; the browser smoke test verified the
bearer header, connected dashboard state, and rendered layer table against a
mocked status response.

One full-suite run concurrent with other validation jobs observed a transient EOF
in the existing `ingress` reconnect-throttle test. The test passed in isolation,
and the complete uncached suite passed on a subsequent uncontended run; no test
was weakened or changed to mask the transient.

Privileged, isolated WSL2 commands:

```sh
CRAFTSMP_PHASE3G_CONFIRM_ISOLATED=YES bash scripts/phase3g_wsl_e2e.sh
CRAFTSMP_PHASE3G_CONFIRM_ISOLATED=YES bash scripts/phase3g_wsl_xdp_regression.sh
```

Both passed when run as root in WSL2. The live-path harness additionally verified:

- The authenticated status API returned a healthy XDP state and all 15 layers.
- The embedded UI and JavaScript assets were served.
- A version-1 rate policy update was applied; a stale version was rejected with
  HTTP 409.
- IPv4 and IPv6 protocol-47 status handshakes traversed generic XDP, the Go
  ingress, and the mock backend.
- Authenticated IPv4 and IPv6 block/unblock operations stopped and restored the
  corresponding test traffic.
- The XDP regression suite passed kernel packet tests, runtime rate-map updates,
  generic/native isolated-veth attachment, pinned-map recovery and compatibility,
  and map pressure/expiry recovery. It filled and expired 131,072 test keys.

These are WSL2/isolated-loopback or private-veth results only. They are not
dedicated-Linux service tests, Internet tests, production capacity measurements,
or proof of protection against upstream bandwidth saturation.

## Remaining work and limitations

- Validate the restricted systemd service on a dedicated Linux host with the
  required kernel, capabilities, bpffs, service file permissions, and installed
  binaries. Prior WSL2 systemd XDP startup did not pass.
- Add an operator identity/RBAC model and per-identity audit attribution if
  multi-administrator operation is required. The current service has one shared
  bearer token.
- Extend control-plane management only when the corresponding enforcement exists.
  Gateway admission, prefix policy, backend policy, and Minecraft inspection are
  not dynamically managed by this API.
- Validate state-file ownership and threat model for the actual deployment account
  and parent directory. The current checks reject symlinks/non-regular files and
  group/world-writable files, but do not implement a multi-user authorization
  system.
- Confirm policy-update rollback behavior on real kernel/map update failures;
  deterministic unit tests cover application and persistence rollback, and kernel
  tests cover successful runtime map updates.
- Run the full adversarial and long-duration matrix on dedicated Linux before
  making capacity or production claims.
- Bedrock UDP remains packet-rate filtering only: no RakNet parsing, UDP session
  relay, or end-to-end Bedrock validation. UDP limiting remains disabled by
  default.
- IPv4 fragments and IPv6 Fragment-header packets are dropped without reassembly.
  This may affect legitimate fragmented traffic.
- Minecraft L7 remains limited to the Gateway's implemented versions and stages.
  The XDP program does not reassemble TCP or inspect Minecraft streams; encrypted,
  compressed, configuration, and play traffic must not be described as fully
  inspected.
- No production readiness, public-network capacity, or upstream-bandwidth DDoS
  protection is claimed.

**Recommended next phase:** dedicated-Linux service and control-plane validation,
then targeted security/adversarial tests for only the supported dataplane and
Minecraft protocol scope. Do not start a deployment installer or claim Phase 3G
complete before those checks pass.
