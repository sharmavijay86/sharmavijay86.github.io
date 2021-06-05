![Kubernetes](k8slogo.png)
# Kubernetes cluster lab with ubuntu 20.04

### Cloud-init-config
If you are using this cloud-init user data file on ubuntu 20.04 it will setup all prerequisite including. apt repo, kernel parameter, kubeadm. Just change the desired kubernetes version.
<script src="https://gist.github.com/sharmavijay86/cf86ca128a166ddd456bf0be1b95e2a6.js"></script>
**Step to follow on all nodes**

``` sudo apt-get update```

``` sudo apt-get install containerd -y```

``` curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add```

``` sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"```

``` sudo apt install kubelet=1.19.0-00 kubeadm=1.19.0-00 kubectl=1.19.0-00 -y```
```
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

**step on master node**

``` sudo kubeadm init --pod-network-cidr=10.244.0.0/16```

``` mkdir -p $HOME/.kube```

``` sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config```

``` sudo chown $(id -u):$(id -g) $HOME/.kube/config```

**join worker node ( step on worker node )**

``` kubeadm join 192.168.122.220:6443 --token gefqt9.oj3kcgubehofxbz8 ```
     ```--discovery-token-ca-cert-hash sha256:a79789ade9c95182522f55b1ab17e93cd6eac9c7eaf8b7b67a6c125bbb5f50ce ```

**deploy a pod network plugin ( on master node )**

``` sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml```
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
## Setup ingres as nginx
 - Daemonset
 ``` 
 helm install ingress-nginx ingress-nginx/ingress-nginx --namespace=ingress --create-namespace=true --set controller.kind=DaemonSet,controller.service.enabled=false,controller.hostNetwork=true,controller.publishService.enabled=false
 ```
 - Deployment
 ```
 helm install ingress-nginx ingress-nginx/ingress-nginx --namespace=ingress --create-namespace=true 
 ```
