# Docker Fleet acceptance evidence

Copy this template into a private evidence artifact or an approved tracking issue.
Do not publish private source, raw secrets, internal network details, or full container inspection output.

## Execution identity

| Field | Value |
| --- | --- |
| Evaluation ID | NOT_SET |
| Operator and reviewer | NOT_SET |
| Date and timezone | NOT_SET |
| Source commit | NOT_SET |
| Control-plane image digest | NOT_SET |
| Manager image digest | NOT_SET |
| Runner image digest | NOT_SET |
| Binary and coding-CLI versions | NOT_SET |
| Docker, kernel, cgroup, and storage versions | NOT_SET |
| Deployment configuration checksum | NOT_SET |
| Firewall policy checksum | NOT_SET |
| Fixture repository and base SHA | NOT_SET |
| Approved model route and spend ceiling | NOT_SET |
| Current gate | G0 / G1 / G2 / G3 |

## Test record

Repeat this section for every test in the runbook.

| Field | Value |
| --- | --- |
| Test ID | NOT_SET |
| Status | NOT_RUN / PASS / FAIL / BLOCKED |
| Preconditions and workload | NOT_SET |
| Procedure or test command | NOT_SET |
| Expected result | NOT_SET |
| Actual result | NOT_SET |
| Run, task, fleet, and runner IDs | NOT_SET |
| Start and finish timestamps | NOT_SET |
| Sanitized evidence location | NOT_SET |
| Evidence checksum | NOT_SET |
| Defect or blocking issue | NOT_SET |
| Reviewer | NOT_SET |

A source review can support a finding. It cannot replace a runtime test result.
A mock test does not prove daemon-level isolation or cgroup enforcement.

## Aggregate results

| Metric | Measured value | Required target | Result |
| --- | --- | --- | --- |
| Correct deterministic outcomes | NOT_MEASURED | 30 of 30 | NOT_RUN |
| Peak active runner containers | NOT_MEASURED | At most configured capacity | NOT_RUN |
| Cached-image queue-to-start p95 | NOT_MEASURED | Below 60 seconds with capacity | NOT_RUN |
| Terminal cleanup delay | NOT_MEASURED | At most 120 seconds | NOT_RUN |
| Recovery convergence | NOT_MEASURED | At most 5 minutes after recovery | NOT_RUN |
| Cross-task canary leaks | NOT_MEASURED | Zero | NOT_RUN |
| Unauthorized side effects | NOT_MEASURED | Zero | NOT_RUN |
| Unrelated resource deletions | NOT_MEASURED | Zero | NOT_RUN |
| Soak duration and resource growth | NOT_MEASURED | 24 hours; bounded growth | NOT_RUN |
| Coding work-order outcomes | NOT_MEASURED | Explicit acceptance evidence | NOT_RUN |
| Total model spend | NOT_MEASURED | Within the approved ceiling | NOT_RUN |

## Compatibility and residual risks

Record supported Git services, tested model routes, authentication method, and verified authorization boundaries.
List every untested integration. Record the manager token's privilege and the shared-kernel trust boundary explicitly.
Link risk exceptions to an owner, compensating controls, and an expiry or review date.

## Release decision

Decision: **NOT_EVALUATED**.

Permitted alternatives: `G0_ONLY`, `G1_APPROVED`, `G2_APPROVED`, `G3_APPROVED`, or `REJECTED`.

- Passed gates: NOT_SET.
- Failed or blocked mandatory tests: NOT_SET.
- Approved repositories and trust domain: NOT_SET.
- Approved security exceptions: NOT_SET.
- Rollback owner and procedure: NOT_SET.
- Human approver and approval date: NOT_SET.

Do not mark a gate approved when a mandatory test is unexecuted or blocked.
