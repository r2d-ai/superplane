# Implementation plan: on-prem Docker Fleet

Status: planned work. No package below is implemented by this documentation change.

Baseline: `c7405227a17b25b04c8f274ab779ff93bb719ba4`.

Use the [specification](spec.md) and [acceptance matrix](poc-runbook.md) together.
Keep the first release small: one dedicated worker host, one fleet, two concurrent synthetic tasks.

## 1. Dependency order

```text
W0 baseline evidence
  -> W1 standalone packaging
  -> W2 hardened runtime
  -> W3 lifecycle and recovery
  -> G1 review
  -> W4 identity, network, and credentials
  -> W5 controlled coding workflow
  -> G2 review
  -> W6 operations and second host
  -> G3 approval
```

W2 and W3 can develop in parallel after their interfaces are agreed.
Do not deploy either independently with real source.
Each work package should produce a small reviewable PR, not a rewrite of the factory product.

## 2. W0: verify the baseline

- [ ] Pin the source commit and record the fork's differences before implementation.
- [ ] Run the existing Docker development path on a disposable VM.
- [ ] Run success, failure, and runner-cleanup fixtures without an LLM.
- [ ] Record task IDs, container lifecycle, and source-to-image versions.
- [ ] Confirm the integrated runner route; do not mix it with legacy task-broker settings.

Deliverable: G0 evidence with no production-readiness claim.
A baseline failure must be diagnosed before unrelated hardening changes begin.

## 3. W1: standalone packaging

Primary entry points:

- `release/fleet-manager/Dockerfile`
- `release/runner/Dockerfile`
- `cmd/fleetmanager/main.go`
- `Makefile`
- `.semaphore/`

Tasks:

- [ ] Add a dedicated manager target, such as `docker-runtime`, with Docker CLI and trusted CA support.
- [ ] Start the manager binary directly. Remove development database polling and automatic development-token creation from this target.
- [ ] Add a coding runner target with only the approved task tools and pinned versions.
- [ ] Keep the existing cloud and development targets working.
- [ ] Add separate control-plane and worker deployment examples with placeholder-only configuration.
- [ ] Use the compiled application; do not run Air, Vite, or development hot reload in the pilot deployment.
- [ ] Provide an explicit fleet bootstrap/synchronization procedure and validate it against the selected API version.
- [ ] Add image smoke tests for binary versions, Docker CLI availability in the manager, and non-root task execution.
- [ ] Produce image digests, dependency inventory, and a compatibility manifest.

Proposed deliverable paths, not existing files:

```text
release/docker-fleet/README.md
release/docker-fleet/compose.control-plane.yml
release/docker-fleet/compose.worker.yml
release/docker-fleet/fleet-manager.example.yaml
release/docker-fleet/compatibility.example.json
```

Acceptance: F02 passes without developer tools, a cloud account, or database credentials on the manager.
Do not resolve missing coding tools by installing them interactively on the production host.

## 4. W2: hardened provider and execution policy

Primary entry points:

- `pkg/fleets/config/config.go` and configuration tests
- `pkg/fleets/provider/docker/provider.go` and provider tests
- `cmd/fleetmanager/main.go`
- `cmd/runner/main.go`
- `pkg/runners/agent/agent.go`

Tasks:

- [ ] Add the typed hardened profile described in the specification.
- [ ] Propagate CPU and memory from fleet policy into provider configuration.
- [ ] Enforce positive capacity, PID, runtime, temporary-storage, and log limits.
- [ ] Build Docker arguments as an argument list. Never interpolate them through a shell.
- [ ] Add non-root user, read-only rootfs, capability drop, no-new-privileges, and namespace restrictions.
- [ ] Create unique bounded writable paths compatible with task HOME, Git, CLI tools, and log spool.
- [ ] Reject arbitrary host binds, extra privileges, host networking, devices, and unapproved images in the hardened profile.
- [ ] Reuse existing runner execution ceilings and spool settings where possible.
- [ ] Reject nested Docker task mode at dispatch and execution for this fleet.
- [ ] Check daemon and cgroup capabilities at startup; fail when requested enforcement is unavailable.
- [ ] Add per-run network creation hooks with fail-closed startup and owned cleanup.

Keep the existing cloud-provider interface stable where possible.
If a shared type changes, update every provider's tests. Do not break cloud operation as a shortcut.

Unit tests must cover argument construction and invalid configuration.
Real-daemon tests must verify effective limits and behavior. Mock argument tests alone do not satisfy G1.

Acceptance: F05, F06, F17, F18, S01, S02, and S03 pass.
An initial network policy may allow only the synthetic fixture endpoints. Broader policy belongs to W4.

## 5. W3: lifecycle, ownership, and recovery

Primary entry points:

- `pkg/fleets/provider/docker/provider.go`
- `pkg/fleets/reconcile/reconciler.go`
- `pkg/fleets/provider/provider.go`, only if the ownership contract requires it
- `pkg/runners/agent/agent.go` and protocol/cleanup code reached by the affected paths

Tasks:

- [ ] Add manager identity to ownership labels and validate ownership before adoption or deletion.
- [ ] Populate resource creation time from Docker inspection.
- [ ] Acquire a per-daemon process lock before reconciliation.
- [ ] Inspect deterministic container names after ambiguous create failures.
- [ ] Treat an already absent owned container as successful deletion.
- [ ] Preserve acknowledged logs and results before terminal cleanup.
- [ ] Test cleanup of owned network and temporary resources without global pruning.
- [ ] Verify capacity under queued, pending, idle, busy, failed, and orphaned conditions.
- [ ] Add bounded provisioning backoff and a repeated-error circuit breaker.
- [ ] Define drain separately from cancellation and force cleanup.
- [ ] Confirm local deadline behavior during control-plane partitions.
- [ ] Implement an independent overdue-runner watchdog before G3.

Reuse existing reconciliation and cleanup mechanisms. Extend only missing behavior demonstrated by tests.
Do not promise exactly-once execution. Use remote-effect reconciliation for ambiguous delivery.

Acceptance: F03 through F16 pass, with no unrelated resource deletion.
The evidence must distinguish a recovered running task from a new retry.

## 6. W4: identity, network, and credentials

First inspect existing authorization interceptors, organization roles, integration credential resolution, and task dispatch.
Document the actual coverage. Do not add a parallel identity system before finding the integration points.

Tasks:

- [ ] Resolve task identity, repository authorization, and credential reference server-side.
- [ ] Apply permission checks to dispatch, retry, subtask creation, logs, artifacts, and delivery.
- [ ] Prevent user-controlled task fields from selecting a more privileged credential or fleet.
- [ ] Keep manager and bootstrap credentials separate from Git and model credentials.
- [ ] Add a registration-token input mechanism that avoids long-lived command-line exposure.
- [ ] Test runner-token scope against admin and sibling-task APIs.
- [ ] Add credential expiry, revocation, and per-run cleanup procedures.
- [ ] Enforce egress on the real worker forwarding path; test direct-IP and IPv6 bypasses.
- [ ] Add an allowlist-based sanitized evidence exporter.
- [ ] Document the manager token's broad authority and obtain an exception or implement a narrower machine identity.

Create two test users and two test repositories. User A must not access user B's protected task data.
Do not treat a successful login or different Git author name as authorization evidence.

Prove repository-specific GitHub and Gitea behavior separately.
A full Gitea UI integration is not required for G1, but an untested adapter cannot serve real Gitea repositories.
Do not bypass missing integration support with a shared administrator token.

Acceptance: S04 through S12 pass. Any unknown cross-user boundary blocks the shared pilot.

## 7. W5: controlled coding workflow

Tasks:

- [ ] Add a fixture repository or fixture generator with an independently verifiable defect.
- [ ] Add an approved issue-to-branch workflow with one selected model route.
- [ ] Use commit SHAs and explicit artifacts between fresh stage runners.
- [ ] Execute trusted verification outside the coding agent's editable task contract.
- [ ] Enforce at most three code/test cycles and two infrastructure retries initially.
- [ ] Enforce the approved model-spend ceiling through a trusted mechanism.
- [ ] Deduplicate Git push and draft-PR delivery using a stable work-order identity.
- [ ] Stop on missing tests, unauthorized dependencies, policy changes, or out-of-scope project creation.
- [ ] Keep merge and production deployment behind human approval.

Do not add hiring agents, an autonomous project manager, or unrelated demonstration applications.
Do not assume Cursor subscription sessions, Keycloak, or a particular model provider work without a separate compatibility result.

Acceptance: S13 through S16 and the three coding experiments pass with explicit evidence.
Report workflow quality separately from worker infrastructure reliability.

## 8. W6: operational release

Tasks:

- [ ] Add operator metrics, alert rules, queue visibility, and a cleanup dashboard or equivalent report.
- [ ] Add drain, resume, inspect, recovery, and credential-rotation procedures.
- [ ] Add Worker B with its own fleet ID and tested explicit routing.
- [ ] Test host-local isolation, capacity, and failure behavior on both workers.
- [ ] Complete the 24-hour soak and disk-growth analysis.
- [ ] Restore control-plane state, logs, artifacts, and required encryption material into an isolated environment.
- [ ] Validate image rollback without unsafe database downgrade.
- [ ] Document operational owners, remaining security exceptions, and integration limits.
- [ ] Obtain G3 approval before moving approved repositories to production use.

Acceptance: O01 through O06 pass. G3 is a human release decision, not an agent status message.

## 9. Build and test conventions

Use the pinned repository's Make targets. Read `AGENTS.md` and the test-audit skill before changing tests.
The commands below exist at the inspected baseline. Run only the checks relevant to each implementation PR.

```sh
make dev.up
make dev.setup
make format.go
make lint
make check.build.app
make test PKG_TEST_PACKAGES='./pkg/fleets/... ./pkg/runners/...'
```

Use `make test` for the wider backend regression suite before the hardened release.
Add real-Docker acceptance tests to the existing CI structure or an explicitly approved isolated runner.
Do not assume a GitHub Actions pipeline exists. The baseline uses `.semaphore/`.
Do not mount a shared production Docker daemon into a CI job.

If models or migrations change, follow the repository migration generator and model guidance.
Do not hand-create migrations or drop databases to repair a test environment.
If protobufs change, run the repository generation and field checks. Do not commit generated API artifacts.

## 10. Implementation-agent handoff

Use this instruction with the selected work package:

> Read `AGENTS.md` and `docs/design/docker-fleet-onprem/README.md`.
> Read the specification, runbook, and selected work package before editing code.
> Implement only the selected package and its necessary tests.
> Confirm the current source differs from the pinned baseline before applying assumptions.
> Keep Docker and cloud-provider behavior compatible outside the new hardened profile.
> Do not create projects, hire agents, broaden credentials, or change production infrastructure.
> Do not bypass failing tests, missing capabilities, network policy, or budget limits.
> Do not claim a security test passed without executing it in an approved environment.
> Record blocked work and attach sanitized evidence to the PR.
> Open a reviewable PR. Do not merge or deploy it.

## 11. Completion checklist

- [ ] Each requirement maps to a test ID and an implementation change.
- [ ] Each new configuration field has validation and negative tests.
- [ ] Existing development and cloud configurations retain regression coverage.
- [ ] Effective runtime limits are measured on the selected host configuration.
- [ ] Secret, network, repository, and cross-user tests pass before real-data use.
- [ ] Crash and retry tests show no duplicate protected side effects.
- [ ] Cleanup preserves unrelated resources and retained evidence.
- [ ] Production deployment has no development token/bootstrap dependency.
- [ ] Operator documentation matches the built binaries and selected commit.
- [ ] All unsupported integrations and residual risks remain explicit.
