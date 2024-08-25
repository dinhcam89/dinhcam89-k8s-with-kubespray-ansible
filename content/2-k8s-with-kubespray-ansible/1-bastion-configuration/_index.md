+++
title = "Configuring the Bastion Instance"
date = 2021
weight = 1
chapter = false
pre = "<b>2.1. </b>"
+++

{{% notice note %}}
In this step, we will configure the Bastion Instance to prepare for deploying the Kubernetes Cluster. All configurations related to Master Nodes and Worker Nodes will be done through the Bastion Instance.
{{% /notice %}}

#### Configuring the Instances

- After applying the Terraform configuration files, an SSH key file ".pem" will be created in the directory containing the Terraform files. This key is the one we configured in the ***key.tf*** file for Terraform to create, and it will be used to SSH into the Bastion Instance.
- SSH into the Bastion Instance for configuration.
![SSH Bastion](/images/2.1-bastion/ssh-bastion.png)
- Configure the hostname on the Bastion machine to simplify SSH operations and make it easier to remember, rather than using the IP addresses of the Instances.
- Add the private IP addresses and names of the nodes to the /etc/hosts file.

1. Configure hostname
```bash
127.0.0.1 localhost

10.0.1.114 k8s-master1
10.0.2.151 k8s-master2
10.0.3.26 k8s-master3

10.0.1.98 k8s-worker1
10.0.2.143 k8s-worker2
10.0.3.245 k8s-worker3

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```

2. Necessary configurations for installing the Kubernetes Cluster on EC2 Instances
- After SSH access to the Bastion Instance, SSH into the Master and Worker Nodes to update packages and configure a few things needed before installing the Kubernetes Cluster.
- To install the Kubernetes Cluster on the Instances, we need to disable swap on the Instances.
- Why does Kubernetes require ***swap*** to be disabled? Kubernetes relies on predictable and efficient resource management on nodes. Swap can cause undesirable delays when the operating system decides to move data between RAM and disk. This can lead to inconsistent pod performance, reducing Kubernetes' ability to manage and coordinate resources effectively.

```bash
swapoff -a

sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

#### Installing Kubespray and Ansible
- After successfully configuring the Bastion Instance, we move on to installing the K8s Cluster.
- As mentioned earlier, all installations will be done on the Bastion Instance, so we will install Kubespray and Ansible on the Bastion Instance.

```bash
sudo apt install git python3 python3-pip -y

sudo apt install ansible-core -y
```

- After successfully installing Ansible, clone the [source code của Kubespray](https://github.com/kubernetes-sigs/kubespray.git) from GitHub to proceed with the K8s Cluster installation.
{{% notice note %}}
To install Kubernetes v1.28, you need to select the correct version 2.24 of Kubespray. For information on the Kubernetes versions corresponding to Kubespray versions, refer to the documentation on the [Github của Kubespray](https://github.com/kubernetes-sigs/kubespray)
{{% /notice %}}

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git --branch release-2.24
```