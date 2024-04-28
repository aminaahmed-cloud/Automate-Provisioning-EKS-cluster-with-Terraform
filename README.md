# Automate Provisioning EKS cluster with Terraform

This guide provides step-by-step instructions for automating the provisioning of an Amazon EKS cluster using Terraform. Follow these instructions to create a VPC, set up the EKS cluster, and deploy an nginx application into the cluster.

## 1. Create VPC

- Use the AWS VPC module available in the Terraform registry to create the VPC.
- Create `vpc.tf` file:

```hcl
provider "aws" {
  region = "eu-west-2"
}

variable "vpc_cidr_block" {}
variable "private_subnet_cidr_blocks" {}
variable "public_subnet_cidr_blocks" {}

data "aws_availability_zones" "azs" {}

module "myapp-vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"

  name              = "myapp-vpc"
  cidr              = var.vpc_cidr_block
  private_subnets   = var.private_subnet_cidr_blocks
  public_subnets    = var.public_subnet_cidr_blocks
  azs               = data.aws_availability_zones.azs.names

  enable_nat_gateway        = true
  single_nat_gateway        = true
  enable_dns_hostnames      = true

  tags = {
    "kubernetes.io/cluster/myapp-eks-cluster" = "shared"
  }
  
  public_subnet_tags = {
    "kubernetes.io/cluster/myapp-eks-cluster" = "shared"
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/myapp-eks-cluster" = "shared"
    "kubernetes.io/role/internal-elb" = 1
  }
}
```

**- Create 'terraform.tfvars' file:**

```
vpc_cidr_block = "10.0.0.0/16"
private_subnet_cidr_blocks = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
public_subnet_cidr_blocks = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]
```

. Run **'terraform plan'** to verify the plan.


<img src="https://i.imgur.com/YWyvHa7.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## 2. Create EKS Cluster

**.** Use the AWS EKS cluster module available in the Terraform registry to create the EKS cluster.

Create **'eks-cluster.tf'** file:

```
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.8.5"

  cluster_name               = "myapp-eks-cluster"
  cluster_version            = "1.27"
  cluster_endpoint_public_access = true

  subnet_ids                 = module.myapp-vpc.private_subnets
  vpc_id                     = module.myapp-vpc.vpc_id

  tags = {
    environment = "development"
    application = "myapp"
  }

  eks_managed_node_groups = {
    dev = {
        min_size     = 1
        max_size     = 3
        desired_size = 3

        instance_types = ["t2.small"]
    }
  }
}
```

**.** Apply configuration files:

```
% terraform init             
% terraform apply --auto-approve
```

<img src="https://i.imgur.com/tPQrei8.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>





## 3. Deploy nginx-App into our cluster

**- Update kubeconfig:**

```
% aws eks update-kubeconfig --name myapp-eks-cluster --region eu-west-2

% kubectl get node

% kubectl apply -f ~/Documents/terraform/nginx-config.yaml

% kubectl get pod -w

% kubectl get svc
```

<img src="https://i.imgur.com/x1vNoiq.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>
