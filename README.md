# AWSKubernetes

Terraform scripts to **set up** and **tear down** a managed Kubernetes cluster
(Amazon EKS) in AWS.

## Architecture

```
VPC (10.0.0.0/16)
├── Public Subnet AZ-a  ──┐
├── Public Subnet AZ-b  ──┼── Internet Gateway → Internet
│                          └── NAT Gateway (outbound for private subnets)
├── Private Subnet AZ-a ──┐
└── Private Subnet AZ-b ──┴── EKS Managed Node Group
                               (worker nodes, no public IPs)

EKS Control Plane
└── Managed node group  (auto-scaling, t3.medium by default)
```

| Resource | Details |
|---|---|
| EKS control plane | Managed by AWS, public + private endpoint |
| Worker nodes | Managed node group in private subnets |
| Networking | VPC with 2 public + 2 private subnets across 2 AZs |
| Outbound traffic | Single NAT Gateway in the first public subnet |

## Prerequisites

| Tool | Minimum version |
|---|---|
| [Terraform](https://developer.hashicorp.com/terraform/downloads) | 1.3.0 |
| [AWS CLI](https://aws.amazon.com/cli/) | 2.x |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | matches cluster version |

Your AWS credentials must be configured (`aws configure` or environment
variables) and the IAM principal must have permission to create EKS clusters,
VPCs, IAM roles, and EC2 resources.

## Quick start

### 1 – Configure variables

```bash
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars to match your environment
```

Key variables (see `variables.tf` for the full list):

| Variable | Default | Description |
|---|---|---|
| `aws_region` | `us-east-1` | AWS region |
| `cluster_name` | `my-eks-cluster` | EKS cluster name |
| `kubernetes_version` | `1.29` | Kubernetes version |
| `node_instance_type` | `t3.medium` | Worker-node EC2 type |
| `node_desired_size` | `2` | Desired number of nodes |
| `node_min_size` | `1` | Minimum nodes (auto-scaling) |
| `node_max_size` | `4` | Maximum nodes (auto-scaling) |

### 2 – Set up the cluster

```bash
terraform init      # download AWS provider
terraform plan      # preview changes
terraform apply     # create the cluster (~15 minutes)
```

After `apply` completes, update your local kubeconfig:

```bash
# The exact command is printed in the "kubeconfig_command" output
aws eks update-kubeconfig --region us-east-1 --name my-eks-cluster

kubectl get nodes   # verify worker nodes are Ready
```

### 3 – Tear down the cluster

```bash
terraform destroy   # destroy all resources (~10 minutes)
```

This removes every resource created by Terraform: the EKS cluster, node group,
VPC, subnets, NAT Gateway, IAM roles, etc.

## File layout

```
.
├── versions.tf              # Terraform + AWS provider version constraints
├── variables.tf             # All input variables with defaults
├── vpc.tf                   # VPC, subnets, IGW, NAT Gateway, route tables
├── iam.tf                   # IAM roles for EKS cluster and node group
├── eks.tf                   # EKS cluster + managed node group
├── outputs.tf               # Useful outputs (endpoint, kubeconfig command…)
└── terraform.tfvars.example # Example variable values – copy to terraform.tfvars
```

## Remote state (optional)

For team usage it is recommended to store Terraform state remotely.
Add a `backend` block to `versions.tf`, for example:

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state-bucket"
    key    = "eks/terraform.tfstate"
    region = "us-east-1"
  }
}
```
