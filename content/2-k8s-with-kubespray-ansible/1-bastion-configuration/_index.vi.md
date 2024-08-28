+++
title = "Cấu hình Bastion Instance"
date = 2021
weight = 1
chapter = false
pre = "<b>2.1. </b>"
+++

{{% notice note %}}
Trong bước này, chúng ta sẽ thực hiện cấu hình Bastion Instance để chuẩn bị cho bước triển khai Kubernetes Cluster, mọi cấu hình liên quan đến các Master Node, Worker Node đều được thực hiện thông qua Bastion Instance.
{{% /notice %}}

#### Cấu hình các Instance

- Sau khi đã apply xong các file cấu hình Terraform, một file SSH key ".pem" sẽ được tạo ra trong thư mục chứa các file Terraform, đây chính là key mà chúng ta đã cấu hình để Terraform thực hiện tạo trong file **key.tf**, và dùng để SSH vào Bastion Instance.
- Thực hiện SSH vào Bastion Instance để cấu hình
{{< figure src="https://github.com/dinhcam89/dinhcam89-k8s-with-kubespray-ansible/blob/main/static/images/2.1-bastion/ssh-bastion.PNG?raw=true" title="SSH Bastion" alt="SSH Bastion" >}}

- Thực hiện cấu hình hostname trên máy Bastion để thuận tiện cho các thao tác SSH cũng như để dễ gợi nhớ hơn, thay vì phải sử dụng IP của các Instance.
- Cấu hình thêm địa chỉ IP private và tên của các nodes vào file ***/etc/hosts***

1. Cấu hình hostname
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

2. Các cấu hình cần thiết để có thể cài đặt Kubernetes Cluster trên các EC2 Instance
- Sau khi đã  SSH được vào Bastion Instance, chúng ta sẽ thực hiện SSH vào các Master và Worker Node để thực hiện cập nhật các package cũng như cấu hình một vài thứ cần thiết trước khi cài đặt K8s Clutser.
- Để có thể cài đặt Kubernetes Cluster trên các Instance, chúng ta cần phải tắt **swap** trên các Instance,
- Tại sao Kubernetes yêu cầu tắt swap? Kubernetes dựa vào khả năng dự đoán và hiệu quả trong quản lý tài nguyên trên các node. Swap có thể gây ra tình trạng trì hoãn không mong muốn khi hệ điều hành quyết định di chuyển dữ liệu từ RAM sang đĩa cứng và ngược lại. Điều này có thể dẫn đến hiệu suất không nhất quán của các pod, làm giảm khả năng quản lý và điều phối tài nguyên của Kubernetes.

```bash
swapoff -a

sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

#### Cài đặt Kubespray và Ansible
- Sau khi đã cấu hình thành công cho Bastion Instance, chúng ta đến với bước cài đặt K8s Cluster
- Như mình đã nói ở trước, tất cả mọi cài đặt sẽ được thực hiện ở Bastion Instance, vì vậy cho nên mình sẽ cài đặt Kubespray và Ansible trên Bastion Instance.

```bash
sudo apt install git python3 python3-pip -y

sudo apt install ansible-core -y
```

- Sau khi đã cài đặt thành công Ansible, chúng ta thực hiện clone [source code của Kubespray](https://github.com/kubernetes-sigs/kubespray.git) từ Github về để tiến hành cài đặt K8s Cluster.

{{% notice note %}}
Để có thể cài đặt Kubernetes v1.28 thì các bạn cần chọn đúng phiên bản 2.24 của Kubespray, về phiên bản Kubernetes tương ứng với các phiên bản Kubespray thì các bạn có thể xem ở Document trong [Github của Kubespray](https://github.com/kubernetes-sigs/kubespray)
{{% /notice %}}

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git --branch release-2.24
```