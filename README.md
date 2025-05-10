# **Deploying a Highly Available Web Application on AWS Using Terraform**

Deploying a highly available web application on AWS using Terraform is a powerful way to leverage the cloud's capabilities while ensuring your application remains resilient and scalable. In this blog post, we'll walk through the steps to set up a highly available web app. This guide assumes you have a basic understanding of AWS services and Terraform.

## **Understanding High Availability**

![](https://media2.dev.to/cdn-cgi/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Ftrgsa0oh3nll0broxh6t.png)

High availability (HA) is crucial for applications that demand continuous uptime. It involves deploying your application across multiple Availability Zones (AZs) to mitigate the risk of downtime due to hardware failures or maintenance. By distributing resources, you can ensure that even if one component fails, others can take over seamlessly.

## **Prerequisites**

Before we begin, ensure you have:

- An AWS account
- Terraform installed on your local machine
- Basic knowledge of AWS services such as EC2, VPC, and Load Balancers

## **On to the steps:**

## **Step 1: Setting Up Your Terraform Configuration**

Start by creating a directory for your Terraform configuration and a file named `main.tf`. This file will contain the necessary resources for your deployment.

### **Sample `main.tf` Configuration**

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "allow_http" {
  vpc_id = aws_vpc.main.id

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

resource "aws_launch_configuration" "app" {
  name          = "app-launch-configuration"
  image_id      = "ami-0e86e20dae9224db8"  # Your specified AMI
  instance_type = "t2.micro"
  security_groups = [aws_security_group.allow_http.id]

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "app" {
  launch_configuration = aws_launch_configuration.app.id
  min_size            = 2
  max_size            = 5
  desired_capacity    = 2
  vpc_zone_identifier = [aws_subnet.private.id]
  health_check_type   = "EC2"
  health_check_grace_period = 300

  tag {
    key                 = "Name"
    value               = "AppInstance"
    propagate_at_launch = true
  }
}

resource "aws_elb" "app" {
  name               = "app-load-balancer"
  availability_zones = ["us-east-1a", "us-east-1b"]

  listener {
    instance_port     = 80
    instance_protocol = "HTTP"
    lb_port           = 80
    lb_protocol       = "HTTP"
  }

  health_check {
    target              = "HTTP:80/"
    interval            = 30
    timeout             = 5
    healthy_threshold  = 2
    unhealthy_threshold = 2
  }

  instances = aws_autoscaling_group.app.instances
}

```

## **Step 2: Initialize and Apply Terraform**

After creating your `main.tf`, navigate to your project directory in the terminal and run the following commands:

```
terraform init
terraform apply

```

This will initialize Terraform and apply your configuration, creating the necessary AWS resources.

## **Step 3: Accessing Your Application**

Once the deployment is complete, you can access your application through the Elastic Load Balancer (ELB) DNS name provided in the output of the `terraform apply` command. This DNS name will route traffic to your EC2 instances, ensuring high availability.

## **Architecture Diagram**

![](https://media2.dev.to/cdn-cgi/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fbu18ougpz98j5kn1unav.png)

## **Parting Shot**

Deploying a highly available web application on AWS using Terraform not only simplifies infrastructure management but also enhances your application's resilience. By following the steps outlined above, you can create a robust architecture that scales with your needs. Remember to monitor your infrastructure and make adjustments as necessary to maintain optimal performance and availability.

With Terraform, the possibilities are endless. Explore further by integrating additional AWS services, such as RDS for database management or S3 for storage solutions, to enhance your application's capabilities.
