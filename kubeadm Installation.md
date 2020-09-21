# Kubeadm Install

# 목적

- 1 Master Node, 2 Worker Node를 Kubeadm으로 간단하게 설치하는 실습을 진행함

---

# 실습 Environment

- VM OS : CentOS 7
- Architecture : x86
- Hypervisior : KVM

- Resource
    1. Master : 2 vCPU, RAM 3 GB
    2. Node : 2 vCPU, Ram 2 GB

- IP Settings
    - Master #1 : 192.168.100.10
    - Worker #1 : 192.168.100.11
    - Worker #2 : 192.168.100.12

---

# 초기 작업 (각 노드에서 수행)

- kubeadm Installation 하기 전 해줘야 할 초기 작업들이 있다.
    1. hostname 설정
    2. Docker 설치
    3. 방화벽 설정 (필요한 방화벽 포트를 열어주거나, 아예 방화벽을 해제하거나 둘중 하나)
    4. iptables 설정
    5. swap off
    6. SELinux 해제

- hostname 설정 (아래와 같이 설정)

    → master.test.local

    → node1.test.local

    → node2.test.local

- master, node1, node2 에서 각각 수행

```bash
# vi /etc/hosts

192.168.100.10          master.test.local
192.168.100.11          node1.test.local
192.168.100.12          node2.test.local

# hostname 변경

# master
hostnamectl set-hostname master.test.local

# node1
hostnamectl set-hostname node1.test.local

# node2
hostnamectl set-hostname node2.test.local
```

- Docker 설치

    ( Docker가 이미 설치되었다면 생략해도 좋습니다. )

```bash
# vi docker-install.sh

#! /bin/bash
sudo yum -y install yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo \
https://download.docker.com/linux/centos/docker-ce.repo
sudo yum -y install docker-ce docker-ce-cli containerd.io
sudo systemctl start docker && systemctl enable docker

chmod +x docker-install.sh

sh docker-install.sh

# 간단한 docker 명령어로 설치되었는지 확인
docker ps
docker version
```

- Firewall 설정

    방화벽을 해제하고 실습을 진행하겠습니다.

```bash
systemctl stop firewalld

systemctl disable firewalld

systemctl status firewalld
```

- iptables 설정

```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

net.bridge.bridge-nf-call-iptables = 1 의 의미는?
CentOS와 같은 리눅스 배포판은 net.bridge.bridge-nf-call-iptables의 default값이 0이다.

이는 bridge 네트워크를 통해 송수신 되는 패킷이 iptable 설정을 우회한다는 의미임.
하지만 컨테이너의 네트워크 패킷이 호스트머신의 iptable 설정에 따라 제어되도록 하는 것이 바람직하며, 이를 위해 1로 설정해주어야 한다.

- SELinux 해제

```bash
# Permissive Mode로 변경 (재부팅 되면 다시 돌아옴)
setenforce 0

# selinux 설정 파일에서 영구 설정
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

- SWAP 끄기

```bash
swapoff -a

# swap이 들어간 Line 주석 처리
vi /etc/fstab
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

# kubeadm 설치

- 작업 순서
    - Master Node
        1. k8s repo 추가
        2. kubeadm, kubectl, kubelet 설치
        3. kubeadm init
        4. CNI 설치
    - Worker Node
        1. kubeadm join

---

이제 kubeadm을 사용하여 설치하기 위한 기초 작업은 마무리했으니, kubeadm을 이용하여 설치를 진행하겠다.

- k8s Repo 추가

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

- kubeadm, kubelet, kubectl 설치
    1. kubeadm : 클러스터를 Bootstrap하는 명령어
    2. kubelet : 클러스터의 모든 머신에서 실행되고 파드 및 컨테이너 시작과 같은 작업을 수행하는 구성 요소
    3. kubectl : 클러스터와 통신하기 위한 Command Line Utility

```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable kubelet && systemctl start kubelet
```

## Master Node

위의 작업은 Master, Worker Node 공통으로 해주어야 하는 작업이였다. 이제 Master와 Worker Node 나눠 작업을 진행하겠다.

- Master init (초기화)

    인자로는 —control-plane-endpoint, —pod-network-cidr, —apiserver-advertise-address 등이 있다.

```bash
kubeadm init

..
..
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.100.10:6443 --token f00nid.ygzfr8e0541yz4ys \
    --discovery-token-ca-cert-hash sha256:975b82d0938617f06aeed72a0eaf830008e311490df392887db52ba3d6a4ca23
```

→ 맨 아래 출력물은 나중에 Worker Node에서 Master Node에 Joining할 때 필요하므로 복사해두자.

→ 만약 token값과 discovery-token-ca-cert-hash 값을 잃어버렸다면 아래 URL에 자세히 설명이 되어있음

[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes)

- kubectl 명령어 사용하기 위한 작업

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

[root@master ~]# kubectl get nodes
NAME                STATUS     ROLES    AGE     VERSION
master.test.local   NotReady   master   6m10s   v1.19.2
```

→ Node가 NotReady이므로 CNI (Container Network Interface)를 설치해주자.

- CNI 설치

    → CNI로는 calico를 설치해주었다. 시간이 지나고 확인해보면 Ready 상태로 바뀐 것을 확인할 수 있다. 설치되기까지 몇분 소요된다.

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O

kubectl apply -f calico.yaml

[root@master ~]# kubectl get nodes
NAME                STATUS   ROLES    AGE   VERSION
master.test.local   Ready    master   11m   v1.19.2

[root@master ~]# kubectl get pods -n kube-system
NAME                                        READY   STATUS              RESTARTS   AGE
calico-kube-controllers-c9784d67d-ttzw4     0/1     ContainerCreating   0          116s
calico-node-dw2k4                           0/1     PodInitializing     0          116s
coredns-f9fd979d6-6gk27                     0/1     ContainerCreating   0          11m
coredns-f9fd979d6-7fnvz                     0/1     ContainerCreating   0          11m
etcd-master.test.local                      1/1     Running             0          11m
kube-apiserver-master.test.local            1/1     Running             0          11m
kube-controller-manager-master.test.local   1/1     Running             0          11m
kube-proxy-5jwd5                            1/1     Running             0          11m
kube-scheduler-master.test.local            1/1     Running             0          11m
```

## Join Worker Node

→ Worker Node #1, #2에서 진행

- kubeadm join

```bash
kubeadm join 192.168.100.10:6443 --token f00nid.ygzfr8e0541yz4ys \
    --discovery-token-ca-cert-hash sha256:975b82d0938617f06aeed72a0eaf830008e311490df392887db52ba3d6a4ca23
```

- 아래와 같은 문구가 뜨면 완료

![Untitled](https://user-images.githubusercontent.com/67780144/93744983-b0e8dc00-fc2d-11ea-9e6a-b5fe017bf4c9.png)

## Master Node에서 확인

```bash
# Master Node에서 확인
# 만약 NotReady 상태이면 kube-system 네임스페이스의 파드들 상태를 확인해볼 것
[root@master ~]# kubectl get nodes
NAME                STATUS     ROLES    AGE    VERSION
master.test.local   Ready      master   15m    v1.19.2
node1.test.local    NotReady   <none>   115s   v1.19.2
node2.test.local    NotReady   <none>   107s   v1.19.2

# 아직 각 노드들에서 CNI를 위한 Pod들을 생성중이므로 NotReady이다.
[root@master ~]# kubectl get pods -n kube-system -o wide
NAME                                        READY   STATUS     RESTARTS   AGE     IP               NODE                NOMINATED NODE   READINESS GATES
calico-kube-controllers-c9784d67d-ttzw4     1/1     Running    0          6m23s   172.16.134.1     master.test.local   <none>           <none>
calico-node-47n8l                           0/1     Init:0/3   0          2m19s   192.168.100.11   node1.test.local    <none>           <none>
calico-node-dw2k4                           1/1     Running    0          6m23s   192.168.100.10   master.test.local   <none>           <none>
calico-node-s2lp4                           0/1     Init:0/3   0          2m11s   192.168.100.12   node2.test.local    <none>           <none>
coredns-f9fd979d6-6gk27                     1/1     Running    0          16m     172.16.134.3     master.test.local   <none>           <none>
coredns-f9fd979d6-7fnvz                     1/1     Running    0          16m     172.16.134.2     master.test.local   <none>           <none>
etcd-master.test.local                      1/1     Running    0          16m     192.168.100.10   master.test.local   <none>           <none>
kube-apiserver-master.test.local            1/1     Running    0          16m     192.168.100.10   master.test.local   <none>           <none>
kube-controller-manager-master.test.local   1/1     Running    0          16m     192.168.100.10   master.test.local   <none>           <none>
kube-proxy-5272q                            1/1     Running    0          2m19s   192.168.100.11   node1.test.local    <none>           <none>
kube-proxy-5jwd5                            1/1     Running    0          16m     192.168.100.10   master.test.local   <none>           <none>
kube-proxy-wlswj                            1/1     Running    0          2m11s   192.168.100.12   node2.test.local    <none>           <none>
kube-scheduler-master.test.local            1/1     Running    0          16m     192.168.100.10   master.test.local   <none>           <none>

# Ready 상태!
[root@master ~]# kubectl get nodes
NAME                STATUS   ROLES    AGE     VERSION
master.test.local   Ready    master   17m     v1.19.2
node1.test.local    Ready    <none>   3m28s   v1.19.2
node2.test.local    Ready    <none>   3m20s   v1.19.2
```

---

# 설치 Tips

1. tmux 를 이용하여 Terminal Window를 분할하여 설치를 진행하면 빠르게 작업할 수 있다.

    ( 아래 그림이 잘 안보이지만 3개의 Window를 분할하여 master, node1, node2 로 작업하였음. tmux 사용법은 인터넷에 검색하면 자세히 나온다. )

![Untitled 1](https://user-images.githubusercontent.com/67780144/93744985-b21a0900-fc2d-11ea-956c-b15a02f53bca.png)

2. shell script를 활용하면 반복작업을 sh [xxx.sh](http://xxx.sh) 명령어 하나만 누르고 커피 마시고 오면 된다.


- References

https://gruuuuu.github.io/cloud/k8s-install/#

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
