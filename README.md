# K8s Kind Voting App

The Docker example voting app, deployed to a local multi-node Kubernetes cluster
running on [Kind](https://kind.sigs.k8s.io/). Includes cluster bootstrap scripts,
plain Kubernetes manifests, an optional Argo CD / Dashboard / monitoring walkthrough,
and a CI pipeline that scans and publishes the container images.

## Architecture

![Architecture diagram](k8s-kind-voting-app.png)

The application consists of:

- A front-end web app in [Python](/vote) which lets you vote between two options
- A [Redis](https://hub.docker.com/_/redis/) instance which collects new votes
- A [.NET](/worker/) worker which consumes votes and stores them in…
- A [Postgres](https://hub.docker.com/_/postgres/) database backed by a volume
- A [Node.js](/result) web app which shows the voting results in real time

## Repository layout

| Path | Purpose |
|------|---------|
| [`vote/`](/vote) [`result/`](/result) [`worker/`](/worker) | Application source and Dockerfiles |
| [`k8s-specifications/`](/k8s-specifications) | Kubernetes Deployment and Service manifests for every component |
| [`kind-cluster/`](/kind-cluster) | Kind cluster config, `kind`/`kubectl` install scripts, dashboard admin user |
| [`kind-cluster/commands.md`](/kind-cluster/commands.md) | Full walkthrough: cluster, Argo CD, Dashboard, kube-prometheus-stack |
| [`seed-data/`](/seed-data) | Image + script to generate sample votes against the running app |
| [`healthchecks/`](/healthchecks) | Redis/Postgres healthcheck scripts |
| [`.github/workflows/ci.yaml`](/.github/workflows/ci.yaml) | Scan, build, and push the images |

## Prerequisites

- Docker
- `kind` and `kubectl` — install with the scripts in [`kind-cluster/`](/kind-cluster):

  ```sh
  ./kind-cluster/install_kind.sh
  ./kind-cluster/install_kubectl.sh
  ```

## Deploy

```sh
# 1. create a 3-node cluster (1 control-plane + 2 workers, k8s v1.30.0)
kind create cluster --config kind-cluster/config.yml

# 2. deploy the app
kubectl apply -f k8s-specifications/

# 3. access the apps
kubectl port-forward service/vote 5000:5000 --address=0.0.0.0 &
kubectl port-forward service/result 5001:5001 --address=0.0.0.0 &
```

Vote on <http://localhost:5000>, watch results on <http://localhost:5001>.

Tear down with:

```sh
kind delete cluster --name kind
```

### Optional

- **Generate sample votes:** build and run the image in [`seed-data/`](/seed-data)
  against the forwarded `vote` service.
- **Argo CD, Kubernetes Dashboard, monitoring:** step-by-step commands in
  [`kind-cluster/commands.md`](/kind-cluster/commands.md).

## CI

[`.github/workflows/ci.yaml`](/.github/workflows/ci.yaml) runs on pushes and pull
requests to `main`:

1. **Code scan** — Trivy filesystem scan for vulnerabilities and secrets
   (fails on `CRITICAL`/`HIGH`), plus an advisory misconfiguration scan.
2. **Build / scan / push** — for `vote`, `result`, and `worker` in parallel:
   build the image, run a Trivy image scan (fails on `CRITICAL`, `HIGH` is
   advisory), and on non-PR events push to Amazon ECR as
   `590183746102.dkr.ecr.us-east-1.amazonaws.com/demo-app-<service>` tagged
   `latest` and `sha-<short>`.

### AWS setup required

The push job assumes an IAM role via GitHub OIDC (no static keys).

- Create a GitHub OIDC identity provider in account `590183746102`
  (`token.actions.githubusercontent.com`, audience `sts.amazonaws.com`).
- Create an IAM role trusted by that provider for
  `repo:powarprashant/k8s-kind-voting-app:*`, with permissions for
  `ecr:GetAuthorizationToken` (resource `*`), the ECR push actions
  (`ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`,
  `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`,
  `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`), and — for the
  auto-create step — `ecr:DescribeRepositories` and `ecr:CreateRepository`.
- Add the role ARN as the repository **variable** `AWS_ROLE_ARN`.

Region, registry, and repo prefix are set in `env:` at the top of the workflow.
The `demo-app-vote` / `demo-app-result` / `demo-app-worker` repositories are
created automatically on first run if the role has `ecr:CreateRepository`;
otherwise create them manually and drop that step.
