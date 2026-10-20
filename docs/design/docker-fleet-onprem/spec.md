# Specification: on-prem Docker Fleet

Status: proposed. This document defines requirements; it does not certify a deployed system.

Baseline: `c7405227a17b25b04c8f274ab779ff93bb719ba4`, inspected on 2026-10-20.

## 1. Goals and non-goals

Provide an on-prem runner service for a small engineering team. Use Docker hosts inside existing Proxmox VMs.
Start with two concurrent tasks on one worker host. Add a second host only after the first host passes acceptance.

Required outcomes:

1. A work order produces a traceable branch, test result, and draft PR or patch artifact.
2. Each task has bounded CPU, memory, processes, storage, runtime, retries, and model spend.
3. An agent cannot read another task's workspace or credentials through its configured interfaces.
4. A failed task cannot silently pass, duplicate delivery, or continue without a runtime limit.
5. An operator can drain, inspect, restore, and upgrade the service without losing task ownership.

Do not implement cloud provisioning, a Proxmox API provider, Kubernetes, or dynamic cross-host scheduling in this scope.
Do not add autonomous hiring, unrelated projects, automatic production deployment, or a replacement workflow engine.

## 2. Verified source baseline

The following findings come from the pinned source. They are not runtime test results.

| Component | Observed behavior | Consequence |
| --- | --- | --- |
| [Docker provider][src-provider] | Uses `exec.CommandContext` to call the Docker CLI; implements create, list, and delete | Reuse this provider; the manager image needs the CLI |
| [Docker provider][src-provider] | Passes image, labels, network, volumes, extra hosts, and registration arguments | The current provider does not pass CPU, memory, PID, or hardening flags |
| [Docker provider][src-provider] | Lists resources by managed and fleet labels; returns an empty `CreatedAt` | Add reliable ownership and creation-time checks before age-based cleanup |
| [Fleet configuration][src-config] | Supports Docker, fleet specifications, warm capacity, and maximum capacity | Extend typed configuration; do not add an arbitrary Docker-arguments escape hatch |
| [Reconciler][src-reconcile] | Creates ephemeral runners and adjusts capacity; accounts for busy runners | Reuse the reconciler and test its fault behavior |
| [Reconciler][src-reconcile] | Treats `maxCapacity: 0` as unlimited | Reject zero capacity in the hardened deployment profile |
| [Manager image][src-manager-image] | `runtime` contains the manager binary; `local` contains Docker CLI and a development entrypoint | Add a standalone Docker production target; do not ship the development database helper |
| [Runner image][src-runner-image] | `runtime` is minimal; `local` contains coding tools and uses `USER node` | Build a tested coding image; do not assume the minimal image can run coding tasks |
| [Runner command][src-runner-main] | Supports an execution ceiling, log-spool limits, fresh task HOME, and single-use registration | Wire existing controls before adding equivalent mechanisms |
| [Agent execution][src-agent] | Selects host or Docker execution; applies the smaller positive execution ceiling | Test both dispatch policy and actual timeout behavior |
| [Manager command][src-manager-main] | Uses an installation administration token and supports `sync-fleets` | Separate trusted manager credentials from task credentials |

The repository uses Docker-based Make targets and Semaphore CI. Generated API files are not committed. See [repository guidance][src-agents].

### Important execution distinction

`provider: docker` describes where the runner lives.
`execution_mode: host` describes execution inside that runner's own environment.
When the runner is a Docker container, host execution stays inside that container.
It does not mean execution on the Proxmox host.

For the initial hardened coding fleet, allow only this host execution mode inside the runner container.
Reject nested Docker execution before allocation and again inside the runner.
Do not advertise the `docker` task capability for this fleet.
Use a separate build service or disposable build VM for tasks that need a Docker daemon.

## 3. Architecture and trust

```text
Users / approved triggers
           |
           v
Control-plane VM
  SuperPlane API/UI + workflow state
  PostgreSQL + RabbitMQ + durable task artifacts
           |
           | HTTPS / WebSocket; authenticated runner protocol
           |
Worker VM A                         Worker VM B (later)
  Fleet Manager A                     Fleet Manager B
  dedicated Docker daemon             dedicated Docker daemon
  fleet: coding-a                      fleet: coding-b
  fresh runner containers             fresh runner containers
           |
           v
Approved Git service / model gateway / package mirror
           |
           v
Independent verification -> draft PR -> human approval
```

These VM names and fleet IDs are examples. Keep real network details in private deployment configuration.

### Trust boundaries

| Actor | Allowed authority | Must not receive |
| --- | --- | --- |
| Platform administrator | Configure hosts, policies, fleets, and trusted integrations | No claim that the platform hides data from its host administrator |
| Fleet Manager | Manage runner containers on its dedicated daemon | Production application mounts or unrelated host workloads |
| Task submitter | Submit authorized work orders and inspect permitted results | Host settings, unrestricted credentials, or policy changes |
| Coding agent | Work on one authorized task and repository | Docker socket, installation admin token, sibling workspaces, production credentials |
| Verifier | Read the selected commit and execute approved checks | Permission to waive checks or modify their trusted definitions |
| Delivery service | Publish an approved branch or draft PR | Permission to merge protected branches or deploy production |

Container isolation shares a host kernel. It does not provide VM-level separation between mutually hostile tenants.
Use separate worker VMs for different trust domains. Use disposable per-task VMs for workloads that require a stronger boundary.
This specification does not implement those VM lifecycles.

Treat daemon access as host-administration authority. Mount the socket only in the trusted manager environment.
A read-only socket mount does not make the Docker API read-only. See [Docker daemon access][docker-access].

### Host ownership

Use one manager process, one dedicated daemon, and distinct fleet IDs per host.
Hold a host-local process lock before reconciliation. A second manager for the same daemon must fail closed.
Do not run active-active managers against one fleet in this scope.

The workflow must select a fleet explicitly or use an existing selector whose behavior has been tested.
Adding another host does not imply automatic balancing, failover, or a global concurrency limit.
For several fleets on one host, validate the sum of their resource reservations before startup.

## 4. Configuration and runtime contract

### 4.1 Existing fields versus proposed fields

Existing fields include `id`, `superplaneUrl`, `fleets[].provider`, `spec`, `warmCapacity`, `maxCapacity`, and `docker.image`.
The Docker block also supports `architecture`, `runnerApiUrl`, `network`, `volumes`, and `extraHosts` at the baseline.

The following block is a DESIGN CONTRACT. It will not work on the baseline without implementation.
Replace placeholders only in private deployment files. Do not paste this block into an unmodified release.

```yaml
id: worker-a
superplaneUrl: https://factory.example.invalid
installationAdminToken: "" # Inject securely into the manager only.
reconciliation:
  pollTimeout: 30s
  errorRetryInterval: 10s
http:
  requestTimeout: 60s
fleets:
  - id: coding-a
    provider: docker
    runnerVersion: "<version-built-from-the-selected-commit>"
    warmCapacity: 0
    maxCapacity: 2
    spec:
      operatingSystem: linux
      architecture: amd64
      cpuMillicores: 2000
      memoryMb: 4096
      diskGb: 2
      capabilities: [] # No nested Docker capability.
    docker:
      image: registry.example.invalid/runner@sha256:<approved-digest>
      architecture: amd64
      runnerApiUrl: https://factory.example.invalid
      profile: hardened                 # NEW
      pidsLimit: 256                     # NEW
      maxExecutionSeconds: 1800          # NEW
      workspaceTmpfsMiB: 2048            # NEW
      tempTmpfsMiB: 256                  # NEW
      networkIsolation: per-runner       # NEW
      logMaxSizeMiB: 10                  # NEW
      logMaxFiles: 3                     # NEW
```

The proposed fields must use typed validation. Reject unknown keys and unsafe combinations.
Do not permit task payloads to override this policy.
Resolve resource limits from the fleet specification and pass them to the provider explicitly.
Reject an API fleet specification that conflicts with the configured resource policy.

Preserve existing development configurations for upstream compatibility.
The new production deployment must require the hardened profile explicitly.
Do not silently fall back to development settings when validation fails.

### 4.2 Enforced container properties

| Property | Initial hardened requirement | Evidence |
| --- | --- | --- |
| CPU | Map 2000 millicores to a 2-CPU quota | Docker inspect plus cgroup measurement |
| Memory | Map the example memory value to 4096 MiB; document the conversion | Hard cgroup limit and bounded allocation test |
| Swap | Set total memory-plus-swap equal to memory | Inspect and host capability check |
| Processes | Maximum 256 PIDs | Inspect and bounded process test |
| User | Explicit non-root UID/GID; no sudo | Effective UID and image inspection |
| Capabilities | Drop all; no added privileges | Inspect and negative test |
| Privilege escalation | `no-new-privileges`; default seccomp or a reviewed stricter profile | Inspect |
| Root filesystem | Read-only | Write-denial test |
| Writable data | Unique task HOME/workspace, bounded temporary storage, bounded spool | Paths, limits, and deletion evidence |
| Namespaces | No host PID, IPC, or network namespace | Inspect |
| Devices | No host devices or privileged mode | Inspect |
| Docker access | No socket, daemon credentials, or reachable remote daemon | Filesystem and network tests |
| Published ports | None for coding workers | Inspect |
| Image | Approved digest and compatible runner binary version | Build manifest and registration record |
| Runtime | Initial example ceiling: 1800 seconds | Timeout and kill tests |
| Container logs | Rotated and size-bounded | Log-driver configuration and disk test |

Docker resource limits must be measured, not inferred from configuration labels. See [resource constraints][docker-limits].
For rootless Docker, verify cgroup v2 and systemd delegation before acceptance. Unsupported limits must stop startup. See [rootless limits][docker-rootless].

The initial workspace can use a size-bounded tmpfs. Its memory use counts against the container memory limit.
Do not mount the same tmpfs at separate HOME and workspace locations with independent, unbudgeted limits.
Put HOME and the checkout under one writable task root. Keep runtime logs under a separate bounded directory.
Permit execution where build tools require it. Do not apply `noexec` to the build workspace without compatibility tests.

The example `diskGb` describes the workspace bound. It is not a quota for Docker images, host logs, or named volumes.
Do not claim that `diskGb` automatically limits Docker storage.
A disk-backed workspace requires an implemented quota mechanism and the same exhaustion tests before use.

Do not share writable HOME, SSH, package caches, browser profiles, or credentials between tasks.
Shared Git worktrees are not an isolation boundary. Their common Git directory can reveal other worktrees or repository state.
A read-only cache can be evaluated later. It must be scoped by repository and trust domain.

### 4.3 Network policy

Create a network boundary per runner. A shared bridge named `isolated` does not prove isolation.
Apply the deny policy before the task starts. Clean network resources only after their owned container stops.

Workers may reach only the runner API, approved Git endpoints, model gateway, package mirror, and controlled DNS.
Block access to control-plane databases, message brokers, hypervisor management, storage administration, and other worker containers.
Block direct internet access when an approved proxy is required.

A proxy environment variable is not enforcement. Validate rules on the actual Docker forwarding path.
Test IPv4, IPv6, DNS, host-gateway access, link-local addresses, and direct IP bypasses.
Persist and revalidate these rules after daemon or VM restart. See [Docker firewall behavior][docker-firewall].

Allowed model or Git endpoints can still carry source data. Network policy alone cannot prevent exfiltration to every allowed endpoint.
Use approved providers, repository-scoped credentials, restricted destinations, and audit. Record this residual risk.

### 4.4 Credentials and identity

Resolve the effective principal server-side before dispatch:

```text
requester -> organization -> authorized repository -> credential reference
          -> approved workflow version -> fleet policy -> task/run ID
```

Do not trust user IDs, credential IDs, repository URLs, or fleet choices supplied by an agent.
Recheck permissions before dispatch, retry, and delivery. A subtask inherits or reduces authority; it cannot increase authority.
Scheduled tasks use an explicitly approved service principal, not an arbitrary employee's cached session.

The manager can use the existing installation token in an isolated administrator-controlled environment during PoC.
Never inject it into a runner. Record its broad scope as a production risk.
Prefer a dedicated fleet-scoped machine identity in a separate security change when supported.
A global manager token needs an explicit security exception before G3; it is not a multi-user credential.

Registration tokens are single-use bootstrap credentials. The baseline passes them in arguments.
For hardened operation, add a secure file or input channel and remove the token after registration.
Do not replace argument exposure with an unrestricted environment dump.
The runner access token must be limited to its own task operations.
The task and runner share an execution boundary; do not promise to hide runner memory from a compromised task.

Use short-lived, repository-scoped Git credentials where supported.
Otherwise use a dedicated test identity during PoC, then an approved per-user or per-project identity with revocation.
Never copy an administrator's HOME, SSH directory, CLI login cache, or global Git credential helper.
Protect branches server-side. Remove PR creation and repository creation rights from the coding process when a delivery service can own them.

Test UI and API access to task details, logs, artifacts, secrets, and retry controls.
Organization-level roles are not proof of project-level isolation.
If the application cannot enforce the required boundary, separate organizations or installations and test those boundaries instead.
Do not onboard users to a shared deployment with unverified ACL behavior.

## 5. Task lifecycle and correctness

Keep the existing runner state model. Map all proposed behavior to existing states before adding new states.
The intended lifecycle is:

```text
Authorize -> queue -> allocate -> register -> execute -> report -> verify
                                                        |
                                                        v
                                        acknowledge -> cleanup -> retain evidence
```

Required invariants:

- One runner container executes one task in this fleet.
- No successful terminal status exists without an acknowledged result.
- The authoritative task result and retained logs survive runner deletion.
- A successful Docker create with a lost response does not create an unrelated replacement blindly.
- A retry cannot publish a second PR or duplicate a protected side effect.
- Cleanup never selects unrelated containers, volumes, or networks.
- A network partition does not turn an unknown task outcome into success.

Use stable runner identity, deterministic resource names, and ownership labels.
Add manager identity to resource ownership checks. Validate fleet, runner, manager, and image before adopting a resource.
Inspect existing resources after an ambiguous create response. Reject ownership conflicts.
Deleting an already absent owned resource must succeed idempotently.

Do not assume exactly-once execution. Use a stable work-order delivery key and deduplicate external writes.
For an uncertain Git push or PR creation, inspect the remote result before retrying.
Do not automatically rerun an unknown build or deployment side effect.

Separate runtime retry, infrastructure retry, and model rework.
Initial pilot policy: at most two infrastructure retries and three code/test cycles per work order.
Use backoff and a circuit breaker for repeated image, registration, credential, or daemon errors.
Budget exhaustion or an unknown permission condition must stop the run and notify an operator.

A model-spend ceiling must be enforced by a trusted gateway or an equivalent hard control.
A reported token count is not a spending limit. Use wall-clock and attempt limits as additional controls.
Choose the currency ceiling before each real-model experiment; this specification does not grant unlimited API spend.

### Drain, recovery, and cleanup

Drain stops new allocations and lets running tasks finish within their existing deadlines.
Do not implement drain as force-delete of every container.

During API failure, stop speculative new allocation. Preserve state for later reconciliation.
Enforce an absolute execution deadline locally, even when cancellation cannot reach the worker.
Use an independent worker watchdog for overdue containers when the manager fails.
Do not kill an active task merely because the control plane is temporarily unavailable.

Use authoritative termination, verified ownership, and bounded age rules for orphan cleanup.
Collect the final failure evidence before deleting local files where possible.
When a worker is permanently lost, report lost or failed work explicitly. Do not label it complete.

Never use global `docker system prune`, volume deletion, or unfiltered name matching in automatic cleanup.

## 6. Coding workflow and independent checks

The first coding workflow uses an approved fixture issue, one repository, one branch, and a fixed acceptance contract.

```text
Approved issue -> bounded plan -> code -> independent tests
                                      ^        |
                                      | fail   |
                                      +--------+
                           pass -> draft PR -> human review
```

Each stage can use a fresh runner. Transfer state through a commit SHA and explicit artifacts, not a shared writable host directory.
The coding agent cannot edit the trusted verification commands or waive their exit status.
Run verification in a fresh environment with its own reduced credentials.
Record changes to tests, workflow files, dependencies, and protected paths for human review.
A green test alone does not prove that an agent stayed within scope.

Required output: base SHA, result SHA or patch checksum, changed files, test commands, exit codes, artifacts, model usage, and delivery identity.
The final result must identify skipped or unavailable checks.
Do not merge, deploy production, create new projects, or expand repository access automatically.

## 7. Operations and release gates

Record queue age, active runners, allocation failures, registration failures, task duration, timeout count, cleanup backlog, and host capacity.
Also record image digests, task-to-principal mapping, result IDs, log retention, and budget events.
Avoid secrets and raw prompts in operational metrics.

Initial acceptance targets are engineering targets, not measured claims:

| Signal | Target |
| --- | --- |
| Cached-image task start | p95 below 60 seconds while capacity is available |
| Terminal resource cleanup | Within 120 seconds after authoritative completion and log acknowledgement |
| Orphan convergence | Within 5 minutes after connectivity and manager operation recover |
| Cancellation | Stop the workload within 60 seconds; retain a canceled result |
| Capacity | Never exceed the configured active-container limit during normal and fault tests |
| Deterministic reliability | 30 consecutive fixture tasks complete correctly; zero unrelated deletions |
| Secret isolation | Zero canary-secret leaks across tasks, user-visible logs, or published artifacts |
| Pilot soak | 24 hours without unbounded resource growth or unexplained tasks |

Adjust thresholds only in a reviewed change with a reason. Do not rewrite a failing result as a relaxed target silently.

Back up the database, configuration, encryption material, retained logs, and artifacts needed to restore the selected deployment.
Store backups outside the worker pool. Test restoration into an isolated environment.
Pin manager, runner, control-plane, and CLI versions in a compatibility manifest.
Drain before rollout. Roll back to the previous validated images and configuration.
Do not perform an automatic database downgrade; use the approved migration and restore procedure.

G3 requires all mandatory security tests, recovery tests, a restore drill, named operational owners, and approval of residual risks.
Unknown results are blockers. See the [runbook](poc-runbook.md) and [evidence template](evidence-template.md).

## 8. Source and platform references

All source links below use the inspected commit. Recheck them when rebasing implementation.

[src-provider]: https://github.com/r2d-ai/superplane/blob/c7405227a17b25b04c8f274ab779ff93bb719ba4/pkg/fleets/provider/docker/provider.go
[src-config]: https://github.com/r2d-ai/superplane/blob/c7405227a17b25b04c8f274ab779ff93bb719ba4/pkg/fleets/config/config.go
[src-reconcile]: https://github.com/r2d-ai/superplane/blob/c7405227a17b25b04c8f274ab779ff93bb719ba4/pkg/fleets/reconcile/reconciler.go
[src-manager-image]: https://github.com/r2d-ai/superplane/blob/c7405227a17b25b04c8f274ab779ff93bb719ba4/release/fleet-manager/Dockerfile
[src-runner-image]: https://github.com/r2d-ai/superplane/blob/c7405227a17b25b04c8f274ab779ff93bb719ba4/release/runner/Dockerfile
[src-runner-main]: https://github.com/r2d-ai/superplane/blob/c7405227a17b25b04c8f274ab779ff93bb719ba4/cmd/runner/main.go
[src-agent]: https://github.com/r2d-ai/superplane/blob/c7405227a17b25b04c8f274ab779ff93bb719ba4/pkg/runners/agent/agent.go
[src-manager-main]: https://github.com/r2d-ai/superplane/blob/c7405227a17b25b04c8f274ab779ff93bb719ba4/cmd/fleetmanager/main.go
[src-agents]: https://github.com/r2d-ai/superplane/blob/c7405227a17b25b04c8f274ab779ff93bb719ba4/AGENTS.md
[docker-access]: https://docs.docker.com/engine/security/protect-access/
[docker-limits]: https://docs.docker.com/engine/containers/resource_constraints/
[docker-rootless]: https://docs.docker.com/engine/security/rootless/tips/
[docker-firewall]: https://docs.docker.com/engine/network/packet-filtering-firewalls/
