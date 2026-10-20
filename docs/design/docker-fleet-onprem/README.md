# On-prem Docker Fleet: PoC and implementation

Status: proposed specification. No deployment or runtime test is complete.

Date: 2026-10-20.

Source baseline: `r2d-ai/superplane@c7405227a17b25b04c8f274ab779ff93bb719ba4`.

## Objective

Run SuperPlane coding tasks on dedicated Docker hosts inside Proxmox VMs.
Use the existing Docker provider and runner protocol. Do not add a cloud dependency or replace the workflow engine.

A successful task must produce verifiable results. Starting a container is not sufficient.
A production deployment must also control resource use, credentials, network access, task scope, and recovery.

## Documents

| Document | Purpose |
| --- | --- |
| [Specification](spec.md) | Architecture, source findings, security boundaries, configuration contract, and acceptance gates |
| [PoC runbook](poc-runbook.md) | Baseline setup, split-host validation, fault tests, security tests, and stop conditions |
| [Implementation plan](implementation-plan.md) | Ordered work packages, code locations, tests, and implementation-agent instructions |
| [Evidence template](evidence-template.md) | Reproducible test records and a release decision |

Read the specification before implementation. Use the runbook to collect evidence, not to assume a passing result.

## Selected approach

- Keep one active Fleet Manager per dedicated Docker host.
- Assign a distinct fleet ID to each host. Do not treat one fleet as an automatic multi-host scheduler.
- Run each coding task in a fresh, non-root runner container.
- Keep the Docker socket and installation administration token outside coding containers.
- Use separate execution for Docker image builds. Do not fix build failures by mounting the host socket into an agent.
- Keep the control plane outside worker VMs after the baseline experiment.

Use synthetic repositories and test credentials first. Do not import production source, personal CLI sessions, or shared administrator Git credentials.

## Delivery gates

| Gate | Result required | Data allowed |
| --- | --- | --- |
| G0: baseline | Existing development stack executes a synthetic task and records its result | Disposable fixtures only |
| G1: Docker PoC | Standalone manager and runner images pass split-host, limit, lifecycle, and basic isolation tests | Disposable fixtures only |
| G2: controlled pilot | Identity, repository ACL, secret, egress, independent verification, and budget tests pass | Explicitly approved internal repositories |
| G3: production | All required tests pass; restore, rollback, monitoring, and operational ownership are approved | Repositories approved for this deployment |

G0 does not approve G1. G1 does not approve multi-user use. Authentication alone does not approve G2.

The work packages are implementation scope, not proof of implementation. All acceptance tests start as `NOT_RUN`.

## Scope limits

This change specifies Docker Fleet support. It does not implement a Proxmox provider, Kubernetes, a new scheduler, or a new agent organization.
It does not grant agents permission to create projects, hire agents, change access policies, merge code, or deploy production.

Native Keycloak login, a full Gitea product integration, and Cursor subscription support are separate compatibility decisions.
Do not claim these capabilities from the presence of a CLI in a runner image.
A GitHub-only fixture can pass G1. Gitea repositories must pass their own authorization and delivery tests before onboarding.

## Working agreement

Create implementation PRs in the order described in the plan.
Keep upstream-compatible changes small. Use the repository Make targets and Semaphore conventions.
Do not commit generated API code, real credentials, internal addresses, or raw private-source logs.
Do not merge or deploy without the repository owner's approval.
