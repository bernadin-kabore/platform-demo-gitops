# Platform Engineering Demo: Getting Started

This runbook brings up the current demo in the order its dependencies require:

1. AWS credentials and tools
2. Terraform state storage
3. AWS networking, EKS, IAM, and ECR
4. Terraform outputs wired into GitOps and service repositories
5. Argo CD and platform add-ons
6. An optional service and Backstage setup

> **Cost warning:** this creates billable AWS resources, including EKS, EC2,
> NAT Gateway, load balancers, storage, logging, and potentially Karpenter
> capacity. Review `terraform plan`, use a non-production AWS account, and
> destroy the demo when finished.

This same runbook is included in all four platform repositories. Commands
assume they have been cloned as sibling directories under one workspace:

```text
YOUR_WORKSPACE/
|-- platform-demo-terraform-modules/
|-- platform-demo-gitops/
|-- platform-demo-backstage/
`-- platform-demo-hello-world-template/
```

In PowerShell, set the workspace once before following the commands:

```powershell
$workspace = "C:\path\to\YOUR_WORKSPACE"
```

## 1. Prerequisites

Install and authenticate:

- Terraform 1.7 or newer
- AWS CLI v2
- `kubectl`
- Helm 3
- Git
- Optional: GitHub CLI (`gh`) for setting repository variables and secrets
- Optional: Node.js/npm for running Backstage locally

Verify the tools:

```powershell
terraform version
aws --version
kubectl version --client
helm version
git --version
```

Authenticate to the AWS account that will own the demo and verify the active
identity:

```powershell
aws configure
aws sts get-caller-identity
```

Use the same AWS account and region for every Terraform step. The examples
below use `us-east-1`.

## 2. Prepare the repositories

The GitOps and Backstage configuration currently contains URLs for the
`bernadin-kabore` GitHub account. If you forked the repositories, replace that
owner with your GitHub user or organization throughout:

- `platform-demo-gitops`
- `platform-demo-backstage`
- `platform-demo-hello-world-template`

Argo CD reads the **remote** GitOps repository. Local changes, including the
current architecture fixes, must be reviewed, committed, and pushed before
Argo CD can see them.

## 3. Bootstrap Terraform remote state

The state backend module must be applied once using local state. Choose a
globally unique S3 bucket name:

```powershell
Set-Location "$workspace\platform-demo-terraform-modules\modules\state-backend"

terraform init
terraform plan `
  -var='bucket_name=YOUR-GLOBALLY-UNIQUE-STATE-BUCKET' `
  -var='lock_table_name=platform-demo-terraform-locks' `
  -out=tfplan
terraform apply tfplan
```

Keep the local state file from this bootstrap module safe. The S3 bucket has
`prevent_destroy = true` to reduce the chance of accidentally deleting all
Terraform state.

Create `platform-demo-terraform-modules/envs/dev/backend.hcl`:

```hcl
bucket         = "YOUR-GLOBALLY-UNIQUE-STATE-BUCKET"
key            = "platform-demo/dev/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "platform-demo-terraform-locks"
encrypt        = true
```

Do not commit `backend.hcl` if it contains environment-specific information
you do not want in source control.

## 4. Configure the dev environment

Copy the example variables file:

```powershell
Set-Location "$workspace\platform-demo-terraform-modules\envs\dev"
Copy-Item terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`. Prefer the `services` map; its keys must exactly match
the GitHub service repository names:

```hcl
aws_region         = "us-east-1"
cluster_name       = "platform-demo"
kubernetes_version = "1.30"
vpc_cidr           = "10.20.0.0/16"
az_count            = 2

# Replace this with the public IP/CIDR used by the operator.
admin_cidrs = ["YOUR_PUBLIC_IP/32"]

# One entry per application the Backstage scaffolder has created. Each produces
# one ECR repository per service, named <application>-<service>, and one CI role
# for the application federated from <application>-source on main. Leave it
# empty until you have scaffolded something.
applications = {}

# The platform's own single-image repositories, which are not applications and
# keep their repository's own name because platform-demo-gitops references them
# that way.
platform_services = {
  platform-demo-ai-agent = { github_owner = "YOUR_GITHUB_OWNER" }
}

owner = "YOUR_NAME_OR_TEAM"
```

Before applying, verify that the configured Kubernetes version is supported
by EKS in the selected region and understand whether extended-support charges
apply. Update `kubernetes_version` when needed.

## 5. Provision AWS infrastructure

Initialize this root against the backend created above:

```powershell
Set-Location "$workspace\platform-demo-terraform-modules\envs\dev"

terraform init -backend-config=backend.hcl
terraform fmt -recursive -check ..\..
terraform validate
terraform plan -out=tfplan
```

Review the plan carefully. It includes much more than ECR and IAM: it owns the
VPC, EKS cluster, managed system node group, EKS add-ons, Karpenter IAM,
controller identities, service ECR repositories, and service CI roles.

If this environment was applied previously, use the same remote state. Do not
apply against a new/empty state because Terraform may try to create duplicate
infrastructure.

Apply only after the plan is correct:

```powershell
terraform apply tfplan
terraform output
```

Configure `kubectl` using the generated command:

```powershell
$configureKubectl = terraform output -raw configure_kubectl
Invoke-Expression $configureKubectl
kubectl get nodes
```

## 6. Wire Terraform outputs into GitOps

Read the controller identity outputs:

```powershell
terraform output -raw karpenter_controller_role_arn
terraform output -raw karpenter_interruption_queue_name
terraform output -raw crossplane_provider_aws_role_arn
terraform output -raw opencost_role_arn
terraform output -raw aws_lb_controller_role_arn
terraform output -raw external_dns_role_arn
```

Update the matching GitOps values:

| Terraform output | GitOps destination |
|---|---|
| `karpenter_controller_role_arn` | `apps/karpenter/values.yaml` service-account annotation |
| `karpenter_interruption_queue_name` | `apps/karpenter/values.yaml` interruption queue |
| `crossplane_provider_aws_role_arn` | `apps/crossplane/runtime-config.yaml` service-account annotation |
| `opencost_role_arn` | `apps/opencost/values.yaml` service-account annotation |
| `aws_lb_controller_role_arn` | AWS Load Balancer Controller values when that controller is enabled |
| `external_dns_role_arn` | ExternalDNS values when that controller is enabled |

Search for unresolved deployment placeholders before bootstrapping:

```powershell
Set-Location $workspace
rg -n "REPLACE_ACCOUNT_ID|REPLACE_REGION" platform-demo-gitops platform-demo-hello-world-template
```

Replace every placeholder required by an enabled component. Then commit and
push the GitOps changes so Argo CD can fetch them:

```powershell
Set-Location "$workspace\platform-demo-gitops"
git status
git add bootstrap apps clusters
git commit -m "configure dev cluster GitOps"
git push
```

Review `git status` and the staged diff before committing; the workspace may
contain other in-progress changes.

## 7. Bootstrap Argo CD

From the GitOps repository:

```powershell
Set-Location "$workspace\platform-demo-gitops"

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd `
  --namespace argocd `
  --create-namespace `
  --values bootstrap/argocd-values.yaml

kubectl apply -f bootstrap/project.yaml
kubectl apply -f clusters/dev/workloads-project.yaml
kubectl apply -f bootstrap/root-app.yaml
```

The root application then discovers and reconciles the platform applications.
Watch the rollout:

```powershell
kubectl get pods -n argocd
kubectl get applications -n argocd
kubectl get pods -A
```

For local Argo CD access:

```powershell
kubectl port-forward service/argocd-server -n argocd 8080:443
```

Open `https://localhost:8080`. The initial admin password can be read with:

```powershell
argocd admin initial-password -n argocd
```

This requires the optional Argo CD CLI. Alternatively, read and decode the
`argocd-initial-admin-secret` with `kubectl`.

## 8. Configure a service repository

For each service listed in `terraform.tfvars`, get its ECR URL and CI role:

First, tell Terraform the application exists. It creates one ECR repository per
service and one CI role for the application, so nothing can be pushed until it
has run. In `envs/dev/terraform.tfvars`:

```hcl
applications = {
  checkout-platform = {
    github_owner = "YOUR_GITHUB_OWNER"
    services     = ["frontend", "auth", "payments", "worker"]
  }
}
```

Then apply, and read back what it made:

```powershell
Set-Location "$workspace\platform-demo-terraform-modules\envs\dev"
terraform apply
terraform output -json ecr_repository_urls
terraform output -json ci_role_arns
```

The scaffolder sets `ECR_REGISTRY` and `AWS_REGION` on the source repository
itself, so only the role ARN has to be set by hand. Note that the pipeline
derives each service's image repository as
`<ECR_REGISTRY>/<application>-<service>` — the same convention Terraform names
them by — so there is no per-service variable to configure.

| Name | Type | Value |
|---|---|---|
| `AWS_REGION` | Repository variable | Set by the scaffolder |
| `ECR_REGISTRY` | Repository variable | Set by the scaffolder; the registry host only, no repository path |
| `AWS_CI_ROLE_ARN` | Repository secret | `ci_role_arns["APPLICATION_NAME"]` |

With GitHub CLI, an example for `checkout-platform`:

```powershell
$repo = "YOUR_GITHUB_OWNER/checkout-platform-source"
$roles = terraform output -json ci_role_arns | ConvertFrom-Json

gh secret set AWS_CI_ROLE_ARN --repo $repo --body $roles.'checkout-platform'
```

The pipeline also expects these GitHub App secrets, which it uses to open its
pull request against the application's GitOps repository:

- `PLATFORM_DEPLOY_BOT_APP_ID`
- `PLATFORM_DEPLOY_BOT_PRIVATE_KEY`

They are not Terraform outputs. Create and install the GitHub App separately and
give it pull-request and contents permission on the GitOps repository. It needs
no ruleset bypass anywhere — it opens pull requests and merges nothing. Until it
is configured, builds succeed and images are published, but no deployment pull
request is opened.

Finally, merge the registration pull request the scaffolder opened against
`platform-demo-gitops`. It adds one `applications/<application>.json`, and that
is what makes ArgoCD read the application's own GitOps repository and start
deploying. Until it is merged, both repositories exist and CI works, and nothing
is running.

## 9. Backstage status and optional local setup

`platform-demo-backstage` is currently a customization layer, not a complete
buildable Backstage application. The infrastructure and GitOps stack can run
without it.

To experiment with the portal, first generate a stock Backstage app, then
integrate this repository's configuration, catalog, custom action, and required
plugins as described in `platform-demo-backstage/README.md`:

```powershell
npx @backstage/create-app@latest
```

Backstage configuration expects at least:

- `GITHUB_TOKEN`
- `AUTH_GITHUB_CLIENT_ID`
- `AUTH_GITHUB_CLIENT_SECRET`
- `PLATFORM_DEPLOY_BOT_APP_ID`
- `K8S_CLUSTER_URL`
- `ARGOCD_URL`
- `ARGOCD_AUTH_TOKEN`
- `GRAFANA_URL`

Production mode additionally expects PostgreSQL and TechDocs S3 settings.
Do not commit tokens, OAuth secrets, GitHub App private keys, or database
passwords.

## 10. Validation checklist

- `terraform plan` reports no unexpected replacements.
- `kubectl get nodes` shows the managed system node.
- Argo CD's `platform` and `workloads` projects both exist.
- Platform Argo CD applications become `Synced` and `Healthy`.
- Karpenter can provision a workload node when a workload is pending.
- Enabled controllers use the expected IRSA role annotations.
- No required `REPLACE_ACCOUNT_ID` or `REPLACE_REGION` placeholders remain.
- A service workflow can assume its AWS role and push to only its own ECR repo.
- The service chart uses the pushed immutable tag and digest.
- The service namespace has Pod Security labels, quota, limits, and network policy.

Useful diagnostic commands:

```powershell
kubectl get applications -n argocd
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl get resourcequota,limitrange,networkpolicy -A
```

## 11. Teardown

Argo CD resources with finalizers can hold AWS-backed Kubernetes resources in
place. Delete application workloads and platform applications cleanly before
destroying the cluster, and confirm that load balancers and persistent volumes
have been removed.

Then preview Terraform destruction:

```powershell
Set-Location "$workspace\platform-demo-terraform-modules\envs\dev"
terraform plan -destroy
```

Run `terraform destroy` only when you intend to delete the demo AWS
environment. The remote-state S3 bucket is protected and is managed separately
by the bootstrap module.

## Current automation gaps

The following steps are still manual in the current implementation:

- Transferring Terraform controller-role outputs into GitOps values.
- Populating service GitHub variables and secrets.
- Creating and installing the deployment GitHub App.
- Provisioning a complete, deployable Backstage application.
- Replacing all account, region, organization, and repository placeholders.
- Promoting immutable releases through stage and production environments.

