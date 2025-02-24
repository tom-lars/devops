# How to create an EC2 Instance using Terraform

### `main.tf` file
```
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

resource "aws_security_group" "my_sg" {
  name        = "my_security_group"
  description = "Allow SSH and TCP traffic"

  ingress {
    from_port  = 22
    to_port    = 22
    protocol   = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port  = 80
    to_port    = 80
    protocol   = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port  = 0
    to_port    = 0
    protocol   = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "my_instance" {
  ami           = "ami-03b8adbf322415fd0"
  instance_type = "t2.micro"

  vpc_security_group_ids = [aws_security_group.my_sg.id]

  tags = {
    Name = "MyEC2Instance"
  }
}
```
