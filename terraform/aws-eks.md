
# EKS Cluster with Terraform

This project provisions an Amazon EKS cluster in the `us-east-2` region using Terraform. It creates the cluster control plane, a managed node group, and installs essential EKS add-ons such as VPC CNI, CoreDNS, and kube-proxy. The required IAM roles for the cluster and nodes must already exist and are provided via input.

## Prerequisites

- AWS CLI configured
- Terraform CLI installed (>= 1.0.0)
- Existing IAM roles for:
  - EKS cluster control plane (`cluster_role_arn`)
  - EKS node group (`node_role_arn`)

## Files Overview

- `provider.tf`: Required Terraform providers
- `variables.tf`: Input variables
- `terraform.tfvars`: Values for input variables
- `main.tf`: Infrastructure definition
- `outputs.tf`: Useful outputs


## provider.tf

```hcl
provider "aws" {
  region = var.region
}

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.11"
    }
    kubectl = {
      source  = "gavinbunney/kubectl"
      version = "~> 1.14.0"
    }
  }

  required_version = ">= 1.0"
}
```

## variables.tf

```hcl
variable "region" {
  default = "us-east-2"
}

variable "cluster_name" {
  default = "demo_cluster"
}

variable "node_group_name" {
  default = "mynodegroup"
}

variable "min_size" {
  default = 1
}

variable "max_size" {
  default = 2
}

variable "desired_size" {
  default = 2
}

variable "cluster_role_arn" {
  description = "IAM role ARN for EKS cluster"
  type        = string
}

variable "node_role_arn" {
  description = "IAM role ARN for EKS node group"
  type        = string
}
```


## terraform.tfvars

```hcl
region           = "us-east-2"
cluster_name     = "demo_cluster"
node_group_name  = "mynodegroup"

min_size         = 1
max_size         = 2
desired_size     = 2

cluster_role_arn = "arn:aws:iam::<ACCOUNT_ID>:role/<YourClusterRole>"
node_role_arn    = "arn:aws:iam::<ACCOUNT_ID>:role/<YourNodeRole>"
```

## main.tf

```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

resource "aws_eks_cluster" "demo" {
  name     = var.cluster_name
  role_arn = var.cluster_role_arn

  vpc_config {
    subnet_ids = data.aws_subnets.public.ids
  }
}

resource "aws_eks_node_group" "demo" {
  cluster_name    = aws_eks_cluster.demo.name
  node_group_name = var.node_group_name
  node_role_arn   = var.node_role_arn
  subnet_ids      = data.aws_subnets.public.ids

  scaling_config {
    desired_size = var.desired_size
    max_size     = var.max_size
    min_size     = var.min_size
  }

  depends_on = [aws_eks_cluster.demo]
}

resource "aws_eks_addon" "vpc_cni" {
  cluster_name                 = aws_eks_cluster.demo.name
  addon_name                   = "vpc-cni"
  service_account_role_arn     = var.node_role_arn
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"
  depends_on                   = [aws_eks_node_group.demo]
}

resource "aws_eks_addon" "coredns" {
  cluster_name                 = aws_eks_cluster.demo.name
  addon_name                   = "coredns"
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"
  depends_on                   = [aws_eks_node_group.demo]
}

resource "aws_eks_addon" "kube_proxy" {
  cluster_name                 = aws_eks_cluster.demo.name
  addon_name                   = "kube-proxy"
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"
  depends_on                   = [aws_eks_node_group.demo]
}
```

## outputs.tf

```hcl
output "cluster_name" {
  value = aws_eks_cluster.demo.name
}

output "cluster_endpoint" {
  value = aws_eks_cluster.demo.endpoint
}

output "cluster_certificate_authority" {
  value = aws_eks_cluster.demo.certificate_authority[0].data
}

output "node_group_name" {
  value = aws_eks_node_group.demo.node_group_name
}
```
## Usage

1. Save all files in the same directory.
2. Update `terraform.tfvars` with your AWS role ARNs.
3. Initialize the project:

```bash
terraform init
```

4. Apply the configuration:

```bash
terraform apply
```

Type `yes` when prompted to proceed.

##

## Accessing the Cluster

To use `kubectl` to manage the cluster, you can configure your local kubeconfig:

```bash
aws eks --region us-east-2 update-kubeconfig --name demo_cluster
```

Alternatively, construct the kubeconfig manually using the outputs for endpoint and certificate.

##

## Notes

* This configuration assumes public subnets in the default VPC.
* The VPC CNI, CoreDNS, and kube-proxy add-ons are installed via the `aws_eks_addon` resource.
* You are responsible for ensuring that the specified IAM roles have the required policies attached.
