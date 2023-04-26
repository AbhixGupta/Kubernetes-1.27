# Kubernetes Installation 1.27

## Minimum System Requirements

- Centos 7/8 3 machine
- 4 GB RAM
- 20 GB Disk Space
- 3 CPU

## Installation

#### Run the following commands to turn off:

- Selinux
- Swap
- Firewalld

```bash
swapoff -a
systemctl disable firewalld
setenforce 0
```

To permanent off the selinux make changes in config file:

```bash
vim /etc/selinux/config
```

Edit the permission to "permissive" in place of enforcing

```bash
SELINUX=permissive
```

#### Kubernetes can be operated in two ways:

- Through root user
  For root user you don't need to do any other configuration.

- Through Specific user
  If you want to run k8 with specific user than, first add the user and set the password:

```bash
useradd kop
```

Then give this user sudo permission, by configuring file:

```bash
visudo
kop ALL=(ALL) NOPASSWD:ALL
```

#### Now you need to add the hosts IP and host names inside the hosts file. For now I am adding 3 hosts.

```bash
vim /etc/hosts
```

```bash
143.244.129.224         master.example.com      master
143.244.129.205         node1.example.com       node1
143.244.129.206         node2.example.com       node2
```

#### Now Install the Docker.

First you need to configure the docker repository.

```bash
wget https://download.docker.com/linux/centos/docker-ce.repo
cp docker-ce.repo /etc/yum.repo.d/
yum install docker-ce -you
systemctl enable --now docker
```

#### Now we need to install the kubectl, kubelet and kubelet

First goto a [Kubernetes.io](https://kubernetes.io/) and under kubernetes blog section search kubeadm installation. Select your os and install.

First set up kubernets repository.

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

Make sure to off the selinux.

```bash
# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Now install the kubectl, kubelet and kubeadm:

```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```

#### Make changes in the containerd file

```bash
vim /etc/containerd/config.toml
```

Comment the following line and add the following contents:

```bash
#disabled_plugins = ["cri"]
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
   [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```

Make sure to paster in this order only.

#### Initialize the Kubernetes Cluster

```bash
kubeadm init
```

If you want to make in specific subnet then run:

```bash
kubeadm init --pod-network-cidr=1.244.0.0/16
```

#### Copy config file

Before running any commands we need to copy admin.config file in the kube directory.

- If you running on root then,

```bash
cd /root
mkdir .kube
cp /etc/kubernetes/admin.conf .kube/config
```

- If you want to run with specific user then run:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Now you need to add nodes to this master node. For this run the following command and change the parameters according to the output of your kubeadm init command:

```bash
kubeadm join 143.244.129.223:6443 --token 8dod6b.746x5hjsflryw0ir  --discovery-token-ca-cert-hash sha256:586b7dd85e7cc7dd6c5f63b8098c12237f2e1c87ee11b9f4467f47d1cde3171b
```

#### Connect the nodes and master to the CNI (Container Netwrok Interface). You can use any of the available adds-on from this [kubernetes.io](https://kubernetes.io/docs/concepts/cluster-administration/addons/) link. Here we are using weave netwrok.

Run the following commands:

```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
kubectl get pods -n kube-system -l name=weave-net -o wide
```

If weave pods are not running then:

```bash
wget https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
vim weave-daemonset-k8s.yaml
```

Now make changes inside this yaml file. Add the following content below the container section:

```bash
 containers:
            - name: weave
              command:
                - /home/weave/launch.sh
              env:
                - name: IPALLOC_RANGE
                  value: 10.142.0.0/24
                - name: INIT_CONTAINER
```

Make this change in all the nodes.

Now check for the pods and the nodes:

```bash
kubectl get pods -n kube-system -l name=weave-net -o wide
kubectl get nodes
```

```bash
kubeadm reset
```

Run this to reset the kubernetes cluster.

## Now you are ready to use Kubenetes.
