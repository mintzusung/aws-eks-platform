# AI Usage Declaration

AI tools (Claude by Anthropic, ChatGPT by OpenAI) were used throughout this project
as a **development accelerator and decision validator**, as permitted by the assignment
guidelines.

## Prompting Strategy

AI was used extensively throughout development, including generating code from scratch,
validating architectural decisions, and accelerating debugging. Prompts were structured to:
- **Generate boilerplate**: Scaffold Terraform, Helm, and GitHub Actions code quickly
- **Validate decisions**: Confirm architectural choices and trade-offs
- **Debug faster**: Identify root causes without lengthy trial and error
- **Review trade-offs**: Compare options across security, cost, and complexity

## Areas of AI Assistance

| Area | Usage Pattern |
|------|--------------|
| Terraform | EKS cluster configuration, VPC setup, S3 backend |
| Dockerfile | nginx customization |
| Helm Chart | Chart structure, HPA, probes, multi-env values |
| GitHub Actions | CI/CD pipeline design, multi-environment strategy |
| Auto-scaling | Node scaling vs Pod scaling design, resource allocation |
| Cost Optimization | Region selection, K8s version, bill analysis |
| Debugging | EKS AMI compatibility, S3 region conflicts, LB deletion |

## Key Prompts Used

**Infrastructure (Q1)**
- "Write Terraform code to create an EKS cluster on AWS with VPC, private subnets, NAT Gateway and managed node groups"
- "How to set up S3 backend for Terraform state management"
- "Review node group scaling settings: min 1, desired 2, max 5 for t3.micro - appropriate for demo workload?"
- "Validate Cluster Autoscaler integration with EKS managed node groups"

**Dockerfile (Q2)**
- "Write a Dockerfile based on nginx:alpine that serves sre.txt containing Hello SRE!"

**Helm Chart (Q3)**
- "Create a Helm chart for nginx with HPA supporting both CPU and Memory autoscaling"
- "Add readiness and liveness probe to Helm deployment template"
- "Review conditional replicas implementation when HPA is enabled to avoid conflicts"
- "Is using Release.Name as fullname a valid approach for multi-environment namespace isolation?"
- "Validate probe design: httpGet /sre.txt vs tcpSocket - which provides better application-level verification?"
- "Review resource requests and limits allocation: cpu 50m/100m, memory 64Mi/128Mi for nginx"

**Auto-scaling Design**
- "Review two-layer scaling strategy: HPA for pods (CPU 70%, Memory 75%) + Cluster Autoscaler for nodes (max 5)"
- "Validate decision to disable HPA in alpha/beta and enable in staging/production for resource efficiency"
- "Confirm Kubernetes scheduler distributes pods across nodes to prevent memory contention"
- "Review HPA thresholds: CPU 70% and Memory 75% - appropriate for nginx workload?"

**Multi-environment Resource Design**
- "Review progressive resource allocation strategy across alpha/beta/staging/production environments"
- "Validate HPA enable/disable decision per environment based on resource efficiency trade-offs"
- "Review Helm values override pattern for environment-specific configurations"
- "Confirm replicaCount per environment: alpha(1), beta(1), staging(2), production(3) for HA requirements"
- "How does PodDisruptionBudget work with multiple replicas in production"

**CI/CD (Q4)**
- "Write GitHub Actions workflow to build Docker image, push to ECR and deploy to EKS using Helm"
- "How to implement multi-environment deployment in GitHub Actions based on branch strategy"
- "Review security trade-offs: AWS Credentials vs OIDC for GitHub Actions authentication"

**GitOps Multi-Environment (Q5)**
- "How to use GitOps concept to support multi-environment deployment from same codebase"
- "How does branch-based strategy (main→alpha, release/beta→beta, tags→production) implement GitOps"
- "How to use same Helm chart with different values.yaml for alpha/beta/staging/production environments"

**Terraform Output Integration (Q6)**
- "How to reference Terraform-created resources in Helm chart annotations"
- "How to pass Terraform output (e.g. cluster name, ECR URL) to Helm via CI/CD --set"

**Cost Optimization**
- "Compare AWS region costs for EKS deployment, which region is cheapest"
- "Analyze AWS bill: EKS Extended Support fee - confirm version upgrade resolves this"
- "Validate region migration decision: ap-southeast-2 → us-east-1 for cost reduction"
- "Does cross-region S3 backend for Terraform state incur significant data transfer costs"
- "Review AWS Budget alert strategy: daily + monthly thresholds for cost control"
- "What is NAT Gateway data transfer cost and how to minimize it"

**Debugging**
- "EKS Node Group AMI not supported error in us-east-1 - root cause and resolution"
- "Terraform destroy failing due to subnet dependency - correct cleanup sequence"
- "S3 backend region conflict when migrating from ap-southeast-2 to us-east-1"
- "Helm uninstall required before terraform destroy to release Load Balancer dependencies"
- "kubectl get hpa showing unknown metrics - missing component diagnosis"

## Author's Contribution

All AI outputs were reviewed, tested, and validated by the author:

**Infrastructure & Deployment**
- All infrastructure was actually provisioned and verified on AWS
- All deployments were tested end-to-end (Hello SRE! confirmed working)
- Debugging and troubleshooting were done collaboratively with AI assistance

**Auto-scaling Design Decisions**
- Designed two-layer auto-scaling strategy:
  - Node scaling: min 1, desired 2, max 5 (AWS Cluster Autoscaler)
  - Pod scaling: HPA with CPU 70% and Memory 75% thresholds
- Decided to disable HPA in alpha/beta environments to save resources
- Configured appropriate resource requests and limits per environment

**Multi-environment Resource Allocation**
- Designed progressive resource allocation across environments:
  - alpha/beta: minimal resources, HPA disabled
  - staging: moderate resources, HPA enabled
  - production: full resources, HPA enabled with higher replica counts
- Implemented conditional replicas in deployment template to avoid HPA conflicts

**Cost Optimization**
- Migrated from ap-southeast-2 (Sydney) to us-east-1 (N. Virginia) for lower costs
- Upgraded from k8s v1.31 (Extended Support) to v1.33 (Standard Support) to avoid ~$0.60/hr surcharge
- Analyzed AWS bill breakdown to identify EKS Extended Support as the largest cost driver
- Set up daily and monthly AWS Budget alerts to prevent unexpected charges
- Identified that cross-region S3 backend traffic cost is negligible (few KB state file)

**Architecture & Security**
- All architectural decisions were made by the author
- Chose IAM User with GitHub Secrets for simplicity, with documented recommendation to migrate to OIDC
- The author understands every component and can explain all implementation details