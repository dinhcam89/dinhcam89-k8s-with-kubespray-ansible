+++
title = "Using Terraform to Deploy Infrastructure on AWS"
date = 2020-05-14T00:38:32+07:00
weight = 1
chapter = false
pre = "<b>1. </b>"
+++

**Contents:**
- [Structure of Terraform Configuration Files](#structure-of-terraform-configuration-files)
- [Configuration files](#configuration-files)
- [Apply cấu hình lên AWS](#apply-cấu-hình-lên-aws)

#### Structure of Terraform Configuration Files

{{% notice note %}}
Since this LAB focuses only on manually deploying a Kubernetes Cluster, I won't go into detail about Terraform or Ansible. If you want to learn more about these tools, you can find many excellent and detailed guides on sites like Medium or Viblo VN.
{{% /notice %}}

```plaintext
terraform/
│
├── 0-provider.tf         # Configuration file to define the Cloud Provider you will use Terraform to deploy
├── 1-vpc.tf              # Defines the configuration for AWS VPC
├── 2-instances.tf        # Defines the configuration for AWS EC2 Instances
├── 3-security-groups.tf  # Defines security groups
├── 4-load-balancers.tf  # Configures Load Balancing for Master Nodes
├── 5-vars.tf             # Defines variables used in .tf files
├── 6-key.tf              # Configures the creation of SSH key for SSH access to Bastion Instance
├── 7-output.tf           # Configuration for output after apply
```
#### Configuration files

File `0-provider.tf`:
- This file defines the Cloud Provider (in this case AWS) with which Terraform will interact with its resources. It sets up the required provider and specifies the region (region) of AWS where the resources will be exploited.
```terraform
provider "aws" {
  region = var.region
}
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

File `1-vpc.tf`:
- This is a Terraform file used to deploy a VPC, which stands for "Virtual Private Cloud" on the AWS environment.
- With Terraform, you can define the components of a VPC such as network, subnets, internet gateway, NAT gateway, security groups, etc. However, in this LAB, I will use a publicly available module from the [Terraform Registry](https://registry.terraform.io/?product_intent=terraform) and only need to adjust the attribute values according to the module's documentation to customize the VPC as needed.
  - For example: 
    - **single_nat_gateway = true**: means that all subnets will use a single NAT gateway; by default, this module will create one NAT gateway per subnet.
    - **cidr**: Defines the CIDR block for the VPC, which is the IP address range that the VPC can use. For example: "10.0.0.0/16" allows you to use IP addresses from 10.0.0.0 to 10.0.255.255.
    - **azs**: List of Availability Zones where you will deploy subnets in the VPC (such as us-east-1a, us-east-1b, us-east-1c,...). This ensures high availability and load balancing for the resources.
    - Other attributes and options can be found in the documentation of each module on the [Terraform Registry](https://registry.terraform.io/?product_intent=terraform)
```terraform
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "self-managed-k8s-vpc"
  cidr = "10.0.0.0/16"

  azs             = var.availability_zones
  private_subnets = var.private_subnets
  public_subnets  = var.public_subnets

  enable_nat_gateway = true
  single_nat_gateway = true
  one_nat_gateway_per_az = false

  map_public_ip_on_launch = true    //assign public ip for BASTION instance

  tags = {
    Terraform = "true"
    Environment = "dev"
  }
}
```
File `2-instances.tf`:

{{% notice warning  %}}
Make sure to create instances with at least **1 CPU và 2GB RAM** otherwise, you may encounter resource errors during the K8s installation process.
{{% /notice %}}

- This file will define the configurations related to **instances**, including the **Bastion Instance**, **Master Nodes** and **Worker Nodes**
- For example:
  - **ami_id**            :lThe ID of the "image" that your EC2 instances will use, such as Linux, Ubuntu, CentOS, etc.
  - **instance_type**     :The configuration of the EC2 instances you want to deploy, such as **t2.micro, t2.small, t3.medium, t3.large,...**
  - **subnet_id**         :The subnet ID where the EC2 instance will be located.
  - **security_groups**   :The IDs of the security groups that need to be assigned to the instance.
  - **key_name**          :The name of the SSH key to be imported into the instance.
  - **user_data** :c      :Configuration to import the SSH key into the instance.

{{% notice note %}}
In this LAB, I use Terraform to generate an SSH key and import it into the Bastion Instance. In practice, you can also create a key on AWS and assign it to the instance or create a key on your local machine and then import the public key into the instance's user data after successfully deploying the infrastructure with Terraform.
{{% /notice %}}
```terraform
// Bastion Instance
resource "aws_instance" "bastion" {
  ami                         = var.ami_id
  instance_type               = var.bastion_instance_type
  subnet_id                   = module.vpc.public_subnets[0]
  associate_public_ip_address = "true"
  security_groups             = [aws_security_group.allow_ssh.id]
  key_name                    =   aws_key_pair.k8_ssh.key_name
  user_data = <<-EOF
                #!bin/bash
                echo "PubkeyAcceptedKeyTypes=+ssh-rsa" >> /etc/ssh/sshd_config.d/10-insecure-rsa-keysig.conf
                systemctl reload sshd
                echo "${tls_private_key.ssh.private_key_pem}" >> /home/ubuntu/.ssh/id_rsa
                chown ubuntu /home/ubuntu/.ssh/id_rsa
                chgrp ubuntu /home/ubuntu/.ssh/id_rsa
                chmod 600   /home/ubuntu/.ssh/id_rsa
                echo "starting ansible install"
                apt-add-repository ppa:ansible/ansible -y
                apt update
                apt install ansible -y
                EOF

  tags = {
    Name = "Bastion"
  }
}

//  Master Nodes
resource "aws_instance" "masters" {
  count           = var.master_node_count
  ami             = var.ami_id
  instance_type   = var.master_instance_type
  subnet_id       = "${element(module.vpc.private_subnets, count.index)}"
  key_name        = aws_key_pair.k8_ssh.key_name
  security_groups = [aws_security_group.k8_nondes.id, aws_security_group.k8_masters.id]

  tags = {
    Name = format("k8s-master-%02d", count.index + 1)
  }
}

//  Worker Nodes
resource "aws_instance" "workers" {
  count           = var.worker_node_count
  ami             = var.ami_id
  instance_type   = var.worker_instance_type
  subnet_id       = "${element(module.vpc.private_subnets, count.index)}"
  key_name        = aws_key_pair.k8_ssh.key_name
  security_groups = [aws_security_group.k8_nondes.id, aws_security_group.k8_workers.id]

  tags = {
    Name = format("k8s-worker-%02d", count.index + 1)
  }
}
```

File `3-security_groups.tf`:
- This section will configure the Security Groups needed for the VPC:
  - SG to allow SSH access to the Bastion Instance
  - SGs required for the Kubernetes Cluster, master nodes, and worker nodes
```terraform
resource "aws_security_group" "allow_ssh" {
    name        = "allow_ssh"
    description = "Allow ssh inbound traffic"
    vpc_id    = module.vpc.vpc_id

    ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port   = -1
        to_port     = -1
        protocol    = "icmp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
}

resource "aws_security_group" "k8_nondes" {
    name = "k8_nodes"
    description = "sec group for k8 nodes"
    vpc_id = module.vpc.vpc_id

    ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["${var.vpc_cidr}"]
    }

    ingress {
        from_port   = -1
        to_port     = -1
        protocol    = "icmp"
        cidr_blocks = ["${var.vpc_cidr}"]
    }

    egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
  
}

resource "aws_security_group" "k8_masters" {
    name = "k8_masters"
    description = "sec group for k8 master nodes"
    vpc_id = module.vpc.vpc_id

    ingress {
        #Kubernetes API server
        from_port   = 6443
        to_port     = 6443
        protocol    = "tcp"
        cidr_blocks = ["${var.vpc_cidr}"]
    }

    ingress {
        #etcd server client API
        from_port   = 2379
        to_port     = 2380
        protocol    = "tcp"
        cidr_blocks = ["${var.vpc_cidr}"]
    }

    ingress {
        #Kubelet API
        from_port   = 10250
        to_port     = 10250
        protocol    = "tcp"
        cidr_blocks = ["${var.vpc_cidr}"]
    }

    ingress {
        #kube-scheduler
        from_port   = 10259
        to_port     = 10259
        protocol    = "tcp"
        cidr_blocks = ["${var.vpc_cidr}"]
    }

    ingress {
        #kube-controller-manager
        from_port   = 10257
        to_port     = 10257
        protocol    = "tcp"
        cidr_blocks = ["${var.vpc_cidr}"]
    }
  
}

resource "aws_security_group" "k8_workers" {
    name = "k8_workers"
    description = "sec group for k8 worker nodes"
    vpc_id = module.vpc.vpc_id

    ingress {
        #Kubelet API
        from_port   = 10250
        to_port     = 10250
        protocol    = "tcp"
        cidr_blocks = ["${var.vpc_cidr}"]
    }

    ingress {
        #NodePort Services†
        from_port   = 30000
        to_port     = 32767
        protocol    = "tcp"
        cidr_blocks = ["${var.vpc_cidr}"]
    }
}
```

File `4-load-balancers.tf`:
```terraform

resource "aws_lb" "k8_masters_lb" {
    name        = "k8-masters-lb"
    internal    = true
    load_balancer_type = "network"
    subnets = module.vpc.private_subnets #[for subnet in module.vpc.private_subnets : subnet.id]
    tags = {
    Terraform = "true"
    Environment = "dev"
  }
  
}

resource "aws_lb_target_group" "k8_masters_api" {
    name = "k8-masters-api"
    port = 6443
    protocol = "TCP"
    vpc_id = module.vpc.vpc_id
    target_type = "ip"

    health_check {
      port      = 6443
      protocol  = "TCP"
      interval  = 30
      healthy_threshold = 2
      unhealthy_threshold = 2
    }
}

resource "aws_lb_listener" "k8_masters_lb_listener" {
    load_balancer_arn = aws_lb.k8_masters_lb.arn
    port = 6443
    protocol = "TCP"

    default_action {
        target_group_arn = aws_lb_target_group.k8_masters_api.id
        type = "forward"
    }
}

resource "aws_lb_target_group_attachment" "k8_masters_attachment" {
    count = length(aws_instance.masters.*.id)
    target_group_arn = aws_lb_target_group.k8_masters_api.arn
    target_id = aws_instance.masters.*.private_ip[count.index]
}
```

File `5-vars.tf`:
```terraform

variable "region" {
   default = "us-east-1"
}

variable "ami_id" {
    type = string
    default = "ami-04a81a99f5ec58529"   // Ubuntu 24.04 LTS
}

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "vpc_cidr" {
    type = string
    default = "10.0.0.0/16"
}

variable "private_subnets" {
    type = list(string)
    default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

variable "public_subnets" {
    type = list(string)
    default = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
}

variable "master_node_count" {
    type = number
    default = 3
}

variable "worker_node_count" {
    type = number
    default = 3
}

variable "ssh_user" {
    type = string
    default = "ubuntu"
}

variable "bastion_instance_type" {
    type = string
    default = "t3.micro"
}

variable "master_instance_type" {
    type = string
    default = "t3.small"
}

variable "worker_instance_type" {
    type = string
    default = "t3.small"
}
```

File `6-key.tf`:
- Use Terraform to generate an SSH key for SSH access to the Bastion Instance.
```terraform
resource "tls_private_key" "ssh" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "local_file" "k8_ssh_key" {
    filename                = "k8_ssh_key.pem"
    file_permission         = "600"
    directory_permission    = "700"
    content                 = tls_private_key.ssh.private_key_pem
}

resource "aws_key_pair" "k8_ssh" {
  key_name   = "k8_ssh"
  public_key = tls_private_key.ssh.public_key_openssh
}
```

File `7-outputs.tf`:
```terraform
resource "tls_private_key" "ssh" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "local_file" "k8_ssh_key" {
    filename                = "k8_ssh_key.pem"
    file_permission         = "600"
    directory_permission    = "700"
    content                 = tls_private_key.ssh.private_key_pem
}

resource "aws_key_pair" "k8_ssh" {
  key_name   = "k8_ssh"
  public_key = tls_private_key.ssh.public_key_openssh
}
```

#### Apply cấu hình lên AWS

- After configuring the Terraform files, use the commands **terraform init** and **terraform apply** to apply the resource configurations to the AWS infrastructure.
![Image](./images/1-terraform/terraform-init.png)

- Successful apply results

![Image](./images/1-terraform/terraform-apply.png)

![Image](./images/1-terraform/aws-instances.png)

