+++
title = "Cài đặt K8s Cluster với Kubespray và Ansible"
date = 2021
weight = 2
chapter = false
pre = "<b>2.2. </b>"
+++

**Nội dung**
- [Cài đặt K8s Cluster với Kubespray và Ansible](#cài-đặt-k8s-cluster-với-kubespray-và-ansible)


{{%notice tip%}}
Nếu bạn không có thiết bị phần cứng , có thể bỏ qua các thao tác dưới đây nhé.
{{%/notice%}}

#### Cài đặt K8s Cluster với Kubespray và Ansible
- Sau khi đã cài đặt thành công Ansible và Kubespray, chúng ta sẽ thực hiện cài đặt K8s Cluster.
- Ở thư mục chứa source code của Kubespray, các bạn hãy tạo một file mới **hosts.ini** trong đường dẫn ***kubespray/inventory/mycluster/hosts.ini***
- Hiểu nôm na thì file này chứa định nghĩa về các Node, Node nào là Master, Node nào là Worker, địa chỉ IP của nó là bao nhiêu,...
- File host này cũng đã có file mẫu trên [Github của Kubespray](https://github.com/kubernetes-sigs/kubespray), các bạn có thể lấy về và chỉnh sửa tùy theo cấu hình của mình, ở dưới là file **hosts.ini** của mình, mọi người có thể xem qua và tham khảo.
```ini
## Configure 'ip' variable to bind kubernetes services on a
## different ip than the default iface
k8s-master1 ansible_host=10.0.1.181 ip=10.0.1.181
k8s-master2 ansible_host=10.0.2.7 ip=10.0.2.7
k8s-master3 ansible_host=10.0.3.17 ip=10.0.3.17
k8s-worker1 ansible_host=10.0.1.95 ip=10.0.1.95
k8s-worker2 ansible_host=10.0.2.10 ip=10.0.2.10
k8s-worker3 ansible_host=10.0.3.69 ip=10.0.3.69

[kube_control_plane]
k8s-master1
k8s-master2
k8s-master3

[etcd]
k8s-master1
k8s-master2
k8s-master3

[kube_node]
k8s-worker1
k8s-worker2
k8s-worker3

[k8s_cluster:children]
kube_node
kube_master
kube_control_plane

[calico-rr]

[vault]
k8s-master1
```
    
- Tiếp theo chúng ta sẽ sử dụng Ansible để bắt đầu quá trình cài đặt K8s-Cluster

```bash
ansible-playbook -i inventory/mycluster/hosts.ini  --become --become-user=root cluster.yml
```

{{%notice note%}}
Quá trình cài đặt có thể kéo dài từ 15-20p
{{%/notice%}}

![ansible](./images/2.2-kubespray/install-spray.png)

- Kết quả sau khi cài đặt thành công
![ansible](./images/2.2-kubespray/install-result.png)

- Tiếp theo chúng ta cần cấu hình kubectl trước khi có thể bắt đầu vào sử dụng K8s Cluster

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Cấu hình kubectl
![ansible](./images/2.2-kubespray/kubectl.png)

- Vậy là chúng ta đã cài đặt thành công một cụm Kubernetes cluster với Kubespray và Ansible, với việc triển khai thủ công theo cách này thì việc chúng ta thêm, xóa, sửa các node trong cluster rất dễ dàng tùy thuộc vào nhu cầu và mục đích sử dụng của từng thời điểm.