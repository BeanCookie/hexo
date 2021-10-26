---
title: Kubeadm概述
date: 2020-04-16 11:43:59
categories:
- 容器化
tags:
- K8S
---

# Kubeadm 概述

Kubeadm 是一个工具，它提供了 `kubeadm init` 以及 `kubeadm join` 这两个命令作为快速创建 kubernetes 集群的最佳实践。

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
```



sudo hostnamectl set-hostname master

```sh
pull_images.sh                                                                                          
vagrant@debian-10:~/script$ cat pull_images.sh                                                          
# cat pull-images.sh                                                                                    
#!/bin/bash                                                                                             
images=(                                                                                                
    kube-apiserver:v1.18.2                                                                              
    kube-controller-manager:v1.18.2                                                                     
    kube-scheduler:v1.18.2                                                                              
    kube-proxy:v1.18.2                                                                                  
    pause:3.2                                                                                           
    etcd:3.4.3-0                                                                                        
    coredns:1.6.7                                                                                       
)                                                                                                       
for imageName in ${images[@]};                                                                          
do                                                                                                      
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/${imageName}                        
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/${imageName} k8s.gcr.io/${imageName} 
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/${imageName}                         
done                                                                                                    
```

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/kubelet.conf $HOME/.kube/config
~/.kube$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://3sa2h8sc.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

```
docker pull quay-mirror.qiniu.com/coreos/flannel:v0.11.0-amd64
docker tag quay-mirror.qiniu.com/coreos/flannel:v0.11.0-amd64 quay.io/coreos/flannel:v0.11.0-amd64

docker rmi quay-mirror.qiniu.com/coreos/flannel:v0.11.0-amd64
```

