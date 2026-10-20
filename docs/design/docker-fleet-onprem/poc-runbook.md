# PoC runbook: Docker Fleet on Proxmox VMs

Status: procedure only. No test in this document has run as part of the specification change.

Use the [specification](spec.md) as the acceptance contract.
Record every test with the [evidence template](evidence-template.md).

## 1. Safety and prerequisites

Use dedicated, disposable VMs. Do not run fault or exhaustion tests on shared production Docker hosts.
Use a synthetic repository, synthetic secrets, and a restricted test Git identity.
Do not mount company source, administrator HOME, production volumes, or a personal coding-subscription session.

The operator must approve the VM, firewall, credentials, and real-model spending ceiling before execution.
These approvals do not grant permission to change production infrastructure.

Suggested starting capacity, not a measured minimum:

| VM | Starting allocation | Purpose |
| --- | --- | --- |
| Baseline lab | 8 vCPU, 16 GiB RAM, sufficient image/build storage | Disposable development experiment only |
| Control plane | 4 vCPU, 8 GiB RAM, persistent storage | Compiled service, database, queue, and evidence |
| Worker A | 8 vCPU, 16 GiB RAM, separate Docker storage | Two runners, each capped at 2 CPU and 4 GiB RAM |
| Worker B | Same profile, added later | Independent fleet and failure-domain test |

Reserve host capacity for the manager, daemon, page cache, logs, and operations.
Do not overcommit memory based only on average task use.
Keep at least 20 percent of the worker storage free during acceptance.

Record these facts before tests:

```sh
git rev-parse HEAD
docker version
docker info
docker compose version
uname -a
```

Redact hostnames, registry credentials, internal addresses, and other sensitive fields before publishing evidence.
Record the storage driver, cgroup version, architecture, and firewall backend explicitly.

## 2. G0: reproduce the baseline

The commands in this section exist at the inspected baseline.
Use a fresh checkout in the disposable baseline VM.

```sh
git switch --detach c7405227a17b25b04c8f274ab779ff93bb719ba4
make dev.up
make dev.setup
make dev.server
make doctor-local
```

Complete owner setup in the private lab interface.
Follow the pinned repository setup instructions for fleet creation and development-token initialization.
Do not copy that token bootstrap into the production manager.

Create a manual test workflow through the supported UI or documented API.
Select the integrated runner path and the lab Docker fleet.
Start with a shell task, not an LLM call.
Use host execution inside the runner container.

Example successful task payload:

```sh
set -eu
id
printf 'fleet-smoke-ok\n'
node -e 'console.log(JSON.stringify({ok: true, check: "smoke"}))'
```

Example failed task payload:

```sh
printf 'expected-test-failure\n'
exit 17
```

Confirm that the successful task records success and the failed task records failure.
Confirm that logs and task state remain visible after runner cleanup.
Record container creation, registration, completion, and deletion timestamps.

G0 proves only that the existing development path works.
Do not call G0 a secure deployment or a production configuration.
Stop before real data or unrestricted employee access.

## 3. G1: split-host Docker PoC

This section requires work packages W1 through W3 from the [implementation plan](implementation-plan.md).
The proposed production files and Make targets do not exist in the specification-only change.
Do not substitute development settings to make a missing implementation appear complete.

### 3.1 Build and bootstrap

1. Build the standalone manager image with the Docker CLI and no development database helper.
2. Build the coding runner image with pinned tools and a non-root user.
3. Export image digests and binary versions into the compatibility manifest.
4. Deploy the compiled control plane on its own VM.
5. Enable the integrated runner API, task cleanup, log retention, and required workers for the selected version.
6. Place the manager on Worker A with its dedicated daemon.
7. Inject the manager credential through private deployment configuration.
8. Create or synchronize only the intended installation fleet.
9. Apply network and capacity policy before admitting any task.
10. Start the manager with one fleet, `warmCapacity: 0`, and `maxCapacity: 2`.

The runner must reach the control plane through its private HTTPS origin.
Test certificate trust, WebSocket operation, and reconnect behavior.
Do not use disabled TLS verification or a public unauthenticated Docker endpoint.

The manager must not need database credentials or direct PostgreSQL access.
The coding runner must not contain the manager token, a Docker socket, or remote-daemon credentials.

### 3.2 Inspect effective settings

Use operator access to identify the disposable runner container.
Never give these Docker commands to the coding agent.

```sh
CID='<runner-container-id>'
docker inspect --format '{{.HostConfig.NanoCpus}}' "$CID"
docker inspect --format '{{.HostConfig.Memory}} {{.HostConfig.MemorySwap}}' "$CID"
docker inspect --format '{{.HostConfig.PidsLimit}}' "$CID"
docker inspect --format '{{.Config.User}} {{.HostConfig.ReadonlyRootfs}}' "$CID"
docker inspect --format '{{json .HostConfig.SecurityOpt}}' "$CID"
docker inspect --format '{{json .HostConfig.CapDrop}}' "$CID"
docker inspect --format '{{json .HostConfig.LogConfig}}' "$CID"
```

Inspect mounts, network settings, and namespaces privately.
Do not publish full container inspection output; arguments or environment fields can contain bootstrap credentials.
Use an allowlist-based evidence collector rather than a blacklist of known secret names.

Test writes to the task workspace and writes outside writable mounts.
Only the permitted task paths must be writable.
Test the smallest supported resource profile with a bounded allocation fixture.
Confirm the task fails without killing the manager, daemon, or another task.
Do not use an unbounded fork bomb or uncontrolled host-wide disk fill.

## 4. Required acceptance matrix

All rows begin as `NOT_RUN`. A documented explanation does not convert a blocked test to a pass.
Tests marked G2 or G3 remain release gates even if G1 passes.

| ID | Gate | Procedure | Pass condition |
| --- | --- | --- | --- |
| F01 | G0 | Run the success and exit-17 fixtures | Correct exit status and retained logs |
| F02 | G1 | Register and run from Worker A through HTTPS | No cloud dependency, no dev helper, correct fleet identity |
| F03 | G1 | Run 30 sequential deterministic fixtures | 30 correct outcomes, no unexplained runners |
| F04 | G1 | Queue 10 bounded tasks with capacity 2 | At most 2 active containers; queue drains correctly |
| F05 | G1 | Run a sleeping fixture past its task deadline | Timed-out result; no remaining child process |
| F06 | G1 | Cancel a running fixture | Workload stops within 60 seconds; state remains canceled |
| F07 | G1 | Restart only the manager during a running task | Existing ownership is recovered; no duplicate task delivery |
| F08 | G1 | Kill a runner container during execution | Explicit failure or approved retry; no false success |
| F09 | G1 | Interrupt the create response after container creation | Existing owned container is reconciled; no duplicate side effects |
| F10 | G1 | Restart the worker daemon, then the worker VM | Task state converges; unrelated resources remain untouched |
| F11 | G1 | Block worker-to-control-plane traffic temporarily | No speculative runaway allocation; deadlines still apply |
| F12 | G1 | Fail image pull or runner registration | Bounded retry/backoff; no accumulating pending resources |
| F13 | G1 | Delete an already removed owned container | Cleanup remains idempotent |
| F14 | G1 | Create an unrelated sentinel container and volume | Cleanup never changes the sentinels |
| F15 | G1 | Drain the fleet with one busy and one queued task | Busy task finishes; no new allocations until resume |
| F16 | G1 | Start a second manager for the same daemon | Startup fails before reconciliation |
| F17 | G1 | Test CPU, memory, PIDs, temporary space, and log limits | Effective cgroup and storage behavior matches policy |
| F18 | G1 | Fill only a bounded disposable task filesystem | Task fails safely; host retains free space |
| S01 | G1 | Check non-root user, capabilities, rootfs, namespaces, and mounts | All hardened properties match the specification |
| S02 | G1 | Attempt local and remote Docker access from the task | No daemon access; no nested Docker fallback |
| S03 | G1 | Run two tasks with distinct canary files and HOME paths | Neither task can read or modify the other's data |
| S04 | G2 | Submit as user A using user B's task, artifact, or credential IDs | All unauthorized UI and API requests fail |
| S05 | G2 | Attempt an unauthorized repository or Git write | Git service and dispatch policy reject the operation |
| S06 | G2 | Revoke access between queue and dispatch, then before retry | Dispatch and retry recheck authorization |
| S07 | G2 | Probe approved and denied destinations over IPv4 and IPv6 | Only policy-approved connections succeed |
| S08 | G2 | Probe direct IP, DNS, host gateway, and sibling containers | No proxy bypass or internal-management access |
| S09 | G2 | Emit synthetic secrets through error and output paths | No canary reaches unauthorized logs or artifacts |
| S10 | G2 | Replay an expired or consumed registration token | Registration fails without exposing another task |
| S11 | G2 | Use runner credentials against admin and sibling-task APIs | Access is rejected |
| S12 | G2 | Request a privileged image, mount, fleet, or nested Docker mode | Policy rejects the request before execution |
| S13 | G2 | Request a new project, new agent, broader credential, or production deployment | No unauthorized side effect occurs |
| S14 | G2 | Change verification commands or replace test evidence | Trusted verification still runs against the selected commit |
| S15 | G2 | Exhaust retry, runtime, and approved model budgets | Run stops; no silent paid-provider fallback |
| S16 | G2 | Retry after an uncertain Git push or PR creation | Delivery is reconciled; no duplicate PR or branch corruption |
| O01 | G3 | Add Worker B with a different fleet ID | Explicit routing works; host failure stays contained |
| O02 | G3 | Perform a 24-hour deterministic soak | Stable resource use; no orphan growth or capacity violations |
| O03 | G3 | Restore control-plane state and artifacts in an isolated VM | Selected task history and required secrets are recoverable |
| O04 | G3 | Drain, update images, then roll back | Versions and task ownership remain consistent |
| O05 | G3 | Trigger queue, disk, cleanup, and registration alerts | Named operator receives an actionable alert |
| O06 | G3 | Disable one manager and exceed a task's absolute deadline | Independent watchdog stops overdue owned resources only |

Fault injection must stay inside the approved lab scope.
Use a proxy or fake client to reproduce lost responses before attempting live network faults.
Do not modify shared infrastructure routing to simulate a worker failure.

## 5. G2: real coding experiment

Do not start this section until infrastructure and required security tests pass.
Use one approved small repository with a reproducible defect and independent acceptance tests.
GitHub and Gitea are separate compatibility results; do not infer one from the other.

Prepare three work orders:

| Fixture | Input | Required output |
| --- | --- | --- |
| Bug fix | A failing regression test and an allowed source area | Minimal fix, passing independent test, draft PR |
| Small feature | Explicit acceptance criteria and permitted dependencies | Implementation, tests, API behavior evidence, draft PR |
| Refactor | Passing baseline suite and a limited refactor goal | Unchanged public behavior and passing verification |

Use one approved model route initially. Do not compare several subscriptions during infrastructure validation.
Select a hard spend ceiling, runtime ceiling, and retry ceiling before dispatch.
Record the exact provider and model used; fail on unauthorized fallback.

The agent must not create another project, invent a hiring workflow, or continue unrelated work.
The workflow must not let the agent approve its own change.
Review test modifications and protected-path changes separately from the test result.

Measure completion correctness, human interventions, end-to-end duration, model spend, rework cycles, and scope violations.
These metrics evaluate the coding workflow. They do not replace the infrastructure acceptance matrix.

## 6. Evidence and stop conditions

Store sanitized results under the proposed private evidence artifact location.
Do not put confidential logs or screenshots into this public repository.
Each case needs a test ID, commit, image digests, run ID, timestamps, expected result, actual result, and evidence checksum.

Stop immediately when any task can access the Docker daemon, manager token, another user's secret, or unauthorized source.
Also stop on uncontrolled spend, production-network access, duplicate protected side effects, or loss of authoritative task state.
Disable new dispatch. Preserve evidence and revoke the affected test credentials.
Do not continue by widening permissions or bypassing a failing check.

A failed test creates an implementation defect linked to its evidence.
A blocked test remains `BLOCKED` and prevents its release gate.
Do not convert an unexecuted test to `PASS` from source inspection alone.

## 7. Cleanup and handover

Drain the experiment fleet. Wait for task acknowledgement and evidence retention.
Remove only resources whose ownership matches this deployment and run.
Revoke test credentials. Remove sensitive temporary files and disposable VM disks through the approved process.
Keep the sanitized evidence bundle, compatibility manifest, and release decision.

Do not run global Docker cleanup commands or delete control-plane data as a shortcut.
The final handover must identify passed gates, failed tests, untested integrations, approved risks, and operational owners.
