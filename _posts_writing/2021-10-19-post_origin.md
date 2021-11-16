---
title: Kubernetes 설치, 클러스터 구성
date: 2021-11-16
categories: Study
tags:
- Kubernetes
- Docker
---


# 개념

한번 쭉 돌려본 뒤에 정리하도록 하고, 일단은 설치-실행까지 먼저 해보자..

# 실행

## 쿠버네티스 설치

도커 설치가 먼저 필요함

```shell
sudo apt-get update

sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

sudo systemctl enable docker
sudo docker version
```

쿠버네티스 설치 전 환경설정
```shell
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

systemctl stop firewalld 
systemctl disable firewalld
```

쿠버네티스 설치 (kubelet, kubeadm, kubectl)

```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
 
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl start kubelet
sudo systemctl enable kubelet
```

## 클러스터 구성

### Control-Plane
```shell
kubeadm init
```


### Worker Node

## 도커 설치

<!--
클러스터 구성
마스터
(마스터에서만) (컨트롤 플레인 구성)
kubeadm init

-> 하는중에 에러 남. kubelet 실행이 안 되는 중

sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF


이후 재부팅, sudo kubeadm reset 하고 다시 init 하니까 됨
이러면 마스터에 컴포넌트가 생성됨 (api, 컨트롤러, 스케줄러, etcd, coreDNS)
설치 완료시 then you can join any number of worker nodes by running the following on each as root: 아래 토큰이 생성됨
저장해두기 -> 워커노드 등록시 필요함
근데 이거 유효기간이 엄청 짧아서 금방 사라짐.. 나는 여기까지 해놓고 워커 등록을 며칠 후에 했는데, 토큰 재생성 했음
토큰 재생성 방법
kubeadm token create --ttl 0 (ttl은 유효기간. 10m, 100h 등. 0은 만료기간 없음)
해시값 구하기 (불변인듯?) : openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
kubeadm token list (토큰 리스트 보기)

kubectl get nodes -> 현재 노드 보기
근데 실행 안됨. 왜와이? path 등록 안되어있으니까.

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config  이후 다시 kubectl get nodes 하면 잘 뜸

마스터가 not ready일텐데, 컨테이너 네트워크가 설치되지 않아서 그럼.
네트워크 애드온 설치 (컨테이너끼리 통신 가능하게)
 kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
(마스터에서만 실행하면 됨)

워커
앞서 만들어둔 토큰을 그대로 노드들에 복사해서 실행하면 됨
kubeadm join [master-node-ip]:[port] --token [token] --discovery-token-ca-cert-hash sha256:[hash키]
-> 실행 안됨.. (에러로그 보려면 --v=5)
워커의 kubelet이 계속 죽어서 재실행되는 현상 -> 알고보니 원래 그럼

/etc/docker 에
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
추가 후
sudo systemctl daemon-reload
sudo systemctl restart docker

sudo vi /etc/fstab
swap 라인 주석처리

이후 sudo reboot
다시 쿠브아담 조인 하면 잘 됨.
마스터에서 kubectl get nodes 실행시 노드 나옴

-- kube 명령어들 자동완성 등록방법
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
kubectl->kubeadm 동일하게 하면 kubeadm도 됨

__>