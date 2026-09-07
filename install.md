# GitHub Actions Runner Controller (ARC) — Quick Notes

## What it is
Kubernetes operator that runs self-hosted GitHub Actions runners as pods, autoscaled based on queued jobs.

## Helm installs (2, minimum)
1. **Controller chart** — once per cluster
   - Deploys the **controller-manager** pod (e.g. in `arc-systems` namespace)
   - No GitHub credentials needed at this step
2. **Runner scale set chart** — once per pool/team/toolchain (repeatable)
   - Creates an `AutoscalingRunnerSet` resource
   - Requires GitHub auth (PAT or GitHub App) + `githubConfigUrl` (org/repo/enterprise)
   - Triggers creation of the **listener pod**

## Core components
| Component | Created by | Role |
|---|---|---|
| Controller-manager pod | Controller chart | Calls GitHub API to register scale set, fetch runner group ID |
| Listener pod | AutoscalingListener controller (per scale set) | Long-polls GitHub Actions Service for "Job Available" |
| EphemeralRunnerSet | Controller | Scaled up/down by listener via K8s API patch |
| EphemeralRunner controller | Controller | Requests JIT config, creates runner pod |
| Runner pod | EphemeralRunner controller | Runs the actual job |

## Job flow (end to end)
1. Workflow triggered → GitHub dispatches job to scale set matching `runs-on`
2. Listener pod gets "Job Available" → acknowledges → patches EphemeralRunnerSet replica count (via K8s API, using its ServiceAccount/Role)
3. EphemeralRunner controller requests a **JIT config** (single-use, pre-registers runner + embeds a short-lived credential) from GitHub API
4. Runner pod created, JIT config injected as a Secret/env var
5. Runner binary starts with `--jitconfig <blob>` → registers **and** starts in one step (no separate config.sh/run.sh)
6. Runner picks up job, executes steps, streams status/logs back via long-poll
7. On completion: GitHub auto-deregisters the ephemeral runner (service-side, since `--ephemeral` = one job only); ARC also calls `RemoveRunner` as a backup/cleanup, then deletes the pod
8. Scale set scales back down

## Authentication (not a per-runner token!)
- One PAT (`repo` + `admin:org` scopes) or GitHub App (App ID + Installation ID + private key), set **once at scale-set install time**
- No manual per-runner registration token loop — GitHub's JIT mechanism replaces that entirely
- Runner groups (optional, for RBAC) are set via `runnerGroup:` in `values.yaml` — group must already exist in GitHub, ARC just joins it

## Images involved
| Image | Who builds it |
|---|---|
| Controller-manager / listener | GitHub (`ghcr.io/actions/gha-runner-scale-set-controller`) — used as-is |
| **Runner pod** | **Your team** — extend `ghcr.io/actions/actions-runner` with Bazel/Node/Java/Maven/Docker CLI etc. This is the one mandatory custom image. |
| DinD sidecar (if used) | Usually stock `docker:dind`, sometimes hardened by platform team |
| Job container (`container:` in workflow YAML) | Chosen per-workflow by the app team, not platform-owned |

## Key gotcha
- If a workflow step runs with **no `container:` block**, it executes directly in the runner pod's own filesystem — so any tool it needs (Maven, etc.) must already be baked into the runner pod image, or installed at runtime via a `run:`/`setup-*` step.
- Docker CLI in the image ≠ working `docker build`. Need a daemon: DinD sidecar, host socket mount (DooD), or a daemonless builder like Kaniko.

## Mapping to GitLab K8s executor (for reference)
| GitLab | ARC |
|---|---|
| Runner registration token | PAT/GitHub App creds (once, at scale-set install) |
| `config.toml` default image | Runner pod's own custom image |
| Job `image:` keyword | Workflow's `container:` block |
| Runner tag/group in `config.toml` | `runnerGroup:` in `values.yaml` |
| One runner process per EC2 | One scale-set install per pool |
