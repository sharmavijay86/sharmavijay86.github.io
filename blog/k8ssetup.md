![Kubernetes](k8slogo.jpg)
# Kubernetes cluster lab with ubuntu 18.04

**Step to follow on all nodes**

```$ sudo apt-get update```

```$ sudo apt-get install docker.io -y```

```$ sudo systemctl enable docker```

```$ sudo systemctl start docker```

```$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add```

```$ sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"```

```$ sudo apt install kubeadm -y```


**step on master node**

```$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16```

```$ mkdir -p $HOME/.kube```

```$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config```

```$ sudo chown $(id -u):$(id -g) $HOME/.kube/config```

**join worker node ( step on worker node )**

```$ kubeadm join 192.168.122.220:6443 --token gefqt9.oj3kcgubehofxbz8 ```
     ```--discovery-token-ca-cert-hash sha256:a79789ade9c95182522f55b1ab17e93cd6eac9c7eaf8b7b67a6c125bbb5f50ce ```

**deploy a pod network plugin ( on master node )**

```$ sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml```
