# AWS Application Infrastructure — Terraform Starter

Reusable Terraform baseline for deploying an application for another client on AWS. It creates a VPC, EKS, managed worker nodes, ECR repositories, CloudWatch log groups, and optional PostgreSQL RDS, Redis ElastiCache, S3 storage, Route 53 record, and Kong Gateway.

> **Important:** This is a reusable baseline, not a copy of your production account. Review CIDRs, sizing, security, backup requirements, DNS, and cost before applying. It does not import or modify your existing AWS resources. Run it in a separate client account/workspace.

## Architecture

```text
                         Route 53 (optional)
                                |
                         App Load Balancer*
                                |
                     EKS cluster (control plane)
                                |
              +-----------------+-----------------+
              |                                   |
        Private worker nodes                Kong Gateway*
              |                                   |
      Application workloads  <--------------------+
              |
        +-----+------+-------------------+
        |            |                   |
     RDS Postgres  Redis Cache       S3 bucket
        private      private          private
```

`*` Route 53 records need the DNS name and hosted-zone ID of the actual ingress/load balancer. Kong is optional and installed by Helm into the EKS cluster. This project does not create an application ALB: deploy AWS Load Balancer Controller and an Ingress/Service for your application separately, or use Kong's service endpoint.

## Requirements

- Terraform >= 1.6
- AWS CLI v2 configured for the target client account
- IAM identity allowed to create VPC, EKS, IAM roles/policies, EC2, ECR, RDS, ElastiCache, S3, CloudWatch, and (if enabled) Route 53 resources
- `kubectl` and `helm` for interacting with EKS/Kong
- Review current AWS pricing. NAT Gateway, EKS control plane, worker nodes, RDS, and Redis can incur charges.

## Quick start

1. Copy `terraform/terraform.tfvars.example` to `terraform/terraform.tfvars`.
2. Edit the values marked **CHANGE ME**. Never commit `terraform.tfvars`.
3. From the `terraform/` directory:

   ```bash
   terraform init
   terraform fmt -recursive
   terraform validate
   terraform plan -out=tfplan
   terraform apply tfplan
   ```

4. Configure `kubectl`:

   ```bash
   aws eks update-kubeconfig --region ap-south-1 --name client-app-eks
   kubectl get nodes
   ```

5. Push application images to ECR (repository URLs are Terraform outputs), deploy your application manifests/Helm chart, then configure ingress/DNS.

## Stages

### Stage 1 — Network and EKS
Keep `enable_rds`, `enable_redis`, `enable_s3`, and `enable_kong` false initially if you want a smaller first deployment. Apply and confirm nodes join the cluster.

### Stage 2 — Container registry and application
Build and push your image to one of the ECR repositories. Example:

```bash
aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS --password-stdin YOUR_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com
docker build -t YOUR_REPOSITORY:latest .
docker tag YOUR_REPOSITORY:latest YOUR_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/YOUR_REPOSITORY:latest
docker push YOUR_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/YOUR_REPOSITORY:latest
```

Create Kubernetes Deployment/Service/Ingress manifests for the app. EKS nodes use private subnets; the default single NAT Gateway allows outbound image pulls and updates. For production, consider NAT per AZ or VPC endpoints, private cluster API access, and stronger network controls.

### Stage 3 — Data and storage
Set `enable_rds`, `enable_redis`, and/or `enable_s3` to `true`. Set a strong database password through `db_password` using a local untracked tfvars file or a secure CI secret. **Terraform state can contain secrets**; use encrypted remote state with restricted access and locking for shared/production use.

### Stage 4 — Kong (optional)
Set `enable_kong = true` and apply. Kong is installed via Helm with a `LoadBalancer` service by default, so AWS may provision a public load balancer. Review exposure, allowed source ranges, TLS, and authentication before using it. This starter does not configure Kong routes/plugins for your application; add those after the application Service exists. If you want Kong behind an ALB instead, configure the relevant ingress architecture deliberately and avoid unintentionally exposing both endpoints.

### Stage 5 — DNS (optional)
Set `enable_route53_record = true`, `route53_zone_id`, `app_dns_name`, and `load_balancer_dns_name`. The supplied DNS name must be an AWS load balancer DNS name. Terraform creates an alias A record. The hosted zone must already exist.

## Where to change things

| File / variable | What to edit |
|---|---|
| `terraform/terraform.tfvars.example` | Client name, AWS region, VPC CIDR, AZs, node size/count, feature toggles |
| `terraform/variables.tf` | Allowed inputs and defaults |
| `terraform/main.tf` | Network/EKS/data-service resources |
| `terraform/outputs.tf` | Values shown after apply |
| `k8s/` | Starter location for your app manifests (not deployed automatically) |
| `GITHUB_ACTIONS_NOTES.md` | CI/CD and OIDC setup guidance |

## Security notes before production

- Do not commit `.tfstate`, plan files, real `terraform.tfvars`, passwords, or access keys.
- Use a remote S3 backend with encryption, versioning, strict bucket policy, and state locking appropriate to your Terraform version/team workflow. Backend resources must be created separately before initializing the backend.
- The default cluster endpoint is public and private access is enabled. Restrict `cluster_endpoint_public_access_cidrs` to your office/VPN egress IPs; do not leave `0.0.0.0/0` for production.
- DB/Redis are in private subnets and have no public ingress. Only EKS node security groups are permitted by the data-service security groups.
- Rotate credentials and store application secrets in AWS Secrets Manager or Parameter Store. This starter passes the DB password to Terraform, which can appear in state.
- Enable backups, maintenance windows, deletion protection, monitoring, and multi-AZ where required.
- Review costs and service quotas before `apply`. Use `terraform destroy` only after confirming the environment is disposable and backups are preserved.

## Tear down

For a disposable lab only:

```bash
terraform plan -destroy
terraform destroy
```

RDS/Redis deletion protection is disabled in this starter for easier lab cleanup. Change that before production. S3 bucket deletion can fail if it contains objects; empty it deliberately before destroying.

## What this project does not do

- It does not deploy your application image or create app-specific Kubernetes Deployment/Service resources.
- It does not install AWS Load Balancer Controller, ExternalDNS, cert-manager, Prometheus, Grafana, or Loki.
- It does not import, discover, or alter resources in your existing AWS account.
- It does not guarantee production readiness without review and security testing.
