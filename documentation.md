# Deploy A Secure And Scalable Three-Tier Architecture On AWS With Terraform

A three-tier architecture is a software design pattern or architecture that divides an application into three main logical layers or tiers, each responsible for specific functionalities and tasks. These tiers are loosely coupled, meaning they can be developed and maintained independently, allowing for easier scalability and maintenance of the overall system. The three tiers are typically named as follows:

**Presentation Tier (Frontend):**
The presentation tier, also known as the frontend, is the top layer that interacts directly with users and handles user interfaces and user interactions. It is responsible for presenting the data and results to the end-users in a human-readable format. In a web application, this tier includes the web browser or mobile app through which users interact with the application. The primary goal of this tier is to provide a user-friendly interface and receive user input, which is then forwarded to the application tier.

**Application Tier (Backend):**
The application tier, also known as the business logic or backend, is the middle layer that processes user requests, performs data processing, and implements the business logic of the application. It acts as an intermediary between the frontend and the database tier. This layer handles tasks such as data validation, business rules, and workflows. It is also responsible for processing and manipulating data retrieved from the database and preparing it for presentation in the frontend.

**Database Tier:**
The database tier is the bottom layer responsible for storing and managing data used by the application. It stores and retrieves data requested by the application tier. This layer could use various database systems like SQL databases (e.g., MySQL, PostgreSQL) or NoSQL databases (e.g., MongoDB, Cassandra) based on the application's requirements. The database tier ensures data persistence and integrity, enabling the application to maintain and manage large amounts of structured or unstructured data.

<br>

<br>

## Advantages of Three-Tier Architecture:

- **Scalability:** The architecture's modularity allows each tier to be scaled independently, making it easier to handle increasing workloads.
- **Maintainability:** The separation of concerns makes it easier to update or modify one tier without affecting the others, reducing the risk of introducing bugs or issues.
- **Security:** By restricting direct access between tiers, it enhances the overall security of the system.
- **Flexibility:** Each tier can be developed using different technologies and languages, providing flexibility for the development team.
- **Reusability:** The separation of concerns allows for better code re-use across different projects.

<br>

Overall, the three-tier architecture provides a structured and organized way to design and build complex applications, making them more manageable, scalable, and maintainable over time.

<br>

<br>

## Project Steps

<br>

<br>

## Set up the Terraform environment:
Before you proceed, make sure you have Terraform installed and configured to interact with your AWS account.
Create a new directory for your Terraform configuration and initialize it with the necessary files:

```

mkdir three-tier
cd three-tier

```

<br>

<br>

## Create Terraform configuration files
Inside the directory, create the following files:

- **versions.tf:** terraform providers file.
- **network.tf:** sets up the VPC and its subnets, as well as security groups and rules for the web, app, and database tiers
- **database.tf:** creates a MySQL RDS instance, security group, and rules for the database tier
- **compute.tf:** sets up the compute resources for the web and app tiers, including launch configurations, auto-scaling groups, application load balancers, target groups, and listeners
- **variables.tf:** Declare variables here.
- **outputs.tf:** Define outputs to display after deployment.
- **terraform.tfvars:** Set values for variables in this file. Make sure to keep sensitive information, such as AWS access keys, out of version control.

<br>

<br>

## Write Terraform Configuration

In the **versions.tf** add the 
### Define Terraform configuration block
```

terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = ">= 4.8.0"
    }
  }
}

```

<br>

This block defines the terraform settings for the configuration. In this case, we're using the AWS provider from HashiCorp, and we're specifying that we need at least version 4.8.0 or newer. By specifying the minimum required version, we ensure that our configuration is compatible with the features and resources we need.

<br>

### Define Providers Block
```

provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile
}

```

<br>

Providers allow Terraform to interact with cloud providers, SaaS providers, and other APIs. In this case, we set the `AWS region` and `profile` using variables defined elsewhere in our configuration (var.aws_region and var.aws_profile).

<br>

### Define Locals Block
```

locals {
  common_tags = {
    Terraform   = "true"
    Environment = var.environment
  }
}

```

<br>

The `locals` block defines local values that can be used throughout the Terraform configuration.

<br>

<br>

<img width="1146" alt="versions" src="https://github.com/earchibong/terraform-3tier/assets/92983658/2e7b3355-73d0-4158-ac02-89c898e5a6e3">

<br>

<br>

## Set Up Network Configuration
in the `network.tf` file, we will set up the VPC, subnets, and security groups for our infrastructure.

### set Up VPC configuration
```

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "main"
  cidr = var.vpc_cidr

  azs              = ["${var.aws_region}a", "${var.aws_region}b"]
  public_subnets   = var.public_subnets
  private_subnets  = var.private_subnets
  database_subnets = var.database_subnets

  enable_nat_gateway = true
  enable_vpn_gateway = false

  public_subnet_names   = ["web-subnet-1", "web-subnet-2"]
  private_subnet_names  = ["app-subnet-1", "app-subnet-2"]
  database_subnet_names = ["db-subnet-1", "db-subnet-2"]

  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}

```

<br>

### Configure Security Groups for web tier
Here, we'll define the security group for the web tier allowing inbound traffic. This security group rule will allow inbound traffic on port 80 (HTTP) and allow SSH access to the web tier.

<br>

```

resource "aws_security_group" "web" {
  name        = "web"
  description = "Allow inbound traffic for web tier"
  vpc_id      = module.vpc.vpc_id
}


resource "aws_security_group_rule" "web" {
  security_group_id = aws_security_group.web.id

  type        = "ingress"
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}


resource "aws_security_group_rule" "web_ssh" {
  security_group_id = aws_security_group.web.id

  type        = "ingress"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"] # Replace with your desired CIDR blocks for SSH access
}


```

<br>

<br>

<img width="965" alt="ntwork_1" src="https://github.com/earchibong/terraform-3tier/assets/92983658/a632fa3f-0b94-450c-baf3-219ab2c3214b">

<br>

<br>

## Create Compute Resources and Autoscaling
The `compute.tf file`, sets up the compute resources for our application

### Web Tier

we'll create an `aws_launch_template` and an `aws_autoscaling_group`. The launch template specifies the instance configuration, such as the AMI ID, instance type, and security group. The autoscaling group uses this template to launch instances and manages the scaling policies.


```

resource "aws_launch_template" "web" {
  name_prefix   = "web-lt-"
  image_id      = "ami-007855ac798b5175e" # AMI ID for Ubuntu 22.04; replace it with the appropriate AMI ID for your use case
  instance_type = var.web_instance_type   #change instance size to your specifications in .tfvars file, such as M5 General

  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = base64encode(templatefile("${path.module}/userdata_web.tpl", {}))

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "web" {
  name_prefix          = "web-asg-"
  launch_configuration = aws_launch_template.web.id
  min_size             = 1
  max_size             = 4
  desired_capacity     = 2
  vpc_zone_identifier  = module.vpc.public_subnets
  target_group_arns    = [aws_lb_target_group.web.arn]

  tag {
    key                 = "Name"
    value               = "web"
    propagate_at_launch = true
  }

  tag {
    key                 = "Terraform"
    value               = "true"
    propagate_at_launch = true
  }

  tag {
    key                 = "Environment"
    value               = "dev"
    propagate_at_launch = true
  }

  lifecycle {
    create_before_destroy = true
  }
}

```

<br>

<br>


