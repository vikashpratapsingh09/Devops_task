Please find the task here.
###############################################################################
terraform-aws-infra/
├── main.tf
├── variables.tf
├── outputs.tf
├── userdata.sh
└── github-actions-workflow.yml
variable "aws_region" {
  description = "AWS region"
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  default     = "t3.medium"
}

variable "key_name" {
  description = "Key pair name"
  default     = "my-key-pair"
}

variable "vpc_id" {
  description = "VPC ID"
  default     = ""
}

variable "subnet_ids" {
  description = "List of subnet IDs"
  type        = list(string)
  default     = []
}

variable "db_username" {
  description = "RDS master username"
  default     = "admin"
}

variable "db_password" {
  description = "RDS master password"
  sensitive   = true
}

variable "db_name" {
  description = "RDS database name"
  default     = "mydatabase"
}

variable "github_repo" {
  description = "GitHub repository name (owner/repo)"
  default     = "myorg/myrepo"
}

variable "github_token" {
  description = "GitHub personal access token"
  sensitive   = true
}

variable "domain_name" {
  description = "Domain name for CloudFront"
  default     = "example.com"
}

variable "certificate_arn" {
  description = "ACM certificate ARN"
  default     = ""
}

terraform {
  required_version = ">= 1.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
    github = {
      source  = "integrations/github"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

provider "github" {
  token = var.github_token
}

# Create KMS key for Secrets Manager
resource "aws_kms_key" "secrets_key" {
  description             = "KMS key for Secrets Manager"
  enable_key_rotation     = true
  deletion_window_in_days = 7
}

# Create secret for RDS with rotation
resource "aws_secretsmanager_secret" "rds_secret" {
  name       = "rds-credentials"
  kms_key_id = aws_kms_key.secrets_key.key_id
}

resource "aws_secretsmanager_secret_version" "rds_secret_version" {
  secret_id = aws_secretsmanager_secret.rds_secret.id
  secret_string = jsonencode({
    username = var.db_username
    password = var.db_password
    engine   = "mysql"
    host     = aws_db_instance.default.address
    dbname   = var.db_name
  })
}

resource "aws_secretsmanager_secret_rotation" "rds_secret_rotation" {
  secret_id           = aws_secretsmanager_secret.rds_secret.id
  rotation_lambda_arn = aws_lambda_function.rotation_lambda.arn

  rotation_rules {
    automatically_after_days = 7
  }
}

# Create RDS instance
resource "aws_db_instance" "default" {
  allocated_storage      = 20
  storage_type           = "gp2"
  engine                 = "mysql"
  engine_version         = "5.7"
  instance_class         = "db.t3.micro"
  name                   = var.db_name
  username               = var.db_username
  password               = var.db_password
  parameter_group_name   = "default.mysql5.7"
  skip_final_snapshot    = true
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  db_subnet_group_name   = aws_db_subnet_group.default.name
}

resource "aws_db_subnet_group" "default" {
  name       = "main-db-subnet-group"
  subnet_ids = var.subnet_ids
}

# Security Group for RDS
resource "aws_security_group" "rds_sg" {
  name        = "rds-security-group"
  description = "Allow traffic to RDS"

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.web_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security Group for Web
resource "aws_security_group" "web_sg" {
  name        = "web-security-group"
  description = "Allow HTTP/HTTPS and SSH traffic"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
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

# IAM Role for Session Manager
resource "aws_iam_role" "session_manager_role" {
  name = "session-manager-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "session_manager_attach" {
  role       = aws_iam_role.session_manager_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# Launch Template
resource "aws_launch_template" "web_server" {
  name_prefix   = "web-server-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  key_name      = var.key_name

  iam_instance_profile {
    name = aws_iam_instance_profile.session_manager_profile.name
  }

  network_interfaces {
    associate_public_ip_address = true
    security_groups             = [aws_security_group.web_sg.id]
  }

  user_data = filebase64("userdata.sh")
}

resource "aws_iam_instance_profile" "session_manager_profile" {
  name = "session-manager-profile"
  role = aws_iam_role.session_manager_role.name
}

# Auto Scaling Group
resource "aws_autoscaling_group" "web_asg" {
  desired_capacity    = 2
  max_size            = 4
  min_size            = 1
  vpc_zone_identifier = var.subnet_ids

  launch_template {
    id      = aws_launch_template.web_server.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "web-server"
    propagate_at_launch = true
  }
}

# Application Load Balancer
resource "aws_lb" "web_alb" {
  name               = "web-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web_sg.id]
  subnets            = var.subnet_ids
}

resource "aws_lb_target_group" "web_tg" {
  name     = "web-target-group"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

resource "aws_lb_listener" "web_listener" {
  load_balancer_arn = aws_lb.web_alb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web_tg.arn
  }
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "web_distribution" {
  origin {
    domain_name = aws_lb.web_alb.dns_name
    origin_id   = "alb-origin"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "http-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Web distribution"
  default_root_object = "index.php"

  aliases = [var.domain_name]

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "alb-origin"

    forwarded_values {
      query_string = true
      cookies {
        forward = "all"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  price_class = "PriceClass_100"

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = var.certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
}

# GitHub Actions IAM Role
resource "aws_iam_role" "github_actions" {
  name = "github-actions-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::123456789012:root" # Replace with your account ID
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
            "token.actions.githubusercontent.com:sub" = "repo:${var.github_repo}:ref:refs/heads/main"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "github_actions_attach" {
  role       = aws_iam_role.github_actions.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess" # Narrow this down for production
}

# Lambda function for secret rotation
resource "aws_lambda_function" "rotation_lambda" {
  filename      = "lambda_function_payload.zip"
  function_name = "secret-rotation-lambda"
  role          = aws_iam_role.iam_for_lambda.arn
  handler       = "index.handler"
  runtime       = "python3.8"
}

resource "aws_iam_role" "iam_for_lambda" {
  name = "iam_for_lambda"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}

#!/bin/bash
apt-get update -y
apt-get install -y apache2 php libapache2-mod-php php-mysql

# Configure Apache
cat > /var/www/html/index.php <<EOF
<?php
phpinfo();
?>
EOF

systemctl enable apache2
systemctl start apache2

# Install AWS CLI and SSM Agent
apt-get install -y awscli amazon-ssm-agent
systemctl enable amazon-ssm-agent
systemctl start amazon-ssm-agent

output "alb_dns_name" {
  value = aws_lb.web_alb.dns_name
}

output "cloudfront_domain" {
  value = aws_cloudfront_distribution.web_distribution.domain_name
}

output "rds_endpoint" {
  value = aws_db_instance.default.endpoint
}

output "github_actions_role_arn" {
  value = aws_iam_role.github_actions.arn
}

output "secret_arn" {
  value = aws_secretsmanager_secret.rds_secret.arn
}

name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.0'
        tools: composer:v2
        
    - name: Install dependencies
      run: composer install --prefer-dist --no-progress --no-suggest
      
    - name: Run tests
      run: vendor/bin/phpunit
      
  code-quality:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: PHP Code Sniffer
      uses: docker://oskarstark/php-cs-fixer-ga
      with:
        args: --dry-run --diff --diff-format=udiff
        
    - name: PHPStan
      uses: docker://ghcr.io/phpstan/phpstan
      with:
        args: analyse --no-progress --level=8 src tests
        
  deploy:
    needs: [test, code-quality]
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      
    steps:
    - uses: actions/checkout@v2
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
        aws-region: us-east-1
        
    - name: Deploy to AWS
      run: |
        # Add your deployment commands here
        echo "Deploying to AWS..."
