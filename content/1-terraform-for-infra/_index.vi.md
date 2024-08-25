+++
title = "Sử dụng Terraform để triển khai Hạ tầng trên AWS"
date = 2020-05-14T00:38:32+07:00
weight = 1
chapter = false
pre = "<b>1. </b>"
+++


**Nội dung:**
- [Cấu trúc các file cấu hình Terraform](#cấu-trúc-các-file-cấu-hình-terraform)
- [Các files cấu hình](#các-files-cấu-hình)
- [Apply cấu hình lên AWS](#apply-cấu-hình-lên-aws)


#### Cấu trúc các file cấu hình Terraform

{{% notice note %}}
Vì trong bài LAB này mình sẽ chỉ tập trung vào triển khai Kubernetes Cluster một cách thủ công nên mình sẽ không giải thích quá kĩ về Terraform cũng như Ansible, các bạn muốn tìm hiểu về các công cụ này có thể tìm xem trên các trang như Medium, hoặc ở web Viblo VN mình thấy có rất nhiều anh hướng dẫn rất hay và chi tiết.
{{% /notice %}}

```plaintext
terraform/
│
├── 0-provider.tf         # File cấu hình dùng để định nghĩa Cloud Provider mà bạn sẽ sử dụng file Terraform để deploy
├── 1-vpc.tf              # Định nghĩa các cấu hình cho AWS VPC
├── 2-instances.tf        # Định nghĩa cấu hình cho các AWS EC2 Instances
├── 3-security-groups.tf  # Định nghĩa các security groups
├── 4-load-balanceres.tf  # Cấu hình Load Balancing cho các Master Nodeds
├── 5-vars.tf             # Định nghĩa các biến dùng trong các file .tf
├── 6-key.tf              # Cấu hình tạo SSH key để SSH vào Bastion Instance
├── 7-output.tf           # Cấu hình output sau khi apply
```
#### Các files cấu hình

File `0-provider.tf`:
- File này định nghĩa Cloud Provider (trong trường hợp này là AWS) mà Terraform sẽ tương tác với các tài nguyên của nhà cung cấp đó. Nó thiết lập nhà cung cấp cần thiết và chỉ định vùng (region) của AWS nơi các tài nguyên sẽ được triển khai.
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
- Đây là file Terraform dùng để triển khai VPC - Virtual Private Cloud, dịch sát nghĩa thì là một "mạng riêng ảo" trên môi trường AWS, bạn nào nếu đã xem kĩ các video trong module 1,2 có video anh Hưng đã nói rất kĩ về VPC.
- Với Terraform, các bạn có thể tự định nghĩa các thành phần của một VPC như network, các subnet, internet gateway, natgateway, security group,... nhưng trong bài LAB lần này mình sẽ sử dụng module có sẵn được public trên [registry của Terraform](https://registry.terraform.io/?product_intent=terraform) và mình chỉ cần thay đổi giá trị của các thuộc tính theo Document của module để tùy chỉnh VPC tùy ý.
  - Ví dụ như: 
    - **single_nat_gateway = true**: có nghĩa là tất cả các subnet sẽ dùng chung một NAT gateway, theo mặc định thì module này sẽ tự tạo **một NAT gateway cho mỗi subnet**.
    - **cidr**: Định nghĩa CIDR block cho VPC, tức là dải địa chỉ IP mà VPC có thể sử dụng. Ví dụ: "10.0.0.0/16" cho phép bạn sử dụng các địa chỉ IP từ 10.0.0.0 đến 10.0.255.255.
    - **azs**: Danh sách các vùng khả dụng (Availability Zones) mà bạn sẽ sử dụng để triển khai các subnet trong VPC (như **us-east-1a, us-east-1b, us-east-1c,...**). Nhằm đảm bảo khả năng dự phòng cũng như cân bằng tải cho các tài nguyên.
    - Các thuộc tính, tùy chọn khác các bạn có thể tham khảo trên Document của từng module tương ứng ở [registry của Terraform](https://registry.terraform.io/?product_intent=terraform)
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
Ở đây mọi người chú ý tạo các Instance có tối thiểu **1 CPU và 2GB RAM** nếu không lúc cài đặt K8s spray sẽ gặp lỗi không đủ tài nguyên 
{{% /notice %}}

- File này sẽ định nghĩa các cấu hình liên quan đến các **instances**, bao gồm **Bastion Instance**,các **Master Nodes** và các **Worker Nodes**
- Như:
  - **ami_id**            :là id của "image" mà các instance EC2 của bạn sẽ sử dụng, như Linux, Ubuntu, Centos,...
  - **instance_type**     :là cấu hình của các instance EC2 mà bạn muốn deploy, ví dụ như **t2.micro, t2.small, t3.medium, t3.large,...**
  - **subnet_id**         :cấu hình subnet ID nơi mà EC2 instance thuộc về
  - **security_groups**   :cấu hình các ID của security groups mà chúng ta cần gán vào instance
  - **key_name**          :tên của SSH để import vào instance
  - **user_data** :c      :cấu hình import SSH key vào instance

{{% notice note %}}
Trong bài LAB này, mình sử dụng Terraform để gen ra SSH key rồi import vào Bastion Instance, trên thực tế bạn cũng có thể tự tạo key trên AWS rồi gán vào instance trên AWS hoặc tự tạo key ở trên máy local sau đó import public key vào mục user data của Instance sau khi đã triển khai hạ tầng thành công với Terraform
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
- Phần này sẽ cấu hình các Security Groups cần cho VPC:
  - SG để cho phép SSH vào Bastion Instance
  - Các SG cần thiết cho Kubernets Cluster, master node, worker node
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

# target_type instance not working well when we bound this LB as a control-plane-endpoint. hence had to use IP target_type
#https://stackoverflow.com/questions/56768956/how-to-use-kubeadm-init-configuration-parameter-controlplaneendpoint/70799078#70799078

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
- Sử dụng Terraform để sinh ra SSH key dùng để SSH vào Bastion Instance
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

- Sau khi đã cấu hình các file Terraform, ta sử dụng câu lệnh **terraform init** và **terraform apply** để áp dụng các cấu hình tài nguyên lên hạ tầng AWS
![Image](/images/1-terraform/terraform-init.png)

- Kết quả apply thành công
![Image](/images/1-terraform/terraform-apply.png)

![Image](/images/1-terraform/aws-instances.png)

