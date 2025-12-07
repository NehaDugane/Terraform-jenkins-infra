# terraform-jenkins-infra-repo

This repo contains a complete, tested Terraform + Jenkins pipeline setup that creates a single VPC and 4 EC2 machines (Tomcat, Jenkins, Nexus, SonarQube) using modular Terraform structure (VPC module + EC2 module). It also includes a Jenkinsfile which runs Terraform from Jenkins to provision the infrastructure.

---

## Repo layout

```
terraform-jenkins-infra/
├── README.md                # this file
├── backend.tf               # terraform backend configuration (S3 + DynamoDB)
├── main.tf                  # root terraform that calls modules
├── variables.tf             # root variables
├── outputs.tf               # root outputs
├── terraform.tfvars.example # example variables
├── userdata/
│   ├── tomcat.sh
│   ├── jenkins.sh
│   ├── nexus.sh
│   └── sonarqube.sh
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── ec2/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── Jenkinsfile
```

---

> **Important:** The `backend.tf` contains a placeholder for your S3 bucket & DynamoDB table. Create those first (or remove backend block to use local state for testing).

---

## backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "s3-bucket-071225"
    key            = "dev/infrastructure.tfstate"
    region         = "ap-south-1"
    encrypt        = true
  }
}
```

---

## main.tf (root)

```hcl
provider "aws" {
  region = var.region
}

module "vpc" {
  source              = "./modules/vpc"
  vpc_cidr            = var.vpc_cidr
  public_subnet_cidrs = var.public_subnet_cidrs
  name                = var.vpc_name
}

resource "aws_security_group" "dev_sg" {
  name   = "${var.vpc_name}-sg"
  vpc_id = module.vpc.vpc_id

  description = "Allow SSH and app ports"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.allowed_ssh_cidr]
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = [var.allowed_http_cidr]
  }

  ingress {
    from_port   = 8081
    to_port     = 8081
    protocol    = "tcp"
    cidr_blocks = [var.allowed_http_cidr]
  }

  ingress {
    from_port   = 9000
    to_port     = 9000
    protocol    = "tcp"
    cidr_blocks = [var.allowed_http_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.vpc_name}-sg" }
}

# AMI data
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# optional: create or reference keypair
resource "aws_key_pair" "default" {
  key_name   = "02-december-2025"
  public_key = file(var.public_key_path)
}

module "tomcat" {
  source            = "./modules/ec2"
  ami               = data.aws_ami.amazon_linux_2.id
  instance_type     = var.instance_type
  subnet_id         = module.vpc.public_subnet_ids[0]
  key_name          = aws_key_pair.default.key_name
  security_group_ids = [aws_security_group.dev_sg.id]
  user_data         = file("${path.module}/userdata/tomcat.sh")
  tags = { Name = "tomcat-server" }
}

module "jenkins" {
  source            = "./modules/ec2"
  ami               = data.aws_ami.amazon_linux_2.id
  instance_type     = var.instance_type
n  subnet_id         = module.vpc.public_subnet_ids[1]
  key_name          = aws_key_pair.default.key_name
  security_group_ids = [aws_security_group.dev_sg.id]
  user_data         = file("${path.module}/userdata/jenkins.sh")
  tags = { Name = "jenkins-server" }
}

module "nexus" {
  source            = "./modules/ec2"
  ami               = data.aws_ami.amazon_linux_2.id
  instance_type     = var.instance_type
  subnet_id         = module.vpc.public_subnet_ids[2]
  key_name          = aws_key_pair.default.key_name
  security_group_ids = [aws_security_group.dev_sg.id]
  user_data         = file("${path.module}/userdata/nexus.sh")
  tags = { Name = "nexus-server" }
}

module "sonarqube" {
  source            = "./modules/ec2"
  ami               = data.aws_ami.amazon_linux_2.id
  instance_type     = var.instance_type
  subnet_id         = module.vpc.public_subnet_ids[3]
  key_name          = aws_key_pair.default.key_name
  security_group_ids = [aws_security_group.dev_sg.id]
  user_data         = file("${path.module}/userdata/sonarqube.sh")
  tags = { Name = "sonarqube-server" }
}
```

---

## variables.tf (root)

```hcl
variable "region" { default = "ap-south-1" }
variable "vpc_name" { default = "devops-vpc" }
variable "vpc_cidr" { default = "10.0.0.0/16" }
variable "public_subnet_cidrs" { default = ["10.0.1.0/24","10.0.2.0/24","10.0.3.0/24","10.0.4.0/24"] }
variable "instance_type" { default = "t3.medium" }
variable "key_name" { default = "02-december-2025" }
variable "public_key_path" { default = "C:\Users\Neha\Desktop\.ssh/id_rsa.pub" }
variable "allowed_ssh_cidr" { default = "0.0.0.0/0" }
variable "allowed_http_cidr" { default = "0.0.0.0/0" }
```

---

## outputs.tf (root)

```hcl
output "tomcat_ip" { value = module.tomcat.public_ip }
output "jenkins_ip" { value = module.jenkins.public_ip }
output "nexus_ip" { value = module.nexus.public_ip }
output "sonarqube_ip" { value = module.sonarqube.public_ip }
```

---

## modules/vpc/main.tf

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags = { Name = var.name }
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags = { Name = "${var.name}-igw" }
}

resource "aws_subnet" "public" {
  for_each = toset(var.public_subnet_cidrs)
  vpc_id = aws_vpc.this.id
  cidr_block = each.value
  map_public_ip_on_launch = true
  tags = { Name = "${var.name}-public-${replace(each.value,"/24","")}" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  route { cidr_block = "0.0.0.0/0" gateway_id = aws_internet_gateway.this.id }
  tags = { Name = "${var.name}-public-rt" }
}

resource "aws_route_table_association" "public_assoc" {
  for_each = aws_subnet.public
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}
```

---

## modules/vpc/variables.tf

```hcl
variable "vpc_cidr" { type = string }
variable "public_subnet_cidrs" { type = list(string) }
variable "name" { type = string }
```

---

## modules/vpc/outputs.tf

```hcl
output "vpc_id" { value = aws_vpc.this.id }
output "public_subnet_ids" { value = [for s in aws_subnet.public : s.id] }
```

---

## modules/ec2/main.tf

```hcl
resource "aws_instance" "this" {
  ami                    = var.ami
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  key_name               = var.key_name
  vpc_security_group_ids = var.security_group_ids
  user_data              = var.user_data
  associate_public_ip_address = true

  tags = merge({ Name = lookup(var.tags, "Name", "ec2-instance") }, var.tags)
}
```

---

## modules/ec2/variables.tf

```hcl
variable "ami" { type = string }
variable "instance_type" { default = "t3.medium" }
variable "subnet_id" { type = string }
variable "key_name" { type = string }
variable "security_group_ids" { type = list(string) }
variable "tags" { type = map(string) }
variable "user_data" { type = string }
```

---

## modules/ec2/outputs.tf

```hcl
output "instance_id" { value = aws_instance.this.id }
output "public_ip" { value = aws_instance.this.public_ip }
output "private_ip" { value = aws_instance.this.private_ip }
```

---

## userdata/tomcat.sh

```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y java-1.8.0-openjdk wget
cd /tmp
wget https://downloads.apache.org/tomcat/tomcat-9/v9.0.73/bin/apache-tomcat-9.0.73.tar.gz
sudo mkdir -p /opt/tomcat
sudo tar -xzf apache-tomcat-9.0.73.tar.gz -C /opt/tomcat --strip-components=1
sudo chmod +x /opt/tomcat/bin/*.sh
sudo /opt/tomcat/bin/startup.sh
```

---

## userdata/jenkins.sh

```bash
#!/bin/bash
sudo yum update -y
sudo amazon-linux-extras install java-openjdk11 -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install -y jenkins
sudo systemctl enable --now jenkins
```

---

## userdata/nexus.sh

```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y java-1.8.0-openjdk wget
cd /tmp
wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
sudo tar -xzf latest-unix.tar.gz -C /opt
sudo mv /opt/nexus-* /opt/nexus
sudo useradd nexus || true
sudo chown -R nexus:nexus /opt/nexus /opt/sonatype-work || true
echo 'run_as_user="nexus"' | sudo tee /opt/nexus/bin/nexus.rc
sudo /opt/nexus/bin/nexus start
```

---

## userdata/sonarqube.sh

```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y java-1.8.0-openjdk unzip wget
cd /tmp
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.8.zip
unzip sonarqube-7.8.zip
sudo mv sonarqube-7.8 /opt/sonarqube
sudo useradd sonar || true
sudo chown -R sonar:sonar /opt/sonarqube
sudo /opt/sonarqube/bin/linux-x86-64/sonar.sh start
```

---


## terraform.tfvars.example

```hcl
region = "ap-south-1"
vpc_name = "devops-vpc"
key_name = "02-december-2025"
public_key_path = "C:\Users\Neha\Desktop\.ssh/id_rsa.pub"
allowed_ssh_cidr = "0.0.0.0/32"
allowed_http_cidr = "0.0.0.0/0"
```

---

## Usage / Quick start

1. Create an S3 bucket and DynamoDB table for terraform state (if using backend). Update `backend.tf` with the bucket & table names.
2. Commit this repo to GitHub.
3. Create Jenkins credentials (AWS access key & secret) with ID `aws-creds`.
4. Configure a Multibranch Pipeline or Pipeline job in Jenkins pointing at this repo (Jenkinsfile).
5. Run the Jenkins job and approve the Apply stage.
6. After successful apply, use `terraform output` to get public IPs for each instance.

---

## Security notes

- Replace wide `0.0.0.0/0` SSH with your IP (allowed_ssh_cidr).
- Use IAM least privilege for the AWS credentials.

---

## License

MIT

