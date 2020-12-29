# Deploying to AWS with Terraform and Ansible Labs

## Table of Contents

- [Deploying to AWS with Terraform and Ansible Labs](#deploying-to-aws-with-terraform-and-ansible-labs)
  - [Table of Contents](#table-of-contents)
  - [About](#about)
  - [Getting Started](#getting-started)
    - [Prerequisites](#prerequisites)
  - [Persisting Terraform State in S3 Back End](#persisting-terraform-state-in-s3-back-end)
  - [Setting Up Multiple AWS Providers in Terraform](#setting-up-multiple-aws-providers-in-terraform)
  - [Network Setup - VPCs, Internet GWs, and Subnets](#network-setup---vpcs-internet-gws-and-subnets)
  - [Network Setup - Multi-Region VPC Peering](#network-setup---multi-region-vpc-peering)
  - [Network Setup - Security Groups](#network-setup---security-groups)
  - [Using Data Source (SSM Parameter Store) to Fetch AMI IDs](#using-data-source-ssm-parameter-store-to-fetch-ami-ids)
  - [Deploying Key Pairs for App Nodes](#deploying-key-pairs-for-app-nodes)
  - [Deploying Jenkins Master and Worker Instances](#deploying-jenkins-master-and-worker-instances)
  - [Configuring Terraform Provisioners for Config Management via Ansible](#configuring-terraform-provisioners-for-config-management-via-ansible)

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

## Network Setup - VPCs, Internet GWs, and Subnets

- Create `networks.tf` to declare the VPCs:

``` python
# Create VPC in us-east-1
resource "aws_vpc" "vpc_master" {
  provider             = aws.region-master
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "master-vpc-jenkins"
  }

}

# Create VPC in us-west-2
resource "aws_vpc" "vpc_master_oregon" {
  provider             = aws.region-worker
  cidr_block           = "192.168.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "worker-vpc-jenkins"
  }

}

# Create IGW in us-east-1
resource "aws_internet_gateway" "igw" {
  provider = aws.region-master
  vpc_id   = aws_vpc.vpc_master.id
}

# Create IGW in us-west-2
resource "aws_internet_gateway" "igw-oregon" {
  provider = aws.region-worker
  vpc_id   = aws_vpc.vpc_master_oregon.id
}
```

- Create a `data` resource to identify the availability zones

``` python
# Get all available AZ's in VPC for master region
data "aws_availability_zones" "azs" {
  provider = aws.region-master
  state    = "available"
}
```

- Use the `element()` function to assign one of the AZs to each subnet

``` python
# Create subnet # 1 in us-east-1
resource "aws_subnet" "subnet_1" {
  provider          = aws.region-master
  availability_zone = element(data.aws_availability_zones.azs.names, 0)
  vpc_id            = aws_vpc.vpc_master.id
  cidr_block        = "10.0.1.0/24"
}

# Create subnet #2  in us-east-1
resource "aws_subnet" "subnet_2" {
  provider          = aws.region-master
  vpc_id            = aws_vpc.vpc_master.id
  availability_zone = element(data.aws_availability_zones.azs.names, 1)
  cidr_block        = "10.0.2.0/24"
}
```

- Create a single `aws_subnet` in the oregon region

``` python
# Create subnet in us-west-2
resource "aws_subnet" "subnet_1_oregon" {
  provider   = aws.region-worker
  vpc_id     = aws_vpc.vpc_master_oregon.id
  cidr_block = "192.168.1.0/24"
}
```

- Format and Validate the configuration

``` Python
terraform fmt
terraform validate
```

- Plan and Apply the configuration

``` Python
terraform plan
terraform apply
```

## Network Setup - Multi-Region VPC Peering

- Initiate Peering connection request from us-east-1 in `networks.tf`

``` Python
# Initiate Peering connection request from us-east-1
resource "aws_vpc_peering_connection" "useast1-uswest2" {
  provider    = aws.region-master
  peer_vpc_id = aws_vpc.vpc_master_oregon.id
  vpc_id      = aws_vpc.vpc_master.id
  peer_region = var.region-worker

}
```

- Accept VPC peering request in us-west-2 from us-east-1 in `networks.tf`

``` Python
# Accept VPC peering request in us-west-2 from us-east-1
resource "aws_vpc_peering_connection_accepter" "accept_peering" {
  provider                  = aws.region-worker
  vpc_peering_connection_id = aws_vpc_peering_connection.useast1-uswest2.id
  auto_accept               = true
}
```

- Create route table to support traffic between the peered VPCs (master to oregon)

``` Python

# Create route table in us-east-1
resource "aws_route_table" "internet_route" {
  provider = aws.region-master
  vpc_id   = aws_vpc.vpc_master.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  route {
    cidr_block                = "192.168.1.0/24"
    vpc_peering_connection_id = aws_vpc_peering_connection.useast1-uswest2.id
  }
  lifecycle {
    ignore_changes = all
  }
  tags = {
    Name = "Master-Region-RT"
  }
}

# Overwrite default route table of VPC(Master) with our route table entries
resource "aws_main_route_table_association" "set-master-default-rt-assoc" {
  provider       = aws.region-master
  vpc_id         = aws_vpc.vpc_master.id
  route_table_id = aws_route_table.internet_route.id
}
```

- Create route table to support traffic between the peered VPCs (oregon to master)

``` Python
# Create route table in us-west-2
resource "aws_route_table" "internet_route_oregon" {
  provider = aws.region-worker
  vpc_id   = aws_vpc.vpc_master_oregon.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw-oregon.id
  }
  route {
    cidr_block                = "10.0.1.0/24"
    vpc_peering_connection_id = aws_vpc_peering_connection.useast1-uswest2.id
  }
  lifecycle {
    ignore_changes = all
  }
  tags = {
    Name = "Worker-Region-RT"
  }
}

# Overwrite default route table of VPC(Worker) with our route table entries
resource "aws_main_route_table_association" "set-worker-default-rt-assoc" {
  provider       = aws.region-worker
  vpc_id         = aws_vpc.vpc_master_oregon.id
  route_table_id = aws_route_table.internet_route_oregon.id
}

```

## Network Setup - Security Groups

- Create `security_groups.tf` to define the security groups
  - Internet --> ALB (us-east-1)
    - ALB Security Group
      - Inbound
        - TCP/80 --> 0.0.0.0/0
        - TCP/443 --> 0.0.0.0/0
      - Outbound
        - All TCP --> 0.0.0.0/0
  - ALB to Jenkins Master
    - Jenkins Master - Security Group
      - Inbound
        - ALB --> TCP/8080
        - TCP/22 --> Jenkins Worker
      - Outbound
        - All TCP --> 0.0.0.0/0
  - Jenkins Master to Jenkins Worker
    - Jenkins Worker Security Group
      - Inbound
        - ALB --> TCP/8080
        - TCP/22 --> Jenkins Master
      - Outbound
        - All TCP --> 0.0.0.0/0
- Create the `lb-sg` that allows TCP/80, TCP/443 and outbound access

``` Python
resource "aws_security_group" "lb-sg" {
  provider    = aws.region-master
  name        = "lb-sg"
  description = "Allow 443 and traffic to Jenkins SG"
  vpc_id      = aws_vpc.vpc_master.id
  ingress {
    description = "Allow 443 from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "Allow 80 from anywhere for redirection"
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

- Create the `jenkins-sg` that allowing TCP/8080 from * and TCP/22 from your IP in us-east-1

``` Python
resource "aws_security_group" "jenkins-sg" {
  provider    = aws.region-master
  name        = "jenkins-sg"
  description = "Allow TCP/8080 & TCP/22"
  vpc_id      = aws_vpc.vpc_master.id
  ingress {
    description = "Allow 22 from our public IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.external_ip]
  }
  ingress {
    description     = "allow traffic on port 8080 from lb-sg"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.lb-sg.id]
  }
  ingress {
    description = "allow traffic from us-west-2 CIDR"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["192.168.1.0/24"]
  }
  egress {
    description = "Allow all outbound traffic to the internet"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

- Create the `jenkins-sg-oregon` to allow traffic from us-east-1 CIDR

``` Python
resource "aws_security_group" "jenkins-sg-oregon" {
  provider = aws.region-worker

  name        = "jenkins-sg-oregon"
  description = "Allow TCP/8080 & TCP/22"
  vpc_id      = aws_vpc.vpc_master_oregon.id
  ingress {
    description = "Allow 22 from our public IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.external_ip]
  }
  ingress {
    description = "Allow traffic from us-east-1"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["10.0.1.0/24"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

- Create the new `external_ip` to the `variables.tf`

``` Python
variable "external_ip" {
  type    = string
  default = "0.0.0.0/0"
}
```

- Format and Validate the configuration

``` BASH
terraform fmt
terraform validate
```

- Plan and Apply the changes

``` BASH
terraform plan
terraform apply --auto-approve
```

## Using Data Source (SSM Parameter Store) to Fetch AMI IDs

- Create the `instances.tf` file to define the `data` source for the AMI ID

``` Python
# Get Linux AMI ID using SSM Parameter endpoint in us-east-1
data "aws_ssm_parameter" "linuxAmi" {
  provider = aws.region-master
  name     = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
}

# Get Linux AMI ID using SSM Parameter endpoint in us-west-2
data "aws_ssm_parameter" "linuxAmiOregon" {
  provider = aws.region-worker
  name     = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
}
```

- Format and Validate

``` bash
terraform fmt
terraform validate
```

- Plan and Apply

``` bash
terraform plan
terraform apply
```

- Validate AMI ID in remote state

``` bash
aws s3 cp s3://sandbox-terraform-backend-542056542192/terraformstatefile .
cat terraformstatefile | grep ami
```

## Deploying Key Pairs for App Nodes

- Create key pairs for logging into the instances

``` BASH
ssh-keygen -t rsa
```

- Create key-pair resources to reference the generated key

``` python
# Create key-pair for logging into EC2 in us-east-1
resource "aws_key_pair" "master-key" {
  provider   = aws.region-master
  key_name   = "jenkins"
  public_key = file("~/.ssh/id_rsa.pub")
}

# Create key-pair for logging into EC2 in us-west-2
resource "aws_key_pair" "worker-key" {
  provider   = aws.region-worker
  key_name   = "jenkins"
  public_key = file("~/.ssh/id_rsa.pub")
}
```

## Deploying Jenkins Master and Worker Instances

- Add jenkins master and worker instances to `instances.tf`

``` Python
# Create and bootstrap EC2 in us-east-1
resource "aws_instance" "jenkins-master" {
  provider                    = aws.region-master
  ami                         = data.aws_ssm_parameter.linuxAmi.value
  instance_type               = var.instance-type
  key_name                    = aws_key_pair.master-key.key_name
  associate_public_ip_address = true
  vpc_security_group_ids      = [aws_security_group.jenkins-sg.id]
  subnet_id                   = aws_subnet.subnet_1.id

  tags = {
    Name = "jenkins_master_tf"
  }

  depends_on = [aws_main_route_table_association.set-master-default-rt-assoc]
}

# Create EC2 in us-west-2
resource "aws_instance" "jenkins-worker-oregon" {
  provider                    = aws.region-worker
  count                       = var.workers-count
  ami                         = data.aws_ssm_parameter.linuxAmiOregon.value
  instance_type               = var.instance-type
  key_name                    = aws_key_pair.worker-key.key_name
  associate_public_ip_address = true
  vpc_security_group_ids      = [aws_security_group.jenkins-sg-oregon.id]
  subnet_id                   = aws_subnet.subnet_1_oregon.id

  tags = {
    Name = join("_", ["jenkins_worker_tf", count.index + 1])
  }
  depends_on = [aws_main_route_table_association.set-worker-default-rt-assoc, aws_instance.jenkins-master]
}
```

- Add input variables to `variables.tf`

``` Python
variable "workers-count" {
  type    = number
  default = 1
}

variable "instance-type" {
  type    = string
  default = "t3.micro"
}
```

- Create `outputs.tf` to export IP Addresses for Jenkins Master and Workers

``` Python
output "Jenkins-Main-Node-Public-IP" {
  value = aws_instance.jenkins-master.public_ip
}

output "Jenkins-Worker-Public-IPs" {
  value = {
    for instance in aws_instance.jenkins-worker-oregon :
    instance.id => instance.public_ip
  }
}
```

- Format and Validate

``` BASH
terraform fmt
terraform validate
```

- Plan and Apply

``` BASH
terraform plan
terraform apply
```

- Validate SSH and KeyPair

``` BASH
ssh ec2-user@{Jenkins-Main-Node-Public-IP}
ssh ec2-user@{Jenkins-Worker-Public-IPs}
```

- Destroy

``` BASH
terraform destroy
```

## Configuring Terraform Provisioners for Config Management via Ansible

- Bootstrapping Infrastructure via Terraform Providers
  - Create and Destroy time provisioners
  - If a provisioner fails to run for a resource, it is marked as tainted
  - Provisioners can run commands locally or remotely (using SSH or WinRM)
    - Local provisioners run on the same system where Terraform commands are invoked

- Configure `ansible.cfg`

``` text
inventory       = ./ansible_templates/inventory_aws/tf_aws_ec2.yml
enable_plugins = aws_ec2
interpreter_python = auto_silent
```

- jenkins-master-sample.yml

``` yaml
# Ansible Jenkins Master, sample playbook - jenkins-master-sample.yml
---

- hosts: "{{ passed_in_hosts }}"

  become: yes
  remote_user: ec2-user
  become_user: root
  tasks:

    - name: install Git client

      yum:
        name: git
        state: present

```

- jenkins-worker-sample.yml

``` yaml
# Ansible Jenkins Worker, sample playbook - jenkins-worker-sample.yml
---

- hosts: "{{ passed_in_hosts }}"

  become: yes
  remote_user: ec2-user
  become_user: root
  tasks:

    - name: install jq, JSON parser

      yum:
        name: jq
        state: present
```

- Jenkins Master - Adding terraform provisioners to `instances.tf`

``` python
  # The code below is ONLY the provisioner block which needs to be
  # inserted inside the resource block for Jenkins EC2 master Terraform
  # Jenkins Master Provisioner:
  provisioner "local-exec" {
    command = <<EOF
aws --profile ${var.profile} ec2 wait instance-status-ok --region ${var.region-master} --instance-ids ${self.id}
ansible-playbook --extra-vars 'passed_in_hosts=tag_Name_${self.tags.Name}' ansible_templates/jenkins-master-sample.yml
EOF
  }
```

- Jenkins Worker - Adding terraform provisioners to `instances.tf`

``` python
  # The code below is ONLY the provisioner block which needs to be
  # inserted inside the resource block for Jenkins EC2 worker in Terraform
  provisioner "local-exec" {
    command = <<EOF
aws --profile ${var.profile} ec2 wait instance-status-ok --region ${var.region-worker} --instance-ids ${self.id}
ansible-playbook --extra-vars 'passed_in_hosts=tag_Name_${self.tags.Name}' ansible_templates/jenkins-worker-sample.yml
EOF
  }
```
