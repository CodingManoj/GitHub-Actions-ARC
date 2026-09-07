# What is ARC?

Reference: 

ARC stands for **Actions Runner Controller**.

It's useful when you're using GitHub Actions and want self-hosted runners that run as **ephemeral pods** on a Kubernetes cluster.

## How does it work?

ARC is an operator — an extension to Kubernetes.

With ARC, you don't need to worry about scaling runner pods on your Kubernetes cluster. Everything is automated.

### Prerequisites

You need a GitHub Org (either an Enterprise instance or a public/cloud instance).

### Components

ARC is deployed across two namespaces:

- **GitHub Actions System** — all the ARC system components run here.
- **GitHub Actions** — all the runner pods that execute jobs run here.

The ARC system namespace runs two pods:

1. **Controller Manager Pod**, which includes:
   - Autoscaling RunnerScaleSet Controller
   - Ephemeral Runner Controller
   - Ephemeral RunnerSet Controller
2. **Runner ScaleSet Listener Pod**

### High-level flow

1. The Controller Manager Pod talks to `api.github.com` to resolve the Group ID. It does this by presenting its token, and in return gets a Runner ID.

2. Using that Runner ID, the Listener ScaleSet Pod authenticates with the GitHub Actions service and opens a long-lived HTTPS connection (long polling). The listener then stays idle until it receives a "job available" message from GitHub.

3. When a job matches the runner's `runs-on` label, the listener signals the Ephemeral RunnerSet, which scales up by creating runner pods in the GitHub Actions namespace.

4. Each new pod starts up with the GitHub Actions Runner application and uses a Just-In-Time (JIT) configuration token to register itself as a runner with the GitHub instance. Once registered, the runner requests its job configuration and starts executing the job. During execution, the connection between the runner pod and GitHub stays open as a long poll, and the runner continuously streams job telemetry, logs, and status back to GitHub.

5. When the job finishes, a "completed" signal is sent to GitHub and to the Ephemeral RunnerSet Controller. The controller confirms this with GitHub Actions and then terminates the runner pod — by which point all logs and telemetry have already been delivered.

This is how GitHub Actions runners on a Kubernetes cluster scale up and down automatically.

### Default runner image

The default image used for the runner pod, `ghcr.io/actions/actions-runner`, is intentionally minimal. It ships with just:

- The runner application itself (`Runner.Listener`, `Runner.Worker`)
- Git
- A basic Node.js runtime (needed because many GitHub Actions — like `actions/checkout` — are JS-based and require Node to execute)
- Core OS utilities

Teams typically use this base image as a starting point and build their own custom image on top of it, adding whatever tools their workflows need.


### How installation works ?
Refer install.md



## The GitLab equivalent

GitLab (when self-managed) works in a similar way.

In GitLab, runners are deployed on EC2 instances, each with a `config.toml` file that defines the target Kubernetes cluster, labels, taints, and tolerations.

When a job matches a runner's `config.toml` configuration, that runner schedules a pod in the namespace specified in the config, on the Kubernetes cluster. The job then runs in that pod on EKS, using the container image defined in the job.
