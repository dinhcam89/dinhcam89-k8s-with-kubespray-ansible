+++
title = "Deploying a Kubernetes Cluster on AWS with Terraform and Ansible"
date = 2024
weight = 1
chapter = false
+++

# Deploying a HA Kubernetes Cluster on AWS with Terraform and Ansible

#### Overview
_First of all, let me say hello to all of you. This is my first workshop, so there might be some mistakes, and I hope you can overlook them. I hope you will find useful and interesting knowledge in this workshop. Thank you._

In this lab, we will manually deploy a Kubernetes cluster on AWS using **kubespray** and EC2 instances.

![architecture](./images/1-terraform/architecture2.png)

{{% notice note %}}
To perform this LAB, you need an AWS account to create EC2 instances and the necessary resources to deploy the Kubernetes Cluster. After completing the LAB, you can delete these resources on AWS to avoid additional costs.
{{% /notice %}}

#### Using Terraform to Deploy VPC and EC2 Instances on AWS
**Terraform** is an open-source tool developed by HashiCorp, used to build, change, and manage infrastructure on the cloud safely and efficiently through "infrastructure as code" (IaC).

#### Using Kubespray and Ansible to Deploy a Kubernetes Cluster
- **Kubespray**
  - ***Kubespray*** is an open-source project by Kubernetes SIGs (Special Interest Groups) used to deploy Kubernetes clusters manually, easily, flexibly, and customizable on various environments. It uses Ansible, a configuration automation tool, to install and manage Kubernetes clusters. Although there are many cloud providers offering Kubernetes as a Service, like AWS's EKS (Elastic Kubernetes Service) or Google's GKE (Google Kubernetes Engine), using these services does not give you full control over the control plane, the "brain" of a Kubernetes Cluster. Depending on your usage and management needs, you may choose to use these ready-made Kubernetes services or deploy and manage your own Kubernetes Cluster.
- **Ansible**
  - ***Ansible*** is an open-source tool used to automate system administration tasks, including system configuration, software deployment, and infrastructure management. Developed by Red Hat, Ansible is known for its simplicity, ease of use, and does not require special software installation on target machines.

#### Main Contents

1. [Using Terraform to Deploy Infrastructure on AWS](1-terraform-for-infra/)
2. [Using Kubespray and Ansible to Deploy a Kubernetes Cluster](2-k8s-with-kubespray-ansible/)
3. [Cleaning UP Resource](3-cleanup/)