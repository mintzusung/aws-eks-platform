# AWS EKS Infrastructure & CI/CD

A production-oriented platform covering infrastructure provisioning, containerization, Kubernetes deployment, CI/CD automation, and multi-environment deployment practices on AWS EKS.

---

## Architecture Overview

```
GitHub Repository
↓ push
GitHub Actions CI/CD
↓
├── terraform.yaml → AWS EKS Cluster (Terraform)
└── app.yaml       → Build & Deploy nginx-sre
↓
Amazon ECR (Image Registry)
↓
Amazon EKS Cluster (us-east-1)
├── namespace: alpha
├── namespace: beta
├── namespace: staging
└── namespace: production
↓
AWS Load Balancer
↓
External Users
http://<LB-URL>/sre.txt → "Hello SRE!"
```

---

## Project Structure

```
sre-pretest/
├── README.md
├── .gitignore
├── .github/
│   └── workflows/
│       ├── app.yaml              # Application CI/CD: build, push ECR, helm deploy
│       └── terraform.yaml        # Infrastructure CI/CD: terraform plan & apply
├── docker/
│   └── Dockerfile                # Custom nginx image serving sre.txt
├── terraform/
│   ├── main.tf                   # VPC, EKS cluster, managed node group
│   ├── variables.tf              # Input variables (region, cluster version, node size)
│   ├── outputs.tf                # Output values (cluster name, endpoint, kubectl command)
│   └── versions.tf               # Provider versions + S3 backend configuration
├── helm/
│   └── nginx-sre/
│       ├── Chart.yaml            # Chart metadata and version
│       ├── values.yaml           # Default values, overridden per environment
│       └── templates/
│           ├── _helpers.tpl      # Reusable template functions (fullname, labels)
│           ├── deployment.yaml   # Pod spec, resource limits, readiness/liveness probes
│           ├── service.yaml      # LoadBalancer service for external access
│           └── hpa.yaml          # Conditional HPA (CPU 70%, Memory 75%)
├── envs/
│   ├── alpha/values.yaml         # Alpha: minimal resources, HPA disabled
│   ├── beta/values.yaml          # Beta: moderate resources, HPA disabled
│   ├── staging/values.yaml       # Staging: moderate resources, HPA enabled (max 3)
│   └── production/values.yaml    # Production: full resources, HPA enabled (max 5)
└── docs/
    └── screenshots/              # Evidence screenshots for each question
```

---

## Prerequisites

- AWS Account with IAM User (AdministratorAccess)
- AWS CLI v2
- AWS credentials configured (`aws configure`)
- Terraform >= 1.0
- kubectl >= 1.30
- Helm >= 3.0
- Docker >= 24.0

---

## Infrastructure (Terraform)

### Cloud & Cluster

- **Cloud Provider**: AWS (us-east-1)
- **Kubernetes**: EKS v1.33 (Standard Support until July 2026)
- **Nodes**: 2x t3.micro (auto-scaling: min 1, desired 2, max 5)
- **Network**: VPC with public/private subnets, NAT Gateway
- **State Management**: S3 Backend (`sre-pretest-tfstate`)

### Node Auto-Scaling

- **Min size**: 1 node (cost saving during low traffic)
- **Desired size**: 2 nodes (normal operation with high availability)
- **Max size**: 5 nodes (increased from 3 for better pod distribution and high availability)
  - Production: up to 5 pods × 256Mi = 1280Mi total memory requirement
  - If 3 nodes (1024Mi each) only, uneven pod scheduling could cause memory pressure on a single node
  - 5 nodes ensure pods are properly distributed with sufficient headroom
- Managed by AWS Auto Scaling Group
- When pods cannot be scheduled due to insufficient resources, new nodes are automatically added
- When nodes are underutilized, they are automatically removed

### Cost Optimization

- **Region**: us-east-1 (N. Virginia) selected for lowest cost in AWS
- **Kubernetes version**: v1.33 (Standard Support) to avoid Extended Support surcharge (~$0.60/hr)
- **Instance type**: t3.micro for cost efficiency in non-production environment
- **Best practice**: Always `terraform destroy` when not in use to minimize costs
- **Recommended**: Set up AWS Budget alerts for daily and monthly spending

### Setup

```bash
cd terraform
terraform init
terraform plan        
terraform apply -auto-approve
```

### Configure kubectl

```bash
aws eks update-kubeconfig --region us-east-1 --name sre-pretest-cluster
kubectl get nodes
```

---

## Container (Dockerfile)

Custom nginx image with `sre.txt` accessible at `/sre.txt`.

### Build & Test

```bash
cd docker
docker build -t nginx-sre .
docker run -d -p 8080:80 --name nginx-sre-test nginx-sre
curl http://localhost:8080/sre.txt
# Output: Hello SRE!
docker stop nginx-sre-test && docker rm nginx-sre-test
```

---

## Kubernetes Deployment (Helm)

### Features

- **HPA**: Auto-scaling based on CPU (70%) and Memory (75%)
  - alpha/beta: HPA disabled for resource efficiency
  - staging: min 1, max 3
  - production: min 2, max 5 (minimum 2 ensures high availability)
- **Readiness Probe**: Checks `/sre.txt` every 5s (initialDelay: 5s)
- **Liveness Probe**: Checks `/sre.txt` every 10s (initialDelay: 10s)
- **LoadBalancer**: AWS Classic Load Balancer for external access
- **Conditional Replicas**: When HPA is enabled, replicas are managed by HPA only

### Two-Layer Auto-Scaling (Node & Pod)

```
Traffic increases
↓
HPA detects CPU > 70% or Memory > 75%
↓
HPA adds more Pods (up to maxReplicas)
↓
If nodes don't have enough resources
↓
Cluster Autoscaler adds new Nodes (up to max_size: 5)
```

### Deploy Manually

```bash
# Push image to ECR
aws ecr create-repository --repository-name nginx-sre --region us-east-1
ECR_URI=$(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com/nginx-sre
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_URI
docker tag nginx-sre:latest $ECR_URI:latest
docker push $ECR_URI:latest

# Deploy with Helm
helm upgrade --install alpha helm/nginx-sre \
  -f envs/alpha/values.yaml \
  --namespace alpha \
  --create-namespace \
  --set image.repository=$ECR_URI \
  --set image.tag=latest
```

### Verify

```bash
curl http://$(kubectl get svc alpha -n alpha -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/sre.txt
# Output: Hello SRE!
```

### Known Limitations & Future Improvements

- `_helpers.tpl` currently uses `Release.Name` as fullname for simplicity. Standard Helm practice would implement `nginx-sre.labels` and `nginx-sre.selectorLabels` for better observability and monitoring integration.
- Currently using Classic Load Balancer (CLB). Production recommendation: AWS Load Balancer Controller with ALB Ingress for path-based routing and cost efficiency.
- HPA requires metrics-server installation for CPU/Memory metrics collection.
- PodDisruptionBudget (PDB) is recommended for production to prevent service disruption during node maintenance.

---

## CI/CD Pipeline

Two GitHub Actions workflows:

### `terraform.yaml` — Infrastructure Pipeline

Triggered when `terraform/` files change on `main` branch.

```
Checkout → Configure AWS → Setup Terraform → Init → Format Check → Plan → Apply
```

| Step | Purpose |
|------|---------|
| Checkout | Pull latest code from repository |
| Configure AWS | Set up AWS credentials for Terraform |
| Setup Terraform | Install Terraform CLI |
| Terraform Init | Initialize backend and download providers |
| Format Check | Validate code formatting standards |
| Terraform Plan | Preview infrastructure changes |
| Terraform Apply | Apply changes to AWS (main branch only) |

### `app.yaml` — Application Pipeline

Triggered on push to `main`, `release/beta`, `release/staging`, or release tag.

```
Checkout → Configure AWS → Login ECR → Build Image → Push Image to ECR → Configure kubectl → Install Helm → Set Environment → Deploy
```

| Step | Purpose |
|------|---------|
| Checkout | Pull latest code from repository |
| Configure AWS | Set up AWS credentials for ECR and EKS access |
| Login ECR | Authenticate Docker to Amazon ECR |
| Build Image | Build Docker image from Dockerfile |
| Push Image to ECR | Push image with commit SHA and latest tags |
| Configure kubectl | Connect kubectl to EKS cluster |
| Install Helm | Set up Helm CLI |
| Set Environment | Determine target environment from branch/tag |
| Deploy | Helm upgrade with environment-specific values |

### GitHub Secrets Required

```
AWS_ACCESS_KEY_ID      → AWS IAM User Access Key
AWS_SECRET_ACCESS_KEY  → AWS IAM User Secret Key
```

> **Security Note**: For production use, recommend replacing long-lived AWS credentials with OIDC (OpenID Connect) for keyless, short-lived authentication.

---

## Multi-Environment Deployment (GitOps)

All four environments share the same Helm chart. Environment-specific configuration lives in `envs/<env>/values.yaml` and is applied at deploy time — no chart duplication, no branching logic inside templates. Git is the single source of truth; every deployment is triggered by a Git event.

### Branch Strategy

```
main              → alpha environment (auto deploy)
release/beta      → beta environment (auto deploy)
release/staging   → staging environment (auto deploy)
tag v*.*.*        → production environment (manual gate)
```

### Deployment Flow

```
Developer pushes code to main branch
        ↓
GitHub Actions triggered automatically
        ↓
Auto deploy to alpha (latest version)
        ↓
After testing, create PR to release/beta
        ↓
After merge, auto deploy to beta
        ↓
After testing, create PR to release/staging
        ↓
After merge, auto deploy to staging
        ↓
Publish tag v*.*.* with manual approval gate
        ↓
After approval, deploy to production
```

### Environment Configuration

Each environment has its own `values.yaml` under `envs/`:

| Environment | Replicas | CPU Limit | Memory Limit | HPA | HPA Max |
|-------------|----------|-----------|--------------|-----|---------|
| alpha       | 1        | 100m      | 128Mi        | ❌  | -       |
| beta        | 1        | 200m      | 256Mi        | ❌  | -       |
| staging     | 2        | 200m      | 256Mi        | ✅  | 3       |
| production  | 3        | 500m      | 512Mi        | ✅  | 5       |

### Deploy to Specific Environment

In practice, deployment is handled automatically by `app.yaml` CI/CD pipeline. The following commands can be used for manual deployment if needed:

```bash
# Alpha
helm upgrade --install alpha helm/nginx-sre \
  -f envs/alpha/values.yaml \
  --namespace alpha --create-namespace

# Production
helm upgrade --install production helm/nginx-sre \
  -f envs/production/values.yaml \
  --namespace production --create-namespace
```

---

## Screenshots

Screenshots are available in `docs/screenshots/`:

| Screenshot | Description |
|------------|-------------|
| eks-cluster.png | EKS Cluster on AWS Console (v1.33, Standard Support) |
| ec2-instances.png | EC2 Nodes running in us-east-1 (2x t3.micro) |
| kubectl-nodes.png | kubectl get nodes - 2 nodes Ready |
| hello-sre.png | curl output: Hello SRE! from LoadBalancer |
| ecr-repository.png | ECR repository with nginx-sre image pushed |
| load-balancer.png | AWS Classic Load Balancer created by Service |
| helm-deploy.png | helm list showing deployed release |
| kubectl-pods.png | Pods running in alpha namespace (READY 1/1) |
| probe.png | Readiness/Liveness probe configured on /sre.txt |
| github-actions-app.png | Build and Deploy workflow - all green |
| github-actions-terraform.png | Terraform workflow - all green |
| alpha-namespace.png | Services running in alpha namespace |
| hpa-enabled.png | HPA enabled with CPU/Memory metrics (staging/production) |
| hpa-disabled.png | HPA disabled in alpha environment (cost saving) |



