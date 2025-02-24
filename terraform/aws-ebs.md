# How to create an EBS volume and attach it to EC2 instance using Terraform

### `provider.tf` file

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
- List the Providers in `provider.tf`
## 

### `variables.tf` file

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
- Variables can be declared here and default values can be set here.
- The variables can be overwritten in `terraform.tfvars`.
##

### `user-data.sh` script

```bash
#!/bin/bash
sudo mkfs -t ext4 /dev/xvdf
sudo mkdir /mnt/ebs
sudo mount /dev/xvdf /mnt/ebs

sudo echo "/dev/xvdf /mnt/ebs ext4 defaults,nofail 0 2" >> /etc/fstab
```
- This user data is used to format volume and mount it on instance.
##

### `main.tf` file
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
- Create an Instance
- Create an EBS volume
- Attach the volume to the Instance
##

### `terraform.tfvars` file

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
- The variables can be overwrittern through this file
##

### `outputs.tf` file

```hcl
output "instance_public_ip" {
  value = aws_instance.my_instance.public_ip
}

output "ebs_volume_id" {
  value = aws_ebs_volume.my_ebs_volume.id
}
```
- The output will be displayed through this file
##

To initialize the terraform
```
terraform init
```
##

To format the config file
```
terraform fmt
```
##

To validate the config file
```
terraform validate
``` 
## 

To plan what is about the happen with the config file
```
terraform plan
```
##

To apply the changes
```
terraform apply
```
> To auto approve user `-auto-approve`
##

To undo what has been created with the config file
```
terraform destroy
``` 



