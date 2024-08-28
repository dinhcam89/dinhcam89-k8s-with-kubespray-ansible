+++
title = "Triển khai Kubernetes Cluster trên AWS với Terraform và Ansible"
date = 2024
weight = 1
chapter = false
+++

# Triển khai HA Kubernetes Cluster trên AWS với Terraform và Ansible

#### Tổng quan
_Lời đầu tiên, cho phép em/mình xin chào tất cả các anh/chị, các bạn. Đây là bài workshop đầu tiên của em cho nên có thể còn nhiều sai sót mong mọi người bỏ qua, rất mong mọi người có thể tìm thấy những kiến thức có ích, thú vị trong workshop lần này ạ. Em xin cảm ơn._
 
Trong bài lab này, mình sẽ thực hiện triển khai một Kubernetes cluster trên môi trường AWS một cách thủ công với **kubespray** và các máy chủ EC2

{{< figure src="https://github.com/dinhcam89/dinhcam89-k8s-with-kubespray-ansible/blob/main/static/images/3-cleanup/clean.PNG?raw=true" title="Terraform Destroy" alt="Terraform Destroy" >}}

{{% notice note %}}
Để có thể thực hiện bài LAB này, các bạn cần có tài khoản AWS để tạo các máy chủ EC2 và các tài nguyên cần thiết để triển khai Kubernetes Cluster, sau khi hoàn thành bài LAB thì các bạn có thể xóa các tài nguyên trên AWS đi để tránh tốn thêm chi phí.
{{% /notice %}}

#### Sử dụng Terraform để triển khai VPC và các máy chủ EC2 trên AWS
**Terraform** Terraform là một công cụ mã nguồn mở được phát triển bởi HashiCorp, dùng để xây dựng, thay đổi và quản lý các cơ sở hạ tầng trên cloud một cách an toàn và hiệu quả thông qua "infrastructure as code" (IaC), dịch sát nghĩa thì là "quản lý cơ sở hạ tầng dưới dạng mã"

#### Sử dụng Kubespray và Ansible để triển khai Kubernetes Cluster
- **Kubespray**
  - ***Kubespray*** là một dự án mã nguồn mở của Kubernetes SIGs (Special Interest Groups) dùng để triển khai thủ công các cluster Kubernetes một cách dễ dàng, linh hoạt, và có thể tùy chỉnh trên nhiều môi trường khác nhau. Nó sử dụng Ansible, một công cụ tự động hóa cấu hình, để cài đặt và quản lý Kubernetes clusters. Mặc dù hiện tại có rất nhiều Cloud Provider có cung cấp dịch vụ Kubernetes as a Service như EKS - Elastic Kubernetes Cluster của AWS hay GKE - Google Kubernetes Engine của Google, nhưng với việc sử dụng các dịch vụ này, bạn sẽ không thể có toàn quyền quản lý control plane, "đầu não" của một Kubernetes Cluster, vì vậy tùy vào mục đích sử dụng và quản lý mà các dự án sử dụng các dịch vụ Kubernetes có sẵn hay tự triển khai và quản lý một hệ thống Kubernetes Cluster riêng.
- **Ansible**
  - ***Ansible*** là một công cụ mã nguồn mở được sử dụng để tự động hóa các tác vụ quản trị hệ thống, bao gồm cấu hình hệ thống, triển khai phần mềm và quản lý hạ tầng. Ansible được phát triển bởi Red Hat và nổi bật nhờ tính đơn giản, dễ sử dụng và không yêu cầu cài đặt phần mềm đặc biệt trên các máy đích.


#### Nội dung chính

1. [Sử dụng Terraform để triển khai Hạ tầng trên AWS](1-create-new-aws-account/)
2. [Sử dụng Kubespray và Ansible để triển khai Kubernetes Cluster](2-mfa-setup-for-aws-user-(root)/)
3. [Cấu hình Kubernetes Cluster](3-create-admin-user-and-group/)
4. [Dọn dẹp tài nguyên](4-verify-new-account/)
5. [Tài liệu tham khảo](5-end/)