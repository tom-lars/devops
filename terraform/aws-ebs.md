## How to create an EBS volume and attach it to EC2 instance using Terraform

main.tf
```hcl
resource "aws_instance" "my_instance" {
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name

  user_data = file("user-data.sh")

  tags = {
    name = "MyEC2Instance"
  }
}

resource "aws_ebs_volume" "my_ebs_volume" {
  availability_zone = aws_instance.my_instance.availability_zone
  size              = var.ebs_volume_size

  tags = {
    name = "my_ebs_volume"
  }
}

resource "aws_volume_attachment" "volume_attachment" {
  device_name = "/dev/xvdf"
  volume_id   = aws_ebs_volume.my_ebs_volume.id
  instance_id = aws_instance.my_instance.id
}
```

variables.tf

```hcl
variable "aws_region" {
  description = "AWS region to deploy resources"
  default     = "ap-south-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  default     = "t2.micro"
}

variable "ami_id" {
  description = "Amazon Machine Image (AMI) ID"
  default     = "ami-0c55b159cbfafe1f0"
}

variable "ebs_volume_size" {
  description = "Size of EBS volume in GB"
  default     = 10
}

variable "key_name" {
  description = "SSH key for access"
  default     = "my-key"
}
```

terraform.tfvars

```hcl
variable "aws_region" {
  description = "AWS region to deploy resources"
  default     = "ap-south-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  default     = "t2.micro"
}

variable "ami_id" {
  description = "Amazon Machine Image (AMI) ID"
  default     = "ami-0c55b159cbfafe1f0"
}

variable "ebs_volume_size" {
  description = "Size of EBS volume in GB"
  default     = 10
}

variable "key_name" {
  description = "SSH key for access"
  default     = "my-key"
}
```

outputs.tf

```hcl
output "instance_public_ip" {
  value = aws_instance.my_instance.public_ip
}

output "ebs_volume_id" {
  value = aws_ebs_volume.my_ebs_volume.id
}
```

provider.tf

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"
    }
  }

  required_version = ">1.2.0"
}

provider "aws" {
  region = "ap-south-1"
}
```

user-data.sh

```bash
#!/bin/bash
sudo mkfs -t ext4 /dev/xvdf
sudo mkdir /mnt/ebs
sudo mount /dev/xvdf /mnt/ebs

sudo echo "/dev/xvdf /mnt/ebs ext4 defaults,nofail 0 2" >> /etc/fstab
```
