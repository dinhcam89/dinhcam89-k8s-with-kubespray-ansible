+++
title = "Installing the K8s Cluster with Kubespray and Ansible"
date = 2021
weight = 2
chapter = false
pre = "<b>2.2. </b>"
+++

**Content**
- [Installing the K8s Cluster with Kubespray and Ansible](#installing-the-k8s-cluster-with-kubespray-and-ansible)


#### Installing the K8s Cluster with Kubespray and Ansible
- After successfully installing Ansible and Kubespray, we will proceed with installing the K8s Cluster.
- In the directory containing the Kubespray source code, create a new file named **hosts.ini** in the path ***kubespray/inventory/mycluster/hosts.ini***
- Essentially, this file contains definitions of the Nodes, which nodes are Master, which nodes are Worker, their IP addresses, etc.
-This host file also has a sample file available on the [Github của Kubespray](https://github.com/kubernetes-sigs/kubespray), which you can download and modify according to your configuration. Below is my hosts.ini file for reference:
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
    
- Next, we will use Ansible to start the process of installing the K8s Cluster:
bash


```bash
ansible-playbook -i inventory/mycluster/hosts.ini  --become --become-user=root cluster.yml
```

{{%notice note%}}
The installation process may take 15-20 minutes.
{{%/notice%}}

![ansible](/images/2.2-kubespray/install-spray.png)

- Result after successful installation:
![ansible](/images/2.2-kubespray/install-result.png)

- Next, we need to configure kubectl before we can start using the K8s Cluster:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Configuring kubectl:
![ansible](/images/2.2-kubespray/kubectl.png)

- And that's it! We have successfully installed a Kubernetes cluster with Kubespray and Ansible. With this manual deployment method, adding, removing, or modifying nodes in the cluster is quite straightforward, depending on the needs and objectives at any given time.