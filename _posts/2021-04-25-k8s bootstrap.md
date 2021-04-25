---
layout: post
title: "kubernetes bootstrap on Linux"
tags: [kubernetes]
---
# Kubernetes Cluster Setup on Linux

전반적인 흐름은 먼저 컨테이너 런타임을 설치한다.  
컨테이너 런타임은 일반적으로 도커를 많이 사용한다. 다른 선택지로는 containerd가 있으며, 그 외에도 lightVM을 컨테이너 런타임으로 사용하는 방법도 있다.  
그 뒤, kubeadm과 kubelet, kubectl을 설치한다.
kubeadm은 클러스터를 쉽게 초기화 해주는 툴이며, kubectl은 컨트롤 노드에서 각종 클러스터 세팅을 도와주는 클라이언트다.
kubectl은 `kube-system`의 팟들과 kube-api-server를 통해 컨트롤 한다.  
kubelet은 각 노드를 컨트롤 하는 에어전트이며, kubelet이 팟을 띄우는 역할을 한다.  

그 외로 etcd도 있으며, k8s가 쓰는 storage라고 보면 된다.  

kubeadm을 이용하여 클러스터를 셋업한 뒤, 네트워크를 설정해 준다.  
네트워크는 아래에서는 Flannel을 사용하지만 일반적으로 Calico를 많이 사용하는 것 같다.[^1]

## Install Docker
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu
sudo apt-mark hold docker-ce
```

## Install kubeadm, kubelet, and kubectl

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet=1.14.5-00 kubeadm=1.14.5-00 kubectl=1.14.5-00
sudo apt-mark hold kubelet kubeadm kubectl
```
## Init kube master node 

kubeadm은 kubernetes control plane을 매니징 하는 툴이다.

```bash
sudo kubeadm init --pod-network-cidr=string
# ref https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/

# in kubeadm init log
# kubectl refer this config
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# in master, server/client info will be shown
# in slave, only client info will be shown
kubectl version
```

## Setup Cluster
```bash
# in kubeadm init log
# do this on each slave nodes
sudo kubeadm join $some_ip:6443 --token $some_token --discovery-token-ca-cert-hash $some_hash

# on master node
# check there are master and slave nodes
kubectl get nodes
```
## Configure Network
```bash
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml

kubectl get nodes
```

---

[^1]: https://medium.com/@jain.sm/flannel-vs-calico-a-battle-of-l2-vs-l3-based-networking-5a30cd0a3ebd