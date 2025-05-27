# AWS Application Load Balancer with Terraform

A complete Terraform configuration to deploy an Application Load Balancer (ALB) in AWS with path-based routing for API and Web traffic.

## 📋 Overview

This Terraform setup creates an Application Load Balancer in AWS Mumbai region (`ap-south-1`) that routes traffic to existing EC2 instances based on URL paths:
- `/api/*` requests → API backend instance
- `/web/*` requests → Web backend instance

## 🏗️ Architecture

```
Internet → ALB → Path-based routing → Target Groups → EC2 Instances
                     ├── /api/* → API Target Group → API Instance
                     └── /web/* → Web Target Group → Web Instance
```

## 📁 Project Structure

```
terraform-alb/
├── main.tf           # Main Terraform configuration
├── variables.tf      # Variable definitions
├── terraform.tfvars  # Variable values (customize this)
├── outputs.tf        # Output definitions
└── README.md         # This file
```

## 🔧 Prerequisites

Before you begin, ensure you have:

1. **AWS CLI configured** with appropriate credentials
2. **Terraform installed** (version 0.12 or later)
3. **Two existing EC2 instances** in the Mumbai region
4. **Instance IDs** of your EC2 instances

## 📦 Files Explanation

### `main.tf` - Main Configuration

The main Terraform file that defines all AWS resources:

#### Provider Configuration
```hcl
provider "aws" {
  region = "ap-south-1"
}
```
Sets AWS provider to use Mumbai region.

#### Data Sources
```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}
```
Fetches the default VPC and its subnets automatically.

#### Security Group
```hcl
resource "aws_security_group" "alb_sg" {
  name        = "alb-sg"
  description = "Allow HTTP traffic to ALB"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
Creates a security group allowing HTTP traffic (port 80) from anywhere.

#### Application Load Balancer
```hcl
resource "aws_lb" "app_alb" {
  name               = "my-app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = data.aws_subnets.default_subnets.ids
}
```
Creates an internet-facing ALB across all default subnets.

#### Target Groups
```hcl
resource "aws_lb_target_group" "api_tg" {
  name     = "api-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default.id

  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
    matcher             = "200"
  }
}
```
Creates target groups for API and Web instances with health checks.

#### Target Group Attachments
```hcl
resource "aws_lb_target_group_attachment" "api_attachment" {
  target_group_arn = aws_lb_target_group.api_tg.arn
  target_id        = var.api_instance_id
  port             = 80
}
```
Attaches EC2 instances to their respective target groups.

#### Listener and Routing Rules
```hcl
resource "aws_lb_listener" "http_listener" {
  load_balancer_arn = aws_lb.app_alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "404 Not Found"
      status_code  = "404"
    }
  }
}
resource "aws_lb_listener_rule" "api_rule" {
  listener_arn = aws_lb_listener.http_listener.arn
  priority     = 10

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api_tg.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}

resource "aws_lb_listener_rule" "web_rule" {
  listener_arn = aws_lb_listener.http_listener.arn
  priority     = 20

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web_tg.arn
  }

  condition {
    path_pattern {
      values = ["/web/*"]
    }
  }
}

```
Creates HTTP listener with default 404 response for unmatched paths.

### `variables.tf` - Variable Definitions

```hcl
variable "api_instance_id" {
  description = "Instance ID for API backend"
  type        = string
}

variable "web_instance_id" {
  description = "Instance ID for Web backend"
  type        = string
}
```
Defines input variables for EC2 instance IDs.

### `terraform.tfvars` - Variable Values

```hcl
api_instance_id = "i-0abc123456789def0"
web_instance_id = "i-0123456789abcdef0"
```
**⚠️ Important**: Replace these with your actual EC2 instance IDs.

### `outputs.tf` - Output Values

```hcl
output "alb_dns_name" {
  description = "Public DNS name of the Application Load Balancer"
  value       = aws_lb.app_alb.dns_name
}
```
Outputs the ALB's DNS name for easy access.

## 🚀 Deployment Steps

### 1. Clone or Create the Project
```bash
mkdir terraform-alb
cd terraform-alb
```

### 2. Create the Terraform Files
Create all four files (`main.tf`, `variables.tf`, `terraform.tfvars`, `outputs.tf`) with the content provided above.

### 3. Update Instance IDs
Edit `terraform.tfvars` and replace the example instance IDs with your actual EC2 instance IDs:
```hcl
api_instance_id = "i-your-api-instance-id"
web_instance_id = "i-your-web-instance-id"
```

### 4. Initialize Terraform
```bash
terraform init
```
This downloads the AWS provider and initializes the working directory.

### 5. Plan the Deployment
```bash
terraform plan
```
Review the planned changes before applying.

### 6. Apply the Configuration
```bash
terraform apply
```
Type `yes` when prompted to create the resources.

### 7. Note the ALB DNS Name
After successful deployment, Terraform will output the ALB DNS name:
```
Outputs:
alb_dns_name = "my-app-alb-123456789.ap-south-1.elb.amazonaws.com"
```

## 🧪 Testing the Setup

Once deployed, test the path-based routing:

### API Endpoint
```bash
curl http://your-alb-dns-name/api/health
# or visit in browser: http://your-alb-dns-name/api/anything
```

### Web Endpoint
```bash
curl http://your-alb-dns-name/web/home
# or visit in browser: http://your-alb-dns-name/web/anything
```

### Default Response (404)
```bash
curl http://your-alb-dns-name/unknown
# Should return: 404 Not Found
```

## 📊 Resource Summary

This configuration creates:
- 1 Application Load Balancer
- 1 Security Group (allows HTTP traffic)
- 2 Target Groups (API and Web)
- 2 Target Group Attachments
- 1 HTTP Listener
- 2 Listener Rules (path-based routing)

## 🔍 Key Features

### Path-Based Routing
- **API Route**: `/api/*` → Routes to API instance
- **Web Route**: `/web/*` → Routes to Web instance
- **Default**: Any other path returns 404

### Health Checks
- **Path**: `/` (customize if needed)
- **Interval**: 30 seconds
- **Timeout**: 5 seconds
- **Healthy Threshold**: 2 consecutive successes
- **Unhealthy Threshold**: 2 consecutive failures

### Security
- ALB accepts HTTP traffic from anywhere (0.0.0.0/0)
- EC2 instances should have security groups allowing traffic from ALB

## 🛠️ Customization Options

### Change Health Check Path
If your application uses a different health check endpoint:
```hcl
health_check {
  path = "/health"  # Change this
  # ... other settings
}
```

### Add HTTPS Support
To add SSL/TLS, you'll need:
1. An SSL certificate in AWS Certificate Manager
2. Additional listener on port 443
3. Updated security group rules

### Different Ports
If your applications run on different ports:
```hcl
resource "aws_lb_target_group_attachment" "api_attachment" {
  target_group_arn = aws_lb_target_group.api_tg.arn
  target_id        = var.api_instance_id
  port             = 8080  # Change this
}
```

## 🧹 Cleanup

To destroy all created resources:
```bash
terraform destroy
```
Type `yes` when prompted.

## 📋 Troubleshooting

### Common Issues

1. **Instance Not Healthy**
   - Check if your application is running on port 80
   - Verify security groups allow traffic from ALB
   - Check health check path returns HTTP 200

2. **502 Bad Gateway**
   - Target instance is not responding
   - Check instance security group
   - Verify application is listening on correct port

3. **404 Not Found**
   - Path doesn't match `/api/*` or `/web/*`
   - Check listener rules configuration

### Verification Commands
```bash
# Check ALB status
aws elbv2 describe-load-balancers --region ap-south-1

# Check target group health
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:...

# Check listener rules
aws elbv2 describe-rules --listener-arn arn:aws:elasticloadbalancing:...
```
