# SRE Pretest - Kubernetes Infrastructure on AWS

This project demonstrates a complete SRE workflow including infrastructure provisioning, containerization, Kubernetes deployment, CI/CD automation, 
and multi-environment deployment practices.

> **AI Usage Declaration**: AI tools were extensively used throughout this project as a pair-programming assistant. See [docs/AI_USAGE.md](docs/AI_USAGE.md) for details.

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

## Prerequisites

- AWS Account with IAM User (AdministratorAccess)
- AWS CLI v2
- AWS credentials configured (`aws configure`)
- Terraform >= 1.0
- kubectl >= 1.30
- Helm >= 3.0
- Docker >= 24.0

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
    ├── AI_USAGE.md               # AI tools usage declaration and prompts
    └── screenshots/              # Evidence screenshots for each question
```

---

## Q1 - Terraform Kubernetes Cluster

### Infrastructure
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

## Q2 - Dockerfile

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

## Q3 - Helm Chart Deployment

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
aws ecr create-repository --repository-name nginx-sre --region us-east-1  # Create ECR repository
ECR_URI=$(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com/nginx-sre  # Get ECR URI
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_URI  # Authenticate Docker to ECR
docker tag nginx-sre:latest $ECR_URI:latest  # Tag image
docker push $ECR_URI:latest  # Push to ECR

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

## Q4 - CI/CD Pipeline

Two GitHub Actions workflows:

### `terraform.yaml` - Infrastructure Pipeline
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

### `app.yaml` - Application Pipeline
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
AWS_ACCESS_KEY_ID      → AWS IAM User Access Key
AWS_SECRET_ACCESS_KEY  → AWS IAM User Secret Key

> **Security Note**: For production use, recommend replacing long-lived AWS credentials with OIDC (OpenID Connect) for keyless, short-lived authentication.

---

## Q5 - GitOps Multi-Environment Deployment

### GitOps Concept

This project implements a **GitOps-inspired** approach using GitHub Actions
as the delivery mechanism. While a full GitOps implementation would use
tools like ArgoCD or Flux for pull-based deployment and drift detection,
this project demonstrates the core GitOps principles:

- **Git as single source of truth**: All environment configurations are version-controlled in Git
- **Declarative configuration**: Environment states defined in `envs/` directory
- **Automated deployment**: Git events (push, tag) trigger CI/CD pipeline automatically
- **Environment isolation**: Different branches map to different environments

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

### Branch Strategy
```
main              → alpha environment (auto deploy)
release/beta      → beta environment (auto deploy)
release/staging   → staging environment (auto deploy)
tag v*..          → production environment (manual gate)
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

In practice, deployment is handled automatically by `app.yaml` CI/CD pipeline.
The following commands can be used for manual deployment if needed:

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

> **GitOps Extension**: For full GitOps implementation, tools like ArgoCD or Flux can be integrated to enable pull-based deployment and drift detection.

---

## Q6 - Linking Terraform Outputs to Helm

### Method Used: CI/CD Pipeline Integration

Terraform outputs are read during CI/CD and passed to Helm via `--set`:

**Step 1: Define outputs in `terraform/outputs.tf`**

```hcl
output "cluster_name" {
  value = module.eks.cluster_name
}

output "ecr_repository_url" {
  value = aws_ecr_repository.nginx_sre.repository_url
}
```

**Step 2: Read outputs in GitHub Actions and pass to Helm**

```yaml
# Read Terraform outputs (e.g. CloudArmor policy, cluster name, ECR URL)
- name: Get Terraform outputs
  run: |
    cd terraform
    echo "CLUSTER_NAME=$(terraform output -raw cluster_name)" >> $GITHUB_ENV
    echo "ECR_URL=$(terraform output -raw ecr_repository_url)" >> $GITHUB_ENV
    # Example: echo "ARMOR_POLICY=$(terraform output -raw cloud_armor_policy_name)" >> $GITHUB_ENV

# Pass to Helm deploy
- name: Deploy to EKS
  run: |
    helm upgrade --install ${{ env.ENV }} helm/nginx-sre \
      --set clusterName=${{ env.CLUSTER_NAME }} \
      --set image.repository=${{ env.ECR_URL }}
      # Example: --set annotations.cloudArmor=${{ env.ARMOR_POLICY }}
```

### Comparison of Methods

| Method | Description | Best For |
|--------|-------------|----------|
| CI/CD `--set` | Read terraform output and pass to helm | Simple projects ✅ |
| `local_file` resource | Terraform writes values.yaml directly | Quick validation |
| AWS Parameter Store | Terraform stores to SSM, CI/CD reads | Production |

---

## Screenshots

Screenshots are available in `docs/screenshots/`:

### Q1 - Terraform EKS Cluster
| Screenshot | Description |
|------------|-------------|
| q1-eks-cluster.png | EKS Cluster on AWS Console (v1.33, Standard Support) |
| q1-ec2-instances.png | EC2 Nodes running in us-east-1 (2x t3.micro) |
| q1-kubectl-nodes.png | kubectl get nodes - 2 nodes Ready |

### Q2 - Dockerfile
| Screenshot | Description |
|------------|-------------|
| q2-hello-sre.png | curl output: Hello SRE! from LoadBalancer |

### Q3 - Helm Chart Deployment
| Screenshot | Description |
|------------|-------------|
| q3-ecr-repository.png | ECR repository with nginx-sre image pushed |
| q3-load-balancer.png | AWS Classic Load Balancer created by Service |
| q3-helm-deploy.png | helm list showing deployed release |
| q3-kubectl-pods.png | Pods running in alpha namespace (READY 1/1) |
| q3-probe.png | Readiness/Liveness probe configured on /sre.txt |

### Q4 - CI/CD Pipeline
| Screenshot | Description |
|------------|-------------|
| q4-github-actions-app.png | Build and Deploy workflow - all green |
| q4-github-actions-terraform.png | Terraform workflow - all green |

### Q5 - Multi-Environment
| Screenshot | Description |
|------------|-------------|
| q5-alpha-namespace.png | Services running in alpha namespace |
| q5-hpa-enabled.png | HPA enabled with CPU/Memory metrics (staging/production) |
| q5-hpa-disabled.png | HPA disabled in alpha environment (cost saving) |
---

## AI Usage

See [docs/AI_USAGE.md](docs/AI_USAGE.md) for complete details on AI assistance used in this project.