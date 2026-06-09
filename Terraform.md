

Terraform is an open-source **Infrastructure as Code (IaC)** tool that lets you manage and provision AWS resources using configuration files instead of manual clicks in the AWS Console. 

## Core Concepts You Need to Know


|Concept|What It Does|Analogy|
|---|---|---|
|**Provider**|Connects Terraform to AWS API|Like a driver that talks to AWS|
|**Resource**|An AWS service you create (e.g., EC2, S3)|Building instructions for each Lego brick|
|**State**|Tracks what's actually deployed|Memory of what Terraform already built|
|**Variables**|Parameterize your config (regions, instance types)|Fill-in-the-blank templates|
|**Outputs**|Shows important info after deployment (e.g., IP address)|Report card after building|
|**Module**|Reusable, pre-built infrastructure packages|Pre-assembled Lego kit|
|**HCL**|HashiCorp Configuration Language|The "recipe" language for Terraform|

### The Core Workflow (4 Commands)

1. **`terraform init`** – Downloads providers and prepares your working directory[](https://github.com/manas-shinde/terraform-basics-aws)
    
2. **`terraform plan`** – Shows exactly what will change (safe preview)[](https://github.com/manas-shinde/terraform-basics-aws)
    
3. **`terraform apply`** – Creates/updates real AWS resources[](https://github.com/manas-shinde/terraform-basics-aws)
    
4. **`terraform destroy`** – Removes everything you created[](https://github.com/manas-shinde/terraform-basics-aws)
    

> **💡 Key Insight:** Terraform works declaratively — you describe the _desired_ state of your infrastructure in code, and Terraform figures out what changes are needed to reach that state.



## First AWS Deployment (EC2 Instance + S3 Bucket)

### Prerequisites

**AWS Account**
**AWS CLI installed and configured** – Run `aws configure` with your Access Key and Secret Key

### Create a Project Folder

```bash

mkdir my-terraform-project
cd my-terraform-project

```


### Create `main.tf`

```bash

# Provider configuration
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"  # Change to your preferred region
}

# Create a VPC (Virtual Private Cloud)
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "terraform-vpc"
  }
}

# Create a subnet inside the VPC
resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  tags = {
    Name = "terraform-subnet"
  }
}

# Create an EC2 instance (t2.micro = Free Tier eligible)
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2 (us-east-1)
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.main.id

  tags = {
    Name = "terraform-ec2"
  }
}

# Create an S3 bucket (bucket names must be globally unique)
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name-12345"  # ⚠️ Change this!
  tags = {
    Name = "terraform-s3-bucket"
  }
}

# Output the EC2 public IP after deployment
output "instance_public_ip" {
  value = aws_instance.web.public_ip
}


```


### Deploy!

```bash

terraform init        # Downloads AWS provider
terraform plan        # Previews what will be created
terraform apply

```


### Clean Up

```bash
terraform destroy
```


## AWS Authentication Methods

| Method                                          | When to Use                                    | Command / Setup                                                                                                                                                                               |
| ----------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **AWS CLI credentials** (easiest for beginners) | Local development                              | `aws configure` → enters credentials in `~/.aws/credentials`                                                                                                                                  |
| **Environment variables**                       | CI/CD scripts                                  | `export AWS_ACCESS_KEY_ID="..."` `export AWS_SECRET_ACCESS_KEY="..."`                                                                                                                         |
| **IAM roles**                                   | EC2 instances running Terraform                | Attach an IAM role to the EC2 instance[](https://docs.aws.amazon.com/ja_jp/prescriptive-guidance/latest/terraform-aws-provider-best-practices/terraform-aws-provider-best-practices.pdf#12#1) |
| **OIDC / Workload Identity**                    | GitHub Actions, GitLab CI (no long-lived keys) | Configure OIDC trust to AWS IAM role                                                                                                                                                          |


**Never hardcode credentials in `main.tf`** – this risks exposing them in version control

##  Terraform File Structure

```text


my-terraform-project/
├── main.tf          # Main resource definitions (EC2, S3, VPC)
├── variables.tf     # Input variables (region, instance type)
├── outputs.tf       # Outputs (IP addresses, resource IDs)
├── terraform.tfvars # Variable values (keep secrets out of code)
└── README.md        # Project documentation


```