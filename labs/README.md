# Deploying to AWS with Terraform and Ansible Labs

## Table of Contents

- [Deploying to AWS with Terraform and Ansible Labs](#deploying-to-aws-with-terraform-and-ansible-labs)
  - [Table of Contents](#table-of-contents)
  - [About](#about)
  - [Getting Started](#getting-started)
    - [Prerequisites](#prerequisites)
  - [Persisting Terraform State in S3 Back End](#persisting-terraform-state-in-s3-back-end)
  - [Setting Up Multiple AWS Providers in Terraform](#setting-up-multiple-aws-providers-in-terraform)

## About

Labs for [Deploying to AWS with Terraform and Ansible](https://acloud.guru/overview/8a6f598f-a41f-48ff-99a6-2c7a760b4119)

## Getting Started

### Prerequisites

- AWS CLI v2
- Terraform
- Ansible

``` bash
brew install awscli
brew install terraform
brew install ansible
```

## Persisting Terraform State in S3 Back End

- Create an S3 Bucket

``` BASH
aws s3api create-bucket --bucket sandbox-terraform-backend-542056542192
```

- Create `backend.tf`

``` python
# Set S3 backend for persisting TF state file remotely, ensure bucket already exits
# And that AWS user being used by TF has read/write perms
terraform {
  required_version = ">=0.12.0"
  required_providers {
    aws = ">=3.0.0"
  }
  backend "s3" {
    region = "us-east-1"
    # profile = "default"
    key    = "terraformstatefile"
    bucket = "sandbox-terraform-backend-542056542192" # replace with s3 bucket
  }
}
```

- Initialize the backend

``` BASH
terraform fmt
terraform validate
terraform init
```

## Setting Up Multiple AWS Providers in Terraform

- Create `variables.tf` to define the parameters used in `providers.tf`

``` Python
variable "profile" {
  type    = string
  default = "default"
}

variable "region-master" {
  type    = string
  default = "us-east-1"
}

variable "region-worker" {
  type    = string
  default = "us-west-2"
}
```

- Create `providers.tf` to create two AWS profiles with different regions and an alias

``` Python
# Defining multiple providers using "alias" parameter
provider "aws" {
  profile = var.profile
  region  = var.region-master
  alias   = "region-master"
}

provider "aws" {
  profile = var.profile
  region  = var.region-worker
  alias   = "region-worker"
}
```
