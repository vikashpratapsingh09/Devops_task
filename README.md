Please find the task here.
###############################################################################
provider.tf

provider "aws" {
  region = "us-east-1"
}
variables.tf

variable "vpc_id" {}
variable "subnet_ids" {
  type = list(string)
}
variable "instance_type" {
  default = "t3.micro"
}
variable "key_name" {}

ec2/user_data.sh

#!/bin/bash
apt update -y
apt install apache2 php libapache2-mod-php -y
systemctl enable apache2
systemctl start apache2

ec2/security_group.tf
resource "aws_security_group" "web_sg" {
  name_prefix = "web-sg-"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_IP/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

ec2/instance.tf
resource "aws_launch_template" "app_lt" {
  name_prefix   = "app-lt-"
  image_id      = "ami-0fc5d935ebf8bc3bc" # Ubuntu 22.04 AMI ID in us-east-1
  instance_type = var.instance_type
  key_name      = var.key_name

  user_data = base64encode(file("${path.module}/user_data.sh"))

  network_interfaces {
    associate_public_ip_address = true
    security_groups = [aws_security_group.web_sg.id]
  }

  iam_instance_profile {
    name = aws_iam_instance_profile.ec2_profile.name
  }
}

alb/alb.tf
resource "aws_lb" "app_lb" {
  name               = "app-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web_sg.id]
  subnets            = var.subnet_ids
}

resource "aws_lb_target_group" "app_tg" {
  name     = "app-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id
}

resource "aws_lb_listener" "app_listener" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }
}

asg/asg.tf

resource "aws_autoscaling_group" "app_asg" {
  desired_capacity     = 2
  max_size             = 4
  min_size             = 1
  vpc_zone_identifier  = var.subnet_ids
  target_group_arns    = [aws_lb_target_group.app_tg.arn]
  launch_template {
    id      = aws_launch_template.app_lt.id
    version = "$Latest"
  }

  health_check_type = "EC2"
  health_check_grace_period = 300
}

rds/rds.tf

resource "aws_db_instance" "app_db" {
  allocated_storage    = 20
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.micro"
  name                 = "appdb"
  username             = aws_secretsmanager_secret_version.db_secret.secret_string["username"]
  password             = aws_secretsmanager_secret_version.db_secret.secret_string["password"]
  publicly_accessible  = false
  vpc_security_group_ids = [aws_security_group.web_sg.id]
  db_subnet_group_name = aws_db_subnet_group.main.name
  skip_final_snapshot  = true
}

secrets_manager/kms.tf
resource "aws_kms_key" "secrets_key" {
  description = "KMS key for Secrets Manager"
}

resource "aws_kms_alias" "secrets_key_alias" {
  name          = "alias/secretsKey"
  target_key_id = aws_kms_key.secrets_key.id
}

secrets_manager/secrets.tf
resource "aws_secretsmanager_secret" "db_secret" {
  name = "rds-db-credentials"
  kms_key_id = aws_kms_key.secrets_key.id
  rotation_lambda_arn = aws_lambda_function.db_rotation_lambda.arn
  rotation_rules {
    automatically_after_days = 7
  }
}

ssm/ssm.tf
resource "aws_iam_role" "ec2_ssm_role" {
  name = "ec2-ssm-role"
  assume_role_policy = data.aws_iam_policy_document.assume_role_policy.json
}

data "aws_iam_policy_document" "assume_role_policy" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-profile"
  role = aws_iam_role.ec2_ssm_role.name
}

resource "aws_iam_role_policy_attachment" "ssm_attach" {
  role       = aws_iam_role.ec2_ssm_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

cloudfront/cloudfront.tf
resource "aws_cloudfront_distribution" "app_distribution" {
  origin {
    domain_name = aws_lb.app_lb.dns_name
    origin_id   = "alb-origin"
  }

  enabled             = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "alb-origin"
    viewer_protocol_policy = "redirect-to-https"
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

github_actions/github_actions.tf
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"] # GitHub Actions Thumbprint
}

# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v2
      - run: terraform init
      - run: terraform apply -auto-approve

code_quality/sonar.tf

# .github/workflows/code_quality.yml
name: Code Quality

on: [push]

jobs:
  sonarcloud:
    name: SonarCloud
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: SonarCloud Scan
      uses: SonarSource/sonarcloud-github-action@v2
      with:
        organization: YOUR_SONAR_ORG
        projectKey: YOUR_PROJECT_KEY
        token: ${{ secrets.SONAR_TOKEN }}







